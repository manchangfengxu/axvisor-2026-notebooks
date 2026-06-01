# AxVisor OVMF 基础设施协作路线图

## 1. 文档目的

这份文档给团队成员认领 OVMF 适配工作使用。它不是实现方案，也不替上游 maintainer 定义合入标准。

每个模块只说明四件事：

1. 这个基础设施在 OVMF/UEFI 客户机里负责什么。
2. 当前 bring-up 为什么碰到它。
3. AxVisor 现在为了最小跑通做了什么。
4. 代码和 develop 记录在哪里。

事实来源以 `axvisor-2026-notebooks/develop/` 为准。

## 2. 当前已验证的最小链路

当前主线已经验证到：

```text
AxVisor
  -> OVMF_CODE / OVMF_VARS
  -> x86 reset vector
  -> OVMF SEC / PEI / DXE / BDS
  -> PCI / virtio-blk
  -> FAT32 ESP
  -> /EFI/BOOT/BOOTX64.EFI
  -> ArceOS UEFI helloworld
```

这说明 OVMF 固件入口、PEI 内存发现、DXE/BDS 驱动加载、virtio-blk 读盘和 PE32+ UEFI app 加载已经能串起来。当前默认目标不是 Linux EFI，也不是完整 PC 平台模型，而是把为这条链路补过的基础设施整理出来，方便后续按模块认领。

## 3. 总览表

| 顺序 | 模块 | OVMF 阶段 | 基础设施角色 | 当前最小适配 | 临时类型 | 主要记录 |
|---|---|---|---|---|---|---|
| 1 | UEFI 配置链路和 OVMF 镜像加载 | 启动前 | 把 TOML 里的 UEFI 字段交给 VM、镜像加载器和 vCPU 初始化 | 在 `[kernel]` 中加入 UEFI 字段，直接把 OVMF_CODE/VARS 加载到固定 GPA | 直接镜像加载，尚无 pflash 语义 | `config-chain.md`, `develop0` |
| 2 | reset vector 和 vCPU 初始状态 | SEC 前 | 让 BSP 从 x86 复位入口进入固件 | VMX/SVM 识别 `0xffff_fff0` 并设置 CS base/RIP | 最小 reset-state 模拟 | `develop0 Step 1-5` |
| 3 | 早期异常诊断和 DEBUG OVMF | SEC/PEI | 让 OVMF 早期故障能定位到具体阶段和源码 | 增强 VMX 异常日志，改用自编 DEBUG OVMF | 诊断增强 | `develop0 Step 1-6`, `develop1 Step 1-2`, `develop1 Step 4` |
| 4 | debugcon 和 string I/O | PEI 日志路径 | 让 OVMF `DEBUG()` 输出能到宿主日志 | 接入 `0x402` debugcon，OVMF debug 输出改为逐字节 I/O | OVMF workaround | `develop1 Step 5-7`, `develop2 Step 5-7` |
| 5 | fw_cfg | PEI | 固件从 VMM 读取平台信息的 QEMU 风格通道 | 在 AxVM 中加入最小 fw_cfg item、selector、data port 和 DMA read | 最小设备模型，部分语义简化 | `develop2 Step 9-10`, `develop4 Step 3` |
| 6 | MTRR / MSR 最小虚拟化 | PEI 内存初始化 | 给 OVMF 一个稳定的内存类型初始状态 | 拦截 `IA32_MTRR_DEF_TYPE`，返回最小 shadow 值 | 最小 MSR shadow | `develop3` |
| 7 | APIC_BASE、x2APIC 和 vLAPIC 状态 | PEI CpuMpPei | 让 OVMF 读写 APIC 模式状态时看到 guest 视图 | 拦截 `IA32_APIC_BASE`，同步 vLAPIC shadow | APIC 模式最小虚拟化 | `develop4 Step 0-2` |
| 8 | CPUID guest 视图 | PEI CpuMpPei | 给固件暴露 guest CPU 数、APIC ID 和能力位 | 修正 leaf 1 的 APIC ID、logical CPU count、x2APIC bit | 最小 CPUID 过滤 | `develop5 Step 1-2` |
| 9 | xAPIC MMIO 和 AP startup | PEI CpuMpPei | 接住 OVMF 的 LAPIC MMIO 访问和单 CPU AP handoff 路径 | 接通 APIC-access page、APIC write exit 和单 vCPU broadcast no-op | vLAPIC 最小路径，含 OVMF workaround | `develop4 Step 4`, `develop5 Step 3` |
| 10 | PCI MMIO window | DXE/BDS | 让 OVMF 连接 PCI 设备和 BAR 时能访问声明的 MMIO 区间 | 在 VM 配置中加入低端和高端 PCI MMIO passthrough | passthrough | `develop4 Step 5-6` |
| 11 | virtio-blk nested DMA | BDS | 让 OVMF 通过 virtio-blk 读取 ESP 文件系统 | 拦截 legacy I/O BAR，把 queue PFN 和 descriptor 地址从 GPA 改成 HPA | guest memory 原地改写 | `develop6 Step 1-2` |
| 12 | ACPI PM I/O | BDS | 让 OVMF 完成 ACPI PM1_CNT 读写 | `0x600..0x60f` 端口临时透传到宿主 | host passthrough | `develop6 Step 3` |
| 13 | UEFI 可启动测试镜像 | BDS 验证 | 给 OVMF 一个含 ESP 和 `BOOTX64.EFI` 的启动盘 | 手工维护 `tmp/uefi-boot-test.img`，用 `--rootfs` 指定 | 手工验证环境 | `develop6 Step 4-5`, `start.md` |
| 14 | ArceOS UEFI payload | UEFI app | 验证 ArceOS 目录能产出 OVMF 可加载的 PE32+ app | 新增 `ax-uefi-helloworld` 示例，尚未接入 runtime | 外壳级 payload | `develop7 Step 1-2` |

