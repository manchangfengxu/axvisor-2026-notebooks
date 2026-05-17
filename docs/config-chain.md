# AxVisor UEFI 配置链路与程序消费关系

本文说明 AxVisor 中 VM 配置的作用边界，以及 OVMF/UEFI 启动第一阶段需要关注的配置项如何关联到真正执行工作的程序。

重点不是完整理解 AxVisor 的所有配置加载细节，而是明确：

```text
TOML 中配置了什么
        ↓
进入哪个 Rust 配置结构
        ↓
被哪个 AxVisor 程序消费
        ↓
最终影响内存映射、固件加载或 vCPU 初始状态
```

配置链路本身不直接完成 UEFI 启动。它的作用是把用户在 TOML 中写下的值搬运到 AxVisor 内部配置结构中，再由具体程序使用这些值完成内存映射、镜像加载和 vCPU 初始化。

---

## 1. 当前阶段需要关心的三个目标

x86_64 OVMF 启动第一阶段的目标不是启动 Linux EFI，也不是立即补齐 fw_cfg、ACPI、PCI 或 virtio，而是先证明 OVMF 能从 reset vector 取指执行。

因此当前阶段最重要的三个目标是：

```text
1. OVMF_CODE 所在 GPA 区间已经映射到有效 HPA
2. OVMF_CODE.fd 文件内容已经写入该 GPA 区间
3. vCPU 初始状态指向 OVMF reset vector
```

对应到程序职责：

```text
memory_regions
        ↓
建立 GPA -> HPA 映射

ovmf_code_path / ovmf_code_base
        ↓
把 OVMF_CODE.fd 加载到指定 GPA

reset_vector
        ↓
成为 BSP vCPU 的初始入口，后续应进入 x86 VMCS guest-state 初始化
```

---

## 2. 配置文件入口

当前 VM 配置入口是 AxVisor 的 VM TOML 文件，例如：

```text
os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml
```

第一阶段建议把 OVMF 相关字段放在现有 `[kernel]` 段中，而不是立即新增新的顶层 `[uefi]` 段。原因是当前 `AxVMCrateConfig` 的顶层结构只有：

```rust
pub struct AxVMCrateConfig {
    pub base: VMBaseConfig,
    pub kernel: VMKernelConfig,
    pub devices: VMDevicesConfig,
}
```

也就是说，当前稳定可解析的 TOML 结构是：

```toml
[base]
...

[kernel]
...

[devices]
...
```

因此，当前阶段的 OVMF 配置可以先写成：

```toml
[kernel]
boot = "uefi"

ovmf_code_path = "/guest/ovmf/OVMF_CODE.fd"
ovmf_code_base = 0xffc0_0000

ovmf_vars_path = "/guest/ovmf/OVMF_VARS.fd"
ovmf_vars_base = 0xff80_0000

reset_vector = 0xffff_fff0

memory_regions = [
  [0x0000_0000, 0x0800_0000, 0x7, 0],
  [0xff80_0000, 0x0080_0000, 0x7, 2],
]
```

其中每个字段的核心含义是：

| 字段 | 作用 |
|---|---|
| `boot = "uefi"` | 选择 UEFI/OVMF 加载路径，避免继续走 legacy BIOS/trampoline 逻辑 |
| `ovmf_code_path` | OVMF_CODE.fd 在 AxVisor 文件系统中的路径 |
| `ovmf_code_base` | OVMF_CODE.fd 被加载到的客户机物理地址 |
| `ovmf_vars_path` | OVMF_VARS.fd 在 AxVisor 文件系统中的路径 |
| `ovmf_vars_base` | OVMF_VARS.fd 被加载到的客户机物理地址 |
| `reset_vector` | x86 reset vector，第一阶段用于设置 BSP 初始入口 |
| `memory_regions` | 声明 VM 地址空间中哪些 GPA 区间可访问 |

---

## 3. 配置来源：文件系统配置与静态配置

AxVisor 运行时有两类 VM 配置来源。

### 3.1 文件系统配置

运行时会优先尝试从 AxVisor 文件系统读取：

```text
/guest/vm_default/*.toml
```

相关程序在：

```text
os/axvisor/src/vmm/config.rs
```

函数：

```rust
config::filesystem_vm_configs()
```

它的作用是：

