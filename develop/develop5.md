## 本阶段行为准则
- 日志多多益善,确定修改是否正确,通过日志代码定位回源码位置.
- 每推进一个对主线有意义的小阶段，就更新本文档。
- 每条记录必须说明：看到什么日志/源码逻辑、判断出现什么问题、根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段（如果进入新的阶段需要写,不进入不写）、为此做了什么改动达到了目标（描述简洁，改动位置定位具体）。
- 不为了"继续推进"而盲目扩展范围；如果下一步需要大改，就明确记录边界。

## 基线
- 本阶段用于纠正 `develop4-msr.md` 后半段的错误推进方向。
- 总目标仍然是让 AxVisor 正确运行标准 OVMF / UEFI 客户机，而不是把 OVMF 改成适配当前 AxVisor 的特殊固件。
- `CLAUDE.md` 的规则仍然生效：除 debug 输出和 fw_cfg 读入绕开当前 VMX string I/O 限制外，不应继续修改 OVMF / EDK2 逻辑代码来让当前 run pass。


## Step 1：按 guest 视图修正 CPUID leaf 1

### 看到的日志
- 参考项目：已拉取 `cloud-hypervisor` 到 `references/cloud-hypervisor`。
- 参考位置：`references/cloud-hypervisor/arch/src/x86_64/mod.rs`。
- cloud-hypervisor 的做法不是原样透传 host CPUID，而是：
  - `CPUID.1:ECX[31] = 1`，设置 hypervisor bit。
  - Intel guest 不暴露 nested VMX：清 `CPUID.1:ECX[5]`。
  - `CPUID.1:EBX[31:24] = guest x2APIC/APIC ID`。
  - `CPUID.1:EBX[23:16] = guest logical CPU count per package`。
  - topology leaf `0xb` / `0x1f` 里也填 guest x2APIC ID。
- 当前 AxVisor 修改后的关键日志：
  - `[VMX] CPUID leaf=0x1: host_ebx=0x800 guest_ebx=0x10800 host_ecx=0xfffab227 guest_ecx=0xfffab207 host_edx=0xfabfbff guest_edx=0xfabfb7f apic_id=0 logical_cpus=1 x2apic_exposed=true`
- 该日志说明：
  - `guest_ebx=0x10800`，即 `EBX[31:24] = 0`，BSP guest APIC ID 为 0。
  - `EBX[23:16] = 1`，当前 `smp1` guest logical CPU count 为 1。
  - `guest_ecx=0xfffab207`，x2APIC capability 继续暴露给 OVMF。
  - `guest_edx=0xfabfb7f`，VMX 侧清 MCE 已从错误的 EAX 修正到 EDX。
- 同一次运行仍能跨过早期 fw_cfg / MTRR / PEI memory 路径：
  - `QemuFwCfgInitCache Pass!!!`
  - `PublishPeiMemory: PhysMemAddressWidth=46 PeiMemoryCap=66088 KB`
  - `TemporaryRamMigration(0x814000, 0x179A000, 0xC000)`
  - `Loading PEIM ... CpuMpPei.efi`
  - `AP Loop Mode is 1`
  - `AP Vector: 16-bit = 9F000/39, ExchangeInfo = 9F040/A5`
- 等待约 15s 未进入 DXE，也没有新的 VMX panic / NestedPageFault 输出；已手动结束 smoke run。
- 本次 smoke 中只观察到 `IA32_APIC_BASE` read intercept：
  - `[VMX] IA32_APIC_BASE read intercepted: value=0xfee00900`
- 尚未观察到 OVMF 写 `IA32_APIC_BASE` 开启 x2APIC 的日志。

### 判断
- 根据 `docs/OVMF-Boot-Overview.md`，当前仍处于 **PEI**，具体仍在 `CpuMpPei.efi` / AP startup 附近。
- CPUID leaf 1 现在不是 host 原样透传，而是最小 guest 视图：
  - 保留 x2APIC capability，让 OVMF 可以按真实硬件语义决定是否写 `IA32_APIC_BASE` 开启 x2APIC。
  - APIC ID 和 logical CPU count 改为 guest 值，避免 OVMF 看到 host APIC ID / host CPU count。
- 当前修复没有让 OVMF 自然跨过 `CpuMpPei`。
- 下一步应继续检查：
  - OVMF 为什么在当前路径只读 `IA32_APIC_BASE`，尚未写入开启 x2APIC。
  - `CpuMpPei` 在单 vCPU 场景下是否仍在等待 broadcast INIT/SIPI 或 AP handoff。
  - xAPIC MMIO ICR / APIC ID / broadcast IPI 的最小语义是否完整。

