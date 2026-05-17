# AxVisor x86_64 OVMF/UEFI 启动适配进展

## 1. 当前目标和阶段性结论

当前工作的目标:先验证可运行链路：

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

说明当前修改已经越过了“固件没有被正确加载”或“reset-vector 没有被正确模拟”的最早期问题。

本文档按五条线复盘当前修改：

1. VM 配置文件线：在哪里声明 UEFI/OVMF 相关信息。
2. 配置结构和配置链路线：这些字段如何进入 AxVisor 的运行时配置。
3. 内存映射线：如何让 OVMF 所在 GPA 真正有 backing memory。
4. 镜像加载线：如何把 OVMF_CODE / OVMF_VARS 写入对应 guest physical address。
5. vCPU reset-vector 线：如何让 VMCS 模拟 x86 reset 状态。

---

## 2. 第一条线：VM 配置文件线

### 2.1 修改位置

当前 OVMF VM 配置主要在：

- [ovmf-x86_64-qemu-smp1.toml](../../../tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml)
- [ovmf-x86_64-qemu-smp1.toml](../../../tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml)

### 2.2 当前关键字段

配置如下：

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

### 2.3 字段含义

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

OVMF_CODE base 是 0xffc8_4000：因为当前系统里的 `OVMF_CODE_4M.fd` 实际文件大小不是完整 4 MiB

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

让配置文件里的值进入程序。

### 3.2 AxVM 运行时配置

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

### 3.3 兼容旧启动链路

当前逻辑是：

```rust
let uefi_boot = cfg.kernel.boot.as_deref() == Some("uefi");
let bsp_entry = if uefi_boot {
    cfg.kernel.reset_vector.unwrap_or(cfg.kernel.entry_point)
} else {
    cfg.kernel.entry_point
};
```

```text
旧启动链路：继续使用 entry_point
UEFI 启动链路：优先使用 reset_vector
```

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

分流代码：

```rust
if self.config.kernel.boot.as_deref() == Some("uefi") {
    return self.load_uefi_images();
}
```

### 5.3 OVMF 镜像加载

UEFI 分支中会执行：

```rust
fs::load_vm_image(ovmf_code_path, ovmf_code_gpa, self.vm.clone())?;
fs::load_vm_image(ovmf_vars_path, ovmf_vars_gpa, self.vm.clone())?;
```

运行日志：

```text
Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000
Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000
```

---

## 6. 第五条线：vCPU reset-vector


改变 vCPU 初始状态模拟 x86 上电后的 reset-vector 行为。

### 6.1 VMX vCPU 修改位置

[vcpu.rs](../../../tgoskits/components/x86_vcpu/src/vmx/vcpu.rs)
VMCS guest state 初始化。

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

x86 reset 后的典型入口语义：

```text
CS.selector = 0xf000
CS.base     = 0xffff_0000
RIP         = 0xfff0
linear addr = CS.base + RIP = 0xffff_fff0
```

### 6.3 当前验证证据

运行日志：

```text
VMX guest entry setup: entry=0xfffffff0, reset_vector=true, cs=0xf000, cs_base=0xffff0000, rip=0xfff0
```

后续日志中 OVMF 运行到：

```text
guest_rip: 0x82997a
cr0: 0x80000023
cr3: 0x800000
cr4: 0x2660
cs: 0x38
```

完成了一部分固件早期初始化，并进入了保护模式 / 分页模式后的执行阶段。

## 7. 当前整体链路图

启动适配链：

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

## 8. 进展汇报

```text
已经完成了 x86_64 OVMF 启动最小链路的第一阶段适配：

1. 在 AxVisor VM 配置中加入 UEFI boot selector、OVMF_CODE/OVMF_VARS 路径和加载 GPA、reset vector 等字段。
2. 修改配置结构和运行时 VM 配置，让这些字段从 TOML 进入 AxVisor 的 VM 创建流程。
3. 通过 memory_regions 为低端 RAM 和高端 4 MiB firmware window 建立 GPA 到 HPA 的 backing。
4. 修改 ImageLoader，在 boot="uefi" 时跳过原 Linux kernel 加载流程，改为从 rootfs 加载 OVMF_CODE 和 OVMF_VARS 到指定 GPA。
5. 修改 x86 VMX vCPU 初始状态，在 reset-vector 启动时把 CS/RIP 设置为接近真实 x86 reset state 的形式：CS.selector=0xf000、CS.base=0xffff0000、RIP=0xfff0。

目前运行结果证明 OVMF 已经不再停在 reset vector 附近，而是能进入后续保护模式/分页模式执行阶段。当前阻塞是 OVMF 后续运行触发 triple fault，下一步要通过新增的 VMX exception/triple-fault 诊断日志定位第一个异常原因。
```