## 4. 分模块基础说明

### 1. UEFI 配置链路和 OVMF 镜像加载

#### 基础设施说明

OVMF 客户机不是把一个 kernel 放到低地址后直接跳转。它需要固件代码、变量区、固件窗口地址、复位入口和低端内存布局。AxVisor 需要先能从 VM 配置里接住这些字段，再把它们交给镜像加载器和 vCPU 初始化。

#### 为什么 bring-up 会碰到它

最早的目标是让 BSP 从 `0xffff_fff0` 进入 OVMF。没有这条配置链路，后面的 fw_cfg、APIC、PCI、virtio 都不会出现，因为固件入口本身还不成立。

#### 当前最小适配状态

当前把 UEFI 字段放在 `[kernel]` 段中，复用现有配置结构。`boot = "uefi"` 让镜像加载器走 UEFI 分支，跳过旧 kernel/header 加载。`OVMF_CODE.fd` 和 `OVMF_VARS.fd` 直接加载到固定 GPA，当前没有 pflash 设备语义。

#### 代码和记录定位

- 详细文档: `axvisor-2026-notebooks/issue/mod0/mod0.md`

#### 认领者需要知道的目标

目标是把 UEFI 配置、固件镜像加载和旧 trampoline 路径分清楚。当前能跑，但 OVMF_CODE/VARS 仍只是普通内存镜像加载，不是 pflash 设备。

### 2. reset vector 和 vCPU 初始状态

#### 基础设施说明

x86 CPU 复位后从物理地址 `0xffff_fff0` 取第一条指令。OVMF 的 reset vector 放在固件窗口顶部，随后跳到 SEC 阶段。VMM 要模拟这组入口寄存器，而不是按普通内核入口设置 RIP。

#### 为什么 bring-up 会碰到它

`develop0` 的基线已经确认 BSP 能进入 OVMF，不再停在镜像加载或固件映射。后续异常发生在 OVMF 早期执行过程中，这说明 reset vector 入口基本成立。

#### 当前最小适配状态

VMX 和 SVM 都识别 `0xffff_fff0` 入口，并设置对应的 CS selector、CS base 和 RIP。这个实现只覆盖当前 OVMF bring-up 需要的 reset vector 入口，没有扩展成完整 x86 reset-state 模拟。

#### 代码和记录定位

- 当前实现代码：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- 当前实现代码：`tgoskits/components/x86_vcpu/src/svm/vcpu.rs`
- 记录来源：`develop0 Step 1-5`

#### 认领者需要知道的目标

