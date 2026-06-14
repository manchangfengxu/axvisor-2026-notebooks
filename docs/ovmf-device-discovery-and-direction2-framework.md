# OVMF 设备发现链路与方向二设备框架

这份笔记只解释一件事：**ACPI、PCI、VM exit、设备模型这些词，放到当前 AxVisor 项目里到底是什么，它们怎样串成一条真实链路。**

目标不是讲标准，而是帮助读源码。下面所有“当前实现”都指当前 OVMF virtio-blk 实验路径；所有“方向二目标”都指设备与中断框架重构之后更正规的接法。

## 1. 先给结论

当前 OVMF virtio-blk 路径大概是这样：

```text
OVMF
  -> 通过 fw_cfg/ACPI/PCI 发现平台和设备
  -> 访问 virtio-blk legacy I/O port 0x6000..0x607f
  -> 触发 VM exit
  -> axvm::vm.rs 手动判断这个 port 属于 OVMF virtio-blk
  -> 调用 axdevice::virtio_blk::LegacyVirtioBlk
  -> LegacyVirtioBlk 读写 guest memory 里的 virtqueue
  -> MemoryDiskBackend 返回磁盘数据
```

方向二希望变成：

```text
OVMF
  -> 通过 AxVisor 生成的 ACPI/PCI 发现设备
  -> 访问设备 BAR / PIO / MMIO
  -> 触发 VM exit
  -> VM exit 统一包装成 BusAccess
  -> BusRouter 按地址找到注册过的设备
  -> 设备通过 DeviceContext 访问 guest memory / 发中断
  -> virtio-blk 后端完成读写
```

区别不在 virtio-blk 内部会不会处理请求。区别在于：**当前是 VM exit 里硬编码认识 virtio-blk；方向二是设备先注册资源，VM exit 只做统一分发。**

## 2. 当前 AxVisor 里已经存在的对象

先不要想 ACPI/PCI 标准。先看当前项目里真的有什么对象。

### 2.1 AxVMInnerMut：一个 VM 的可变运行状态

位置：

```text
tgoskits/components/axvm/src/vm.rs
```

当前相关字段：

```rust
struct AxVMInnerMut {
    address_space: AddrSpace<PagingHandlerImpl>,
    memory_regions: Vec<VMMemoryRegion>,
    config: AxVMConfig,
    vm_status: VMStatus,

    #[cfg(target_arch = "x86_64")]
    fw_cfg: FwCfgState,

    #[cfg(target_arch = "x86_64")]
    ovmf_virtio_blk: LegacyVirtioBlk,
}
```

可以把它理解成：

```text
AxVMInnerMut 是一个 VM 的“状态包”。

address_space:
  这个 VM 的 GPA -> HPA/HVA 翻译能力。
  virtio-blk 处理 virtqueue 时要靠它读写 guest memory。

fw_cfg:
  当前简化版 QEMU fw_cfg 设备状态。
  OVMF 通过 port 0x510/0x511 访问它。

ovmf_virtio_blk:
  当前专门给 OVMF 路径挂上的 virtio-blk 设备模型。
  它不是通过 AxVmDevices 正规注册进来的，而是直接塞在 AxVMInnerMut 里。
```

这里最关键的是 `ovmf_virtio_blk`。它说明当前实现还不是完整设备框架，而是为了 OVMF 启动链先特化接上一个设备。

### 2.2 AxVmDevices：当前通用设备容器

位置：

```text
tgoskits/components/axdevice/src/device.rs
```

它现在把设备分成三类：

```rust
pub struct AxVmDevices {
    emu_mmio_devices: AxEmuMmioDevices,
    emu_sys_reg_devices: AxEmuSysRegDevices,
    emu_port_devices: AxEmuPortDevices,
    ivc_channel: Option<Mutex<RangeAllocator<usize>>>,
}
```

含义：

```text
emu_mmio_devices:
  处理 MMIO，也就是 guest 访问某段内存地址触发的设备访问。

emu_sys_reg_devices:
  处理系统寄存器访问，主要是 ARM 这类架构。

emu_port_devices:
  处理 x86 port I/O，也就是 in/out 指令。
```

当前查找方式很直接：

```rust
pub fn handle_port_read(&self, port: Port, width: AccessWidth) -> AxResult<usize> {
    if let Some(emu_dev) = self.find_port_dev(port) {
        return emu_dev.handle_read(port, width);
    }
    panic_device_not_found("port", port, true, width);
}
```

这表示：

```text
AxVmDevices 会在线性设备列表里找“谁拥有这个 port”。
找到后调用设备的 handle_read / handle_write。
找不到就 panic。
```

这个设计能工作，但对 virtio-blk 这种设备不够。原因后面讲。

### 2.3 BaseDeviceOps：当前设备接口

位置：

```text
tgoskits/components/axdevice_base/src/lib.rs
```

核心接口：

```rust
pub trait BaseDeviceOps<R: DeviceAddrRange>: Any {
    fn emu_type(&self) -> EmuDeviceType;
    fn address_range(&self) -> R;
    fn handle_read(&self, addr: R::Addr, width: AccessWidth) -> AxResult<usize>;
    fn handle_write(&self, addr: R::Addr, width: AccessWidth, val: usize) -> AxResult;
}

pub trait BasePortDeviceOps = BaseDeviceOps<PortRange>;
```

它的含义很朴素：

```text
address_range:
  设备说自己占用哪段地址或者 port。

handle_read:
  guest 读设备寄存器时调用。

handle_write:
  guest 写设备寄存器时调用。
```

问题是：这个接口没有给设备传 `guest memory`。

