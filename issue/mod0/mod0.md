# AxVisor x86_64 OVMF/UEFI 启动适配进展

## 1. 当前目标和阶段性结论

当前工作的目标:先验证一条最小可运行链路：

```text
AxVisor 能读取 VM 配置
  -> 为 OVMF 固件和低端 RAM 建立 GPA -> HPA 映射
  -> 从 rootfs 加载 OVMF_CODE / OVMF_VARS 到指定 GPA
  -> x86 BSP vCPU 以 reset-vector 语义进入 OVMF
  -> 通过 VM-Exit / fault 日志观察 OVMF 后续缺什么
```

目前已经验证到的关键进展是：

```text
OVMF_CODE 被加载到 GPA 0xffc8_4000
OVMF_VARS 被加载到 GPA 0xffc0_0000
BSP vCPU 以 entry=0xffff_fff0 进入 reset-vector 流程
VMCS 中写入了接近 x86 reset state 的 CS/RIP 状态
OVMF 不再停在 reset-vector 附近的 #UD
OVMF 已能运行到较后阶段，并进入保护模式 / 分页模式后触发 triple fault
```

这说明当前修改已经越过了“固件没有被正确加载”或“reset-vector 没有被正确模拟”的最早期问题。
后续问题：OVMF 继续执行后，平台设备、异常路径、内存映射、CPU 特性或中断模型仍不完整，导致最终 triple fault。

本文档按五条线复盘当前修改：

1. VM 配置文件线：在哪里声明 UEFI/OVMF 相关信息。
2. 配置结构和配置链路线：这些字段如何进入 AxVisor 的运行时配置。
3. 内存映射线：如何让 OVMF 所在 GPA 真正有 backing memory。
4. 镜像加载线：如何把 OVMF_CODE / OVMF_VARS 写入对应 guest physical address。
5. vCPU reset-vector 与诊断线：如何让 VMCS 模拟 x86 reset 状态，以及如何观察后续 fault。

---

## 2. 第一条线：VM 配置文件线

### 2.1 修改位置

当前 OVMF VM 配置主要在：

- [ovmf-x86_64-qemu-smp1.toml](../../../tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml)
- [ovmf-x86_64-qemu-smp1.toml](../../../tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml)

其中 `configs/vms` 下的文件更像模板或静态配置来源，`tmp/configs` 下的文件是当前手动测试时实际传给 axvisor 启动流程的临时配置。

### 2.2 当前关键字段

核心配置如下：

```toml
[kernel]
entry_point = 0xffff_fff0
boot = "uefi"
image_location = "fs"
kernel_path = "/guest/uefi/empty-kernel-placeholder.bin"
kernel_load_addr = 0x20_0000
enable_bios = false

ovmf_code_path = "/guest/ovmf/OVMF_CODE.fd"
ovmf_code_base = 0xffc8_4000
ovmf_vars_path = "/guest/ovmf/OVMF_VARS.fd"
ovmf_vars_base = 0xffc0_0000
reset_vector = 0xffff_fff0

memory_regions = [
  [0x0000_0000, 0x0400_0000, 0x7, 0],
  [0xffc0_0000, 0x0040_0000, 0x7, 0],
]
```

### 2.3 每个字段解决的问题

| 字段 | 作用 |
| --- | --- |
| `boot = "uefi"` | 告诉 AxVisor 这不是原来的 Linux/trampoline/BIOS 启动链路，而是 UEFI 链路。 |
| `entry_point = 0xffff_fff0` | 把 VM 的入口语义指向 x86 reset vector。 |
| `reset_vector = 0xffff_fff0` | 显式记录 UEFI reset-vector 入口，避免和普通 kernel entry 混在一起。 |
| `image_location = "fs"` | 当前 OVMF 镜像从 AxVisor rootfs 中读取。 |
| `ovmf_code_path` | AxVisor 内部 rootfs 路径，不是宿主机路径。 |
| `ovmf_code_base` | OVMF_CODE 写入 guest physical address 的起始地址。 |
| `ovmf_vars_path` | OVMF 变量存储镜像路径。 |
| `ovmf_vars_base` | OVMF_VARS 写入 guest physical address 的起始地址。 |
| `memory_regions` | 真正决定哪些 GPA 范围会被分配或映射到 HPA。 |

### 2.4 为什么 OVMF_CODE base 是 0xffc8_4000

最初曾尝试把 OVMF_CODE 放到 `0xffc0_0000`，但后来发现当前系统里的 `OVMF_CODE_4M.fd` 实际文件大小不是完整 4 MiB，而是：

