# virtio-blk 设备模型实现原理

## 1. 整体架构

virtio 设备模型的核心是**共享内存 + 事件通知**：

```
┌─────────────────────────────────────────────────┐
│                   Guest (OVMF)                  │
│  virtio-blk driver                              │
│    │  写 descriptor table / avail ring          │
│    │  写 notify 寄存器                          │
│    ▼                                            │
│  ┌─────────────────────────────┐                │
│  │  Virtqueue (guest memory)   │                │
│  │  ┌────────┐ ┌──────┐ ┌────┐│                │
│  │  │descr tbl│ │avail │ │used││                │
│  │  └────────┘ └──────┘ └────┘│                │
│  └─────────────────────────────┘                │
└──────────────────┬──────────────────────────────┘
                   │ notify (I/O write)
                   ▼
┌─────────────────────────────────────────────────┐
│              AxVisor (VMM)                      │
│  ┌─────────────────────────────┐                │
│  │ VirtioBlk Device Model      │                │
│  │  1. 读 avail ring 取请求    │                │
│  │  2. 读 descriptor 取数据地址│                │
│  │  3. 通过 GuestMemoryAccessor│                │
│  │     读写 guest buffer       │                │
│  │  4. 调用 backend 做实际 I/O │                │
│  │  5. 写 used ring 返回结果   │                │
│  └────────────┬────────────────┘                │
│               │                                 │
│  ┌────────────▼────────────────┐                │
│  │  Backend                    │                │
│  │  当前为 MemoryDiskBackend   │                │
│  └─────────────────────────────┘                │
└─────────────────────────────────────────────────┘
```

guest 和 VMM 之间通过 guest memory 中的 virtqueue 结构通信，不通过 I/O 端口传数据。I/O 端口只用来做配置和通知。

当前源码对照：

- 设备模型核心：`tgoskits/components/axdevice/src/virtio_blk/`
  - `legacy.rs`: `LegacyVirtioBlk`，legacy I/O BAR、队列 notify、请求执行。
  - `queue.rs`: `LegacyQueue`，计算 split queue 布局、读取 avail ring、写 used ring。
  - `request.rs`: `VirtioBlkRequest`，解析/校验 virtio-blk descriptor chain。
  - `backend.rs`: `MemoryDiskBackend`，当前内存 raw disk 后端。
- AxVM 接线：`tgoskits/components/axvm/src/vm.rs`
  - `AxVMInnerMut::ovmf_virtio_blk`
  - `AxVmGuestMemory`
  - `handle_ovmf_virtio_blk_io_read()`
  - `handle_ovmf_virtio_blk_io_write()`

Note：这里的 virtio-blk 仍是 OVMF 适配阶段的实验性实现。 I/O exit 分发还没有完全接入 AxVisor 标准 `AxVmDevices` port device 注册路径。

## 2. Virtqueue 结构

一个 virtqueue 在内存中是这样布局的（legacy split queue，OVMF 当前用的）：

```
queue_pfn 给出 queue 的起始 GPA 页号，queue_gpa = queue_pfn << 12。
descriptor table、avail ring、used ring 从这个起点开始排布；整个 queue 不一定只占一页。

┌─────────────────────────────────────────────────┐  queue_gpa = queue_pfn << 12
│  Descriptor Table: queue_size 个 entry          │
│  每个 entry 16 字节:                             │
│    addr  (u64): 数据 buffer 的 guest 物理地址     │
│    len   (u32): 数据长度                         │
│    flags (u16): NEXT / WRITE 等标志              │
│    next  (u16): 下一个 descriptor 的索引          │
├─────────────────────────────────────────────────┤  queue_gpa + queue_size * 16
│  Available Ring:                                │
│    flags (u16): NO_INTERRUPT 等                  │
│    idx   (u16): 下一个写入位置（guest 维护）      │
│    ring[queue_size] (u16[]): descriptor 索引      │
├─────────────────────────────────────────────────┤  align up to 4KiB
│  Used Ring:                                     │
│    flags (u16): NO_NOTIFY 等                     │
│    idx   (u16): 下一个写入位置（VMM 维护）        │
│    ring[queue_size]: {id (u32), len (u32)}[]    │
└─────────────────────────────────────────────────┘
```

