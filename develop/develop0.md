# OVMF 早期启动适配记录

## 当前基线
- `OVMF_CODE` / `OVMF_VARS` 已经能从 AxVisor guest rootfs 加载。
- BSP vCPU 已经能从 x86 reset vector 进入 OVMF，不再停在最初的 firmware 加载、映射或 reset-vector 入口问题。
- 当前失败点已经进入 OVMF 早期执行阶段，表现为 VMX `#GP` / IDT vectoring / descriptor 相关异常，而不是镜像缺失或 OVMF_CODE base 错误。
- 本阶段工作边界：只做小步、可解释、可回滚的诊断增强或单点修正；如果需要大规模平台能力、跨层 API 重构、复杂设备模型或 sudo/系统权限，就停止。

## 本阶段行为准则
- 每推进一个有意义的小阶段，就更新本文档。
- 每条记录必须说明：做了什么、看到什么日志、判断处于哪个阶段、下一步最小动作是什么。
- 不为了“继续推进”而盲目扩展范围；如果下一步需要大改，就明确记录边界。

## Step 1：选择最小诊断增强点

### 目标
先不改 OVMF 行为，只增强 AxVisor 的 VMX 异常日志，让当前 `#GP` 不再只是原始字段。

### 检查到的现状
- VMX 异常日志位置在：
  - `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
- 相关函数：
  - `dump_exception_exit_info()`
  - `dump_triple_fault_exit_info()`
  - `dump_guest_descriptor_tables()`
  - `dump_guest_segments()`
- 现有日志已经能输出：
  - `GDTR/IDTR`
  - `CS/SS/DS/ES/FS/GS/TR`
  - `CR0/CR3/CR4/EFER`
  - `VMEXIT_INTERRUPTION_INFO`
  - `IDT_VECTORING_INFO`
- 但当时还不能直接看出：
  - `#GP err_code=0x11` 到底对应哪个 selector
  - `idt_info=0x80000305` 到底是什么 vectoring 事件

### 修改内容
修改文件：
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`

新增/增强内容：
- 增加 `dump_interruption_error_code(prefix, vector, err_code)`：
  - 解码 `#TS/#NP/#SS/#GP` 这类 selector-style error code。
  - 解码 `#PF` 的 page fault error code。
- 增加 `dump_idt_vectoring_info(prefix, raw_idt_info, idt_err)`：
  - 解码 VMX `IDT_VECTORING_INFO`。
  - 复用 error-code 解码函数。
- 在 `dump_exception_exit_info()` 中调用上述新日志。
- 在 `dump_triple_fault_exit_info()` 中也加入 IDT-vectoring 解码。

### 编译验证
运行：

```bash
cargo check -p x86_vcpu --target x86_64-unknown-none
```

结果：通过。

### 阶段结论
这是纯诊断增强，没有改变 guest 行为。下一步需要重新运行 OVMF smoke test，看新日志能否把当前 `#GP` 定位到具体 descriptor / vectoring 事件。

## Step 2：重新运行 OVMF smoke test，确认 `#GP` 含义

### 运行命令
在 `tgoskits/os/axvisor` 下运行：

```bash
sg kvm -c 'timeout 240 cargo xtask qemu \
  --config /home/ssdns/work/axvisor-uefi/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml \
  --qemu-config /home/ssdns/work/axvisor-uefi/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml \
  --vmconfigs /home/ssdns/work/axvisor-uefi/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml'
```

### 关键结果
构建成功，QEMU/AxVisor/OVMF VM 均正常进入启动流程。`timeout` 最终返回 `124`，这是外层超时结束，不是编译失败。

关键日志：

```text
Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000
Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000
VMX guest entry setup: entry=0xfffffff0, reset_vector=true, cs=0xf000, cs_base=0xffff0000, rip=0xfff0
```

当前异常：

```text
VMX exception/NMI VM-Exit:
  exit_reason: EXCEPTION_NMI
  guest_rip: 0x82997a
  vector: 0xd
  err_code: 0x11
  vector_name=#GP general protection
```

新增解码日志：

