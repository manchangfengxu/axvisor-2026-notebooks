## 本阶段行为准则
- 每推进一个有意义的小阶段，就更新本文档。
- 每条记录必须说明：看到什么日志、判断出现什么问题、根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段（如果进入新的阶段需要写,不进入不写）、为此做了什么改动（描述简洁，改动位置定位具体）。
- 不为了"继续推进"而盲目扩展范围；如果下一步需要大改，就明确记录边界。

## 基线
- develop3.md 的最后停点：`CpuMpPei.efi` 访问 x2APIC ICR MSR `rcx=0x830`，vLAPIC 报 `Illegal read attempt of ICR register at width Qword with X2APIC disabled`，随后 VMX panic。
- 根因已经确认：CPUID leaf 1 ECX bit 21 透传，guest 看到 x2APIC 可用；OVMF 写 `IA32_APIC_BASE (0x1b)` 启用 x2APIC，但该 MSR 没有被拦截，写到了真实硬件，vLAPIC 内部的 `apic_base` 字段始终是 0；`is_x2apic_enabled()` 永远返回 false，所以 ICR 的 Qword 访问被拒绝。

## Step 0：根因分析 —— CPUID x2APIC bit 透传 + APIC_BASE 未拦截

### CPUID 侧
- 位置：`x86_vcpu/src/vmx/vcpu.rs:1073-1081` `handle_cpuid()` leaf 1 (LEAF_FEATURE_INFO)。
- `handle_cpuid()` 对 leaf 1 只 mask 了 VMX(bit5)、MCE(bit7)，set 了 HYPERVISOR(bit31)。
- **ECX bit 21 (x2APIC) 无任何屏蔽**，直通 host 值 → guest 认为 x2APIC 可用。
- 判断：不应该屏蔽这个 bit。OVMF 确实需要 x2APIC（PEI 阶段 CpuMpPei 使用 x2APIC 做 CPU 间通信），问题出在 APIC_BASE 没有正确虚拟化。

### MSR bitmap 侧
- 位置：`x86_vcpu/src/vmx/vcpu.rs:445` `setup_msr_bitmap()`。
- `IA32_APIC_BASE (0x1b)` 的拦截被**注释掉了**，guest 写 `0x1b` 直接落到 host 物理 MSR。
- x2APIC MSR 范围 `0x800..=0x83f` **已经拦截**，会触发 VM-exit 走 `handle_apic_msr_access()` 进入 vLAPIC。
- 但 vLAPIC 的 `apic_base` 字段初始化为 0，没有 setter，`is_x2apic_enabled()` 永远 false → x2APIC 的 Qword 访问被拒绝。
- SVM 侧同样没有拦截 `0x1b`。

### 结论
- 两条链路各自缺失：CPUID 侧告知 guest x2APIC 可用，但 APIC_BASE 侧没有拦截导致 guest 的 x2APIC 使能写不到 vLAPIC shadow。
- 修复方向：不屏蔽 CPUID bit（保持 x2APIC 通路），拦截 IA32_APIC_BASE 并正确更新 vLAPIC 状态。

## Step 1：拦截 IA32_APIC_BASE，修正 vLAPIC 初始状态

### 做了什么
- Step 0 结论落地：拦截 IA32_APIC_BASE（VMX/SVM 两侧），同步 vLAPIC shadow 状态。
- 同时修复 vLAPIC 初始状态：reset 时 `apic_base` 应为 `0xFEE00900`（xAPIC enabled, BSP for vcpu0），而非 0。

### 改动

`tgoskits/components/x86_vlapic/src/vlapic.rs`
- `VirtualApicRegs::new()` 把 `apic_base` 初始值从 0 改为正确的 reset 值：`base=0xFEE00000, EN=1, BSP=1(vcpu0)`，这样 `is_xapic_enabled()` 在 guest 启动时就返回 true。
- 新增 `set_apic_base(value: u64)` 方法。

`tgoskits/components/x86_vlapic/src/lib.rs`
- `EmulatedLocalApic` 新增 `apic_base()` getter 和 `set_apic_base()` 公开接口。