## 3. 请求处理流程

以 guest 读一个 block 为例：

```
Guest driver 做的事:
  1. 把请求头(type, reserved, sector) 放到一个 buffer
  2. 把"数据目标 buffer" 放到另一个 buffer
  3. 在 descriptor table 里建立 chain:
       desc[0] -> 请求头 (device-readable)
       desc[1] -> 数据 buffer (device-writable), NEXT flag
       desc[2] -> 状态字节 (device-writable)
  4. 把 desc[0] 的索引写入 avail.ring[avail.idx % size]
  5. avail.idx += 1
  6. 写 notify 寄存器 (I/O port 0x6010)

VMM 做的事 (notify 触发):
  1. 读 avail.idx，知道有新请求
  2. 从 avail.ring 取 descriptor head index
  3. 遍历 descriptor chain:
       - 用 GuestMemoryAccessor 把请求头从 guest memory 读出来
       - 解析出 type (read/write) 和 sector LBA
  4. 用 backend 做实际读写:
       - 把数据从 backend 读到临时 buffer
       - 用 GuestMemoryAccessor 写回 guest 的数据 buffer
  5. 写 used ring:
       - used.ring[used.idx % size] = {id: head, len: written_bytes}
       - used.idx += 1
  6. (可选) 通过 vLAPIC 注入中断通知 guest
```

`used.len` 表示设备写入到 device-writable descriptor 的字节数。对读请求，这通常包含数据 buffer 的长度，也包含 1 字节 status；对写请求，当前实现同样把写入 status 的 1 字节计入 used length。

当前源码里对应为：

- `LegacyVirtioBlk::execute_read()`: 返回 `data_len + 1`，因为设备写了 guest data buffer 和 status byte。
- `LegacyVirtioBlk::execute_write()`: 返回 `1`，因为写请求的 data buffer 是 device-readable，设备只写 status byte。
- `LegacyVirtioBlk::execute_get_id()`: 返回写入的 serial 字节数加 status byte。

## 4. Rust 伪代码

### 4.1 virtqueue 抽象

```rust
// 内存中 virtqueue 的布局描述
struct Virtqueue {
    queue_size: u16,        // queue 能容纳多少个 descriptor
    queue_pfn: u32,         // guest 告诉我们 queue 在哪一页
    // 以下由 queue_pfn 计算得到
    desc_table_gpa: usize,  // descriptor table 起始地址
    avail_ring_gpa: usize,  // available ring 起始地址
    used_ring_gpa: usize,   // used ring 起始地址
    last_avail_idx: u16,    // 我们已处理到哪个位置
}

#[repr(C)]
struct VirtqDesc {
    addr: u64,    // guest physical address
    len: u32,
    flags: u16,
    next: u16,
}

const VIRTQ_DESC_F_NEXT: u16 = 1;
const VIRTQ_DESC_F_WRITE: u16 = 2;  // device writable (读请求的数据目标)
```

### 4.2 读 notify 寄存器时触发处理

```rust
// 这是设备模型的核心入口
// 当 guest 写 I/O port 0x6010 (queue notify) 时调用
fn handle_queue_notify(&mut self, mem: &impl GuestMemoryAccessor) {
    let vq = &mut self.queue;

    // 读 avail ring 的 idx，看 guest 发了多少新请求
    let avail_idx: u16 = mem.read_obj(
        GuestPhysAddr::from(vq.avail_ring_gpa + 2)
    ).unwrap();

    // 逐个处理从 last_avail_idx 到 avail_idx 的请求
    while vq.last_avail_idx != avail_idx {
        let ring_slot = vq.last_avail_idx % vq.queue_size;

        // 从 avail ring 取 descriptor head index
        let head: u16 = mem.read_obj(
            GuestPhysAddr::from(vq.avail_ring_gpa + 4 + ring_slot as usize * 2)
        ).unwrap();

        // 遍历 descriptor chain，收集所有 buffer
        self.process_chain(head, mem);

        // 把结果写入 used ring
        self.write_used(head, bytes_written, mem);

        vq.last_avail_idx = vq.last_avail_idx.wrapping_add(1);
    }
}
```