```text
扫描 /guest/vm_default
        ↓
读取 .toml 文件内容
        ↓
检查是否包含 [base] / [kernel] / [devices]
        ↓
返回 Vec<String>
```

这里返回的是 TOML 文本字符串，不是已经创建好的 VM。

### 3.2 编译期静态配置

如果文件系统中没有 VM 配置，AxVisor 会回退到编译期静态配置：

```rust
config::static_vm_configs()
```

静态配置由：

```text
os/axvisor/build.rs
```

根据环境变量生成：

```text
AXVISOR_VM_CONFIGS
```

构建时链条是：

```text
AXVISOR_VM_CONFIGS
        ↓
os/axvisor/build.rs
        ↓
OUT_DIR/vm_configs.rs
        ↓
include!(concat!(env!("OUT_DIR"), "/vm_configs.rs"))
        ↓
config::static_vm_configs()
```

如果 VM 配置里使用：

```toml
image_location = "memory"
```

`build.rs` 还会把相关 kernel / bios / dtb / ramdisk 通过 `include_bytes!()` 编译进 AxVisor，并生成：

```rust
config::get_memory_images()
```

如果使用：

```toml
image_location = "fs"
```

则配置文本可以被编译进 AxVisor，但镜像文件仍需要运行时从 AxVisor 文件系统读取。

OVMF 第一阶段更适合使用：

```toml
image_location = "fs"
```

因为 OVMF_CODE.fd 和 OVMF_VARS.fd 通常作为固件文件放在 AxVisor 文件系统中，由运行时 ImageLoader 读取。

---

## 4. 配置进入 Rust 结构

不论配置来自 `/guest/vm_default/*.toml`，还是来自 `static_vm_configs()`，最终都会进入：

```text
os/axvisor/src/vmm/config.rs
```

函数：

```rust
init_guest_vm(raw_cfg: &str)
```

核心步骤是：

```rust
let vm_create_config = AxVMCrateConfig::from_toml(raw_cfg)?;
```

`AxVMCrateConfig::from_toml()` 定义在：

```text
components/axvmconfig/src/lib.rs
```

它负责把 TOML 文本反序列化成 Rust 配置结构。

当前 OVMF 第一阶段需要的字段位于：

```rust
pub struct VMKernelConfig {
    pub boot: Option<String>,
    pub ovmf_code_path: Option<String>,
    pub ovmf_code_base: Option<usize>,
    pub ovmf_vars_path: Option<String>,
    pub ovmf_vars_base: Option<usize>,
    pub reset_vector: Option<usize>,
}
```

这一步只完成“配置值进入 Rust 变量”。例如：

```toml
ovmf_code_base = 0xffc0_0000
```

会变成：

```rust
vm_create_config.kernel.ovmf_code_base == Some(0xffc0_0000)
```

这一步本身不会建立内存映射，也不会加载 OVMF 文件，更不会写 VMCS。

---

## 5. 配置链路总览

运行时主链条可以简化为：

```text
VM TOML 文本
        ↓
AxVMCrateConfig::from_toml()
        ↓
AxVMCrateConfig
        ↓
AxVMConfig::from(vm_create_config.clone())
        ↓
VM::new(vm_config)
        ↓
vm_alloc_memorys(&vm_create_config, &vm)
        ↓
ImageLoader::new(...).load()
        ↓
vm.init()
        ↓
vCPU 初始化 / 运行
```

其中需要重点区分两个配置结构：

| 配置结构 | 位置 | 作用 |
|---|---|---|
| `AxVMCrateConfig` | `components/axvmconfig` | 贴近 TOML，保存用户原始配置语义 |
| `AxVMConfig` | `components/axvm` | 运行时 VM 创建使用的配置 |

`AxVMCrateConfig` 更像“从 TOML 读出来的原始配置”。

`AxVMConfig` 更像“VM 子系统真正用来创建 VM 的运行时配置”。

---

## 6. 字段消费关系总表

第一阶段最重要的是知道每个 OVMF 相关字段最终被哪个程序消费。