### 为此做了什么改动
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
  - `VmxVcpu` 保存 `vcpu_id`，用于构造 guest-visible APIC ID。
  - CPUID leaf 1 中保留 x2APIC capability：`res.ecx |= FEATURE_X2APIC`。
  - 清 nested VMX：`res.ecx &= !FEATURE_VMX`。
  - 设置 hypervisor bit：`res.ecx |= FEATURE_HYPERVISOR`。
  - 修正 `CPUID.1:EBX[31:24]` 为 guest APIC ID。
  - 修正 `CPUID.1:EBX[23:16]` 为当前 VM vCPU 数。
  - 修正 VMX 侧 MCE 清位错误：从 `res.eax &= !FEATURE_MCE` 改为 `res.edx &= !FEATURE_MCE`。
  - 扩展一次性 CPUID 日志，输出 host/guest EBX、ECX、EDX、APIC ID、logical CPU count、x2APIC 暴露状态。
- `tgoskits/components/x86_vcpu/src/svm/vcpu.rs`
  - 同步保存 `vcpu_id`。
  - 同步修正 CPUID leaf 1 的 x2APIC capability、guest APIC ID、guest logical CPU count 和日志，保持 VMX/SVM 行为一致。

### 验证
- 已执行：`cargo fmt --all`。
- 结果：通过。
- 已执行：`cargo check -p x86_vcpu`。
- 结果：通过。
- 已执行 OVMF smoke run。
- 结果：AxVisor 日志确认 `apic_id=0 logical_cpus=1 x2apic_exposed=true`；当前仍停在 PEI `CpuMpPei` / AP startup，未进入 DXE。

## Step 2：最新 smoke 复核 —— 当前仍是 xAPIC 模式下的 CpuMpPei/AP startup 等待

### 看到的日志
- 第一次按 `CLAUDE.md` 中的相对路径 smoke 命令运行失败：`--vmconfigs tmp/configs/ovmf-x86_64-qemu-smp1.toml` 被 xtask/config loader 解析到 `tgoskits/tmp/configs/`，实际文件在 `tgoskits/os/axvisor/tmp/configs/`。
- 已修正 `CLAUDE.md`，后续 smoke 命令应使用 `$WORKSPACE/tgoskits/os/axvisor/tmp/configs/...` 绝对路径。
- 使用绝对路径重新运行后，日志仍能跨过 fw_cfg 和 PEI memory 路径：
  - `QemuFwCfgInitCache Pass!!!`
  - `PublishPeiMemory: PhysMemAddressWidth=46 PeiMemoryCap=66088 KB`
  - `TemporaryRamMigration(0x814000, 0x179A000, 0xC000)`
  - `Loading PEIM ... CpuMpPei.efi`
  - `AP Loop Mode is 1`
  - `AP Vector: non-16-bit = 3F3A000/48D`
  - `AP Vector: 16-bit = 9F000/39, ExchangeInfo = 9F040/A5`
- APIC 相关日志：
  - host/AxVisor 自身平台初始化显示 `Using x2APIC`，这是宿主侧 local APIC 模式，不等于 guest OVMF 已进入 x2APIC。
  - guest vLAPIC 初值仍为 `IA32_APIC_BASE shadow=0xfee00900`，即 `EN=1, EXTD=0, BSP=1`。
  - CPUID leaf 1 仍暴露 `x2apic_exposed=true`，但本次只看到大量 `IA32_APIC_BASE read intercepted: value=0xfee00900`。
  - 没看到 `IA32_APIC_BASE write intercepted`，也没看到 x2APIC ICR MSR `0x830` 的读写日志。

### 判断
- 根据 Intel SDM Vol.3A 11.12.1 / Table 11-5，`IA32_APIC_BASE[11]=1, [10]=0` 表示 local APIC 处于 **xAPIC mode**；只有 guest software 写 `IA32_APIC_BASE[10]=1` 后才进入 x2APIC mode。
- 当前 OVMF 在 `CpuMpPei` 附近只读 `IA32_APIC_BASE`，没有写 bit10，因此可信判断是：guest 仍在 xAPIC 路径，而不是 x2APIC 路径。
- 对当前 `smp1`、guest APIC ID 0 的场景，不应为了推进而强行把 OVMF 引向 x2APIC。Intel SDM 11.12.7 也说明 ACPI 默认应让 OS 使用 xAPIC，除非 APIC ID 需要 x2APIC。
- 下一步主线应先补齐/验证 xAPIC 语义：APIC ID、LDR/DFR/SVR、xAPIC MMIO ICR、broadcast INIT/SIPI 在单 vCPU 下的处理，以及是否存在 vLAPIC/APIC-access 没有正确接住 xAPIC MMIO 的问题。
- x2APIC MSR path 继续保留，但暂时不是当前主线；只有看到 OVMF 写 `IA32_APIC_BASE` 开启 EXTD 或访问 `0x830/0x83f` 时，才转入 x2APIC MSR 细节。

### 为此做了什么改动
- `CLAUDE.md`：修正 OVMF smoke run 示例，改为使用绝对配置路径，避免 `tmp/configs/...` 被解析到错误目录。
- `axvisor-2026-notebooks/docs/x86_64/inode.md`：新增 Intel SDM x2APIC 官方页码索引，当前可直接读取 PDF pages 3421-3427 / Vol.3A 11-37..11-43。