```text
OVMF_CODE_4M.fd size = 0x37c000
```

x86 reset vector 是：

```text
0xffff_fff0
```

OVMF reset-vector 指令通常位于固件镜像末尾附近，因此需要让：

```text
ovmf_code_base + (OVMF_CODE 文件大小 - 0x10) = 0xffff_fff0
```

所以：

```text
ovmf_code_base = 0x1_0000_0000 - 0x37c000 = 0xffc8_4000
```

这次修正后，运行日志显示 OVMF 不再在 reset-vector 附近立刻 `#UD`，而是能继续运行到更后面的地址，证明这个 base 修正是有效的。

---

## 3. 第二条线：配置结构和配置链路线

这条线的作用是：让 TOML 里新增的 OVMF/UEFI 字段能被 AxVisor 的 Rust 配置结构接住，并转化为 VM 创建时真正会使用的运行时配置。

### 3.1 TOML 反序列化结构

修改位置：

[lib.rs](../../../tgoskits/components/axvmconfig/src/lib.rs)

在 `VMKernelConfig` 中新增了 UEFI/OVMF 相关字段：

```rust
pub boot: Option<String>,
pub ovmf_code_path: Option<String>,
pub ovmf_code_base: Option<usize>,
pub ovmf_vars_path: Option<String>,
pub ovmf_vars_base: Option<usize>,
pub reset_vector: Option<usize>,
```

这些字段本身不完成启动，它们只是让配置文件里的值能进入程序。

### 3.2 模板默认值

修改位置：

[templates.rs](../../../tgoskits/components/axvmconfig/src/templates.rs)

这里给模板配置补了默认值：

```rust
boot: None,
ovmf_code_path: None,
ovmf_code_base: None,
ovmf_vars_path: None,
ovmf_vars_base: None,
reset_vector: None,
```

这样做的目的是保持旧配置还能正常构造，不要求所有旧 VM 配置都必须写 UEFI 字段。

### 3.3 配置反序列化测试

修改位置：

[test.rs](../../../tgoskits/components/axvmconfig/src/test.rs)

测试中加入了 UEFI 字段的解析和断言，目的是确认 TOML 中的：

```toml
boot = "uefi"
ovmf_code_path = "OVMF_CODE.fd"
ovmf_code_base = 0xffc0_0000
ovmf_vars_path = "OVMF_VARS.fd"
ovmf_vars_base = 0xff80_0000
reset_vector = 0xffff_fff0
```

确实能进入 `VMKernelConfig`。

### 3.4 AxVM 运行时配置

修改位置：

[config.rs](../../../tgoskits/components/axvm/src/config.rs)

这里新增了 OVMF 运行时信息：

```rust
pub struct OvmfInfo {
    pub code_load_gpa: GuestPhysAddr,
    pub vars_load_gpa: Option<GuestPhysAddr>,
}
```

并在 `VMImageConfig` 中增加：

```rust
pub ovmf: Option<OvmfInfo>,
```

这一步把 TOML 里的 `ovmf_code_base` / `ovmf_vars_base` 转成 AxVisor VM 运行时会使用的 GPA 信息。

### 3.5 兼容旧启动链路的关键点

这里最重要的兼容性处理是：只有当 `boot == "uefi"` 时，才让 BSP entry 使用 `reset_vector`。

当前逻辑是：

```rust
let uefi_boot = cfg.kernel.boot.as_deref() == Some("uefi");
let bsp_entry = if uefi_boot {
    cfg.kernel.reset_vector.unwrap_or(cfg.kernel.entry_point)
} else {
    cfg.kernel.entry_point
};
```

这意味着：

```text
旧启动链路：继续使用 entry_point
UEFI 启动链路：优先使用 reset_vector
```

因此，这个修改目标上是兼容旧启动流程的，不会让所有旧 VM 都被强行改成 reset-vector 启动。

---

## 4. 第三条线：内存映射线

这条线回答的问题是：配置里写了 OVMF 的 GPA 后，AxVisor 是否真的为这些 GPA 建立了可访问的 guest memory backing。

### 4.1 VM 内存分配入口

修改和关注位置：

[config.rs](../../../tgoskits/os/axvisor/src/vmm/config.rs)

创建 VM 后，会调用：

```rust
vm_alloc_memorys(&vm_create_config, &vm);
```

它遍历 TOML 中的：

```toml
memory_regions = [ ... ]
```

并根据 `map_type` 选择不同的映射方式。

当前 OVMF 测试使用的是：

```toml
memory_regions = [
  [0x0000_0000, 0x0400_0000, 0x7, 0],
  [0xffc0_0000, 0x0040_0000, 0x7, 0],
]
```