| 配置目标 | TOML 字段 | Rust 配置结构 | 消费程序 | 最终影响 |
|---|---|---|---|---|
| 选择 UEFI 启动路径 | `boot = "uefi"` | `VMKernelConfig.boot` | `ImageLoader::load()` | 进入 UEFI 镜像加载分支，跳过 legacy kernel/BIOS 加载 |
| OVMF 高地址窗口映射 | `memory_regions` | `VMKernelConfig.memory_regions` | `vm_alloc_memorys()` | 建立 GPA -> HPA 映射 |
| OVMF_CODE 文件路径 | `ovmf_code_path` | `VMKernelConfig.ovmf_code_path` | `ImageLoader::load_uefi_images()` | 找到 OVMF_CODE.fd 文件 |
| OVMF_CODE 加载地址 | `ovmf_code_base` | `VMKernelConfig.ovmf_code_base` -> `VMImageConfig.ovmf.code_load_gpa` | `ImageLoader::load_uefi_images()` | 把 OVMF_CODE.fd 写入指定 GPA |
| OVMF_VARS 文件路径 | `ovmf_vars_path` | `VMKernelConfig.ovmf_vars_path` | `ImageLoader::load_uefi_images()` | 找到 OVMF_VARS.fd 文件 |
| OVMF_VARS 加载地址 | `ovmf_vars_base` | `VMKernelConfig.ovmf_vars_base` -> `VMImageConfig.ovmf.vars_load_gpa` | `ImageLoader::load_uefi_images()` | 把 OVMF_VARS.fd 写入指定 GPA |
| BSP 初始入口 | `reset_vector` | `VMKernelConfig.reset_vector` -> `AxVMConfig.cpu_config.bsp_entry` | x86 vCPU 初始化路径 | 后续应写入 VMCS guest-state |

这张表是理解当前配置链路的核心。它说明配置字段不是孤立存在的，每个字段都应该能追踪到一个消费程序。

---

## 7. GPA 到 HPA 映射：`memory_regions` 的作用

OVMF reset vector 是：

```text
0xfffffff0
```

这不是一块单独需要加载的数据，而是 CPU 初始取指地址。

为了让 vCPU 能从这个地址取指，必须满足：

```text
GPA 0xfffffff0
        ↓
能在 VM 地址空间中翻译到某个 HPA
        ↓
该 HPA 上存在 OVMF reset stub 指令
```

第一步由 `memory_regions` 决定。

相关程序是：

```text
os/axvisor/src/vmm/config.rs
```

函数：

```rust
fn vm_alloc_memorys(vm_create_config: &AxVMCrateConfig, vm: &VM)
```

它遍历：

```rust
vm_create_config.kernel.memory_regions
```

并根据 `map_type` 建立映射。

当前 `map_type` 含义是：

```text
0 = MAP_ALLOC
1 = MAP_IDENTICAL
2 = MAP_RESERVED
```

对 OVMF 第一阶段而言，配置中必须有一个区间覆盖：

```text
0xfffffff0
```

例如：

```toml
memory_regions = [
  [0xff80_0000, 0x0080_0000, 0x7, 2],
]
```

这个区间是：

```text
0xff80_0000 .. 0xffff_ffff
```

它包含：

```text
0xffff_fff0
```

因此从“地址空间是否存在映射”的角度看，reset vector 对应 GPA 有机会被翻译。

需要注意：`memory_regions` 只负责映射地址空间，不负责把 OVMF 文件内容写进去。

---

## 8. OVMF_CODE 加载：`ovmf_code_path` 与 `ovmf_code_base`

OVMF_CODE 文件加载由：

```text
os/axvisor/src/vmm/images/mod.rs
```

中的 `ImageLoader` 完成。

当前 UEFI 分支入口是：

```rust
if self.config.kernel.boot.as_deref() == Some("uefi") {
    return self.load_uefi_images();
}
```

当配置为：

```toml
boot = "uefi"
ovmf_code_path = "/guest/ovmf/OVMF_CODE.fd"
ovmf_code_base = 0xffc0_0000
```

`load_uefi_images()` 会执行：

```text
读取 /guest/ovmf/OVMF_CODE.fd
        ↓
写入 GPA 0xffc0_0000
```

如果 OVMF_CODE.fd 大小是 4 MiB，那么它对应的 GPA 区间通常是：

```text
0xffc0_0000 .. 0xffff_ffff
```

此时 reset vector：

```text
0xffff_fff0
```

位于 OVMF_CODE 映射区间末尾附近。