普通串口设备可能只需要读写寄存器。但 virtio-blk 不一样。virtio-blk 收到 notify 后，要去 guest memory 里读 descriptor table、avail ring、data buffer，再写 used ring 和 status byte。

所以当前 `BasePortDeviceOps` 装不下 virtio-blk 的完整需求。

这也是为什么当前代码没有把 `LegacyVirtioBlk` 注册进 `AxVmDevices`，而是在 `axvm::vm.rs` 里单独走特化路径。

## 3. 当前真实链路：从 OVMF 访问设备到读磁盘

下面按真实时间顺序串。

### 3.1 VM 启动时准备 fw_cfg 和 virtio-blk

当前 `AxVMInnerMut` 里有：

```rust
fw_cfg: FwCfgState,
ovmf_virtio_blk: LegacyVirtioBlk,
```

`FwCfgState` 是一个简化的 QEMU fw_cfg：

```rust
struct FwCfgState {
    selector: u16,
    offset: usize,
    dma_address: u64,
    dma_bytes: [u8; 8],
    dma_bytes_written: usize,
    items: BTreeMap<u16, Vec<u8>>,
}
```

可以理解成：

```text
items:
  fw_cfg 能提供给 OVMF 的数据项。
  当前包括 QEMU signature、interface version、CPU count、E820。

selector:
  OVMF 当前选中了哪个 fw_cfg item。

offset:
  当前从该 item 读到哪里。

dma_address / dma_bytes:
  fw_cfg DMA 访问用的临时状态。
```

当前 `FwCfgState::configure()` 会填：

```rust
self.items.insert(QEMU_FW_CFG_ITEM_SIGNATURE, b"QEMU".to_vec());
self.items.insert(QEMU_FW_CFG_ITEM_INTERFACE_VERSION, FW_CFG_F_DMA.to_le_bytes().to_vec());
self.items.insert(QEMU_FW_CFG_ITEM_SMP_CPU_COUNT, (cpu_count as u16).to_le_bytes().to_vec());
self.items.insert(QEMU_FW_CFG_ITEM_ETC_E820, Self::build_e820(memory_regions));
self.items.insert(QEMU_FW_CFG_ITEM_FILE_DIR, self.build_file_dir());
```

注意：这只是当前最小 fw_cfg。更完整的 UEFI 路线里，ACPI 表也应该能通过 fw_cfg table-loader 交给 OVMF。

### 3.2 OVMF 访问 fw_cfg：当前是 axvm::vm.rs 手动处理

OVMF 访问 fw_cfg 时，会执行 x86 port I/O，比如：

```text
out 0x510, selector
in  0x511
```

这会触发 VM exit，进入：

```text
tgoskits/components/axvm/src/vm.rs
```

当前 VM exit 处理里有：

```rust
AxVCpuExitReason::IoRead { port, width } => {
    if port.0 == 0x402 {
        ...
    } else if let Some(val) = self.handle_fw_cfg_io_read(*port, *width)? {
        vcpu.set_gpr(0, val);
        true
    } else if let Some(val) = self.handle_ovmf_virtio_blk_io_read(*port, *width)? {
        vcpu.set_gpr(0, val);
        true
    } else if let Some(val) = self.handle_acpi_pm_io_read(*port, *width)? {
        vcpu.set_gpr(0, val);
        true
    } else {
        let val = self.get_devices().handle_port_read(*port, *width)?;
        vcpu.set_gpr(0, val);
        true
    }
}
```

写 port 时类似：

```rust
AxVCpuExitReason::IoWrite { port, width, data } => {
    if port.0 == 0x402 {
        ...
    } else if self.handle_fw_cfg_io_write(*port, *width, *data as usize)? {
        true
    } else if self.handle_ovmf_virtio_blk_io_write(*port, *width, *data as usize)? {
        true
    } else if self.handle_acpi_pm_io_write(*port, *width, *data as usize)? {
        true
    } else {
        self.get_devices().handle_port_write(*port, *width, *data as usize)?;
        true
    }
}
```

这段代码非常重要。它说明当前有两条路：

```text
特化路：
  fw_cfg / OVMF virtio-blk / ACPI PM IO
  在 axvm::vm.rs 里先手动判断。

通用路：
  self.get_devices().handle_port_read/write()
  走 AxVmDevices。
```

当前 virtio-blk 走的是特化路，不是通用路。

### 3.3 当前 ACPI/PCI 发现为什么不完整

从项目目标看，标准路线应该是：

```text
AxVisor 生成 ACPI
  -> ACPI 里描述 PCI root bridge
  -> MCFG 描述 PCI ECAM 地址
  -> OVMF 枚举 AxVisor 自己的 PCI config space
  -> OVMF 发现 AxVisor 自己注册的 virtio-blk PCI function
```

但当前实验路径不是这样。

当前 OVMF 能发现 virtio-blk，很大程度上依赖外层 QEMU runtime 暴露了一个 virtio-blk PCI 设备。AxVisor 再对这个设备对应的 legacy I/O port 做特化接管。

所以当前更像：

```text
外层 QEMU 提供 PCI 发现外壳
  -> OVMF 发现 virtio-blk
  -> OVMF 访问 legacy I/O port 0x6000..0x607f
  -> AxVisor 拦截这些 port
  -> AxVisor 自己处理 virtio-blk 数据面
```

这就是为什么当前实现可以跑通 OVMF 启动链，但还不是方向一最终目标里的“AxVisor 自己提供 PC 平台骨架”。

## 4. 当前 virtio-blk 是怎么被读写驱动的

virtio-blk 的源码在：