源码对应关系：

- guest 写 `0x6010` 后进入 `LegacyVirtioBlk::handle_write()`。
- `REG_QUEUE_NOTIFY` 分支调用 `LegacyVirtioBlk::process_queue()`。
- `process_queue()` 循环调用 `LegacyQueue::pop_available()`，再调用 `process_chain()` 和 `LegacyQueue::publish_used()`。

### 4.3 遍历 descriptor chain 并执行 I/O

```rust
fn process_chain(&self, head: u16, mem: &impl GuestMemoryAccessor) {
    let mut desc_idx = head;
    let mut req_header = None;
    let mut data_bufs = Vec::new(); // 数据 buffer (GPA + len)
    let mut status_desc = None;

    // 遍历 chain，收集所有 descriptor
    loop {
        let desc: VirtqDesc = mem.read_obj(
            GuestPhysAddr::from(self.queue.desc_table_gpa + desc_idx as usize * 16)
        ).unwrap();

        if desc_idx == head {
            // 第一个 descriptor 是 virtio-blk 请求头，必须 device-readable。
            req_header = Some(desc);
        } else if desc.flags & VIRTQ_DESC_F_NEXT == 0 {
            // 当前实现把 chain 最后一个 descriptor 当作 status descriptor，
            // 并要求它是 device-writable 且 len > 0。
            status_desc = Some(desc);
        } else {
            data_bufs.push((desc.addr as usize, desc.len as usize, desc.flags));
        }

        if desc.flags & VIRTQ_DESC_F_NEXT == 0 {
            break; // chain 结束
        }
        desc_idx = desc.next;
    }

    // 解析 virtio-blk 请求头
    let header = req_header.unwrap();
    let mut raw_header = [0u8; 16];
    mem.read_buffer(GuestPhysAddr::from(header.addr as usize), &mut raw_header).unwrap();
    let req: VirtioBlkReq = parse_request(&raw_header, data_bufs, status_desc.unwrap());

    match req.type_ {
        VIRTIO_BLK_T_IN => {
            // 读操作：从 backend 读数据，写回 guest
            for (gpa, len, _flags) in &data_bufs {
                let mut buf = vec![0u8; *len];
                self.backend.read_at(req.sector * 512, &mut buf);
                mem.write_buffer(GuestPhysAddr::from(*gpa), &buf).unwrap();
            }
            // 写状态 = VIRTIO_BLK_S_OK
            mem.write_obj(req.status, VIRTIO_BLK_S_OK).unwrap();
        }
        VIRTIO_BLK_T_OUT => {
            // 写操作：从 guest 读数据，写到 backend
            for (gpa, len, _flags) in &data_bufs {
                let mut buf = vec![0u8; *len];
                mem.read_buffer(GuestPhysAddr::from(*gpa), &mut buf).unwrap();
                self.backend.write_at(req.sector * 512, &buf);
            }
            mem.write_obj(req.status, VIRTIO_BLK_S_OK).unwrap();
        }
        _ => {
            mem.write_obj(req.status, VIRTIO_BLK_S_UNSUPP).unwrap();
        }
    }
}
```

### 4.4 写 used ring 完成通知

```rust
fn write_used(&self, head: u16, len: u32, mem: &impl GuestMemoryAccessor) {
    let ring_slot = self.queue.used_idx % self.queue.queue_size;
    let used_entry_gpa = self.queue.used_ring_gpa + 4 + ring_slot as usize * 8;

    // used ring entry: { id: u32, len: u32 }
    mem.write_obj(GuestPhysAddr::from(used_entry_gpa), head as u32).unwrap();
    mem.write_obj(GuestPhysAddr::from(used_entry_gpa + 4), len).unwrap();

    // 更新 used.idx (先写 entry，再更新 idx，顺序不能反)
    // memory barrier 在 virtio spec 里要求，这里简化
    self.queue.used_idx = self.queue.used_idx.wrapping_add(1);
    mem.write_obj(
        GuestPhysAddr::from(self.queue.used_ring_gpa + 2),
        self.queue.used_idx
    ).unwrap();

    // 可选: 如果 guest 没设 NO_INTERRUPT flag，注入中断
    // self.inject_interrupt();
}
```