`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- `setup_msr_bitmap()` 取消注释，正式拦截 `IA32_APIC_BASE (0x1b)`。
- `builtin_vmexit_handler()` 新增 `0x1b` 分支，路由到新的 `handle_apic_base_msr()`。
- 新增 `handle_apic_base_msr()`：读返回 vLAPIC shadow，写更新 vLAPIC shadow。

`tgoskits/components/x86_vcpu/src/svm/vcpu.rs`
- `setup_msr_bitmap()` 同步拦截 `0x1b`。
- `builtin_vmexit_handler()` 新增 `0x1b` 分支。
- 新增 `handle_apic_base_msr()`，与 VMX 侧行为一致。

### 验证
- `cargo check -p x86_vlapic -p x86_vcpu` 通过。
- 已重新跑 OVMF smoke test；fw_cfg init 已通过，当前不再被 fw_cfg DMA 回归阻塞。

## Step 2：参考开源项目，规划 x2APIC MSR 小集合映射（计划阶段，未实施）

### 动机
- Step 1 只解决了 IA32_APIC_BASE 的拦截和 vLAPIC 模式切换。
- 一旦 OVMF 跨过 ICR 访问，还会遇到其他 x2APIC MSR（如 TPR、EOI、SelfIPI、APIC ID 等），当前 vLAPIC 对这些 MSR 的处理可能不完整或有 `unimplemented!()`。
- 需要从成熟开源 hypervisor 拉取参考，适配一个最小集合。

### 参考项目
- 拉取了 crosvm 到 `/tmp/crosvm-ref/`：
  - `devices/src/irqchip/apic.rs` — crosvm 的 APIC 模拟实现，包含 x2APIC MSR 的读写处理模式。
- 参考目标：提取 OVMF PEI 阶段实际访问到的 x2APIC MSR 子集（ICR `0x830`、TPR `0x808`、EOI `0x80B`、LDR `0x80D`、SELF_IPI `0x83F` 等），比对 crosvm/cloud-hypervisor 的处理方式。

### 当前状态
- 仅完成参考代码拉取，尚未做任何 x2APIC MSR handler 实现。
- fw_cfg init 已重新通过；当前停在 `CpuMpPei.efi` / AP startup 附近，尚未确认是否进入新的 x2APIC MSR handler 缺口。

## Step 3：fw_cfg DMA 状态复核（已通过，非当前阻塞项）

### 最终结论
- 使用当前正确的 OVMF_CODE：`$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd`，hash 为 `4c8bf9119a9b725c6fc3c00a8a63efcf1cd31ad4b2d54c329b49b9fc91371bbf`。
- `QemuFwCfgInitCache` 已通过，fw_cfg DMA 死循环不再复现。
- A/B 复查：在正确 OVMF_CODE 下，临时恢复 `handle_fw_cfg_dma()` 中的 verify-readback 和 `info!()` DMA 日志后，仍能看到 `QemuFwCfgInitCache Pass!!!`，并继续推进到 `CpuMpPei.efi`。
- 因此目前不能把死循环归因于 verify-readback 或高频日志；更可靠的结论是：当前干净代码和带临时 DMA 验证代码都能通过 fw_cfg init。
- 曾误用 `$WORKSPACE/edk2/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd` 旧产物时，会退回到 `VMX unsupported IO-Exit ... port: 0x511`。该旧产物早于 `QemuFwCfgPei.c` 的逐字节 `IoRead8()` 改动，不应再作为当前复现镜像。
- `start.md` 已修正：默认 OVMF 产物来源统一为 `$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/`，避免混用旧产物。

### 关键日志
- `OVMF debugcon: QemuFwCfgProbe: Supported 1, DMA 1`
- `OVMF debugcon: AXOVMF InternalQemuFwCfgDmaBytes: ...`
- `OVMF debugcon: Cache FwCfg Name: QemuFwCfgSignature Item:0x0 Size: 0x4`
- `OVMF debugcon: Cache FwCfg Name: QemuFwCfgInterfaceVersion Item:0x1 Size: 0x4`
- `OVMF debugcon: Cache FwCfg Name: QemuFwCfgFileDri Item:0x19 Size: 0x44`
- `OVMF debugcon: Cache FwCfg Name: etc/e820 Item:0x8000 Size: 0x28`
- `OVMF debugcon: Cache FwCfg Name: BootMenu Item:0xE Size: 0x2`
- `OVMF debugcon: QemuFwCfgInitCache Pass!!!`

### 当前停点
- 已经跨过 `QemuFwCfgInitCache`、CMOS dump、动态 MMIO window、PEI memory publish、temporary RAM migration。
- 已进入后续 PEIM 加载：
  - `OVMF debugcon: Loading PEIM ... PeiCore.efi`
  - `OVMF debugcon: Loading PEIM ... DxeIpl.efi`
  - `OVMF debugcon: Loading PEIM ... CpuMpPei.efi`
  - `OVMF debugcon: AP Loop Mode is 1`
  - `OVMF debugcon: AP Vector: 16-bit = 9F000/39, ExchangeInfo = 9F040/A5`
- 之后 90s timeout 内无新输出，也没有复现 Step 3 的 fw_cfg DMA 死循环。

### 当前阶段判断
- 根据 `docs/OVMF-Boot-Overview.md`，当前仍在 **PEI**，更具体是 `CpuMpPei.efi` / AP startup 附近。
- fw_cfg DMA 不是当前阻塞项；Step 1 的 IA32_APIC_BASE 修复已经能支撑 OVMF 推进到 `CpuMpPei` 后，但还未证明 AP startup 已完成。
- 下一步应围绕 `CpuMpPei` 的等待点继续加日志或检查 x2APIC/vLAPIC IPI/AP startup 路径，而不是继续排查 `QemuFwCfgInitCache`。