目标是保证 UEFI 固件入口和旧 kernel entry 不混在一起。后续如果要正规化，需要和 maintainer 确认 AxVisor 是否需要更完整的 reset-state 抽象。

### 3. 早期异常诊断和 DEBUG OVMF

#### 基础设施说明

OVMF 早期故障如果只看到 VM-exit 原始字段，很难判断是 SEC、PEI、IDT、GDT 还是 fw_cfg 问题。bring-up 需要一条能把 VMX 异常、guest 状态和 OVMF DEBUG 日志连起来的诊断路径。

#### 为什么 bring-up 会碰到它

`develop0` 早期看到的是 `#BR` 后投递异常时触发 `#GP`。这些日志证明问题已经越过 reset vector，但还没定位到 PEI 的 fw_cfg 路径。之后改用自编 DEBUG OVMF，才把失败点推进到 `PlatformPei` 和 `QemuFwCfgPei`。

#### 当前最小适配状态

AxVisor 增加了 VMX 异常和 IDT-vectoring 解码日志。OVMF 使用当前工作区自编 DEBUG 版本，并带有 `AXOVMF` 日志点。部分日志增强只是诊断用途，不是长期平台模型。

#### 代码和记录定位

- 当前诊断代码：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- 当前辅助 dump：`tgoskits/os/axvisor/src/vmm/vcpus.rs`
- OVMF 日志点：`edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c`
- 记录来源：`develop0 Step 1-6`, `develop1 Step 1-2`, `develop1 Step 4`

#### 认领者需要知道的目标

目标是保留对 OVMF bring-up 有用的诊断能力，同时区分通用诊断和临时日志。后续清理时不要误删仍用于 smoke 定位的日志路径。

### 4. debugcon 和 string I/O

#### 基础设施说明

OVMF 的 `DEBUG()` 输出可以通过 debugcon I/O port 发给宿主。EDK2 的 I/O helper 可能用 `rep outsb` 或 `rep insb` 做批量端口读写。VMM 如果不处理 string/repeat I/O，固件日志和 fw_cfg probe 都可能直接停住。

#### 为什么 bring-up 会碰到它

`develop1` 先发现 AxVisor 没有把 `0x402` 暴露成 OVMF debugcon。接上后又遇到 `rep outsb`。`develop2` 中 fw_cfg probe 又遇到 `rep insb` 读取 `0x511`。

#### 当前最小适配状态

AxVisor 接入了 `0x402` debugcon 输出。OVMF 侧把 debug 输出和 PEI fw_cfg probe 的 FIFO I/O 改成逐字节 I/O，以避开当前 VMX/SVM 对 string/repeat I/O 的缺口。

#### 代码和记录定位

- 当前 debugcon 设备：`tgoskits/components/axdevice/src/device.rs`
- 当前 I/O 拦截：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- OVMF workaround：`edk2/OvmfPkg/Library/PlatformDebugLibIoPort/DebugLib.c`
- OVMF workaround：`edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c`
- 记录来源：`develop1 Step 5-7`, `develop2 Step 5-7`, `develop2 Step 10`

#### 认领者需要知道的目标

目标是让 AxVisor 自己能处理当前固件会用到的 string/repeat port I/O。清理完成后，应能撤掉 OVMF 侧逐字节 workaround，至少不再依赖修改过的 OVMF 才能跑 smoke。

### 5. fw_cfg

#### 基础设施说明

fw_cfg 是 QEMU 风格的固件配置通道。OVMF 用它读取签名、接口能力、CPU 数量、e820 内存布局和文件目录。PEI 阶段不需要完整 PC 平台，但需要 fw_cfg 提供足够的启动信息。

#### 为什么 bring-up 会碰到它

`develop1` 把失败点定位到 `PlatformPei` 的 `InternalQemuFwCfgDmaBytes()`。`develop2` 说明旧问题已经不是 reset vector，而是 OVMF 认为 fw_cfg DMA 可用后，AxVisor 没有提供对应行为。

#### 当前最小适配状态

AxVM 中实现了最小 `FwCfgState`，支持 selector、data port、DMA address 写入和 DMA read。item 集合只覆盖当前 PEI 需要的内容，例如 signature、interface version、SMP CPU count、file directory 和 `etc/e820`。