## 5. 设备模型 vs 早期临时实现

| 方面 | 早期临时实现 | 当前设备模型做法 |
|------|------------|------------|
| 数据传输 | descriptor 原地改写 GPA→HPA，让 host QEMU DMA | VMM 自己读写 guest memory，自己做 backend I/O |
| guest memory | 被污染（addr 字段变成 HPA） | 不修改，只读/写数据 buffer |
| I/O 来源 | host QEMU virtio-blk | AxVisor backend，当前是内存中的 raw disk image |
| 设备代码位置 | 裸函数写在 `axvm/src/vm.rs` | 核心设备模型在 `axdevice/src/virtio_blk/` |
| AxVisor 接线 | VMX 或 AxVM 里手写透传/改写 | 当前仍由 `axvm` 专门分发，因为 `BasePortDeviceOps` 没有 guest memory 参数 |
| 可扩展性 | 绑死 host QEMU legacy I/O | 可继续抽 backend，可把 transport 接入标准设备注册路径 |

## 6. AxVisor 接口定位

### 6.1 当前接线

当前代码已经把 virtio-blk 的核心逻辑从 `axvm` 拆到 `axdevice/src/virtio_blk/`：

- `legacy.rs`: legacy I/O BAR 寄存器、notify、请求处理。
- `queue.rs`: split virtqueue 布局、avail/used ring 读写、descriptor chain 遍历。
- `request.rs`: virtio-blk 请求类型和 descriptor 形态校验。
- `backend.rs`: 当前内存镜像 backend。

但它还没有作为普通 `AxVmDevices` port device 注册。实际接线是：

```text
x86 I/O exit
  -> axvm::run_vcpu()
  -> AxVM::handle_ovmf_virtio_blk_io_read/write()
  -> LegacyVirtioBlk::handle_read/write(port, width, mem)
  -> notify 时通过 AxVmGuestMemory 访问 guest memory
```

这个接线比早期 descriptor 改写标准很多，因为设备语义已经在 `axdevice`，但还不是 AxVisor 最标准的设备分发形态。

### 6.2 Guest Memory 访问：`GuestMemoryAccessor`

```rust
// axaddrspace::GuestMemoryAccessor
// 已有的 trait，提供:
//   translate_and_get_limit(GPA) -> Option<(HPA, usize)>
//   read_obj<T>(GPA) -> AxResult<T>
//   write_obj<T>(GPA, val) -> AxResult<()>
//   read_buffer(GPA, &mut [u8]) -> AxResult<()>
//   write_buffer(GPA, &[u8]) -> AxResult<()>
//
// 当前 BasePortDeviceOps::handle_read/write 没有 GuestMemoryAccessor 参数。
// 所以 LegacyVirtioBlk 暂时不能直接实现 BasePortDeviceOps 后放进
// AxVmDevices::handle_port_read/write。
```

这里有一个当前源码必须注意的地址语义细节：

- `AddrSpace::translate_and_get_limit(GPA)` 返回的是 HPA。
- `GuestMemoryAccessor` 的默认 `read_buffer()` / `write_buffer()` 会把 `translate_and_get_limit()` 返回值当作 host 可解引用地址。
- 所以 `AxVmGuestMemory::translate_and_get_limit()` 不能直接返回 HPA，必须先调用 `PagingHandlerImpl::phys_to_virt()` 转成 direct-map HVA。

当前实现位置：

```rust
// tgoskits/components/axvm/src/vm.rs
impl GuestMemoryAccessor for AxVmGuestMemory<'_> {
    fn translate_and_get_limit(&self, guest_addr: GuestPhysAddr) -> Option<(HostPhysAddr, usize)> {
        let (host_paddr, limit) = self.address_space.translate_and_get_limit(guest_addr)?;
        let host_vaddr = PagingHandlerImpl::phys_to_virt(host_paddr);
        Some((HostPhysAddr::from_usize(host_vaddr.as_usize()), limit))
    }
}
```

