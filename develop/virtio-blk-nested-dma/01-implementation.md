# implementation

## 本轮架构

`virtio-blk` 作为设备模型进入 `axdevice`。`axvm` 不再拥有设备实现，只负责把 x86_64 I/O exit 分发到设备，并提供 `AxVmGuestMemory`。

## tgoskits 修改

- `components/axdevice/src/lib.rs`
  - `pub mod virtio_blk`

- `components/axdevice/src/virtio_blk/mod.rs`
  - `LegacyVirtioBlk`
  - `MemoryDiskBackend`
  - `LegacyQueue`
  - `VirtioBlkRequest`

- `components/axdevice/src/virtio_blk/backend.rs`
  - `MemoryDiskBackend`
  - `MemoryDiskBackend::new`
  - `MemoryDiskBackend::capacity_sectors`
  - `MemoryDiskBackend::read_sector`
  - `MemoryDiskBackend::write_sector`
  - `BackendStatus`

- `components/axdevice/src/virtio_blk/request.rs`
  - `Descriptor`
  - `RequestKind`
  - `VirtioBlkRequest`
  - `VirtioBlkRequest::from_descriptors`

- `components/axdevice/src/virtio_blk/queue.rs`
  - `LegacyQueue`
  - `LegacyQueue::set_pfn`
  - `LegacyQueue::read_descriptor`
  - `LegacyQueue::collect_chain`
  - `LegacyQueue::pop_available`
  - `LegacyQueue::publish_used`

- `components/axdevice/src/virtio_blk/legacy.rs`
  - `LegacyVirtioBlk`
  - `LegacyVirtioBlk::new`
  - `LegacyVirtioBlk::install_disk_image`
  - `LegacyVirtioBlk::owns_port`
  - `LegacyVirtioBlk::handle_read`
  - `LegacyVirtioBlk::handle_write`
  - `LegacyVirtioBlk::process_queue`
  - `LegacyVirtioBlk::parse_and_execute_chain`

- `components/axdevice/src/virtio_blk/test_mem.rs`
  - `TestGuestMemory`

- `components/axvm/src/vm.rs`
  - `use axdevice::virtio_blk::LegacyVirtioBlk`
  - `AxVMInnerMut::ovmf_virtio_blk`
  - `AxVmGuestMemory`
  - `impl GuestMemoryAccessor for AxVmGuestMemory`
  - `AxVM::install_virtio_blk_disk_image`
  - `AxVM::handle_ovmf_virtio_blk_io_read`
  - `AxVM::handle_ovmf_virtio_blk_io_write`

- `components/axvm/src/config.rs`
  - `DiskImageInfo`
  - `VMImageConfig::disk`
  - `VMImageConfig::try_from`

- `os/axvisor/src/vmm/images/mod.rs`
  - `ImageLoader::load_virtio_blk_disk_from_filesystem`
  - `fs::read_image_file`
  - `AxVM::install_virtio_blk_disk_image`

- `os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`
  - `kernel.disk_path`

## 删除和拒绝的路径

- 不新增 `components/axvirtio_blk`
- 不使用 `components/axvm/src/virtio_blk`
- 不恢复 `OvmfVirtioBlkIoState`
- 不恢复 `translate_ovmf_virtio_blk_queue_pfn`
- 不恢复 `rewrite_ovmf_virtio_blk_descriptors`
- 不恢复 `dump_ovmf_virtio_blk_queue`

## 当前接口限制

- `BasePortDeviceOps::handle_read`
- `BasePortDeviceOps::handle_write`

这两个接口没有 `GuestMemoryAccessor` 参数。当前不强行改 `axdevice_base` 公共 trait，只在 `axvm/src/vm.rs` 通过 `AxVmGuestMemory` 把 guest memory 传给 `LegacyVirtioBlk`。

## 验证记录

- `rg "axvirtio_blk|axvirtio-blk|components/axvirtio_blk|axvm::virtio_blk" Cargo.toml Cargo.lock components os`
  - 无匹配

- `rg "rewrite desc|rewrite_ovmf|translated_queue_pfn|translate_ovmf_virtio_blk_queue_pfn|OvmfVirtioBlkIoState" components/axvm/src os/axvisor/src components/axdevice/src`
  - 无匹配

- `cargo fmt --check --package axdevice --package axvm --package axvisor`
  - passed

- `cargo test -p axdevice virtio_blk -- --nocapture`
  - 15 tests passed

- `cargo test -p axvm --lib`
  - 3 tests passed

- `cargo run --manifest-path xtask/Cargo.toml -- axvisor build --config os/axvisor/configs/board/qemu-x86_64.toml --vmconfigs os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`
  - release build passed

## smoke 限制

- 当前 `$WORKSPACE/tmp/uefi-boot-test.img` 没有 `/guest/disks/uefi-boot-test.img`
- 当前镜像约 60 MB 空闲，放不下 64 MB nested disk
- rootfs smoke 需要先换更大的 UEFI rootfs 或减小 nested disk
