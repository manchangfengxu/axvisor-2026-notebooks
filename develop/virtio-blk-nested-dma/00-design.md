# virtio-blk nested DMA

## 定位

`virtio-blk` 是设备模型，不是 VM 核心功能。

- 设备模型放在 `tgoskits/components/axdevice/src/virtio_blk/`
- `axvm` 只保留 VM exit 路由和 guest memory accessor 胶水
- 不新增 `components/axvirtio_blk`
- 不保留 `components/axvm/src/virtio_blk/`

## 目标

AxVisor 自己消费 OVMF legacy virtio-blk virtqueue。

- `queue_pfn` 保存 guest PFN
- descriptor `addr` 按 GPA 访问
- 不把 descriptor table 原地改成 HPA
- notify 时通过 `GuestMemoryAccessor` 做 nested DMA

## 设备模型

- `tgoskits/components/axdevice/src/lib.rs`
  - `pub mod virtio_blk`

- `tgoskits/components/axdevice/src/virtio_blk/mod.rs`
  - `pub use legacy::LegacyVirtioBlk`
  - `pub use queue::LegacyQueue`
  - `pub use backend::MemoryDiskBackend`
  - `pub use request::VirtioBlkRequest`

- `tgoskits/components/axdevice/src/virtio_blk/legacy.rs`
  - `LegacyVirtioBlk`
  - `LEGACY_BLK_IO_BASE`
  - `LEGACY_BLK_IO_SIZE`
  - `LegacyVirtioBlk::owns_port`
  - `LegacyVirtioBlk::handle_read`
  - `LegacyVirtioBlk::handle_write`
  - `LegacyVirtioBlk::install_disk_image`
  - `LegacyVirtioBlk::process_queue`
  - `LegacyVirtioBlk::process_chain`

- `tgoskits/components/axdevice/src/virtio_blk/queue.rs`
  - `LegacyQueue::set_pfn`
  - `LegacyQueue::desc_table_gpa`
  - `LegacyQueue::avail_ring_gpa`
  - `LegacyQueue::used_ring_gpa`
  - `LegacyQueue::collect_chain`
  - `LegacyQueue::pop_available`
  - `LegacyQueue::publish_used`

- `tgoskits/components/axdevice/src/virtio_blk/request.rs`
  - `Descriptor`
  - `RequestKind`
  - `VirtioBlkRequest::from_descriptors`

- `tgoskits/components/axdevice/src/virtio_blk/backend.rs`
  - `MemoryDiskBackend`
  - `MemoryDiskBackend::read_sector`
  - `MemoryDiskBackend::write_sector`
  - `BackendStatus`

## VM 接线

- `tgoskits/components/axvm/src/vm.rs`
  - `use axdevice::virtio_blk::LegacyVirtioBlk`
  - `AxVMInnerMut::ovmf_virtio_blk`
  - `AxVmGuestMemory`
  - `impl GuestMemoryAccessor for AxVmGuestMemory`
  - `AxVM::install_virtio_blk_disk_image`
  - `AxVM::handle_ovmf_virtio_blk_io_read`
  - `AxVM::handle_ovmf_virtio_blk_io_write`

说明：现有 `BasePortDeviceOps` 没有传入 `GuestMemoryAccessor`。legacy virtio-blk notify 必须读写 guest memory，所以当前由 `axvm` 把 `AxVmGuestMemory` 传给 `LegacyVirtioBlk`。

## 镜像加载

- `tgoskits/components/axvm/src/config.rs`
  - `DiskImageInfo`
  - `VMImageConfig::disk`
  - `VMImageConfig::try_from`

- `tgoskits/os/axvisor/src/vmm/images/mod.rs`
  - `ImageLoader::load_virtio_blk_disk_from_filesystem`
  - `fs::read_image_file`
  - `AxVM::install_virtio_blk_disk_image`

- `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`
  - `kernel.disk_path = "/guest/disks/uefi-boot-test.img"`

## 数据路径

1. OVMF 写 `queue_pfn` 到 `0x6008`。
2. `LegacyVirtioBlk::handle_write` 调 `LegacyQueue::set_pfn`。
3. OVMF 写 notify 到 `0x6010`。
4. `LegacyVirtioBlk::process_queue` 调 `LegacyQueue::pop_available`。
5. `LegacyQueue::collect_chain` 通过 `GuestMemoryAccessor` 读 descriptor chain。
6. `VirtioBlkRequest::from_descriptors` 解析 header、data descriptor、status descriptor。
7. `MemoryDiskBackend` 读写 raw disk bytes。
8. `LegacyVirtioBlk` 写 status byte。
9. `LegacyQueue::publish_used` 写 used ring。
10. 本阶段不注入中断，OVMF 走 polling。

## 删除项

- `tgoskits/components/axvm/src/virtio_blk/`
- `components/axvirtio_blk`
- `OvmfVirtioBlkIoState`
- `translate_ovmf_virtio_blk_queue_pfn`
- `rewrite_ovmf_virtio_blk_descriptors`
- `dump_ovmf_virtio_blk_queue`

## 验收点

- `rg "axvirtio_blk|components/axvm/src/virtio_blk" tgoskits` 无匹配
- `rg "rewrite desc|rewrite_ovmf|translated_queue_pfn|translate_ovmf_virtio_blk_queue_pfn|OvmfVirtioBlkIoState" tgoskits/components tgoskits/os` 无匹配
- `cargo test -p axdevice virtio_blk -- --nocapture` 通过
- `cargo test -p axvm --lib` 通过
- `cargo run --manifest-path xtask/Cargo.toml -- axvisor build --config os/axvisor/configs/board/qemu-x86_64.toml --vmconfigs os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` 通过