含义是：

```text
0x0000_0000 - 0x03ff_ffff：低端 RAM，64 MiB
0xffc0_0000 - 0xffff_ffff：4 MiB firmware window
```

### 4.2 为什么 firmware window 从 0xffc0_0000 开始

当前布局是：

```text
OVMF_VARS base = 0xffc0_0000
OVMF_CODE base = 0xffc8_4000
firmware window = [0xffc0_0000, 0x0040_0000]
```

也就是用 4 MiB 高端地址窗口覆盖：

```text
0xffc0_0000 ... 0xffff_ffff
```

这样既覆盖 OVMF_VARS，又覆盖 OVMF_CODE，且包含最终 reset-vector 地址 `0xffff_fff0`。

### 4.3 当前阶段的限制

这条内存映射线目前只解决了“让 OVMF 镜像所在 GPA 有内存 backing”的问题。

它还没有完整解决：

```text
标准 PC 平台的 MMIO 设备模型
fw_cfg
ACPI 表
PCI/PCIe ECAM
virtio 设备
更完整的中断控制器模型
```

所以 OVMF 能开始运行，不代表它能完成整个 UEFI 平台初始化。

---

## 5. 第四条线：镜像加载线

这条线回答的问题是：AxVisor 如何从 rootfs 里读取 OVMF_CODE / OVMF_VARS，并写到上面已经映射好的 GPA。

### 5.1 修改位置

[mod.rs](../../../tgoskits/os/axvisor/src/vmm/images/mod.rs)

在 `ImageLoader` 中新增了 OVMF 目标地址：

```rust
ovmf_code_gpa: Option<GuestPhysAddr>,
ovmf_vars_gpa: Option<GuestPhysAddr>,
```

在 `load()` 中，从 VM runtime config 读取：

```rust
if let Some(ovmf) = &config.image_config.ovmf {
    self.ovmf_code_gpa = Some(ovmf.code_load_gpa);
    self.ovmf_vars_gpa = ovmf.vars_load_gpa;
}
```

### 5.2 UEFI 与旧镜像加载链路分流

当前关键分流是：

```rust
if self.config.kernel.boot.as_deref() == Some("uefi") {
    return self.load_uefi_images();
}
```

也就是：

```text
boot != "uefi"：继续走原来的 kernel / bios / dtb 加载逻辑
boot == "uefi"：进入 OVMF 加载逻辑
```

这也是兼容旧链路的关键点之一。

### 5.3 OVMF 镜像加载

UEFI 分支中会执行：

```rust
fs::load_vm_image(ovmf_code_path, ovmf_code_gpa, self.vm.clone())?;
fs::load_vm_image(ovmf_vars_path, ovmf_vars_gpa, self.vm.clone())?;
```

当前运行日志已经看到：

```text
Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000
Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000
```

这说明镜像加载线已经跑通。

### 5.4 rootfs 路径问题

这里要特别注意：

```text
/guest/ovmf/OVMF_CODE.fd
/guest/ovmf/OVMF_VARS.fd
```

是 AxVisor 运行时看到的 rootfs 内部路径，不是宿主机 Linux 的路径。

所以宿主机上的：

```text
/usr/share/OVMF/OVMF_CODE_4M.fd
/usr/share/OVMF/OVMF_VARS_4M.fd
```

需要先复制进 AxVisor rootfs image 中，变成：

```text
/guest/ovmf/OVMF_CODE.fd
/guest/ovmf/OVMF_VARS.fd
```

否则 AxVisor 会报：

```text
Failed to open /usr/share/OVMF/OVMF_CODE_4M.fd
```

---

## 6. 第五条线：vCPU reset-vector 与诊断线

这条线是当前最关键的执行线：即使 OVMF 被加载到了正确地址，如果 vCPU 初始状态仍像普通 kernel entry 那样设置，也不能正确模拟 x86 上电后的 reset-vector 行为。

### 6.1 VMX vCPU 修改位置

[vcpu.rs](../../../tgoskits/components/x86_vcpu/src/vmx/vcpu.rs)

当前重点是 VMCS guest state 初始化。

### 6.2 reset-vector 判断

当前逻辑会识别：

```rust
let entry_addr = entry.as_usize();
let reset_vector_entry = matches!(entry_addr, 0xffff_fff0 | 0xfff0);
```

如果入口是 reset-vector，则使用：

```rust
let (cs_selector, cs_base, rip) = if reset_vector_entry {
    (0xf000, 0xffff_0000, 0xfff0)
} else {
    (0, 0, entry_addr)
};
```

