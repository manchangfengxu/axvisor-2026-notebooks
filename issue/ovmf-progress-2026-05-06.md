# OVMF 适配进展：2026 年 5 月底至 6 月

这段时间的工作从 OVMF ArceOS helloworld 稳定跑通之后开始，目标是把为跑通 smoke 而堆在 `axvm/src/vm.rs` 里的临时逻辑拆干净，同时往 Linux guest 推进。

---

## 1. virtio-blk：从 vm.rs 特殊路径到 axdevice 设备模型

smoke 跑通时，virtio-blk 的全部逻辑（queue PFN 翻译、descriptor 遍历、GPA→HPA 改写、ISR 状态、请求解析）都堆在 `axvm/src/vm.rs` 里，和 VM 上下文、guest memory、端口分发完全混在一起。这次把它拆成了两层：

- `axdevice/src/virtio_blk.rs`：设备核心。`LegacyVirtioBlk` 持有 queue 状态、descriptor 解析、request 解析、block backend 接口和 ISR 状态。通过 `GuestMemoryAccessor` 读写 guest memory，不依赖 AxVM 的内部锁。
- `axvm/src/vm.rs`：替换型适配胶水。`AxVMInnerMut::ovmf_virtio_blk` 从 `OvmfVirtioBlkIoState`（只存 queue_pfn/queue_size 的裸结构体）换成了 `Mutex<LegacyVirtioBlk>`。I/O exit 仍然从 vm.rs 分发，但 queue 解析、request 处理和 ISR 管理都进了设备核心。

同步做的清理：
- `LegacyVirtioBlk::handle_write()` 返回 notify result / interrupt-needed 信号，不再是 void
- legacy ISR status 在 `REG_ISR_STATUS = 0x13` 上实现，读 ISR 自动清
- descriptor chain 通过 `GuestMemoryAccessor` 读写，不再做 GPA→HPA 原地改写

这部分涉及约 10 个文件、1000+ 行新增代码。详细记录在 `develop/virtio-blk-nested-dma/` 下的 03 到 09 号文档。

---

## 2. 上游变基 + 基础设施对齐

`uefi-develop` 和上游 `dev` 做了一次变基。上游在这段时间补了不少东西：

- `GuestMemoryAccessor` trait（`axaddrspace`）：read_obj/write_obj/read_buffer/write_buffer，正好替代旧的 descriptor GPA→HPA 原地改写
- `EmulatedDeviceType::{VirtioBlk, VirtioNet, VirtioConsole}` 枚举预留
- x86 vIOAPIC/PIT/serial 的基础实现
- UEFI 配置链路的更多字段（`boot_protocol`、`uefi_firmware_path`、`ovmf_code_path` 等）

变基后的主要工作是把本地实现对接到上游已有的基础设施上，而不是重复造轮子。virtio-blk 的 descriptor 读写从手动 HPA 翻译改成了走 `GuestMemoryAccessor`；设备类型枚举直接复用上游的 `EmulatedDeviceType::VirtioBlk`；fw_cfg 的配置加载对齐了上游的 `VMKernelConfig` 字段。

---

## 3. fw_cfg：从 vm.rs 提取到 axdevice

fw_cfg 原来也是 `vm.rs` 里的特殊路径状态机。这次把它搬到了 `axdevice/src/fw_cfg.rs`：

- `FwCfgDevice` 实现 `BaseDeviceOps<PortRange>`，是标准的 AxVisor 端口设备
- `FwCfgInner`、`FwCfgItem`、`FwCfgContent`、`FwCfgFileDirectory` 拥有 selector/data/DMA/file-directory 行为
- `AxVmDevices::x86_fw_cfg_execute_pending_dma()` 是 VM 侧的 DMA wrapper
- `AxVM` 只提供 `AxVmGuestMemory` 和窄 I/O glue

同时做了 fw_cfg 内容模型的整理（`develop9-fwcfg-content-model.md`），把之前硬编码的 item 列表理清了结构。

---

## 4. ACPI 转发

嵌套 OVMF 需要 ACPI 表才能让 Linux 正确发现设备。AxVisor 自己不生成 ACPI 表，而是从外层 QEMU 读取现成的 fw_cfg blob 转发给嵌套 guest：

