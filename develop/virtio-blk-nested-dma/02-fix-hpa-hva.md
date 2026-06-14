# fix: AxVmGuestMemory HPA/HVA conversion

## 背景

当前 virtio-blk 设备模型已经从早期“改写 descriptor GPA->HPA，再交给外层 QEMU DMA”的临时做法，改成 AxVisor 自己消费 virtqueue：

```text
OVMF virtio-blk driver
  -> legacy I/O BAR 0x6000..0x607f
  -> AxVM I/O exit
  -> axdevice::virtio_blk::LegacyVirtioBlk
  -> GuestMemoryAccessor 读写 guest virtqueue/data buffer
  -> MemoryDiskBackend
```

这条路径需要 AxVisor 自己安全地把 guest GPA 转成 host 可访问地址。

## 问题现象

使用 `--rootfs "$WORKSPACE/tmp/uefi-boot-test.img"` 启动，并在 AxVisor rootfs 内放入 `/guest/disks/uefi-boot-test.img` 后，OVMF 已能看到 virtio-blk 容量：

```text
[OVMF-VIRTIO-BLK-IO] in port=0x6014 width=Dword value=0x10000
[OVMF-VIRTIO-BLK-IO] in port=0x6018 width=Dword value=0x0
VirtioBlkInit: LbaSize=0x200[B] NumBlocks=0x10000[Lba]
```

随后 OVMF 开始读盘时，AxVisor host 自己触发 page fault：

```text
Page fault at VA:0x71b6002 with flags READ
Unhandled #PF ... fault_vaddr=VA:0x71b6002
```

## 根因

`LegacyVirtioBlk` 通过 `GuestMemoryAccessor` 读写 virtqueue 和 data buffer。

问题点在 `tgoskits/components/axvm/src/vm.rs`：

```text
AxVmGuestMemory::translate_and_get_limit()
```

`AddrSpace::translate_and_get_limit(GPA)` 返回的是 HPA。但 `GuestMemoryAccessor` 的默认 `read_buffer()` / `write_buffer()` 会把 `translate_and_get_limit()` 返回值当作 host 可解引用指针使用。

也就是说，旧实现等价于把 HPA 当 HVA 解引用，所以 host 在读取 `0x71b6002` 这类物理地址数值时 #PF。

旧的 `read_guest_bytes_locked()` 没有这个问题，因为它走 `AddrSpace::translated_byte_buffer()`，该路径内部会做 `PagingHandlerImpl::phys_to_virt()`。

## 修改定位

### 1. AxVM guest memory accessor

文件：

```text
tgoskits/components/axvm/src/vm.rs
```

涉及符号：

```text
use ax_page_table_multiarch::PagingHandler
struct AxVmGuestMemory<'a>
impl GuestMemoryAccessor for AxVmGuestMemory<'_>
AxVmGuestMemory::translate_and_get_limit()
PagingHandlerImpl::phys_to_virt()
```

修改描述：

- 先调用 `address_space.translate_and_get_limit(guest_addr)` 得到 HPA 和连续范围 `limit`。
- 再调用 `PagingHandlerImpl::phys_to_virt(host_paddr)` 把 HPA 转成 AxVisor direct-map HVA。
- 最后把 HVA 数值临时包装进 `HostPhysAddr` 返回给 `GuestMemoryAccessor` 默认实现。

当前代码形态：

```rust
let (host_paddr, limit) = self.address_space.translate_and_get_limit(guest_addr)?;
let host_vaddr = PagingHandlerImpl::phys_to_virt(host_paddr);
Some((HostPhysAddr::from_usize(host_vaddr.as_usize()), limit))
```

说明：

`GuestMemoryAccessor` 的 trait 命名仍然偏向 HPA，但默认 buffer 读写逻辑实际需要可解引用地址。这里返回 HVA 数值是当前实验性接线的胶水修复，长期更好的方向是把 trait 接口拆清楚：翻译到 HPA和翻译到 host pointer 不应共用同一个返回类型。

### 2. virtio-blk write request used length

文件：

```text
tgoskits/components/axdevice/src/virtio_blk/legacy.rs
```

涉及符号：

```text
LegacyVirtioBlk::execute_write()
used_len
RequestKind::Write
```

修改描述：

- `VIRTIO_BLK_T_OUT` 写请求的数据 descriptor 是 device-readable，设备从 guest buffer 读数据写到 backend。
- 对这个请求，设备写回 guest 的内容只有 status byte。
- 因此 `execute_write()` 返回的 `used_len` 固定为 `1`，不能把写入 backend 的 data buffer 长度计入 used ring。