```text
tgoskits/components/axdevice/src/virtio_blk/
```

关键文件：

```text
legacy.rs:
  LegacyVirtioBlk，处理 legacy virtio I/O port 寄存器。

queue.rs:
  LegacyQueue，处理 split virtqueue 布局、avail ring、used ring。

request.rs:
  VirtioBlkRequest，解析 block read/write/get_id/flush 请求。

backend.rs:
  MemoryDiskBackend，当前内存磁盘后端。
```

### 4.1 LegacyVirtioBlk 持有什么状态

位置：

```text
tgoskits/components/axdevice/src/virtio_blk/legacy.rs
```

结构体：

```rust
pub struct LegacyVirtioBlk {
    queue: LegacyQueue,
    backend: MemoryDiskBackend,
    driver_features: u32,
    queue_select: u16,
    device_status: u8,
    serial: [u8; 20],
}
```

解释：

```text
queue:
  virtqueue 状态。
  OVMF 把请求放在 guest memory 里，queue 记录这些结构在哪里。

backend:
  真正的磁盘后端。
  当前是 MemoryDiskBackend，也就是 Vec<u8>。

driver_features:
  OVMF 写进来的 feature 协商结果。

queue_select:
  当前选择哪个 virtqueue。当前只支持 queue 0。

device_status:
  virtio 设备状态寄存器。
  OVMF 通过写这个寄存器推进 ACKNOWLEDGE / DRIVER / DRIVER_OK 等状态。

serial:
  GET_ID 请求返回给 guest 的磁盘序列号。
```

这里的“状态机”不是一个叫 `run_state_machine()` 的函数。它是由 OVMF 对设备寄存器的读写一步步推进的。

### 4.2 哪些 port 属于当前 virtio-blk

当前固定：

```rust
pub const LEGACY_BLK_IO_BASE: u16 = 0x6000;
pub const LEGACY_BLK_IO_SIZE: u16 = 0x80;

pub fn owns_port(port: Port) -> bool {
    (LEGACY_BLK_IO_BASE..LEGACY_BLK_IO_BASE + LEGACY_BLK_IO_SIZE).contains(&port.number())
}
```

意思是：

```text
只要 OVMF 访问 0x6000..0x607f，当前实现就认为这是 virtio-blk legacy I/O BAR。
```

这就是特化点。标准方向二里，不应该由 `axvm::vm.rs` 固定知道这个范围，而应该由设备注册时声明：

```text
VirtioBlkLegacyFrontend 注册：
  PIO range = 0x6000..0x6080
```

然后 BusRouter 根据范围分发。

### 4.3 OVMF 读寄存器

当前读路径：

```rust
pub fn handle_read<M: GuestMemoryAccessor>(
    &mut self,
    port: Port,
    width: AccessWidth,
    _mem: &M,
) -> AxResult<Option<usize>> {
    if !Self::owns_port(port) {
        return Ok(None);
    }

    let offset = port.number() - LEGACY_BLK_IO_BASE;
    let mut value = 0usize;
    for byte in 0..width.size() {
        value |= (self.read_register_byte(offset + byte as u16) as usize) << (byte * 8);
    }
    Ok(Some(value))
}
```

这表示：

```text
port 0x6000 对设备来说不是绝对地址。
设备内部先算 offset = port - 0x6000。
然后根据 offset 判断 guest 在读哪个 virtio 寄存器。
```

例如：

```rust
const REG_DEVICE_FEATURES: u16 = 0x00;
const REG_QUEUE_SIZE: u16 = 0x0c;
const REG_DEVICE_STATUS: u16 = 0x12;
const REG_CONFIG: u16 = 0x14;
```

所以：

```text
in 0x6000:
  offset 0x00
  读 device features

in 0x600c:
  offset 0x0c
  读 queue size

in 0x6014:
  offset 0x14
  读 block capacity 低 32 位
```

这也解释了日志：

```text
[OVMF-VIRTIO-BLK-IO] in port=0x6014 width=Dword value=0x10000
```

`0x6014 - 0x6000 = 0x14`，也就是 virtio-blk config 里的 capacity。

### 4.4 OVMF 写寄存器

当前写路径：

```rust
pub fn handle_write<M: GuestMemoryAccessor>(
    &mut self,
    port: Port,
    width: AccessWidth,
    value: usize,
    mem: &M,
) -> AxResult<bool> {
    if !Self::owns_port(port) {
        return Ok(false);
    }

    let offset = port.number() - LEGACY_BLK_IO_BASE;
    match (offset, width) {
        (REG_GUEST_FEATURES, AccessWidth::Dword) => {
            self.driver_features = value as u32;
        }
        (REG_QUEUE_PFN, AccessWidth::Dword) => {
            self.queue.set_pfn(value as u32);
        }
        (REG_QUEUE_SELECT, AccessWidth::Word) => {
            self.queue_select = value as u16;
        }
        (REG_QUEUE_NOTIFY, AccessWidth::Word) => {
            self.process_queue(mem)?;
        }
        (REG_DEVICE_STATUS, AccessWidth::Byte) => {
            self.device_status = value as u8;
            if self.device_status == 0 {
                self.queue.reset();
                self.queue_select = 0;
                self.driver_features = 0;
            }
        }
        _ => {}
    }

    Ok(true)
}
```

这就是设备模型状态机。

用人话说：