```text
VMX exception exception error code decode:
  raw=0x11
  selector=0x10
  index=0x2
  table=GDT
  external=true
```

IDT-vectoring 原始字段：

```text
idt_info=0x80000305
idt_err=0x0
```

当时日志已能看出 vector 是 `0x5`，但 type 还只是数字。

### 阶段分析
- `#GP err_code=0x11` 不再是模糊错误码，已经可以解释为：
  - GDT selector `0x10`
  - GDT index `2`
- 对照 EDK2/OVMF reset-vector GDT：
  - 文件：`edk2/UefiCpuPkg/ResetVector/Vtf0/Ia16/Real16ToFlat32.nasm.inc`
  - selector `0x10` 对应 `LINEAR_CODE_SEL`
- 因此当前不是 OVMF_CODE 加载问题，而是 OVMF 早期异常投递时，CPU/VMX 在使用 GDT selector `0x10` 的路径上触发了 `#GP`。

### 阶段结论
当前失败已经定位到：OVMF 早期 SEC/异常处理路径中，正在处理或投递一个 IDT vectoring 事件时，围绕 GDT selector `0x10` 触发 `#GP`。

## Step 3：进一步解码 IDT-vectoring type

### 目标
`idt_info=0x80000305` 里不只有 vector，还有 VMX interruption type。只打印数字不够直观，所以继续做一个小的日志增强。

### 修改内容
修改文件：
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`

修改函数：
- `dump_idt_vectoring_info()`

新增 interruption type 名称解码：

```text
0 -> external interrupt
2 -> NMI
3 -> hardware exception
4 -> software interrupt
5 -> privileged software exception
6 -> software exception
7 -> other event
```

### 编译验证
运行：

```bash
cargo check -p x86_vcpu --target x86_64-unknown-none
```

结果：通过。

### 重新运行结果
再次运行 OVMF smoke test 后，关键日志变为：

```text
VMX exception IDT-vectoring decode:
  vector=0x5 #BR bound range exceeded
  type=0x3 (hardware exception)
  has_error_code=false
  err=0x0
```

同时仍然触发：

```text
#GP general protection
err_code=0x11
selector=0x10
table=GDT
```

### 阶段分析
这说明当前不是软件 `int 5` 一类路径，而是 guest 正在投递硬件异常 `#BR bound range exceeded` 时，又触发了 `#GP`。

也就是说：

```text
guest 中先发生 #BR
  -> CPU/VMX 尝试通过 IDT 投递 #BR
  -> 投递过程中需要使用 IDT gate 里的 code selector 0x10
  -> 使用 GDT selector 0x10 时触发 #GP
```

当前 guest 状态仍然显示：

```text
GDTR base=0xfffffed0, limit=0x3f
IDTR base=0x81fd70, limit=0x21f
CS=0x38
SS/DS/ES/FS/GS=0x18
TR=0x0
CR0=0x80000023
CR3=0x800000
CR4=0x2660
EFER=0x500
```

### 对照 OVMF 阶段
对照 `axvisor-2026-notebooks/docs/OVMF-Boot-Overview.md` 和 EDK2 源码：
- 已经越过 reset-vector 入口。
- 已经进入 long mode / early SEC 附近。
- 当前更接近 OVMF SEC 中 IDT 初始化 / exception handler 初始化 / early exception delivery 路径。

## Step 4：检查是否能继续做 guest memory dump

### 目标
下一步最有价值的信息是读取：

```text
GDTR.base + 0x10 = 0xfffffed0 + 0x10 = 0xfffffee0
```

也就是当前 `#GP` 指向的 GDT selector `0x10` 的实际 descriptor 内容。

### 检查到的代码位置
已有 guest memory 读取能力：
- `tgoskits/components/axvm/src/vm.rs`
- 函数：`read_from_guest_of<T>()`

