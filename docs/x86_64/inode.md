# x86_64 官方资料索引

## Intel SDM combined volumes

文件：`docs/x86_64/intel/intel-sdm-combined-volumes.pdf`

### VMX (Vol. 3C)

- **VMX preemption timer**: Vol. 3C, Chapter 24.7 (Pin-Based VM-Execution Controls, "Activate VMX-preemption timer" bit). Counts down by 1 per 2^X TSC changes, X from `IA32_VMX_MISC[4:0]`. VM-exit reason 52 (`PREEMPTION_TIMER`) when reaches zero.
- **External-interrupt exiting**: Vol. 3C, Chapter 24.7, pin-based bit 0. VM-exit reason 1. Used by VMMs to gain control on host IRQs for interrupt injection.
- **HLT exiting**: Vol. 3C, Chapter 24.7, primary processor-based bit 7. VM-exit reason 12. Catches guest HLT instruction.
- **PAUSE-loop exiting**: Vol. 3C, Chapter 25.1.2, secondary processor-based bit 15. VM-exit reason 40. Fires when PAUSE loop count exceeds threshold.
- **MWAIT exiting**: Vol. 3C, Chapter 24.7, primary processor-based bit 10. VM-exit reason 36.
- **Interrupt-window exiting**: Vol. 3C, Chapter 24.7, primary processor-based bit 2. VM-exit reason 7. Fires when guest can accept interrupts (IF=1, no blocking).
- **Posted interrupts**: Vol. 3C, Chapter 29.1. Pin-based bit 1. Delivers virtual interrupts without VM exit via posted-interrupt descriptor. Requires APICv.
- **Virtual interrupt delivery**: Vol. 3C, Chapter 29.4, secondary processor-based bit 31. Enables direct injection via APIC page.
- **VMX exit reasons**: Vol. 3C, Table 24-1 (Basic VM-Exit Information). Key: 1=EXTERNAL_INTERRUPT, 7=INTERRUPT_WINDOW, 12=HLT, 36=MWAIT, 40=PAUSE, 52=PREEMPTION_TIMER.

### x2APIC (Vol. 3A, Chapter 11)

- PDF pages 3421-3427 / SDM Vol. 3A, 11-37..11-43：x2APIC architecture。
  - 11.12.1 Detecting and Enabling x2APIC Mode：`CPUID.01H:ECX[21]` 表示 x2APIC capability；`IA32_APIC_BASE[10]` 开启 x2APIC；`IA32_APIC_BASE[11]` 为 local APIC global enable。
  - Table 11-5：`IA32_APIC_BASE[11:10]` 的 xAPIC/x2APIC/disabled/invalid 组合。
  - 11.12.1.1：x2APIC 模式下用 `RDMSR/WRMSR` 访问 APIC registers。
  - 11.12.1.2 / Table 11-6：x2APIC MSR 地址空间；ICR 在 xAPIC 的 `0x300/0x310` 合并为 x2APIC MSR `0x830`，`0x831` reserved；SELF IPI 为 `0x83f`。
  - 11.12.2 / Table 11-7：xAPIC mode 下 MMIO available、MSR GP；x2APIC mode 下 MSR available、MMIO 表现为 globally disabled。
  - 11.12.5：x2APIC state transitions；reset 后 local APIC 处于 xAPIC mode，即 `EN=1, EXTD=0`。
  - 11.12.7：ACPI 默认应让 OS 以 xAPIC 模式操作，除非 APIC ID 需要 x2APIC。