#### 代码和记录定位

- 当前实现代码：`tgoskits/components/axvm/src/vm.rs`
- OVMF 诊断和 workaround：`edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c`
- 记录来源：`develop1 Step 7`, `develop2 Step 9-10`, `develop4 Step 3`

#### 认领者需要知道的目标

目标是把现在的最小 fw_cfg 行为整理成可维护的 AxVisor 侧基础设施。当前已足够通过 OVMF PEI cache，但 file directory、DMA 状态、write item 和更多 `etc/` 内容仍是后续沟通点。

### 6. MTRR / MSR 最小虚拟化

#### 基础设施说明

OVMF PEI 内存初始化会检查 MTRR 默认类型。MTRR 决定内存类型和缓存属性。VMM 如果把 host MTRR 原样透给 guest，guest 可能看到和自己平台不匹配的状态。

#### 为什么 bring-up 会碰到它

`develop2 Step 10` 已经跨过 fw_cfg cache，下一停点变成 `MemDetect.c` 对 `IA32_MTRR_DEF_TYPE` 的断言。`develop3` 判断问题不再是 fw_cfg，而是 guest 读到了 host 的 MTRR 状态。

#### 当前最小适配状态

VMX/SVM 拦截 `IA32_MTRR_DEF_TYPE (0x2ff)`，读写都走 vCPU 内部 shadow。初始值只满足当前 OVMF PEI 假设，没有实现 fixed/variable MTRR 子系统。

#### 代码和记录定位

- 当前实现代码：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- 当前实现代码：`tgoskits/components/x86_vcpu/src/svm/vcpu.rs`
- 记录来源：`develop3`, `develop2 Step 10`

#### 认领者需要知道的目标

目标是避免 OVMF 看到 host MTRR 状态。当前只是针对 `IA32_MTRR_DEF_TYPE` 的最小 shadow，后续是否扩展到完整 MTRR，需要根据新的 guest 需求和 maintainer 意见决定。

### 7. APIC_BASE、x2APIC 和 vLAPIC 状态

#### 基础设施说明

OVMF 的 CPU 初始化会读取或写入 APIC 相关 MSR。`IA32_APIC_BASE` 决定 LAPIC base、xAPIC enable、x2APIC enable 和 BSP 标志。VMM 要让这些状态落到 guest 的 vLAPIC，而不是宿主硬件。

#### 为什么 bring-up 会碰到它

`develop3` 跨过 MTRR 后停在 `CpuMpPei` 附近，日志指向 x2APIC ICR MSR。`develop4 Step 0` 判断根因是 CPUID 暴露了 x2APIC，但 `IA32_APIC_BASE` 写没有被 vLAPIC shadow 接住。

#### 当前最小适配状态

VMX/SVM 拦截 `IA32_APIC_BASE` 读写，vLAPIC 初始化为 xAPIC enabled、BSP for vcpu0。x2APIC MSR 小集合只是参考过，未作为完整实现完成。

#### 代码和记录定位

- 当前实现代码：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- 当前实现代码：`tgoskits/components/x86_vcpu/src/svm/vcpu.rs`
- 当前 vLAPIC 代码：`tgoskits/components/x86_vlapic/src/vlapic.rs`
- 当前 vLAPIC 入口：`tgoskits/components/x86_vlapic/src/lib.rs`
- 记录来源：`develop3`, `develop4 Step 0-2`

#### 认领者需要知道的目标

目标是让 APIC 模式变化都停留在 guest 虚拟状态里。当前能支撑 smp1 的 OVMF 路径，但 x2APIC MSR 处理还不是完整基础设施。

### 8. CPUID guest 视图

#### 基础设施说明

OVMF 通过 CPUID 判断 CPU 能力、APIC ID、逻辑 CPU 数和部分拓扑信息。VMM 不能直接把 host CPUID 原样暴露给 guest，否则固件可能看到错误的 CPU 数或 host APIC ID。

#### 为什么 bring-up 会碰到它

`develop5` 纠正了 `develop4` 后半段的推进方向。当前 OVMF 没有真正走到 x2APIC 主线，而是在 xAPIC 模式下等待 CpuMpPei/AP startup。CPUID leaf 1 需要按 guest 视图修正。