```text
OVMF 写 GUEST_FEATURES:
  设备记录 driver 接受了哪些 feature。

OVMF 写 QUEUE_PFN:
  设备记录 virtqueue 在 guest memory 的哪一页。

OVMF 写 QUEUE_SELECT:
  设备记录当前配置哪个 queue。

OVMF 写 DEVICE_STATUS:
  设备记录 virtio 初始化状态。
  如果写 0，表示 reset，清空 queue/features。

OVMF 写 QUEUE_NOTIFY:
  这不是普通配置了。
  这表示 guest 已经把请求放进 virtqueue。
  设备要开始读 guest memory，处理 block 请求。
```

### 4.5 为什么需要 AxVmGuestMemory

当前 `LegacyVirtioBlk::handle_write()` 的最后一个参数是：

```rust
mem: &M
where M: GuestMemoryAccessor
```

这是因为 virtqueue 不在设备对象内部，而在 guest memory 里。

OVMF 写 `QUEUE_PFN` 后，设备只知道：

```text
queue_pfn = 0x29b5
queue_gpa = 0x29b5000
```

但真正的 descriptor table、avail ring、used ring 都在 guest memory 里。设备要读它们，必须有一个“把 guest physical address 转成 host 可读写地址”的能力。

当前这个能力由 `AxVmGuestMemory` 提供：

```rust
struct AxVmGuestMemory<'a> {
    address_space: &'a AddrSpace<PagingHandlerImpl>,
}

impl GuestMemoryAccessor for AxVmGuestMemory<'_> {
    fn translate_and_get_limit(&self, guest_addr: GuestPhysAddr) -> Option<(HostPhysAddr, usize)> {
        let (host_paddr, limit) = self.address_space.translate_and_get_limit(guest_addr)?;
        let host_vaddr = PagingHandlerImpl::phys_to_virt(host_paddr);
        Some((HostPhysAddr::from_usize(host_vaddr.as_usize()), limit))
    }
}
```

这段代码做的是：

```text
guest GPA
  -> address_space.translate_and_get_limit()
  -> host physical address
  -> PagingHandlerImpl::phys_to_virt()
  -> AxVisor 可以解引用的 host virtual address
```

所以 `AxVmGuestMemory` 不是一个设备。它是设备读写 guest memory 的“翻译适配器”。

当前 `axvm::vm.rs` 特化 virtio-blk 时，会手动构造它：

```rust
let AxVMInnerMut {
    address_space,
    ovmf_virtio_blk,
    ..
} = &mut *g;

let mem = AxVmGuestMemory { address_space };
let handled = ovmf_virtio_blk.handle_write(port, width, val, &mem)?;
```

这就是当前为什么不能直接走 `AxVmDevices::handle_port_write()`。

`AxVmDevices::handle_port_write()` 只能调用：

```rust
emu_dev.handle_write(port, width, val)
```

它没地方传：

```text
&AxVmGuestMemory
```

### 4.6 queue notify 后发生什么

当 OVMF 写：

```text
out 0x6010, 0
```

设备内部看到：

```text
offset = 0x6010 - 0x6000 = 0x10
REG_QUEUE_NOTIFY
```

然后调用：

```rust
self.process_queue(mem)?;
```

核心代码：

```rust
fn process_queue<M: GuestMemoryAccessor>(&mut self, mem: &M) -> AxResult {
    while let Some(head) = self.queue.pop_available(mem)? {
        let used_len = self.process_chain(mem, head);
        self.queue.publish_used(mem, head, used_len)?;
    }
    Ok(())
}
```

这段话翻译成中文：

```text
while guest 的 avail ring 里还有新请求:
  取出一个 descriptor chain 的头编号 head
  解析并执行这个 chain
  把完成结果写进 used ring
```

`LegacyQueue::pop_available()` 在：

```text
tgoskits/components/axdevice/src/virtio_blk/queue.rs
```

它会读 guest memory：

```rust
mem.read_buffer(avail_ring_gpa + 2, &mut idx)?;
mem.read_buffer(avail_ring_gpa + 4 + slot * 2, &mut head)?;
```

含义：

```text
avail.idx:
  guest 已经提交到第几个请求。

avail.ring[slot]:
  某个请求的 descriptor chain 头编号。
```

之后 `process_chain()` 会调用：

```rust
self.parse_and_execute_chain(mem, head)
```

里面做：

```rust
let chain = self.queue.collect_chain(mem, head)?;
let header = chain[0];
mem.read_buffer(header.addr, &mut raw_header)?;
let request_type = u32::from_le_bytes(raw_header[0..4].try_into().unwrap());
let sector = u64::from_le_bytes(raw_header[8..16].try_into().unwrap());
```

翻译：

```text
collect_chain:
  从 descriptor table 里沿 next 指针读完整个 descriptor chain。

header:
  第一个 descriptor 指向 virtio-blk 请求头。

request_type:
  是读盘、写盘、flush，还是 get_id。

sector:
  从哪个扇区开始读写。
```

如果是读盘：

```rust
fn execute_read<M: GuestMemoryAccessor>(
    &self,
    mem: &M,
    request: &VirtioBlkRequest,
) -> Result<u32, ()> {
    let mut disk_offset = request.sector.checked_mul(SECTOR_SIZE as u64).ok_or(())?;
    let mut used_len = 1u32;

    for (gpa, len) in &request.data {
        let mut data = alloc::vec![0u8; *len as usize];
        self.backend.read_at(disk_offset, &mut data);
        mem.write_buffer(*gpa, &data)?;
        disk_offset += *len as u64;
        used_len += *len;
    }

    Ok(used_len)
}
```

含义：

```text
从 MemoryDiskBackend 读数据
  -> 写到 guest 提供的数据 buffer
  -> 最后写 status byte
  -> 更新 used ring
```

这就是 OVMF 能从 virtio-blk 里读 FAT/EFI 文件的根本原因。