因此，OVMF_CODE 启动取指需要同时满足：

```text
memory_regions 覆盖 0xffc0_0000 .. 0xffff_ffff
        ↓
ImageLoader 把 OVMF_CODE.fd 写入 0xffc0_0000
        ↓
reset_vector 0xffff_fff0 落在 OVMF_CODE 内容范围内
```

---

## 9. OVMF_VARS 加载：`ovmf_vars_path` 与 `ovmf_vars_base`

OVMF_VARS 是 UEFI 变量存储区域，用于保存 NVRAM / BootOrder / UEFI variables 等状态。

第一阶段可以先加载它，但不一定立即实现完整 pflash 语义。

当前配置：

```toml
ovmf_vars_path = "/guest/ovmf/OVMF_VARS.fd"
ovmf_vars_base = 0xff80_0000
```

对应程序仍然是：

```text
os/axvisor/src/vmm/images/mod.rs
```

`load_uefi_images()` 会执行：

```text
读取 /guest/ovmf/OVMF_VARS.fd
        ↓
写入 GPA 0xff80_0000
```

需要注意：真正的 UEFI 变量存储通常不是普通 RAM 文件加载这么简单，后续还需要考虑：

```text
pflash 访问语义
写保护 / 可写区域区分
变量区持久化
MMIO 或 flash 设备模型
```

但第一阶段的目标是让 OVMF 能开始执行，因此可以先把 OVMF_VARS 当成普通镜像加载到指定 GPA。

---

## 10. reset_vector 与 vCPU 初始状态

`reset_vector` 当前首先进入：

```text
components/axvmconfig/src/lib.rs
```

字段：

```rust
VMKernelConfig.reset_vector
```

随后在：

```text
components/axvm/src/config.rs
```

转换为：

```rust
AxVMConfig.cpu_config.bsp_entry
```

简化链条是：

```text
TOML reset_vector = 0xffff_fff0
        ↓
VMKernelConfig.reset_vector
        ↓
AxVMConfig.cpu_config.bsp_entry
        ↓
后续 x86 vCPU 初始化路径
```

这一步仍然只是“把 reset vector 传递到运行时 VM 配置”。

它不等于已经完成 x86 reset state。

x86 reset vector 的真实语义通常不是简单设置：

```text
RIP = 0xfffffff0
```

而更接近：

```text
CS.selector = 0xf000
CS.base     = 0xffff0000
IP/EIP      = 0xfff0
linear addr = 0xfffffff0
```

因此，后续必须继续检查 x86 vCPU 初始化路径：

```text
AxVMConfig.cpu_config.bsp_entry
        ↓
VM 创建 / vCPU 创建
        ↓
vcpu.set_entry(...) 或架构相关 init
        ↓
VMCS guest RIP / CS / CR0 / CR4 / EFER 等字段
```

当前配置链路只保证 `reset_vector` 能到达 `bsp_entry`。是否真正写入正确 VMCS guest-state，要看 x86 vCPU 初始化代码。

---

## 11. legacy BIOS 路径与 UEFI 路径的区别

当前 AxVisor 原有 x86_64 启动方式是 legacy BIOS/trampoline 风格。

典型配置是：

```toml
entry_point = 0x8000
enable_bios = true
bios_load_addr = 0x8000
```

这条路径的语义是：

```text
低地址 BIOS/trampoline 镜像
        ↓
加载到 0x8000
        ↓
从 0x8000 附近进入
        ↓
AxVisor 自定义 multiboot 信息 patch
```

相关程序在 `ImageLoader` 中会加载 BIOS 镜像，并在 x86_64 下执行 multiboot 信息写入和 patch。

OVMF 不应该直接塞进这个 legacy BIOS 路径，因为 OVMF 不是 AxVisor 的低地址 trampoline 镜像，也不应该被当作需要 patch EBX multiboot 立即数的 BIOS blob。

因此 UEFI 路径需要：

```text
boot = "uefi"
        ↓
进入 load_uefi_images()
        ↓
跳过 legacy kernel / BIOS multiboot patch
        ↓
加载 OVMF_CODE / OVMF_VARS
```

---

## 12. 当前最小适配的能力边界

当前最小适配目标是：