#### 当前最小适配状态

VMX/SVM 的 CPUID leaf 1 会设置 guest APIC ID、guest logical CPU count、hypervisor 相关位和 x2APIC capability。topology leaf、cache leaf 和 extended leaf 没有作为本阶段目标展开。

#### 代码和记录定位

- 当前实现代码：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- 当前实现代码：`tgoskits/components/x86_vcpu/src/svm/vcpu.rs`
- 记录来源：`develop5 Step 1-2`, `develop4 Step 0-1`

#### 认领者需要知道的目标

目标是让 guest 看到自洽的 CPU 能力和拓扑。当前只修了 OVMF smp1 路径实际暴露出的 leaf 1 问题，不是完整 CPUID 虚拟化。

### 9. xAPIC MMIO 和 AP startup

#### 基础设施说明

OVMF 的 CpuMpPei 会访问 LAPIC 寄存器，并尝试处理 AP startup 路径。即使当前配置只有一个 vCPU，固件仍会走到 APIC ID、SVR、ICR 和 broadcast 相关逻辑。

#### 为什么 bring-up 会碰到它

`develop5 Step 2` 判断当前可信路径是 xAPIC，不是 x2APIC。`develop5 Step 3` 发现 guest LAPIC 页没有正确接到 VMX/vLAPIC 路径，导致访问 `0xfee000f0` 一类地址时停住。

#### 当前最小适配状态

AxVisor 启用 APIC virtualization，设置 APIC-access page，把 APIC write exit 分发给 vLAPIC。单 vCPU 下的 `AllExcludingSelf` broadcast 做 no-op。EDK2 侧还有针对单 CPU handoff 的 workaround。

#### 代码和记录定位

- 当前 vCPU 实现：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- 当前 vLAPIC 实现：`tgoskits/components/x86_vlapic/src/vlapic.rs`
- 当前 VM 映射相关：`tgoskits/components/axvm/src/vm.rs`
- OVMF workaround：`edk2/UefiCpuPkg/Library/MpInitLib/MpLib.c`
- OVMF 诊断日志：`edk2/UefiCpuPkg/Library/BaseXApicX2ApicLib/BaseXApicX2ApicLib.c`
- 记录来源：`develop4 Step 4`, `develop5 Step 2-3`

#### 认领者需要知道的目标

目标是让 OVMF 的单 CPU 路径不依赖 EDK2 workaround。当前只是让 smp1 能过 CpuMpPei，AP startup、多 vCPU、EOI、timer 和中断投递不是已经完成的部分。

### 10. PCI MMIO window

#### 基础设施说明

OVMF 在 DXE/BDS 阶段会枚举 PCI root bridge，连接 PCI 设备，并访问设备 BAR。VMM 至少要让 OVMF 宣告的 MMIO window 可访问，否则连接设备时会触发 EPT fault。

#### 为什么 bring-up 会碰到它

`develop4 Step 5` 进入 BDS 后，在 `0x81000000` 附近出现 `NestedPageFault`。这个地址位于 OVMF 宣告的低端 PCI MMIO window，不是 fw_cfg、APIC 或 PEI 问题。

#### 当前最小适配状态

VM 配置加入低端 `0x80000000..0xdfffffff` 和高端 `0xe0000000..0xefffffff` PCI MMIO passthrough。这样 OVMF 能继续连接 framebuffer 和 virtio-blk 等外层 QEMU 设备。

#### 代码和记录定位

- 当前 VM 配置：`tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`
- 记录来源：`develop4 Step 5-6`, `develop5 Step 3`

#### 认领者需要知道的目标

目标是让 PCI 设备访问路径从临时 passthrough 走向 AxVisor 自己可维护的设备或配置空间模型。当前只解决 BDS 访问 MMIO window 的最小问题。

### 11. virtio-blk nested DMA

#### 基础设施说明

OVMF BDS 通过 virtio-blk 读取磁盘，找到 FAT32 ESP，再加载 `BOOTX64.EFI`。在当前嵌套场景里，L2 guest 写进 virtqueue 的地址是 L2 GPA，外层 QEMU 设备不能直接用它做 DMA。

#### 为什么 bring-up 会碰到它

