# 06 INTx / IOAPIC 运行证据

## 目标

确认当前 nested OVMF + legacy virtio-blk 路径下，guest 是否真的会编程 emulated IOAPIC。

这一步只做证据收集，不做 INTx 接线实现。

---

## 本次改动

1. `tgoskits/virtualization/x86_vlapic/src/vioapic.rs`
   - `EmulatedIoApic::write_selected_register()`
   - 新增 `info!` 日志：打印 `gsi / selector / value / entry / vector / masked / level / delivery`

2. `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml`
   - `emu_devices`
   - 加入 `["X86 IO APIC", 0xfec0_0000, 0x1000, 0, 0x23, []]`

3. `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml`
   - `passthrough_devices`
   - 去掉 `["IO APIC", 0xfec0_0000, ...]`

---

## 验证命令

来自 `axvisor-2026-notebooks/start.md`：

```bash
export WORKSPACE=~/work/axvisor-uefi
cd "$WORKSPACE/tgoskits/os/axvisor"
timeout 45 cargo xtask qemu \
  --config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml" \
  --qemu-config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml" \
  --vmconfigs "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml" \
  --rootfs "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"
```

本次为避免终端截断，把完整输出落盘到 `/tmp/ovmf-ioapic-smoke.log` 后再检索。

---

## 运行结果

### 1. emulated IOAPIC 确实启用了

日志出现：

```text
x86 IO APIC initialized with base GPA 0xfec00000 and length 0x1000
```

说明 `AxVmDevices::init()` 的 `EmulatedDeviceType::X86IoApic` 路径已经生效。

### 2. virtio-blk 完成路径持续触发“应发中断”

日志多次出现：

```text
[OVMF-VIRTIO-BLK-IO] out port=0x6010 ... used_any=true used_count=1 irq=true
```

说明 `AxVM::handle_ovmf_virtio_blk_io_write()` 已经走到 `LegacyNotifyResult.should_raise_irq = true`。

### 3. 没有任何 emulated IOAPIC MMIO 访问

完整日志里没有出现任何：

```text
vIOAPIC IOREGSEL write ...
vIOAPIC IOWIN read ...
vIOAPIC IOWIN write ...
vIOAPIC redirection entry ...
```

这意味着 guest 在这次运行里根本没有碰 `EmulatedIoApic::handle_read()` / `handle_write()` 的关键路径，连 selector / window 都没访问到。

结论不是“GSI 选错了”，而是**nested OVMF 根本没有访问 emulated IOAPIC，更谈不上 unmask 任何 redirection entry**。

### 4. OVMF 仍然在用 outer QEMU 的 PCI 设备路径

日志出现：

```text
PciRoot(0x0)/Pci(0x1,0x0)
FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success
```

外层运行参数 `tmp/configs/qemu-x86_64-runtime.toml` 里有：

```text
-machine q35
-device virtio-blk-pci,drive=disk0
```

同时 `x86_vcpu` 的 `setup_io_bitmap()` 只拦：

- `0x402`
- `0x510..0x517`
- `0x6000..0x607f`

没有拦 `0xcf8/0xcfc`。因此 nested OVMF 仍然能直接枚举 outer QEMU 的 PCI config space。

---

## 结论

1. 当前不能继续做 INTx 接线实现。
2. 阻塞点已经不是“AxVisor 没有 vIOAPIC 基础设施”，而是“guest 完全没有访问 vIOAPIC”。
3. `mptable.rs` 的 GSI 18 不能作为当前 nested OVMF emulated IOAPIC 场景的接线依据。
4. 下一步如果继续查，应回头确认 OVMF 当前拿到的 APIC/IRQ 平台信息来源，而不是继续在 GSI 常量上试错。

---

## 文件 / 符号索引

- `tgoskits/virtualization/x86_vlapic/src/vioapic.rs`
  - `EmulatedIoApic::write_selected_register()`
- `tgoskits/virtualization/axvm/src/vm.rs`
  - `AxVM::handle_ovmf_virtio_blk_io_write()`
- `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs`
  - `SvmVcpu::setup_io_bitmap()`
- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
  - `VmxVcpu::setup_io_bitmap()`
- `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml`
  - `emu_devices`
  - `passthrough_devices`
- `tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml`
  - `args`