## 5. 方向二到底要改什么

方向二不是要改 virtio-blk 读盘算法本身。它要改的是“谁负责把访问送到设备”。

当前是：

```text
VM exit 里写死：
  如果 port 是 fw_cfg，就调 fw_cfg
  如果 port 是 virtio-blk，就调 ovmf_virtio_blk
  如果 port 是 ACPI PM，就调 ACPI PM
  否则再去 AxVmDevices 找
```

方向二希望：

```text
设备启动时声明自己占用哪些资源。
VM exit 不认识具体设备。
VM exit 只把访问交给 BusRouter。
BusRouter 根据资源表找到设备。
设备需要 guest memory / IRQ 时，从 DeviceContext 里拿。
```

### 5.1 方向二里的几个模块是什么

这些名字可以先当成目标设计，不代表当前都已经存在。

#### DeviceRegistry

它是“设备登记册”。

负责记录：

```text
这个 VM 有哪些设备
每个设备的 DeviceId 是什么
每个设备占用哪些资源
资源之间有没有冲突
```

伪代码：

```rust
struct DeviceRegistry {
    devices: Vec<Box<dyn DeviceModel>>,
    resources: Vec<(Resource, DeviceId)>,
}

enum Resource {
    Pio { base: u16, size: u16 },
    Mmio { base: usize, size: usize },
    PciFunction { segment: u16, bus: u8, device: u8, function: u8 },
    IrqLine { gsi: u32 },
}

impl DeviceRegistry {
    fn register(&mut self, device: Box<dyn DeviceModel>, resources: Vec<Resource>) -> Result<DeviceId> {
        self.check_conflict(&resources)?;
        let id = self.alloc_device_id();
        self.resources.extend(resources.into_iter().map(|r| (r, id)));
        self.devices.push(device);
        Ok(id)
    }
}
```

它解决当前什么问题？

```text
当前：
  AxVmDevices 分 MMIO/SysReg/Port 三个 Vec。
  新设备经常要改 EmulatedDeviceType 和 match。
  资源冲突检查弱。

方向二：
  设备自己声明资源。
  注册时统一检查。
  新设备不应该到处改中心 match。
```

#### BusRouter

它是“总线路由器”。

负责把一次 VM exit 产生的访问送到正确设备。

伪代码：

```rust
struct BusAccess {
    kind: BusKind,
    addr: u64,
    width: AccessWidth,
    op: BusOp,
}

enum BusKind {
    Pio,
    Mmio,
    SysReg,
}

enum BusOp {
    Read,
    Write { value: usize },
}

struct BusRouter {
    pio_ranges: RangeMap<u16, DeviceId>,
    mmio_ranges: RangeMap<usize, DeviceId>,
}

impl BusRouter {
    fn route(&self, access: &BusAccess) -> Result<DeviceId> {
        self.lookup(access.kind, access.addr)
    }
}
```

这里故意让 `BusRouter` 只返回 `DeviceId`。真实 Rust 代码里通常不要让 router 同时借设备表和上下文，否则容易把可变借用关系写复杂。更清楚的职责划分是：

```rust
let device_id = vm.bus_router.route(&access)?;
let device = vm.registry.get_mut(device_id)?;
let response = device.handle_access(access, &mut vm.device_context)?;
```

它解决当前什么问题？

```text
当前：
  axvm::vm.rs 直接 if/else 判断 fw_cfg、virtio-blk、ACPI PM。

方向二：
  axvm::vm.rs 不知道具体设备。
  它只把 exit 转成 BusAccess。
```

当前 VM exit 伪代码：

```rust
match exit {
    IoRead { port, width } => {
        if port == FW_CFG_IO_DATA {
            handle_fw_cfg_io_read(port, width)
        } else if LegacyVirtioBlk::owns_port(port) {
            handle_ovmf_virtio_blk_io_read(port, width)
        } else {
            devices.handle_port_read(port, width)
        }
    }
}
```

方向二目标伪代码：

```rust
match exit {
    IoRead { port, width } => {
        let access = BusAccess::pio_read(port, width);
        let device_id = vm.bus_router.route(&access)?;
        let device = vm.registry.get_mut(device_id)?;
        let response = device.handle_access(access, &mut vm.device_context)?;
        vcpu.set_gpr(0, response.value);
    }

    IoWrite { port, width, data } => {
        let access = BusAccess::pio_write(port, width, data);
        let device_id = vm.bus_router.route(&access)?;
        let device = vm.registry.get_mut(device_id)?;
        device.handle_access(access, &mut vm.device_context)?;
    }
}
```

#### DeviceContext

它是“设备处理请求时能用的 VM 能力包”。

设备模型不应该直接拿整个 `AxVM`。但它确实需要一些能力：

```text
读写 guest memory
发中断
访问时间/定时器
访问配置
记录日志
```

所以方向二应该传一个精简上下文：

```rust
struct DeviceContext<'a> {
    guest_memory: &'a dyn GuestMemoryAccessor,
    irq_sink: &'a dyn IrqSink,
    registry: &'a mut DeviceRegistry,
}
```

对 virtio-blk 来说，最重要的是：

```text
guest_memory:
  读 descriptor、读写 data buffer、写 used ring。

irq_sink:
  请求完成后通知 guest。
```

这正好修复当前 `BasePortDeviceOps` 的问题。

当前接口：

```rust
fn handle_write(&self, addr, width, val) -> AxResult;
```

方向二目标接口：

```rust
fn handle_access(&mut self, access: BusAccess, ctx: &mut DeviceContext) -> AxResult<BusResponse>;
```

差异：

