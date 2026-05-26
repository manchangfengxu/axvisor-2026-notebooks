## 本阶段行为准则
- 每推进一个有意义的小阶段，就更新本文档。
- 每条记录必须说明：看到什么日志、判断出现什么问题、根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段（如果进入新的阶段需要写）、为此做了什么改动（描述简洁，改动位置定位具体）。
- 不为了“继续推进”而盲目扩展范围；如果下一步需要大改，就明确记录边界。

## 基线
- 当前失败点是 OVMF 在 `PlatformPei.dll` / `MemDetect.c(1344)` 处断言：`(MtrrSettings.MtrrDefType & 0x00000400) == 0`。
- 这对应 `IA32_MTRR_DEF_TYPE` 的 `FE` 位，也就是 Fixed MTRRs Enable。
- 现场已经跨过 `QemuFwCfgInitCache`，所以当前问题不再是 fw_cfg，而是 PEI 早期 MTRR 初始状态。

## MTRR 透传判断
- 在 `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs` 里，`setup_msr_bitmap()` 目前只显式拦截了 `IA32_UMWAIT_CONTROL (0xe1)` 和 x2APIC MSR 范围 `0x800..=0x83f`，没有看到 `0x2ff`。
- `tgoskits/components/x86_vcpu/src/vmx/structs.rs` 里 `MsrBitmap::passthrough_all()` 是零填充 bitmap，只有显式设置过的 MSR 才会触发 VM-exit。
- 因此 `IA32_MTRR_DEF_TYPE (0x2ff)` 目前更像是 **直接透传给 host 物理 CPU**，guest 读到的就是当前宿主机的真实 MTRR 状态。
- 这和 OVMF 的早期假设不一致：`SecMtrrSetup()` 只会把 default type 设成 WB 并打开 `E` 位，但不会替 OVMF 保证 `FE` 位一定为 0。

## 现有 MSR 路径
- `x86_vcpu` 返回的 `AxVCpuExitReason::SysRegRead/SysRegWrite` 最终由 `axvm` 里的 `handle_sys_reg_read()/handle_sys_reg_write()` 分发。
- 但这条链路只会在 MSR 已经被 bitmap 拦截并触发 VM-exit 时才走到；`0x2ff` 目前还没进入这条链路。
- 当前代码里没有看到任何 `IA32_MTRR_DEF_TYPE` / MTRR 专用的虚拟化实现，所以它不是“被拦截但默认返回 0”，而是更早地直接透传了。

## 最小处理方案
- 先不要扩完整 MTRR 虚拟化；当前只需要给 OVMF 一个稳定的 `IA32_MTRR_DEF_TYPE` 初值。
- 最小做法是在 `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs` 把 `0x2ff` 加入 MSR bitmap 拦截范围，然后在现有 MSR 处理链路里返回一个固定的虚拟值：`Type=WB, E=1, FE=0`，语义上等价于 `0x0000_0000_0000_0806`。
- 写入可以先做最小 shadow，或者先忽略除这几个低位之外的写入；只要下一次 `rdmsr` 不再暴露 host 的 `FE` 位即可。
- 边界先记录在这里：如果后续 OVMF 继续读取 fixed/variable MTRR 再决定是否扩展，不在这一阶段直接补完整 MTRR 子系统。

## 实际改动
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
  - 新增 `IA32_MTRR_DEF_TYPE = 0x2ff`、`GUEST_MTRR_DEF_TYPE_INIT = 0x806`。
  - 在 `VmxVcpu` 中新增 `mtrr_def_type` shadow，初值为 `Type=WB, E=1, FE=0`。
  - 在 `setup_msr_bitmap()` 中对 `0x2ff` 设置 read/write intercept。
  - 在 `builtin_vmexit_handler()` 中优先处理 `IA32_MTRR_DEF_TYPE` 的 `MSR_READ/MSR_WRITE`，读写都在 x86_vcpu 内部完成，避免走通用 `SysRegRead` 后不能正确写回 `EDX:EAX`。
- `tgoskits/components/x86_vcpu/src/svm/vcpu.rs`
  - 同步新增 `mtrr_def_type` shadow 和 `0x2ff` read/write intercept。
  - 在 SVM 内建 MSR exit 路径中处理 `IA32_MTRR_DEF_TYPE`，保持 VMX/SVM 行为一致。

## 验证
- 已执行：`cargo check --manifest-path /home/ssdns/work/axvisor-uefi/tgoskits/Cargo.toml -p x86_vcpu -p axvm`。
- 结果：`x86_vcpu` 和 `axvm` 编译通过。
- 已重新启动 OVMF smoke test。
- 结果：已经跨过 `MemDetect.c(1344)` 的 `FE` 位断言，日志继续推进到 `PublishPeiMemory()`、temporary RAM migration 和后续 PEIM 加载。
- 关键日志包括：
  - `QemuFwCfgInitCache Pass!!!`
  - `PlatformDynamicMmioWindow: using dynamic mmio window`
  - `PublishPeiMemory: PhysMemAddressWidth=46 PeiMemoryCap=66088 KB`
  - `TemporaryRamMigration(0x814000, 0x179A000, 0xC000)`
  - `Loading PEIM ... PeiCore.efi`
  - `Loading PEIM ... CpuMpPei.efi`

## 新停点
- 新的失败点不是 MTRR，而是在 `CpuMpPei.efi` 运行后访问 x2APIC ICR MSR：`rcx=0x830`。
- AxVisor 日志显示：`[VLAPIC] Illegal read attempt of ICR register at width Qword with X2APIC disabled`。
- 随后 VMX 内建 MSR exit handler panic：`VmxVcpu failed to handle a VM-exit that should be handled by itself: MSR_WRITE`。
- 判断：MTRR 最小 shadow 已经有效，当前推进到了 CPU MP / AP 启动相关路径；下一步应检查 AxVisor 的 x2APIC 暴露与 vLAPIC ICR 处理，而不是继续改 MTRR。

## 当前阶段判断
- OVMF 已经从 SEC 进入 PEI，并且已经通过 fw_cfg 路径完成平台内存发现。
- 现在卡在 `PlatformPei` 的 MTRR 初始状态检查，不再是日志、debugcon 或 fw_cfg 设备模型问题。
- 下一步应当优先让 guest 看到一个符合 OVMF 早期假设的 `IA32_MTRR_DEF_TYPE`，再观察是否继续暴露其他 MTRR 相关读取。

## 计划中的最小改动位置
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`：将 `0x2ff` 纳入拦截范围。
- `tgoskits/components/axvm/src/vm.rs`：在 `SysRegRead/SysRegWrite` 分发里补一个 MTRR 最小映射，至少保证 OVMF 读到的 `IA32_MTRR_DEF_TYPE` 不携带 host 的 `FE` 位。
- 如果后续日志继续显示新的 MTRR 读写，再决定是否补完整 MTRR shadow；这一阶段先不扩展到完整虚拟 MTRR。