当前 VMX 异常日志位置：
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`

问题：
- `VmxVcpu` 异常路径里没有直接持有 `AxVM` 或 address-space 句柄。
- 要在 `dump_exception_exit_info()` 里直接读 guest memory，需要把 VM/address-space 访问能力跨层传进 `x86_vcpu`。
- 这不是一两行日志增强，会涉及新的 API 或跨层结构调整。

### 阶段结论
这一步已经到达当前工作边界。继续做 GDT/IDT 实际内存 dump 是合理方向，但需要更改 AxVisor VM/vCPU 层接口，属于比当前“小步适配”更大的改动。

## Step 5：补充 EDK2/OVMF 源码定位

### 查看 OVMF IDT 模板
文件：
- `edk2/OvmfPkg/Sec/SecMain.c`

相关代码：

```c
// Template of an IDT entry pointing to 10:FFFFFFE4h.
IA32_IDT_GATE_DESCRIPTOR  mIdtEntryTemplate = {
  {
    0xffe4,                              // OffsetLow
    0x10,                                // Selector
    0x0,                                 // Reserved_0
    IA32_IDT_GATE_TYPE_INTERRUPT_32,     // GateType
    0xffff                               // OffsetHigh
  }
};
```

这与当前 `#GP err_code=0x11 -> selector=0x10` 高度相关。

### 查看 OVMF reset-vector GDT
文件：
- `edk2/UefiCpuPkg/ResetVector/Vtf0/Ia16/Real16ToFlat32.nasm.inc`

相关 selector：

```asm
LINEAR_CODE_SEL     equ $-GDT_BASE    ; Selector [0x10]
LINEAR_SEL          equ $-GDT_BASE    ; Selector [0x18]
LINEAR_CODE64_SEL   equ $-GDT_BASE    ; Selector [0x38]
```

当前 guest segment 状态：

```text
CS=0x38
SS/DS/ES/FS/GS=0x18
```

说明 OVMF 已进入 64-bit code selector `0x38`，但早期 IDT 模板仍指向 selector `0x10` 的 32-bit interrupt gate target `10:FFFFFFE4h`。

### 当前判断
当前卡点不是“AxVisor 没有进 OVMF”，而是：

```text
OVMF 已进入 SEC 早期；
发生 #BR 硬件异常；
CPU 尝试用 OVMF 早期 IDT gate 投递 #BR；
该 IDT gate 使用 selector 0x10；
selector 0x10 对应 reset-vector GDT 的 32-bit code segment；
在当前 long-mode/VMX guest state 下投递这个异常时触发 #GP。
```

这提示后续方向可能是：
- 检查 OVMF 早期 IDT 模板在当前 build/阶段是否应该仍使用 selector `0x10`。
- 检查 AxVisor 对 long mode 下 IDT gate / compatibility code selector 的 VMX guest state 支持是否完整。
- 检查是否因为最初 VMCS guest state 或后续 guest segment state 同步不完整，导致 VMX 看到的 descriptor 状态与 OVMF 实际 GDT 不一致。

## 当前修改汇总

### 已修改
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
  - 增加异常 error-code 解码日志。
  - 增加 IDT-vectoring 解码日志。
  - 增加 VMX interruption type 名称。

- `tgoskits/os/axvisor/src/vmm/vcpus.rs`
  - 增加一次只读的 OVMF RIP 窗口 dump，用来抓取 `guest_rip=0x82997a` 附近的真实指令字节。

- `axvisor-2026-notebooks/docs/develop.md`
  - 记录当前 OVMF 早期启动推进过程。
  - 记录 `0x82997a` 的离线反汇编结果。

### 未修改
- 没有修改 OVMF 行为。
- 没有修改 OVMF firmware 镜像。
- 没有新增 fw_cfg / ACPI / PCI / virtio 等平台设备。
- 没有引入跨层 VM/address-space API。

## 当前停止点
我停止在 guest memory dump 之后，因为下一步如果要继续确认 `#BR` 的直接来源，需要继续读 `0x82997a` 对应函数的上下文，并进一步对照 OVMF/EDK2 源码和运行时寄存器状态。

## 当前结论补充
本次已经拿到 `guest_rip=0x82997a` 附近的真实 guest 内存字节，并离线反汇编出对应代码：

```text
0x82997a: mov eax, ecx
0x82997c: bswap eax
0x82997e: ret
```

前后上下文还显示这一段紧邻一个 `bit scan / count leading/trailing` 风格的通用整数位操作函数，以及一个 `pause` 自旋函数。