`develop6 Step 1` 确认 OVMF 已经写 legacy virtio queue notify。问题不是 OVMF 没发请求，而是 queue notify 后没有看到请求完成。`develop6 Step 2` 判断是 nested DMA 地址空间不匹配。

#### 当前最小适配状态

virtio-blk legacy I/O BAR `0x6000..0x607f` 从 VMX 层转给 AxVM dispatch。AxVM 在 queue PFN 写入时做 GPA 到 HPA 翻译，在 queue notify 前遍历 descriptor chain，把 `desc.addr` 原地改写为 HPA。

#### 代码和记录定位

- 当前 vCPU 转发：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- 当前 AxVM handler：`tgoskits/components/axvm/src/vm.rs`
- 记录来源：`develop6 Step 1-2`

#### 认领者需要知道的目标

目标是让 OVMF 能稳定读盘，同时清楚当前 descriptor 原地改写会污染 guest memory。后续是否做 AxVisor 侧 virtio-blk device model，需要由认领者和 maintainer 沟通。

### 12. ACPI PM I/O

#### 基础设施说明

OVMF BDS 末尾会读写 ACPI PM1_CNT 一类端口，用来检测或打开 ACPI SCI。即使当前不做完整 ACPI，端口缺失也会让设备连接流程之后直接 panic。

#### 为什么 bring-up 会碰到它

`develop6 Step 2` 后 virtio-blk nested DMA 已能继续推进，但新的停点是 `port=0x604` Word read。`develop6 Step 3` 判断这是 QEMU q35 的 PMBA `0x600` 加偏移 `4`，即 ACPI PM1_CNT。

#### 当前最小适配状态

AxVM 对 `0x600..0x60f` 做 host port passthrough。读写都转发给宿主端口。这样 OVMF 能完成 BDS 启动项枚举和后续 boot option 尝试。

#### 代码和记录定位

- 当前实现代码：`tgoskits/components/axvm/src/vm.rs`
- 记录来源：`develop6 Step 3`

#### 认领者需要知道的目标

目标是把当前 host passthrough 的 ACPI PM 行为整理为 AxVisor 可接受的基础设施。当前只是让 OVMF 不在 PM1_CNT 访问处 panic，不是完整 ACPI PM device model。

### 13. UEFI 可启动测试镜像

#### 基础设施说明

OVMF BDS 要从磁盘启动，需要标准 UEFI 启动盘布局：GPT、FAT32 ESP 和 `/EFI/BOOT/BOOTX64.EFI`。普通 ext4 rootfs 不能验证这条路径。

#### 为什么 bring-up 会碰到它

`develop6 Step 3` 已经能走到 boot option，但默认 Alpine/ext4 rootfs 没有 ESP，BDS 报 `Not Found`。必须换成 UEFI 可启动镜像，才能验证 virtio-blk 读盘和 PE32+ 加载。

#### 当前最小适配状态

当前使用 `tmp/uefi-boot-test.img`，镜像里同时放 OVMF 文件和 ESP。运行时通过 `--rootfs` 指定该镜像，避免 ostool 自动换回默认 rootfs。`BOOTX64.EFI` 可以在 EDK2 Shell、最小 UEFI app 和 ArceOS UEFI helloworld 之间切换。

#### 代码和记录定位

- 运行说明：`axvisor-2026-notebooks/start.md`
- 镜像路径：`tmp/uefi-boot-test.img`
- 记录来源：`develop6 Step 4-5`

#### 认领者需要知道的目标

目标是让团队有一个稳定的 smoke 环境。当前镜像是手工维护的验证资产，不是自动化构建产物。

### 14. ArceOS UEFI payload

#### 基础设施说明

OVMF 能加载 UEFI app 后，还要验证 ArceOS 自己能产出 PE32+ payload。这个阶段的重点是入口形态，不是完整 ArceOS runtime。

#### 为什么 bring-up 会碰到它

`develop6 Step 5` 用最小 UEFI app 证明了 OVMF 到 PE32+ app 的链路。`develop7` 将目标推进到 ArceOS 目录内的 UEFI app，要求不破坏旧 multiboot 入口。

#### 当前最小适配状态