```text
当前设备只能知道“guest 写了哪个寄存器”。
方向二设备还能通过 ctx 读 guest memory、发 IRQ。
```

#### InterruptRouter / IrqSink

它是“设备发中断的出口”。

设备不应该知道 x86 的 vLAPIC/vIOAPIC 怎么实现，也不应该知道 RISC-V 的 vPLIC 怎么实现。

设备只应该说：

```text
我的请求完成了，请帮我发一个中断。
```

伪代码：

```rust
trait IrqSink {
    fn raise(&self, irq: IrqLine);
    fn lower(&self, irq: IrqLine);
    fn pulse(&self, irq: IrqLine);
    fn msi(&self, msg: MsiMessage);
}

struct InterruptRouter {
    routes: Vec<IrqRoute>,
}

impl IrqSink for InterruptRouter {
    fn pulse(&self, irq: IrqLine) {
        match self.resolve(irq) {
            IrqTarget::X86IoApic { gsi } => self.vioapic.set_irq(gsi, true),
            IrqTarget::X86Msi { addr, data } => self.vlapic.inject_msi(addr, data),
            IrqTarget::RvPlic { source } => self.vplic.set_pending(source),
            IrqTarget::ArmGic { intid } => self.vgic.inject(intid),
        }
    }
}
```

对 OVMF 最小 virtio-blk 来说，早期可能 polling 就够了。但要变成标准设备，最终需要这条路。

#### AcpiBuilder

它是“把设备注册表转换成 ACPI 表”的模块。

注意：ACPI 不是设备模型。ACPI 是给 OVMF/OS 看的平台说明。

方向二下，ACPI builder 应该从平台配置和设备注册信息生成：

```text
RSDP
XSDT
FADT
MADT
MCFG
DSDT
```

最重要的是 DSDT/MCFG 里的 PCI root bridge 信息：

```text
Device (PCI0)
  _HID = PNP0A08
  _CID = PNP0A03
  _SEG = 0
  _BBN = 0
  _CRS = bus range + MMIO window + PIO window
  _PRT = INTx routing

MCFG:
  ECAM base address
  segment
  bus range
```

它的输入不应该是“拍脑袋写死”。应该来自：

```text
PciBus / PciHostBridge 的配置
InterruptRouter 的 INTx 路由
VM memory layout
CPU topology
```

伪代码：

```rust
struct AcpiBuilder;

impl AcpiBuilder {
    fn build(vm: &VmPlatform) -> AcpiTables {
        let madt = build_madt(vm.cpu_topology, vm.interrupts);
        let mcfg = build_mcfg(vm.pci.ecam_base, vm.pci.bus_range);
        let dsdt = build_dsdt(vm.pci.root_bridge, vm.devices);
        let fadt = build_fadt(vm.pm);
        let xsdt = build_xsdt([fadt, madt, mcfg, dsdt]);
        let rsdp = build_rsdp(xsdt);

        AcpiTables { rsdp, tables: vec![xsdt, fadt, madt, mcfg, dsdt] }
    }
}
```

#### PciBus / PciConfigSpace

它是“OVMF 枚举设备时读到的 PCI 世界”。

ACPI 只告诉 OVMF：

```text
PCI 总线在哪里。
ECAM 配置空间在哪里。
```

真正让 OVMF 发现 virtio-blk 的是 PCI config space。

方向二应该有类似：

```rust
struct PciBus {
    functions: Vec<PciFunction>,
}

struct VirtioBlkPciDevice {
    bdf: PciBdf,
    config_space: PciConfigSpace,
    frontend: VirtioBlkFrontend,
}
```

当 OVMF 读 ECAM：

```text
MMIO read ECAM_BASE + offset(bus, device, function, register)
```

BusRouter 应该把它分发给 `PciConfigSpace` 设备。

`PciConfigSpace` 返回：

```text
vendor_id = 0x1af4
device_id = virtio-blk 对应 ID
class code = mass storage / block
BAR = virtio-blk 寄存器窗口
interrupt pin / MSI capability
```

然后 OVMF 才知道：

```text
这里有一个 virtio-blk 设备。
我要去访问它的 BAR。
```

## 6. 方向二下，一条完整链路应该怎么走

下面把“发现到触发”完整串起来。

### 6.1 VM 创建阶段

方向二目标伪代码：

```rust
fn create_uefi_vm(config: VmConfig) -> AxVM {
    let mut registry = DeviceRegistry::new();
    let mut irq_router = InterruptRouter::new();

    let fw_cfg = FwCfgDevice::new();
    registry.register(
        Box::new(fw_cfg),
        vec![Resource::Pio { base: 0x510, size: 0x08 }],
    )?;

    let pci_host = PciHostBridge::new(
        ecam_base = 0xe000_0000,
        bus_start = 0,
        bus_end = 0,
    );
    registry.register(
        Box::new(pci_host),
        vec![
            Resource::Mmio { base: 0xe000_0000, size: 0x1000_0000 },
            Resource::Pio { base: 0xcf8, size: 0x08 },
        ],
    )?;

    let virtio_blk = VirtioBlkPciDevice::new(
        bdf = "00:03.0",
        backend = MemoryDiskBackend::from_image(config.disk_image),
    );
    registry.register(
        Box::new(virtio_blk),
        vec![
            Resource::PciFunction { segment: 0, bus: 0, device: 3, function: 0 },
            Resource::Pio { base: 0x6000, size: 0x80 },
            Resource::IrqLine { gsi: 11 },
        ],
    )?;

    irq_router.add_route(IrqLine(11), IrqTarget::X86IoApic { gsi: 11 });

    let acpi_tables = AcpiBuilder::build(&registry, &irq_router, &config);
    registry.get_mut::<FwCfgDevice>().add_acpi(acpi_tables);

    AxVM {
        registry,
        bus_router: BusRouter::from_registry(&registry),
        irq_router,
        ...
    }
}
```