## Step 3：接通 guest LAPIC 页，跨过 CpuMpPei 进入 DXE/BDS

### 看到的日志
- 去掉 `Local APIC` passthrough 并启用 VMX APIC virtualization 后，第一次 smoke 立即失败在 VM-entry control 校验：
  - `VM entry with invalid control field(s)`
- 修正 VMX primary control 后，VM-entry 可以进入 guest，但 OVMF 早期 LAPIC 读变成反复 EPT fault：
  - `NestedPageFault { addr: GPA:0xfee000f0, access_flags: READ }`
- `0xfee000f0` 是 xAPIC MMIO 的 LAPIC version register 附近访问；这说明 guest LAPIC 页没有被当前 EPT 映射接住，CPU 还没走到预期的 APIC virtualization 行为。
- 将 guest LAPIC GPA 映射到 VMX APIC-access backing page 后，重新 smoke 已跨过原来的 PEI `CpuMpPei` 停点：
  - `Loading PEIM ... CpuMpPei.efi`
  - `AP Vector: 16-bit = 9F000/39, ExchangeInfo = 9F040/A5`
  - `DXE IPL Entry`
  - `Loading DXE CORE at 0x00003F03000 EntryPoint=0x00003F1A053`
  - `MpInitLibInitialize: ProcessorIndex=0 CpuCount=1`
  - `Loading driver ... BdsDxe.efi`
  - `[Bds] Entry...`
- 本次 smoke 没有再出现 `NestedPageFault { addr: GPA:0xfee... }` 的早期循环。

### 判断
- 根据 `docs/OVMF-Boot-Overview.md`，本次已经从 PEI `CpuMpPei` / AP startup 推进到 **DXE**，并继续进入 **BDS**。
- 当前主线问题不是 OVMF 必须立即进入 x2APIC，而是 AxVisor 去掉 host LAPIC passthrough 后，guest xAPIC MMIO 页没有正确接到 VMX/vLAPIC 路径。
- `CpuMpPei` 卡住的直接原因可定位为：OVMF 在 AP startup / LAPIC 探测路径访问 `0xfee000f0` 等 xAPIC MMIO 时，AxVisor 没有给 guest LAPIC GPA 建立可用的 APIC-access backing 映射，导致反复 EPT fault 而不是继续执行。
- 这次修复是 AxVisor 侧主线有效小步：不改 OVMF 逻辑，标准 OVMF 能跨过 `CpuMpPei`。
- 后续边界：现在只证明 `smp1` OVMF 所需的 LAPIC 页接入足以跨过 PEI；真正多 vCPU 的 INIT/SIPI、active vCPU mask、self IPI、NMI/interrupt injection 仍未完成，不应把本步误记为完整 APIC/SMP 支持。

### 为此做了什么改动
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
  - VMX primary controls 增加 `USE_TPR_SHADOW`，满足 APIC-register virtualization 的控制位依赖。
  - secondary controls 启用 `VIRTUALIZE_APIC` 和 `VIRTUALIZE_APIC_REGISTER`。
  - VMCS 写入 `APIC_ACCESS_ADDR = 0xfee0_0000`，`VIRT_APIC_ADDR = self.vlapic.virtual_apic_page_addr()`。
  - 保留 `APIC_WRITE` exit 分发到 `self.vlapic.handle_xapic_write_offset(offset)`，并把意外 `APIC_ACCESS` exit 打成诊断日志。
- `tgoskits/components/x86_vlapic/src/lib.rs`
  - 新增 `handle_xapic_write_offset()`，从 virtual-APIC page 读出 APIC-write exit 对应寄存器值，再交给 vLAPIC `handle_write()`。
- `tgoskits/components/x86_vlapic/src/vlapic.rs`
  - 对单 vCPU VM 的 `AllExcludingSelf` 目的地做 no-op 日志处理，避免 `smp1` broadcast 唤醒路径落到尚未实现的 active-vCPU mask。
- `tgoskits/components/x86_vcpu/src/lib.rs` / `tgoskits/components/x86_vcpu/src/vmx/mod.rs` / `tgoskits/components/axvm/src/vcpu.rs`
  - 导出 `EmulatedLocalApic`，让 VM 初始化层能取得 VMX APIC-access backing page 的 HPA。
- `tgoskits/components/axvm/src/vm.rs`
  - x86_64 VM setup 中将 guest LAPIC GPA `0xfee0_0000` 映射到 `EmulatedLocalApic::virtual_apic_access_addr()`，权限为 `DEVICE | READ | WRITE | USER`。
- `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`
  - 删除 `Local APIC` passthrough。
- `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml`
  - 同步删除 `Local APIC` passthrough，避免 guest LAPIC MMIO 直通 host LAPIC。

### 验证
- 已执行：`cargo fmt --all`。
- 结果：通过。
- 已执行：`cargo check -p axvm -p x86_vcpu -p x86_vlapic`。
- 结果：通过。
- 已执行 OVMF smoke run。
- 结果：不再卡在 PEI `CpuMpPei`，已进入 DXE 和 BDS；本步目标达成。