这和 `execute_read()` 不同：读请求的数据 descriptor 是 device-writable，设备会把磁盘数据写入 guest buffer，所以 `execute_read()` 返回 `data_len + 1`。

### 3. 嵌套磁盘加载路径

文件：

```text
tgoskits/components/axvm/src/config.rs
tgoskits/os/axvisor/src/vmm/images/mod.rs
tgoskits/components/axvm/src/vm.rs
tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml
```

涉及符号：

```text
kernel.disk_path
VMImageLoader::load_uefi_images()
VMImageLoader::load_virtio_blk_disk_from_filesystem()
AxVM::install_virtio_blk_disk_image()
LegacyVirtioBlk::install_disk_image()
MemoryDiskBackend::new()
```

修改描述：

- `disk_path = "/guest/disks/uefi-boot-test.img"` 表示 AxVisor rootfs 内的嵌套 raw disk 文件。
- `VMImageLoader::load_virtio_blk_disk_from_filesystem()` 用 `fs::read_image_file(disk_path)` 把该文件读入内存。
- `AxVM::install_virtio_blk_disk_image()` 把 `Vec<u8>` 安装进 `LegacyVirtioBlk`。
- `MemoryDiskBackend` 以这个 `Vec<u8>` 作为 OVMF 可见 virtio-blk 后端。

注意：

`--rootfs "$WORKSPACE/tmp/uefi-boot-test.img"` 是外层 QEMU 提供给 AxVisor 的 rootfs；OVMF 看到的 virtio-blk 不是这个 rootfs 本身，而是 rootfs 内的 `/guest/disks/uefi-boot-test.img`。

## 验证

代码验证：

```bash
cargo fmt --check --package axdevice --package axvm --package axvisor
cargo test -p axdevice virtio_blk -- --nocapture
cargo test -p axvm --lib
cargo run --manifest-path xtask/Cargo.toml -- axvisor build --config os/axvisor/configs/board/qemu-x86_64.toml --vmconfigs os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml
```

结果：

```text
fmt: pass
axdevice virtio_blk tests: 15 passed
axvm --lib tests: 3 passed
xtask axvisor build: pass
```

smoke 命令：

```bash
timeout 240 cargo xtask qemu \
  --config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml" \
  --qemu-config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml" \
  --vmconfigs "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml" \
  --rootfs "$WORKSPACE/tmp/uefi-boot-test.img" \
  2>&1 | tee "$WORKSPACE/tmp/ovmf-owned-virtio-blk-smoke.log"
```

关键日志：

```text
Loading virtio-blk disk image from /guest/disks/uefi-boot-test.img (33554432 bytes)
[OVMF-VIRTIO-BLK-IO] in port=0x6014 width=Dword value=0x10000
[OVMF-VIRTIO-BLK-IO] in port=0x6018 width=Dword value=0x0
VirtioBlkInit: LbaSize=0x200[B] NumBlocks=0x10000[Lba]
[OVMF-VIRTIO-BLK] notify head=0 used_idx=...
FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success
ArceOS UEFI helloworld
ArceOS UEFI shell-stage boot OK
```

日志检查：

```text
未出现 panic / Unhandled #PF / FAIL PATTERN
```

## 总结

这次问题不是 OVMF virtio-blk 协议流程本身失败，而是 AxVM 胶水层把 HPA 当成 HVA 交给 `GuestMemoryAccessor` 默认实现解引用，导致 host #PF。

修复后，AxVisor-owned virtio-blk 的路径已经能完成：

```text
OVMF virtio-blk init
  -> 识别 32MiB capacity
  -> 读取 GPT/FAT
  -> 打开 \EFI\BOOT\BOOTX64.EFI
  -> 启动 ArceOS UEFI hello
```

当前实现的标准度定位：

- `axdevice::virtio_blk` 内部的 legacy queue/request/backend 逻辑已经是相对标准的设备模型拆分。
- `axvm::vm` 中的 `handle_ovmf_virtio_blk_io_*` 仍是 OVMF 适配阶段的专用接线，因为现有 `BasePortDeviceOps` 没有 guest memory accessor 参数。
- `MemoryDiskBackend` 是实验性内存后端，guest 写盘只更新内存中的 `Vec<u8>`，不会回写 rootfs 文件。