这段伪代码表达的不是具体语法，而是职责：

```text
先注册设备。
注册时声明资源。
根据注册好的平台生成 ACPI。
把 ACPI 交给 fw_cfg 或放到 guest memory。
VM exit 后只走 BusRouter。
```

### 6.2 OVMF 读取 ACPI

方向二目标：

```text
OVMF 访问 fw_cfg
  -> VM exit
  -> BusAccess::pio_read/write(0x510/0x511)
  -> BusRouter 找到 FwCfgDevice
  -> FwCfgDevice 返回 ACPI table-loader / RSDP / tables
```

伪代码：

```rust
fn handle_vm_exit(exit: AxVCpuExitReason) {
    match exit {
        IoRead { port, width } => {
            let access = BusAccess::pio_read(port, width);
            let device_id = self.bus_router.route(&access)?;
            let device = self.registry.get_mut(device_id)?;
            let response = device.handle_access(access, &mut self.device_context)?;
            vcpu.set_gpr(0, response.value);
        }
    }
}
```

这里 VM exit 不知道 `FwCfgDevice`。

### 6.3 OVMF 发现 PCI root bridge

OVMF 拿到 ACPI 后，读到：

```text
DSDT:
  Device (PCI0) ...

MCFG:
  ECAM base = 0xe0000000
```

这一步没有调用 virtio-blk。它只是让 OVMF 知道：

```text
PCI 配置空间可以通过 ECAM 地址访问。
```

### 6.4 OVMF 枚举 PCI config space

OVMF 访问：

```text
MMIO read 0xe0000000 + bdf/register offset
```

方向二链路：

```text
OVMF MMIO read ECAM
  -> VM exit MmioRead
  -> BusAccess::mmio_read(ecam_addr, width)
  -> BusRouter 找到 PciHostBridge / PciConfigSpace
  -> PciConfigSpace 根据 BDF 返回 vendor_id/device_id/BAR
```

伪代码：

```rust
impl DeviceModel for PciConfigSpace {
    fn handle_access(&mut self, access: BusAccess, ctx: &mut DeviceContext) -> AxResult<BusResponse> {
        let (bdf, register) = decode_ecam(access.addr);
        let function = self.functions.get(bdf);

        if access.is_read() {
            return Ok(BusResponse::value(function.read_config(register, access.width)));
        }

        function.write_config(register, access.value);
        Ok(BusResponse::none())
    }
}
```

如果 BDF 是 virtio-blk，比如 `00:03.0`，它会返回：

```text
vendor_id = 0x1af4
device_id = virtio block
BAR0 = 0x6000 或某段 MMIO
IRQ pin = INTA
```

于是 OVMF 加载 virtio-blk driver。

### 6.5 OVMF 初始化 virtio-blk

OVMF 访问 virtio-blk BAR：

```text
in  0x6000       读 device features
out 0x6004       写 driver features
out 0x600e       选择 queue 0
in  0x600c       读 queue size
out 0x6008       写 queue_pfn
out 0x6012       写 device status
```

方向二链路：

```text
OVMF in/out 0x6000..0x607f
  -> VM exit IoRead/IoWrite
  -> BusAccess::pio_read/write
  -> BusRouter 找到 VirtioBlkPciDevice 的 legacy frontend
  -> frontend 修改 LegacyVirtioBlk 状态
```

设备伪代码：

```rust
impl DeviceModel for VirtioBlkLegacyFrontend {
    fn handle_access(&mut self, access: BusAccess, ctx: &mut DeviceContext) -> AxResult<BusResponse> {
        let offset = access.addr - self.pio_base;

        match access.op {
            BusOp::Read => {
                let value = self.legacy.read_register(offset, access.width);
                Ok(BusResponse::value(value))
            }

            BusOp::Write { value } => {
                match offset {
                    REG_GUEST_FEATURES => self.legacy.driver_features = value as u32,
                    REG_QUEUE_PFN => self.legacy.queue.set_pfn(value as u32),
                    REG_QUEUE_SELECT => self.legacy.queue_select = value as u16,
                    REG_DEVICE_STATUS => self.legacy.set_status(value as u8),
                    REG_QUEUE_NOTIFY => {
                        self.legacy.process_queue(ctx.guest_memory)?;
                        ctx.irq_sink.pulse(self.irq_line);
                    }
                    _ => {}
                }
                Ok(BusResponse::none())
            }
        }
    }
}
```

这和当前 `LegacyVirtioBlk::handle_write()` 的逻辑很接近。区别是：

```text
当前：
  mem 从 axvm::vm.rs 手动构造后传进去。

方向二：
  mem 从 DeviceContext 里拿。

当前：
  axvm::vm.rs 负责判断 0x6000 是 virtio-blk。

方向二：
  BusRouter 根据注册资源判断。
```

### 6.6 OVMF 发起读盘

OVMF 想读 `\EFI\BOOT\BOOTX64.EFI` 时，会：

```text
在 guest memory 写 descriptor table
在 guest memory 写 avail ring
out 0x6010, 0
```

方向二后半段和当前基本一样：

```text
VirtioBlk 收到 QUEUE_NOTIFY
  -> LegacyQueue::pop_available()
  -> LegacyQueue::collect_chain()
  -> VirtioBlkRequest::from_descriptors()
  -> MemoryDiskBackend::read_at()
  -> GuestMemoryAccessor::write_buffer()
  -> LegacyQueue::publish_used()
  -> IrqSink::pulse()
```