- `os/axvisor/src/x86_fw_cfg.rs`：读外层 QEMU 的 `etc/acpi/tables`、`etc/acpi/rsdp`、`etc/table-loader`
- `os/axvisor/src/config.rs`：在 `x86_64 + UEFI + OVMF` 路径上把这三个 blob 注册到嵌套 fw_cfg
- 失败策略：缺少任何一个 blob 就直接报 VM 创建失败，不做部分转发

记录在 `develop/uefi-acpi-passthrough/` 下的三份文档。

---

## 5. Linux guest 启动尝试

OVMF helloworld 稳定之后，开始往 Linux guest 推进。这条路比预想的深：

### 5.1 EFI stub 和早期 #PF

Linux EFI stub 的 decompression 和早期 #PF 都已经越过。这两个阶段的诊断记录在 `develop/linux-uefi-pf-diagnostics/`。

### 5.2 PIT reset

Linux 启动时依赖 PIT channel 0 在 reset 后立即以 mode 3 运行。AxVisor 的 PIT 实现原来没有这个语义，补了 `PitState::new(now_ns)` 让 channel 0 在 reset 后自动进入 mode 3、divisor 0x10000。单测和 smoke 都验证了 `x86 PIT IRQ0 due` 日志出现。

### 5.3 APIC LVT0 路由

Linux `check_timer()` 会写 `APIC_LVT0`（xAPIC MMIO 写 `0xfee00350`）来配置 ExtINT/Fixed 路由。AxVisor 开了 APICv（virtual-APIC access + APIC-register virtualization），这些写入被 CPU 直接虚拟化到了 virtual-APIC page，不产生 VM-exit。

诊断发现 virtual-APIC page 上的 LVT0 确实从 0x0 变成了 0x700（ExtINT），但 AxVisor 的软件 shadow（`lvt_last`）没有跟上。修了 `lint0_route_from_lvt()` 的 tock-registers `.mask()` 误用，让 ExtINT decode 正确。

### 5.4 PIC 端口 intercept

VMX 的 I/O bitmap 从 `passthrough_all()` 出发，PIC 端口（0x20/0x21/0xa0/0xa1/0x4d0/0x4d1）原来没有加入 intercept 集。Linux 对 8259 的 ICW/OCW 写根本不会形成 VM-exit。补了 `configure_default_io_intercepts()`，PIC 端口写入现在能到达 `x86_vlapic::pic`。

### 5.5 PIC ISR stuck

intack() 在注入路径里无条件 set PIC ISR，但 guest 只发 LAPIC EOI 不发 PIC EOI。对比 QEMU/KVM 参考实现后发现，真实硬件 ExtINT 路径中 APIC 透明处理 INTA cycle，PIC ISR 根本不会被 set。加了 `read_irq_vector_extint()` 只清 IRR 不动 ISR，匹配真实硬件行为。

### 5.6 当前状态

ExtINT 修复后前 2 个 tick 成功注入，guest handler 跑了、EOI 发了、ISR 不再 stuck。但第 3 个 tick 没有到达 guest。`timer_irq_works()` 需要 5 个 jiffies，只拿到 2 个，kernel panic。

怀疑方向是 `inject_pending_events()` 的传统 VMCS injection 和 APICv virtual-interrupt delivery 之间的交互问题。这部分还在调查中。

详细记录在 `develop/linux-uefi-timer-apic/` 下的四份文档和 `subagent_watch/` 下的八份审计文档。

---

## 涉及的主要文件

| 区域 | 文件 | 说明 |
|---|---|---|
| virtio-blk 设备核心 | `axdevice/src/virtio_blk.rs`（新增） | LegacyVirtioBlk、LegacyQueue、request 解析 |
| fw_cfg 设备模型 | `axdevice/src/fw_cfg.rs` | 从 vm.rs 提取 |
| PIC 8259 设备模型 | `x86_vlapic/src/pic.rs`（新增） | master/slave PIC、OCW2 EOI、read_irq_vector_extint |
| PIT 语义 | `x86_vlapic/src/pit.rs` | reset 后 mode 3 自动运行 |
| LINT0 路由 | `x86_vlapic/src/vlapic.rs`、`lib.rs` | Lint0Observation、ExtINT decode |
| VM 适配层 | `axvm/src/vm.rs` | virtio-blk/fw_cfg 适配、LINT0 注入路径 |
| VMX vCPU | `x86_vcpu/src/vmx/vcpu.rs` | PIC 端口 intercept、APICv 诊断 |
| ACPI 转发 | `os/axvisor/src/x86_fw_cfg.rs` | 外层 QEMU ACPI blob 读取 |