```text
配置能表达 OVMF
        ↓
AxVisor 能解析这些字段
        ↓
ImageLoader 能根据 boot = "uefi" 加载 OVMF_CODE / OVMF_VARS
        ↓
reset_vector 能传递到 bsp_entry
```

这意味着已经覆盖了第一阶段的一部分需求：

| 能力 | 当前状态 |
|---|---|
| TOML 表达 OVMF_CODE 路径和地址 | 已有字段 |
| TOML 表达 OVMF_VARS 路径和地址 | 已有字段 |
| TOML 表达 reset vector | 已有字段 |
| OVMF 高地址窗口由配置声明 | 依赖 `memory_regions` |
| OVMF_CODE 文件加载到 GPA | 由 `load_uefi_images()` 执行 |
| OVMF_VARS 文件加载到 GPA | 由 `load_uefi_images()` 执行 |
| reset_vector 传给 BSP entry | 由 `AxVMConfig::from()` 执行 |
| VMCS guest-state 完整 reset 初始化 | 仍需检查/实现 |
| fw_cfg / ACPI / PCI / virtio | 后续阶段 |

---

## 13. 当前阶段的检查清单

为了确认“配置链路已经服务于 OVMF 第一阶段”，可以按以下顺序检查。

### 13.1 配置是否可解析

检查点：

```text
AxVMCrateConfig::from_toml()
```

目标：

```text
boot == Some("uefi")
ovmf_code_path != None
ovmf_code_base != None
reset_vector == Some(0xfffffff0)
```

### 13.2 OVMF 固件窗口是否有 GPA -> HPA 映射

检查点：

```text
vm_alloc_memorys()
```

目标：

```text
memory_regions 覆盖 ovmf_code_base .. ovmf_code_base + OVMF_CODE_SIZE
memory_regions 覆盖 0xfffffff0
```

如果这里失败，vCPU 从 reset vector 取指时会遇到地址翻译失败或 EPT violation。

### 13.3 OVMF_CODE 是否加载到映射窗口

检查点：

```text
ImageLoader::load_uefi_images()
```

目标：

```text
OVMF_CODE.fd 被读取
OVMF_CODE.fd 被写入 ovmf_code_base
0xfffffff0 落在 OVMF_CODE 文件内容对应区间
```

### 13.4 BSP entry 是否指向 reset vector

检查点：

```text
AxVMConfig::from()
```

目标：

```text
cpu_config.bsp_entry == 0xfffffff0
```

### 13.5 x86 VMCS guest-state 是否正确

检查点：

```text
x86 vCPU 初始化 / set_entry / VMCS guest-state 设置路径
```

目标不是简单确认 RIP，而是确认是否接近 x86 reset state：

```text
CS.selector = 0xf000
CS.base     = 0xffff0000
IP/EIP      = 0xfff0
linear addr = 0xfffffff0
CR0 / CR4 / EFER / segment attributes 合法
```

如果这里不正确，可能出现：

```text
invalid guest state
triple fault
立即 VM-Exit
无法按预期从 OVMF reset stub 取指
```

---

## 14. 大方向结论

配置链路的核心作用是传递值，不是完成启动。

当前 OVMF 第一阶段只需要抓住三条关键关联：

```text
memory_regions
        ↓
vm_alloc_memorys()
        ↓
建立 OVMF 固件窗口 GPA -> HPA 映射
```

```text
ovmf_code_path / ovmf_code_base
        ↓
ImageLoader::load_uefi_images()
        ↓
把 OVMF_CODE.fd 写入固件窗口
```

```text
reset_vector
        ↓
AxVMConfig.cpu_config.bsp_entry
        ↓
后续 x86 vCPU 初始化应设置 VMCS guest-state
```

因此，当前不需要完整掌握所有配置加载细节，只需要确认：

1. 需要配置的字段写在 VM TOML 的 `[kernel]` 中。
2. 这些字段能进入 `VMKernelConfig`。
3. 内存映射相关字段被 `vm_alloc_memorys()` 消费。
4. OVMF 镜像相关字段被 `ImageLoader::load_uefi_images()` 消费。
5. reset vector 字段能到达 `AxVMConfig.cpu_config.bsp_entry`。
6. 下一步应继续检查 x86 vCPU 初始化代码是否把 `bsp_entry` 转换为正确的 VMCS guest-state。

这就是当前阶段理解配置链路所需的大方向。