这说明当前 `axdevice::virtio_blk` 的内部逻辑有迁移价值。

需要替换的是外面的接线：

```text
从 axvm::vm.rs 特判
替换成
DeviceRegistry + BusRouter + DeviceContext
```

## 7. 当前实现和方向二目标对照

| 问题 | 当前实现 | 方向二目标 |
| --- | --- | --- |
| fw_cfg 在哪里 | `AxVMInnerMut::fw_cfg` | `FwCfgDevice` 注册到 `DeviceRegistry` |
| virtio-blk 在哪里 | `AxVMInnerMut::ovmf_virtio_blk` | `VirtioBlkPciDevice` / `VirtioBlkLegacyFrontend` 注册到 registry |
| VM exit 怎么分发 | `axvm::vm.rs` 里 if/else 特判 | 统一生成 `BusAccess`，交给 `BusRouter` |
| 设备怎么声明地址 | 当前 virtio-blk 固定 `0x6000..0x607f` | 设备注册 `Resource::Pio { base, size }` 或 BAR |
| 设备怎么读 guest memory | `axvm::vm.rs` 手动构造 `AxVmGuestMemory` | `DeviceContext::guest_memory` |
| 设备怎么发中断 | 当前 virtio-blk 基本没有正规 IRQ 路由 | `IrqSink` -> `InterruptRouter` -> vIOAPIC/vLAPIC |
| ACPI 怎么来 | 当前最小 fw_cfg/E820，PC 平台 ACPI 不完整 | `AcpiBuilder` 根据 VM 平台和设备生成 |
| PCI 谁提供 | 当前很大程度借外层 QEMU 的发现外壳 | AxVisor 自己的 `PciHostBridge` / `PciConfigSpace` |
| 新设备接入方式 | 改 `EmulatedDeviceType` / match / VM exit 特判 | 实现 `DeviceModel`，注册资源 |

## 8. 读源码建议顺序

建议按这条线看，不要先看 ACPI 标准。

### 第一步：看 VM exit 当前怎么分发

文件：

```text
tgoskits/components/axvm/src/vm.rs
```

重点找：

```text
AxVCpuExitReason::IoRead
AxVCpuExitReason::IoWrite
handle_fw_cfg_io_read()
handle_fw_cfg_io_write()
handle_ovmf_virtio_blk_io_read()
handle_ovmf_virtio_blk_io_write()
handle_acpi_pm_io_read()
handle_acpi_pm_io_write()
```

读的时候只问一个问题：

```text
一次 in/out 指令最后调用到了谁？
```

### 第二步：看为什么 virtio-blk 不能走普通 AxVmDevices

文件：

```text
tgoskits/components/axdevice_base/src/lib.rs
tgoskits/components/axdevice/src/device.rs
```

重点看：

```text
BaseDeviceOps::handle_read()
BaseDeviceOps::handle_write()
AxVmDevices::handle_port_read()
AxVmDevices::handle_port_write()
```

读的时候只问：

```text
这个接口有没有地方传 guest memory？
```

答案是没有。所以当前需要特化。

### 第三步：看 virtio-blk 状态机

文件：

```text
tgoskits/components/axdevice/src/virtio_blk/legacy.rs
```

重点看：

```text
LegacyVirtioBlk
owns_port()
handle_read()
handle_write()
read_register_byte()
process_queue()
parse_and_execute_chain()
execute_read()
execute_write()
```

读的时候只问：

```text
OVMF 写哪个寄存器，会改变设备哪个字段？
OVMF 写 notify 后，怎么开始读 guest memory？
```

### 第四步：看 virtqueue 是怎么从 guest memory 读出来的

文件：

```text
tgoskits/components/axdevice/src/virtio_blk/queue.rs
```

重点看：

```text
LegacyQueue::set_pfn()
desc_table_gpa()
avail_ring_gpa()
used_ring_gpa()
pop_available()
collect_chain()
publish_used()
```

读的时候只问：

```text
queue_pfn 怎么变成 descriptor table / avail ring / used ring 地址？
```

### 第五步：再看方向二应该怎么替换外壳

读完前四步，再把当前代码映射成方向二：

```text
AxVMInnerMut::fw_cfg
  -> FwCfgDevice

AxVMInnerMut::ovmf_virtio_blk
  -> VirtioBlkPciDevice / VirtioBlkLegacyFrontend

handle_*_io_read/write 特判
  -> BusRouter::route() + DeviceModel::handle_access()

AxVmGuestMemory 手动构造
  -> DeviceContext::guest_memory

未来中断注入
  -> DeviceContext::irq_sink
```

## 9. 最后再压缩成一句话

当前实现是：

```text
AxVisor 在 VM exit 里认出几个 OVMF 需要的 port，
然后手动调用对应的小设备模型。
virtio-blk 的核心读盘逻辑已经在 axdevice 里，
但设备发现、资源注册、DMA 上下文、中断路由还没有正规化。
```

方向二要做的是：

```text
把“VM exit 认识具体设备”改成“设备注册资源，BusRouter 自动分发”；
把“设备临时拿 guest memory”改成“DeviceContext 正式提供 guest memory / IRQ 能力”；
把“ACPI/PCI 发现依赖外部或特化”改成“AxVisor 根据注册好的平台生成 ACPI，并自己提供 PCI 配置空间”。
```

这样方向一和方向二才能交汇：

```text
方向一提供 OVMF 真实需要什么。
方向二提供这些需要应该怎样正规接入 AxVisor。
```