这说明 `0x82997a` 不像是 OVMF 固件主流程里“阶段跳转点”本身，而更像是某个被调用的通用 helper / library 函数内部地址。

当前 `#BR` 仍然是运行时真实发生的硬件异常，不是 reset-vector 问题，也不是 OVMF_CODE 未加载问题。下一步应继续做两件事：

```text
1. 用 `0x82997a` 周边的反汇编和 OVMF 源码函数签名继续定位具体函数名。
2. 顺着该函数所在源码路径，判断为什么当前 guest state 会在这里触发 `#BR`。
```
 结论：

  - AxVisor 现有的只读 guest memory 能力足够做这件事；
  - 我临时在 tgoskits/os/axvisor/src/vmm/vcpus.rs 加了一个只读 dump；
  - smoke test 已经跑起来，并成功把 0x82997a 附近的 guest 内存打出来了。
  
  我把那段字节离线反汇编后，0x82997a 对应的是：

  mov eax, ecx
  bswap eax
  ret

  这说明：

  - 0x82997a 不像是 OVMF 主启动流程里的“阶段入口”；
  - 它更像是 MdePkg/Library/BaseLib/SwapBytes32.c 这一类通用 helper 的实现点；
  - 周边那段代码还和 HighBitSet32 / HighBitSet64 / CpuPause 这些 BaseLib helper 很接近。
  
  所以现在能更明确地说：

  1. RIP=0x82997a 对应的是 OVMF/EDK2 里的一个通用基础库函数区域，最像 SwapBytes32。
  2. 这也意味着当前 #BR 不是“卡在某个 SEC 大阶段入口”，而是 OVMF 早期执行过程中，进入了某个基础库 helper 后发生的异常。

  这次验证方式是日志 dump，不需要 GDB。
  下一步如果你要，我可以继续做两件事之一：

  - 把 0x82997a 这个 helper 的调用者往上追，找是谁调用到了它；
  - 继续顺着 #BR 的现场状态，判断为什么会在这个 helper 附近投递 #BR。

## Step 6：尝试自编 DEBUG OVMF 并处理构建依赖

### 目标
把 AxVisor 运行时使用的 OVMF 换成自己构建的 DEBUG 版本，尽量借助 OVMF 自身的符号和日志把 `0x82997a` 对应的 helper 调用链继续往上追。

### 已完成的准备
- EDK2 workspace 选在：`/home/ssdns/work/axvisor-uefi/edk2`
- 已执行 `source edksetup.sh`
- 已将 toolchain 从已移除的 `GCC5` 改为当前树可用的 `GCC`
- 已补齐 EDK2 子模块
- 已在 `edk2/BaseTools` 下执行 `make`，让 `GenSec` / `GenFfs` / `GenFw` 等真实工具可用

### 当前构建命令
```bash
build -a X64 -t GCC -b DEBUG -p OvmfPkg/OvmfPkgX64.dsc -D FD_SIZE_4MB
```

### 当前失败点
构建推进到了 `MdeModulePkg/Universal/Disk/RamDiskDxe/RamDiskDxe.inf`，然后卡在 `iasl` 缺失：

```text
"iasl" .../RamDisk.aml .../RamDisk.iiii
/bin/sh: 行 1: iasl: 未找到命令
make: *** ... RamDisk.aml 错误 127
:error 7000: Failed to execute command
:error F002: Failed to build module
  /home/ssdns/work/axvisor-uefi/edk2/MdeModulePkg/Universal/Disk/RamDiskDxe/RamDiskDxe.inf [X64, GCC, DEBUG]
```

### 当前判断
- 这不是 OVMF 源码逻辑失败，而是宿主环境缺少 `iasl`。
- 该树的 OVMF 默认配置把 `RamDiskDxe` 编进了 `OvmfPkg/OvmfPkgX64.dsc` 和 `OvmfPkg/OvmfPkgX64.fdf`。
- 为了继续推进而不引入 sudo/系统安装，下一步先尝试用最小配置绕过 `RamDiskDxe`，再继续跑 smoke test。