Note：这里返回类型仍叫 `HostPhysAddr`，但为了适配 trait 默认实现，AxVM 胶水层实际塞进去的是可解引用的 HVA 数值。这是当前实验性接线的 API 妥协点。

### 6.3 AxVisor 标准设备注册路径

```
VM TOML 配置
  └─ [devices].emu_devices = [...]
       └─ EmulatedDeviceConfig { emu_type, base_gpa, length, irq_id, cfg_list }

AxVmDevices::new(configs)
  └─ AxVmDevices::init()
       └─ match EmulatedDeviceType
            └─ add_mmio_dev/add_port_dev/add_sys_reg_dev

VM exit dispatch
  └─ AxVmDevices::handle_port_read/write()
       └─ find_port_dev(port)
            └─ BasePortDeviceOps::handle_read/write()
```

virtio-blk 想完全符合这条路径，需要先解决“port 设备处理函数拿不到 guest memory”的 API 边界。直接让设备持有 `AddrSpace` 引用会把 VM 生命周期、锁和设备对象强绑定，通常不是最干净的方向。更合理的方向是给需要 DMA 的设备建立一条 VMM 调用入口，例如扩展设备分发接口或新增 DMA-capable port device trait。

### 6.4 当前实现和下一步正规化

| 文件 | 当前状态 | 下一步 |
|------|------|
| `axdevice/src/virtio_blk/` | 已承载设备模型核心逻辑 | 保持，不应退回 `axvm` |
| `axvm/src/vm.rs` | 保留 `handle_ovmf_virtio_blk_*` 胶水，提供 `AxVmGuestMemory` | 等设备分发 API 支持 DMA 后再删除 |
| `axvmconfig/src/lib.rs` | 还没有 `VirtioBlk` emu type | 正规注册时新增 |
| `axdevice/src/device.rs` | 还没有从 `EmulatedDeviceConfig` 创建 virtio-blk | 正规注册时新增分支 |
| `ovmf-x86_64-qemu-smp1.toml` | `disk_path` 暂放在 `[kernel]` | 后续应迁到设备配置或 VM image/device 明确边界 |
| `x86_vcpu/src/vmx/vcpu.rs` | I/O bitmap 仍需拦截 `0x6000..0x607f` | 保持 exit，分发目标可调整 |

## 7. Backend 选择

最小可行 backend 只需要一个 trait：

```rust
trait BlockBackend {
    fn read_at(&self, offset: u64, buf: &mut [u8]) -> io::Result<()>;
    fn write_at(&self, offset: u64, buf: &[u8]) -> io::Result<()>;
    fn num_sectors(&self) -> u64;
}
```

当前实现使用 `MemoryDiskBackend`，在镜像加载阶段把 raw disk 文件读成 `Vec<u8>`，再安装到设备模型里。这足够验证 OVMF 读盘链路，但不是长期存储后端：写请求只改内存镜像，不会回写 rootfs 文件。

当前启动路径还有一层容易混淆的磁盘关系：

- `--rootfs "$WORKSPACE/tmp/uefi-boot-test.img"` 是外层 QEMU 提供给 AxVisor 的 rootfs。
- `disk_path = "/guest/disks/uefi-boot-test.img"` 是 AxVisor 在自己的 rootfs 里读取的嵌套 raw disk 文件。
- OVMF 看到的 virtio-blk 后端是这个嵌套 raw disk 被读入内存后的 `MemoryDiskBackend`，不是外层 `--rootfs` 本身。

加载位置：

- `tgoskits/os/axvisor/src/vmm/images/mod.rs`
  - `VMImageLoader::load_uefi_images()`
  - `VMImageLoader::load_virtio_blk_disk_from_filesystem()`
- `tgoskits/components/axvm/src/vm.rs`
  - `AxVM::install_virtio_blk_disk_image()`

后续更标准的 backend 可以抽成 trait，并支持：

- raw 文件 backend：按 offset 读写文件。
- 只读 raw image：用于固件启动 smoke。
- qcow2 或 vhost-user：更接近成熟 VMM 的块设备后端。
