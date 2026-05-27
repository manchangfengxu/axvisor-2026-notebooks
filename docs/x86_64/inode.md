# x86_64 官方资料索引

## Intel SDM combined volumes

文件：`docs/x86_64/intel/intel-sdm-combined-volumes.pdf`

- PDF pages 3421-3427 / SDM Vol. 3A, 11-37..11-43：x2APIC architecture。
  - 11.12.1 Detecting and Enabling x2APIC Mode：`CPUID.01H:ECX[21]` 表示 x2APIC capability；`IA32_APIC_BASE[10]` 开启 x2APIC；`IA32_APIC_BASE[11]` 为 local APIC global enable。
  - Table 11-5：`IA32_APIC_BASE[11:10]` 的 xAPIC/x2APIC/disabled/invalid 组合。
  - 11.12.1.1：x2APIC 模式下用 `RDMSR/WRMSR` 访问 APIC registers。
  - 11.12.1.2 / Table 11-6：x2APIC MSR 地址空间；ICR 在 xAPIC 的 `0x300/0x310` 合并为 x2APIC MSR `0x830`，`0x831` reserved；SELF IPI 为 `0x83f`。
  - 11.12.2 / Table 11-7：xAPIC mode 下 MMIO available、MSR GP；x2APIC mode 下 MSR available、MMIO 表现为 globally disabled。
  - 11.12.5：x2APIC state transitions；reset 后 local APIC 处于 xAPIC mode，即 `EN=1, EXTD=0`。
  - 11.12.7：ACPI 默认应让 OS 以 xAPIC 模式操作，除非 APIC ID 需要 x2APIC。