这对应 x86 reset 后的典型入口语义：

```text
CS.selector = 0xf000
CS.base     = 0xffff_0000
RIP         = 0xfff0
linear addr = CS.base + RIP = 0xffff_fff0
```

也就是说，虽然配置文件中写的是物理 reset vector：

```text
0xffff_fff0
```

但 VMCS 中不是简单写 `RIP = 0xffff_fff0`，而是写成更接近真实 x86 reset state 的：

```text
CS.base = 0xffff_0000
RIP     = 0xfff0
```

### 6.3 当前验证证据

运行日志已经出现：

```text
VMX guest entry setup: entry=0xfffffff0, reset_vector=true, cs=0xf000, cs_base=0xffff0000, rip=0xfff0
```

这说明 reset-vector 分支确实被触发。

后续日志中 OVMF 能运行到：

```text
guest_rip: 0x82997a
cr0: 0x80000023
cr3: 0x800000
cr4: 0x2660
cs: 0x38
```

这说明它不再只是停留在 reset-vector 附近，而是已经完成了一部分固件早期初始化，并进入了保护模式 / 分页模式后的执行阶段。

### 6.4 SVM vCPU 同步适配

另一个修改位置：

[vcpu.rs](../../../tgoskits/components/x86_vcpu/src/svm/vcpu.rs)

这里也做了类似 reset-vector 特判：

```rust
let (cs_selector, cs_base, rip) = if entry.as_usize() == 0xffff_fff0 {
    (0xf000, 0xffff_0000, 0xfff0)
} else {
    (0, 0, entry.as_usize() as u64)
};
```

当前主要测试路径是 Intel VMX，但 SVM 侧也先保持了同方向适配。

### 6.5 诊断日志修改

为了知道 OVMF 为什么失败，VMX exit handling 中增加了异常和 triple fault 诊断。

关注位置：

[vcpu.rs](../../../tgoskits/components/x86_vcpu/src/vmx/vcpu.rs)

当前对这些异常设置了拦截：

```rust
let exception_bitmap: u32 = (1 << 6) | (1 << 8) | (1 << 13) | (1 << 14);
```

对应：

```text
#UD：invalid opcode
#DF：double fault
#GP：general protection
#PF：page fault
```

同时增加了：

```text
VMX exception/NMI VM-Exit
VMX exception raw fields
VMX guest state
VMX guest CS
```

以及 triple fault 分支：

```text
VMX triple fault VM-Exit
VMX triple fault raw fields
VMX triple fault guest state
VMX triple fault guest CS
```

这条诊断线的目的不是修复 OVMF，而是捕获 triple fault 之前更早、更具体的异常原因。

---

## 7. 另外一处关键修补：UEFI 模式跳过 placeholder kernel header

修改位置：

[config.rs](../../../tgoskits/os/axvisor/src/vmm/config.rs)

原有 `init_guest_vm()` 会读取 Linux kernel header。UEFI 模式下我们目前没有真正的 Linux kernel 文件，而是用 placeholder 保持旧配置结构完整，因此曾出现：

```text
Failed to open /guest/uefi/empty-kernel-placeholder.bin
```

当前修复是：

```rust
if vm_create_config.kernel.boot.as_deref() != Some("uefi") {
    if let Some(linux) = super::images::get_image_header(&vm_create_config) {
        ...
    }
}
```

含义是：

```text
旧 Linux/trampoline 链路：继续读取 kernel header
UEFI 链路：不再读取 placeholder kernel
```

这也是保持旧链路兼容、同时让 UEFI 分支独立前进的一处小修补。

---

## 8. 当前整体链路图

可以把目前的启动适配理解成下面这条链：

```text
ovmf-x86_64-qemu-smp1.toml
  |
  |-- boot = "uefi"
  |-- reset_vector = 0xffff_fff0
  |-- ovmf_code_base = 0xffc8_4000
  |-- ovmf_vars_base = 0xffc0_0000
  |-- memory_regions = firmware window + low RAM
  v
AxVMCrateConfig / VMKernelConfig
  |
  |-- 接住 TOML 字段
  v
AxVMConfig / VMImageConfig / OvmfInfo
  |
  |-- UEFI 模式下 BSP entry 使用 reset_vector
  |-- OVMF image GPA 进入 runtime config
  v
vm_alloc_memorys()
  |
  |-- 为 low RAM 和 firmware window 建立 GPA -> HPA backing
  v
ImageLoader::load_uefi_images()
  |
  |-- 从 /guest/ovmf/OVMF_CODE.fd 读取并写入 0xffc8_4000
  |-- 从 /guest/ovmf/OVMF_VARS.fd 读取并写入 0xffc0_0000
  v
x86_vcpu VMX setup
  |
  |-- entry=0xffff_fff0 被识别为 reset-vector
  |-- VMCS 写入 CS.selector=0xf000, CS.base=0xffff_0000, RIP=0xfff0
  v
OVMF 开始执行
  |
  |-- 当前已能运行到保护模式 / 分页模式后的地址
  v
triple fault
  |
  |-- 下一步通过 exception bitmap 和 triple fault raw fields 定位第一异常
```