新增 `ax-uefi-helloworld` 示例，使用 `x86_64-unknown-uefi` target，导出 `efi_main`，打印 UEFI memory map 摘要，然后停机。它已经能作为 `BOOTX64.EFI` 由 OVMF BDS 加载执行。

#### 代码和记录定位

- 当前示例：`tgoskits/os/arceos/examples/uefi-helloworld/`
- 当前测试 payload：`tgoskits/target/x86_64-unknown-uefi/debug/ax-uefi-helloworld.efi`
- 记录来源：`develop7 Step 1-2`, `develop6 Step 5`

#### 认领者需要知道的目标

目标是把 UEFI entry 接到 ArceOS runtime，同时保留旧 multiboot 路径。当前还没有接入 `axruntime::rust_main(0, boot_context_ptr)`，也没有处理 `ExitBootServices` 和 UEFI memory map 转换。

## 5. 污染代码和 workaround 索引

| 类型 | 文件或区域 | 当前行为 | 关联模块 | 记录来源 |
|---|---|---|---|---|
| OVMF workaround | `edk2/OvmfPkg/Library/PlatformDebugLibIoPort/DebugLib.c` | debugcon 输出从 FIFO I/O 改成逐字节 I/O | debugcon 和 string I/O | `develop1 Step 6-7` |
| OVMF workaround | `edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` | fw_cfg probe 从 FIFO I/O 改成逐字节 I/O，并保留 `AXOVMF` 日志 | debugcon 和 fw_cfg | `develop1 Step 7`, `develop2 Step 10` |
| OVMF workaround | `edk2/UefiCpuPkg/Library/MpInitLib/MpLib.c` | 单 CPU handoff fallback 和 APIC ID 规避 | xAPIC MMIO 和 AP startup | `develop4 Step 4` |
| OVMF 诊断日志 | `edk2/UefiCpuPkg/Library/BaseXApicX2ApicLib/BaseXApicX2ApicLib.c` | IPI 路径增加 `AXOVMF SendIpi` 日志 | xAPIC MMIO 和 AP startup | `develop4 Step 4` |
| AxVisor 诊断增强 | `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs` | VMX 异常、IDT-vectoring、CPUID、APIC 日志 | 早期诊断、CPUID、APIC | `develop0`, `develop5` |
| 直接镜像加载 | `tgoskits/os/axvisor/src/vmm/images/mod.rs` | OVMF_CODE/VARS 直接加载到固定 GPA | UEFI 配置链路 | `config-chain.md` |
| 最小设备模型 | `tgoskits/components/axvm/src/vm.rs` | fw_cfg item 和 DMA 语义只覆盖当前 PEI 需求 | fw_cfg | `develop2 Step 9-10` |
| 最小 MSR shadow | `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`, `tgoskits/components/x86_vcpu/src/svm/vcpu.rs` | 只 shadow `IA32_MTRR_DEF_TYPE` | MTRR / MSR | `develop3` |
| passthrough | `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` | 低端和高端 PCI MMIO 直接 passthrough | PCI MMIO window | `develop4 Step 5-6` |
| guest memory 改写 | `tgoskits/components/axvm/src/vm.rs` | virtio-blk descriptor 地址原地从 GPA 改成 HPA | virtio-blk nested DMA | `develop6 Step 2` |
| host passthrough | `tgoskits/components/axvm/src/vm.rs` | ACPI PM I/O 端口转发到宿主 | ACPI PM I/O | `develop6 Step 3` |
| 手工验证环境 | `tmp/uefi-boot-test.img` | 手工维护 GPT/FAT32 ESP 和 `BOOTX64.EFI` | UEFI 可启动测试镜像 | `develop6 Step 4-5` |

## 6. 写作和维护规则

- 新增或修改条目时，先从 `develop*.md` 找出处，再写结论。
- `issue/ing.md` 只能作为对照，不作为事实来源。
- 每个模块都要能回溯到 `developX Step Y` 或 `config-chain.md`。
- 当前实现和污染代码分开写。不要把能跑通说成已经完成正式基础设施。
- 不在本文写具体实现方案。认领者可以在自己的设计或 PR 中决定。
- 不在本文写上游合入标准。需要和 maintainer 沟通的地方，只描述当前状态和问题边界。
