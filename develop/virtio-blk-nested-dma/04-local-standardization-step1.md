# virtio-blk 局部标准化 step 1

本文记录 2026-06-14 在 `uefi-develop` 上继续整理 virtio-blk 设备核心的第一步。范围只包括 legacy virtio-blk 的 queue notify 结果和 ISR status，不接完整 INTx。

## 修改点

代码：

- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - 新增 `REG_ISR_STATUS = 0x13`
  - 新增 `VIRTIO_ISR_QUEUE`
  - `LegacyVirtioBlk::isr_status`
  - `LegacyVirtioBlk::handle_write()` 返回 `AxResult<bool>`
  - `LegacyVirtioBlk::process_queue()` 返回 `AxResult<bool>`
  - `LegacyVirtioBlk::handle_read()` 读取 ISR status 后清零
- `tgoskits/virtualization/axvm/src/vm.rs`
  - `AxVM::handle_ovmf_virtio_blk_io_write()` 接住 `LegacyVirtioBlk::handle_write()` 的完成状态

测试：

- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - `notify_without_available_request_returns_false_and_leaves_isr_clear`
  - `completed_request_sets_queue_isr_until_guest_reads_it`

## 当前语义

`REG_QUEUE_NOTIFY` 触发后：

- 如果没有可用 descriptor，`LegacyVirtioBlk::handle_write()` 返回 `false`，ISR 保持 0。
- 如果至少发布了一个 used buffer，`LegacyVirtioBlk::handle_write()` 返回 `true`，并置位 ISR queue bit。
- guest 读 `REG_ISR_STATUS` 会拿到当前 ISR 值，并把 ISR 清零。

这一步让设备核心具备标准 legacy virtio 的中断状态语义。后续接 INTx 时，VM 侧 adapter 可以根据 `handle_write()` 的返回值调用 `AxVmDevices::x86_ioapic_assert_gsi()`。

## 还没做

暂时没有接 INTx 注入。原因是当前 OVMF VM 配置里仍是：

- `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`
  - `emu_devices = []`

补 INTx 前需要先让 OVMF VM 使用 `EmulatedDeviceType::X86IoApic`，否则直接从 virtio-blk adapter 注入会变成半条路径。

## 验证

已运行：

```bash
cargo test -p axdevice virtio_blk -- --nocapture
```

结果：

```text
running 5 tests
test virtio_blk::tests::completed_request_sets_queue_isr_until_guest_reads_it ... ok
test virtio_blk::tests::read_request_copies_sector_into_guest_buffer_and_publishes_used_ring ... ok
test virtio_blk::tests::notify_without_available_request_returns_false_and_leaves_isr_clear ... ok
test virtio_blk::tests::reports_capacity_from_installed_disk_image ... ok
test virtio_blk::tests::write_request_updates_backend_and_reports_only_status_byte_used ... ok

test result: ok. 5 passed; 0 failed
```