---

## 9. 当前进展如何汇报

可以这样汇报当前阶段性成果：

```text
我们已经完成了 x86_64 OVMF 启动最小链路的第一阶段适配：

1. 在 AxVisor VM 配置中加入 UEFI boot selector、OVMF_CODE/OVMF_VARS 路径和加载 GPA、reset vector 等字段。
2. 修改配置结构和运行时 VM 配置，让这些字段从 TOML 进入 AxVisor 的 VM 创建流程。
3. 通过 memory_regions 为低端 RAM 和高端 4 MiB firmware window 建立 GPA 到 HPA 的 backing。
4. 修改 ImageLoader，在 boot="uefi" 时跳过原 Linux kernel 加载流程，改为从 rootfs 加载 OVMF_CODE 和 OVMF_VARS 到指定 GPA。
5. 修改 x86 VMX vCPU 初始状态，在 reset-vector 启动时把 CS/RIP 设置为接近真实 x86 reset state 的形式：CS.selector=0xf000、CS.base=0xffff0000、RIP=0xfff0。

目前运行结果证明 OVMF 已经不再停在 reset vector 附近，而是能进入后续保护模式/分页模式执行阶段。当前阻塞是 OVMF 后续运行触发 triple fault，下一步要通过新增的 VMX exception/triple-fault 诊断日志定位第一个异常原因。
```

---

## 10. 当前风险和未完成事项

### 10.1 兼容性风险

当前设计上尽量通过 `boot = "uefi"` 分支隔离旧链路：

```text
只有 UEFI VM 使用 reset_vector 和 load_uefi_images()
旧 VM 继续使用 entry_point、kernel image、bios image 等旧逻辑
```

但目前仍属于快速验证阶段，后续需要重点复查：

```text
旧 x86 guest 是否仍能启动
非 x86 架构是否受新增字段影响
static config / filesystem config 两种来源是否都正常
```

### 10.2 临时性实现

当前存在一些偏验证性质的代码，需要后续清理或确认：

```text
VMX reset-vector 相关日志后续应降级或移除
异常/triple fault 日志后续应变成更系统的调试能力
RSP 当前写法需要复查是否符合真实 reset state 或仅是临时调试
临时 link.x symlink 如果不再需要应清理
```

### 10.3 UEFI 平台能力仍缺失

当前只是让 OVMF 能被加载并开始执行，不代表已经具备完整 PC UEFI 平台。

后续大概率仍需要补：

```text
fw_cfg
ACPI
PCI/PCIe
virtio-blk / virtio-net
串口/控制台路径
中断控制器模型
更完整的 CPUID/MSR 暴露
更完整的 MMIO/EPT fault handling
```

这些不应该在当前阶段一次性展开，而应该根据 OVMF 下一步日志逐个补。

---

## 11. 下一步定位方向

下一次运行时重点看新增日志。

如果先抓到：

```text
VMX exception/NMI VM-Exit
VMX exception raw fields
VMX guest state
VMX guest CS
```

则根据 vector 判断：

| vector | 方向 |
| --- | --- |
| `#PF` | 看 `GUEST_LINEAR_ADDR` / `GUEST_PHYSICAL_ADDR`，判断是页表访问、GPA 映射还是 MMIO 缺失。 |
| `#GP` | 看 err code、CS、CR0/CR4/EFER、MSR 或 segment 状态。 |
| `#UD` | 看是否执行了当前虚拟 CPU 暴露不支持的指令，可能涉及 CPUID/MSR 能力。 |
| `#DF` | 说明前一个异常处理失败，需要反推 IDT/GDT/栈/异常注入路径。 |

如果仍然只看到：

```text
VMX triple fault VM-Exit
VMX triple fault raw fields
VMX triple fault guest state
VMX triple fault guest CS
```

则优先根据 raw VMCS 字段判断 triple fault 前是否留下 IDT-vectoring 或 exit qualification 信息。

当前不建议立刻转向完整 ACPI/PCI/virtio 实现，而应继续沿“最小可观测链路”推进：先确认第一个明确失败点，再针对性补最小缺口。
