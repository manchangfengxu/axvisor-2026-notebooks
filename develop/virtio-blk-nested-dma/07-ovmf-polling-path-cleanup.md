# 07 OVMF 轮询路径收口

## Scope

本轮不再沿着 legacy virtio-blk INTx 接线继续推进。

根因已经在 `subagent_watch/ovmf-vioapic-root-cause.md` 闭合：

- OVMF DXE 的 virtio-blk 路径是同步轮询
- `LegacyNotifyResult.should_raise_irq` 在当前 OVMF 启动链路里只是设备核心的协议级提示
- nested OVMF 启动期间不访问 emulated IOAPIC

因此这里只做代码语义收口和临时取证日志回退。

## Modified Symbols

- `tgoskits/virtualization/axvm/src/vm.rs`
  - `AxVM::handle_ovmf_virtio_blk_io_write`
    - 日志字段从 `irq={}` 改为 `should_raise_irq={}`，避免把设备核心的 notify 结果误读成“guest 需要 INTx”。
    - `if notify.should_raise_irq` 分支里的 `trace!` 文案改成：
      - 当前 OVMF 路径通过 polling `Used.Idx` 完成请求
      - AxVisor 在这里不做 INTx 注入
    - 这次没有改 `notify.should_raise_irq` 的计算语义，只改了 AxVM adapter 对它的解释。

- `tgoskits/virtualization/x86_vlapic/src/vioapic.rs`
  - `EmulatedIoApic::write_selected_register`
    - 删除本轮取证期间临时加入的 `info!` redirection-entry 日志。
  - `EmulatedIoApic::handle_read`
    - 删除 `IOWIN read` 的临时 `info!` 日志。
  - `EmulatedIoApic::handle_write`
    - 删除 `IOREGSEL write` 和 `IOWIN write` 的临时 `info!` 日志。

## Behavior Summary

1. `LegacyNotifyResult.should_raise_irq` 仍然保留在 `axdevice::virtio_blk` 设备核心里。
2. 但在当前 nested OVMF boot 路径里，它只表示：
   - 本次 queue notify 至少发布了一个 used buffer
   - 设备核心按 legacy virtio 语义认为“可以触发 IRQ”
3. AxVM adapter 现在明确写清楚：
   - 当前 OVMF DXE 驱动是 polling path
   - 因此这里不把 `should_raise_irq` 继续向下翻译成 INTx 注入动作
4. `vioapic.rs` 恢复到无取证噪音的状态；相关证据已经沉淀在：
   - `06-intx-ioapic-evidence.md`
   - `subagent_watch/ovmf-vioapic-root-cause.md`

## Not Changed

- 没有改 `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - `LegacyNotifyResult`
  - `LegacyVirtioBlk::process_queue`
  - `LegacyVirtioBlk::handle_write`
- 没有引入 INTx 注入
- 没有新增 GSI 常量
- 没有改 OVMF VM 配置
- 没有开始 ACPI / MP table / IRQ routing 平台实现

## Verification

- `cargo fmt --package axvm --package x86_vlapic`
- `cargo test -p axvm --lib`
- `cargo test -p x86_vlapic --lib`

## Next

下一步不该继续写 `virtio-blk` INTx。

更合理的下一步是先查清并决定：**nested guest 的 IRQ / ACPI 平台信息由谁提供。**

最小落点有三个：

1. `tgoskits/virtualization/axdevice/src/fw_cfg.rs`
   - 查当前 fw_cfg file directory 里是否完全没有 ACPI 相关表项
   - 目标变量/结构：fw_cfg file directory / file entries 构建路径

2. `tgoskits/os/axvisor/src/images/mod.rs`
   - 查 UEFI 路径是否有意完全不提供 MP table / IRQ 平台信息
   - 目标函数：`load_uefi_ovmf_images()` / UEFI filesystem loading path

3. `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs`
   - `SvmVcpu::setup_io_bitmap()`
   - `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
   - `VmxVcpu::setup_io_bitmap()`
   - 确认 PCI config / ECAM 当前继续 passthrough 是不是我们下一阶段还接受的前提

这一步做完，才能决定后面是：

- 继续保持“OVMF 轮询 + outer QEMU PCI passthrough”作为当前边界
- 还是开始补一条最小 ACPI / IRQ 平台信息链
