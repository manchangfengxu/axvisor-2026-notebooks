# AxVisor OVMF 设备模型重构计划

## 0. 背景知识：虚拟化中的设备访问与拦截

本节为不熟悉虚拟化的同学准备。已了解的同学可跳过。

### 0.1 地址翻译：EPT 是什么

普通操作系统中，CPU 把虚拟地址（VA）翻译成物理地址（PA），页表由 OS 管理。

在虚拟化场景中，Guest OS 以为自己在把 VA 翻译成 PA，但 CPU 实际上会再做一次翻译——把 Guest 的 PA（称为 GPA，Guest Physical Address）翻译成真正的 Host PA（HPA）。这次翻译由 VMM（AxVisor）通过 EPT（Extended Page Tables，Intel VMX 的二级页表）控制。

```text
Guest 程序看到的：  VA → GPA
CPU 实际做的：      VA → GPA → HPA
                           ↑
                    EPT 页表由 AxVisor 管理
```

AxVisor 在 EPT 页表中建立 GPA → HPA 的映射关系。Guest 访问一个 GPA 时，CPU 查 EPT 找到 HPA，然后访问真正的物理内存或设备。

### 0.2 设备穿透与模拟的三种机制

Guest 访问设备时，AxVisor 有三种处理方式，取决于设备类型和映射关系：

**穿透（passthrough）—— EPT 有映射**：AxVisor 在 EPT 里把 guest 的设备地址直接映射到外层 QEMU 的物理地址。Guest 访问时 CPU 能翻译，直接落到外层 QEMU 的设备上，AxVisor 不参与，不触发 VM exit。当前 x86 UEFI 的 PCI MMIO、IOAPIC、HPET 走这条路。

**模拟 MMIO 设备 —— EPT 无映射**：AxVisor 不在 EPT 里映射这个地址。Guest 访问时 CPU 翻译不了，触发 EPT violation（类似 page fault，但在虚拟化层），VM exit，AxVisor 收到 `NestedPageFault`，调用 `get_devices().handle_mmio_read/write()` 分发给模拟设备。后续的 vIOAPIC、PCI 配置空间等 MMIO 设备走这条路。

**模拟 port I/O 设备 —— I/O bitmap 标记**：x86 的 IN/OUT 指令访问 I/O 端口（和内存地址空间分开的独立地址空间）。AxVisor 通过 I/O bitmap 控制哪些端口触发 VM exit——bitmap 中标记的端口会被拦截，未标记的直接穿透。当前 x86 UEFI 的 debugcon（0x402）、fw_cfg（0x510）、QEMU exit（0x604）、virtio-blk I/O（0x6000）走这条路。ACPI PM（0x600）当前也走这条路，但重构后将保持穿透（见 5.6 节）。

```text
穿透设备（PCI MMIO 等）：EPT 有映射 → 不拦截 → 外层 QEMU 处理
模拟 MMIO 设备：          EPT 无映射 → EPT violation → AxVisor 拦截 → handle_mmio_read/write
模拟 port I/O 设备：      I/O bitmap 标记 → VM exit → AxVisor 拦截 → handle_port_read/write
```

### 0.3 AxVisor 设备模型的三层结构

AxVisor 的设备模型由三层 crate 组成：

**第一层：trait 定义**（`axdevice_base`）。定义 `BaseDeviceOps<R>` trait，是所有模拟设备的统一接口。`R` 是地址范围类型，对应三种设备：
- `BaseMmioDeviceOps` = `BaseDeviceOps<GuestPhysAddrRange>`（MMIO 设备）
- `BasePortDeviceOps` = `BaseDeviceOps<PortRange>`（port I/O 设备）
- `BaseSysRegDeviceOps` = `BaseDeviceOps<SysRegAddrRange>`（系统寄存器设备，仅 aarch64）

**第二层：设备注册表**（`axdevice`）。`AxVmDevices` 持有三组设备集合（`emu_mmio_devices`、`emu_port_devices`、`emu_sys_reg_devices`），提供 `handle_mmio_read/write`、`handle_port_read/write` 等分发方法。设备不在这里实现，只在这里注册和分发。

**第三层：设备实现**（独立 crate）。每个设备实现 `BaseDeviceOps` trait。aarch64 的 GIC 在 `arm_vgic` crate，riscv64 的 PLIC 在 `riscv_vplic` crate。本次重构新建的 `x86_qemu_device` crate 放 x86 QEMU 平台设备。

```text
Guest I/O 访问 → CPU 拦截（EPT violation 或 I/O bitmap）
  → AxVM::run_vcpu() 收到 exit reason
  → AxVmDevices::handle_mmio_read/write 或 handle_port_read/write
  → find_dev() 找到设备
  → 调用设备的 handle_read/write 方法（在 x86_qemu_device crate 中实现）
```

### 0.4 当前 x86 UEFI 的穿透与模拟分工

当前 OVMF 启动链路中，部分设备穿透、部分设备模拟：

| 设备 | 地址 | 类型 | 处理方式 | 原因 |
|------|------|------|---------|------|
| PCI Low MMIO | 0x8000_0000 | MMIO | 穿透 | OVMF 需要访问外层 QEMU 的 PCI 设备 |
| PCI MMIO | 0xE000_0000 | MMIO | 穿透 | 同上 |
| IO APIC | 0xFEC0_0000 | MMIO | 穿透 | 让外层 QEMU 提供中断控制器 |
| HPET | 0xFED0_0000 | MMIO | 穿透 | 让外层 QEMU 提供定时器 |
| debugcon | 0x402 | port | 模拟 | OVMF 日志输出，AxVisor 自己处理 |
| fw_cfg | 0x510-0x51B | port | 模拟 | OVMF 读取固件配置，嵌套场景下外层 QEMU 的 fw_cfg 是给 AxVisor 用的，不是给 guest 用的 |
| ACPI PM | 0x600-0x60F | port | 当前模拟，重构后穿透 | host passthrough shim，不是真正模拟设备 |
| virtio-blk I/O | 0x6000-0x607F | port | 模拟 | OVMF 读磁盘，需要 AxVisor 做 nested DMA 地址翻译 |
| QEMU exit | 0x604 | port | 模拟 | guest 关机信号，AxVisor 控制 VM 生命周期 |

穿透是临时方案——后续 roadmap 会逐步用模拟设备替换穿透设备，最终 AxVisor 可以独立运行在裸机上。

---

## 1. 背景与动机

### 1.1 当前状态

`uefi-develop-io` 分支已初步跑通 OVMF 启动链路：

```text
AxVisor → OVMF_CODE/VARS → x86 reset vector → SEC/PEI/DXE/BDS → virtio-blk → FAT32 ESP → BOOTX64.EFI
```

这条链路证明了 OVMF 固件入口、PEI 内存发现、DXE/BDS 驱动加载、virtio-blk 读盘和 PE32+ UEFI app 加载可以串起来。但这条链路的实现方式是"能跑就行"——设备模拟代码全部内联在 `vm.rs`（1690 行）中，没有遵循 AxVisor 原有的设备模型架构。

### 1.2 为什么需要重构

这条启动链路涉及 4 个模拟 port I/O 设备（debugcon、fw_cfg、virtio-blk I/O、QEMU exit）和 1 个穿透 shim（ACPI PM）。当前分布在两个文件里：`vm.rs` 持有 fw_cfg、virtio-blk I/O、ACPI PM 的完整实现；`axdevice/src/device.rs` 持有 debugcon 的实现。ACPI PM 重构后保持穿透。这两个文件同时承担了其他职责——`vm.rs` 负责 VM 生命周期管理，`device.rs` 负责设备注册表和分发。

按 `ovmf-infra-roadmap.md` 规划，后续还需要加入 PCI 主桥、virtio-pci 设备模型、vIOAPIC、i8259、ACPI 表生成、MSI/MSI-X 等大量设备。这些新设备如果继续用"内联在 vm.rs"的方式添加，每加一个设备都要同时修改 `vm.rs`（添加处理逻辑）和 `vcpu.rs`（添加 I/O bitmap 端口范围），且两处的端口范围必须手动保持同步。设备之间的联动（中断路由、PCI BAR 配置）也没有统一的注册和发现机制。

因此，在继续添加新设备之前，必须先把现有设备的架构规范好。

### 1.3 重构目标

本次重构有四个目标。第一，把内联在 `vm.rs` 中的设备模拟代码抽出为独立的设备实现。第二，通过 `AxVmDevices` 设备注册表统一注册和分发所有设备。第三，建立端口声明到 I/O bitmap 的自动联动机制——这个机制在 `dev` 分支就已规划为 TODO，但一直未完成。第四，为后续 roadmap 模块提供清晰的扩展点。

---

## 2. 穿透与模拟：当前架构的两条路径

### 2.1 为什么同时存在穿透和模拟

AxVisor 当前运行在 QEMU 内部（嵌套虚拟化场景）。Guest OVMF 访问硬件时，AxVisor 有两条路径可选。

**穿透路径**：AxVisor 不拦截 guest 的访问，让访问直接落到外层 QEMU 的设备上。OVMF 以为自己在访问真实硬件，实际上在和外层 QEMU 通信。这条路径用于 PCI 设备、IOAPIC、HPET 等——这些设备由外层 QEMU 提供，AxVisor 不需要自己实现。

**模拟路径**：AxVisor 自己处理 guest 的访问，不传给外层 QEMU。这条路径用于 fw_cfg、debugcon 等——这些设备在嵌套场景下有特殊含义：外层 QEMU 的 fw_cfg 端口是给 AxVisor 用的，不是给 guest OVMF 用的。如果 AxVisor 把 guest 的 fw_cfg 请求穿透给外层 QEMU，外层 QEMU 返回的是 AxVisor 自己的配置数据，而不是 guest OVMF 需要的数据。因此 AxVisor 必须自己模拟这些设备。

### 2.2 当前 TOML 配置中的体现

这两种路径在 TOML 配置中分别对应不同的字段。`emu_devices` 字段声明需要模拟的设备——当前 x86 UEFI 配置中该字段为空，因为 QEMU 平台设备（debugcon、fw_cfg 等）是在代码中硬编码注册的，不在 TOML 中声明。`passthrough_devices` 字段声明需要穿透的地址范围——当前 x86 UEFI 配置中包含 PCI Low MMIO、PCI MMIO、IO APIC、HPET 四个穿透条目。

```toml
# ovmf-x86_64-qemu-smp1.toml 的 [devices] 段
[devices]
interrupt_mode = "passthrough"
emu_devices = []    # 空——QEMU 平台设备不在这里声明
passthrough_devices = [
  ["PCI Low MMIO", 0x8000_0000, 0x8000_0000, 0x6000_0000, 0x1],
  ["PCI MMIO", 0xe000_0000, 0xe000_0000, 0x1000_0000, 0x1],
  ["IO APIC", 0xfec0_0000, 0xfec0_0000, 0x1000, 0x1],
  ["HPET", 0xfed0_0000, 0xfed0_0000, 0x1000, 0x1],
]
```

### 2.3 穿透是临时方案，模拟是最终目标

穿透机制让 AxVisor 不用自己实现 PCI 设备就能让 OVMF 启动。但 AxVisor 的最终目标是独立运行在裸机上，不依赖外层 QEMU。在裸机场景下没有外层 QEMU 可以穿透，所有设备必须由 AxVisor 自己模拟。

`ovmf-infra-roadmap.md` 规划了从穿透到模拟的演进路线：PCI 主桥和配置空间（模块 10）将替换 PCI MMIO 穿透；virtio-pci 设备模型（模块 11）将替换 virtio-blk I/O 穿透；vIOAPIC 将替换 IO APIC 穿透。每实现一个模拟设备，就可以减少一项穿透依赖。

本次重构不涉及穿透到模拟的替换——替换是后续 roadmap 模块的工作。本次重构只管模拟设备的架构规范化。穿透机制保持现状不动。

---

## 3. 现有架构分析

### 3.1 `dev` 分支的 crate 层次

`dev` 分支建立了一套清晰的设备模型架构。这套架构由以下 crate 组成：

```
axdevice_base     -- 定义 BaseDeviceOps trait（设备接口）
axdevice          -- 设备注册表 AxVmDevices（纯分发，不含设备实现）
arm_vgic          -- ARM 中断控制器实现（独立 crate）
riscv_vplic       -- RISC-V 中断控制器实现（独立 crate）
axvmconfig        -- TOML 配置结构和 EmulatedDeviceType 枚举
axvm              -- VM 生命周期管理
x86_vcpu          -- vCPU 管理和 VMX/SVM 退出处理
```

这套架构的设计原则是：`axdevice` 作为纯注册表和分发层，不包含任何设备实现；设备实现在各自独立的 crate 中；`AxVmDevices::init()` 从配置中实例化设备并注册到注册表中。

### 3.2 `uefi-develop-io` 分支引入的问题

`uefi-develop-io` 分支在实现 OVMF 启动链路时，没有遵循上述设计原则。具体表现为以下五个问题：

**问题一：设备代码内联在 `vm.rs` 中。** `FwCfgState` 结构体（`vm.rs:155`）、`OvmfVirtioBlkIoState` 结构体（`vm.rs:136`）、ACPI PM 处理函数（`vm.rs:1389-1442`）、debugcon 处理函数（`vm.rs:1059`）全部作为 `AxVM` 的方法直接写在 `vm.rs` 中。`run_vcpu()` 方法（`vm.rs:730`）中有一条约 160 行的 if/else 链（`vm.rs:758-822`），依次检查 debugcon（0x402）、fw_cfg（0x510）、virtio-blk（0x6000）、ACPI PM（0x600）的端口，最后才走设备注册表。这导致 `vm.rs` 膨胀到 1690 行，混合了 VM 生命周期管理和设备模拟两种不同的职责。

**问题二：`OvmfDebugConDevice` 写在 `axdevice/src/device.rs` 中。** debugcon 设备的实现（`device.rs:39-70`）被放在了设备注册表 crate 里，违反了 `axdevice` 纯注册表的设计原则。`dev` 分支的 `device.rs` 不含任何设备实现，只有注册表和分发逻辑。

**问题三：端口常量在三处重复定义。** `vm.rs:46-86` 定义了 `FW_CFG_IO_SELECTOR`（0x510）、`OVMF_VIRTIO_BLK_IO_BASE`（0x6000）、`ACPI_PM_IO_BASE`（0x600）等常量。`vcpu.rs:433` 的 `setup_io_bitmap()` 中硬编码了相同的端口范围。`device.rs:37` 定义了 `OVMF_DEBUGCON_PORT`（0x402）。这三处定义的是相同的端口号，修改时容易遗漏。

**问题四：I/O bitmap 硬编码。** `vcpu.rs:433` 的 `setup_io_bitmap()` 方法中硬编码了需要拦截的端口范围。每添加一个新设备，必须手动在这里添加对应的端口范围，否则新设备不会收到 VM exit。

**问题五：未处理端口直接 panic。** `device.rs:163` 的 `panic_device_not_found()` 函数在 guest 访问未注册端口时直接 panic，导致整个 hypervisor 崩溃。该函数在 `handle_mmio_read`（`device.rs:488`）、`handle_mmio_write`（`device.rs:503`）、`handle_port_read`、`handle_port_write` 四处被调用。

### 3.3 `dev` 分支遗留的 TODO

`dev` 分支的 `vcpu.rs:512`（当前分支 `vcpu.rs:435`）已有注释表明，原设计就打算把 I/O bitmap 和设备注册关联起来：

```rust
// Todo: these should be combined with emulated pio device management,
// in `modules/axvm/src/device/x86_64/mod.rs` somehow.
```

当时只拦截了 QEMU exit 端口（0x604），这个 TODO 一直未完成。本次重构将完成这个 TODO。

---

## 4. 设计参考

### 4.1 QEMU 的设备组织方式

QEMU 的设备源码位于 `hw/` 目录下，按设备功能分类：`hw/nvram/` 放 NVRAM 和 fw_cfg，`hw/char/` 放串口和控制台，`hw/intc/` 放中断控制器，`hw/acpi/` 放 ACPI 相关设备。每个设备一个文件——fw_cfg 在 `fw_cfg.c`，8259 PIC 在 `i8259.c`，16550 UART 在 `serial.c`。不按平台分目录。

QEMU 的设备通过 `memory_region_init_io()` 注册自己的端口或 MMIO 范围。框架根据注册信息自动路由 guest 的访问。设备不需要知道其他设备的存在。

QEMU 的设备通过 `AddressSpace *dma_as` 指针访问 guest memory。这个指针在设备创建时由 board 代码传入，设备存下来供 DMA 使用。具体到 fw_cfg 设备，`include/hw/nvram/fw_cfg.h:71-72` 定义了 `dma_addr_t dma_addr` 和 `AddressSpace *dma_as` 两个字段。`fw_cfg_init_mem_dma()` 函数（`hw/nvram/fw_cfg.c:1059`）在初始化时接收 `AddressSpace *dma_as` 参数并存入设备状态。DMA 传输时，`fw_cfg_dma_transfer()` 函数（`hw/nvram/fw_cfg.c:334`）通过 `dma_memory_read(s->dma_as, ...)` 读写 guest memory。这个模式——设备持有地址空间引用、初始化时注入、DMA 时使用——是 QEMU 设备访问 guest memory 的标准方式。

fw_cfg 的 DMA 协议、端口布局、ACPI 表结构都是 QEMU 约定的，OVMF 按 QEMU 行为实现。因此 AxVisor 的 fw_cfg 设备必须兼容 QEMU 的行为。

### 4.2 Cloud Hypervisor 的设备组织方式

Cloud Hypervisor 是一个 Rust VMM 项目，采用一个 crate 一个设备类别、crate 内一个文件一个设备的组织方式。其 `devices` crate 包含所有平台和 legacy 设备：

```
devices/src/
  legacy/
    fw_cfg.rs        -- QEMU fw_cfg（33KB）
    serial.rs        -- 16550 串口
    cmos.rs          -- CMOS/NVRAM
    i8042.rs         -- 键盘控制器
    debug_port.rs    -- 调试端口
  acpi.rs            -- ACPI PM
  ioapic.rs          -- I/O APIC
  gic.rs             -- ARM GIC
```

fw_cfg、serial、debug_port 全在同一个 `devices` crate 里，用文件区分模块。其中 fw_cfg（`devices/src/legacy/fw_cfg.rs`，33KB）是最大的一个，通过 Cargo feature flag `fw_cfg` 条件编译，不需要时可以完全排除。其他 legacy 设备如 `serial.rs`、`cmos.rs`、`i8042.rs`、`debug_port.rs` 都是几 KB 的小文件。只有 virtio 设备因数量多（block、net、console、balloon、iommu 等 11 个设备）且代码量大才单独一个 `virtio-devices` crate。

Cloud Hypervisor 的设备通过 `Arc<dyn GuestMemoryAccess>` trait object 访问 guest memory。这个 trait object 在设备创建时传入，设备存下来供 DMA 使用。这与 QEMU 的 `AddressSpace *dma_as` 模式一致。fw_cfg 设备（`devices/src/legacy/fw_cfg.rs`，33KB）被归类为 legacy 设备，和 serial、cmos、i8042 放在同一个 `devices` crate 的 `legacy/` 子模块中。fw_cfg 通过 Cargo feature flag 条件编译（`[features] fw_cfg = [...]`），不需要时可以完全排除。

### 4.3 AxVisor 现有模式

AxVisor 的 ARM 中断控制器 `arm_vgic` 是独立 crate，因为它是一个完整的子系统——包含 GICv2/v3、distributor、redistributor、ITS，代码量大，逻辑自成体系。这不是"一个设备一个 crate"的规则，而是"一个有独立边界的子系统一个 crate"。

---

## 5. 设计决策

### 5.1 新建 `x86_qemu_device` crate

**决策**：在 `components/` 下新建 `x86_qemu_device` crate，只放 QEMU 特有设备（debugcon、fw_cfg、qemu_exit、virtio_blk_io）。标准 x86 硬件（vIOAPIC、vLAPIC、PCI 主桥、i8259 等）放在各自独立的 crate 中，和 `arm_vgic`、`riscv_vplic` 平级。所有设备通过同一套 TOML emu_devices + emu_type 机制注册。

**理由**：参考 Cloud Hypervisor 的 `devices/src/legacy/` 模式——fw_cfg、serial、debug_port 等平台设备放在同一个 `devices` crate 中，用文件区分模块。QEMU x86 的这些设备（debugcon、fw_cfg、qemu_exit、virtio_blk_io）都服务于 OVMF/UEFI 启动链条，都是 QEMU 平台固有设备。ACPI PM 保持穿透，不作为模拟设备注册。除 fw_cfg（约 300 行）外，其他设备都很小（10 到 50 行），单独开 crate 过于碎片化。该 crate 与 `arm_vgic`、`riscv_vplic` 平级放在 `components/` 下。

**结构**：

```
components/x86_qemu_device/
  Cargo.toml
  src/
    lib.rs              -- 模块入口，pub use 各设备
    debugcon.rs         -- OvmfDebugConDevice（从 axdevice 搬出）
    fw_cfg.rs           -- FwCfgDevice（从 vm.rs 搬出）
    virtio_blk_io.rs    -- OvmfVirtioBlkIoDevice（从 vm.rs 搬出）
    qemu_exit.rs        -- QemuExitDevice（从 vcpu.rs 搬出）
    # ACPI PM 保持穿透，不在此 crate 中实现
```

### 5.2 扩展 `BaseDeviceOps` 支持 string I/O

**决策**：给 `BaseDeviceOps` trait（`axdevice_base/src/lib.rs:192`）增加 `handle_string_read` 和 `handle_string_write` 两个默认方法。这两个方法接收 `GuestMemoryBytes` 引用和 guest 地址，让设备自己读写 guest memory，与 DMA 路径保持一致。

**理由**：`rep insb` 和 `rep outsb` 是 x86 标准指令，fw_cfg 和 debugcon 都会用到。当前 trait 只有 `handle_read` 和 `handle_write` 两个方法（`axdevice_base/src/lib.rs:222-241`），不支持批量 I/O，这是这些设备被内联在 `vm.rs` 里的根本原因——`vm.rs:823-853` 的 `IoStringRead`/`IoStringWrite` 处理分支无法走设备注册表。

当前 `vm.rs` 的 string I/O 处理方式不一致：debugcon 和 fw_cfg data 由 axvm 读写 guest memory 后传 buffer 给设备（`vm.rs:829-853`），但 fw_cfg DMA 由设备自己持有 guest memory 引用直接读写（`vm.rs:1500`）。重构后统一为：string I/O 也传 `GuestMemoryBytes` 引用和 guest 地址给设备，设备自己读写。这样 fw_cfg 的 string read 和 DMA 逻辑一致，debugcon 也只需从 guest memory 读字节打印。

默认方法返回 `Ok(false)`（未处理），对现有所有设备实现（`Vgic`、`VGicR`、`VPlicGlobal`、`OvmfDebugConDevice`）零影响。

**已知折中**：`BaseDeviceOps<R>` 同时服务 MMIO、sysreg、port 三种地址类型。string I/O（`rep insb`/`rep outsb`）只属于 x86 port I/O，对 MMIO 和 sysreg 设备没有意义。将 string I/O 方法加到通用 base trait 上，抽象不够干净。更严谨的做法是将 `BasePortDeviceOps` 从 trait alias 升级为独立 trait，在 port 层面扩展 string I/O。但本轮为小步重构接受这个折中——默认方法对非 port 设备零影响，后续有需要时再拆分。

### 5.3 fw_cfg 设备持有 guest memory 引用

**决策**：`FwCfgDevice` 构造时接收 `Arc<dyn GuestMemoryBytes>` 引用，DMA 和 string I/O 都通过该引用访问 guest memory。

**背景**：`axaddrspace` 已有 `GuestMemoryAccessor` trait（`axaddrspace/src/memory_accessor.rs:26`），提供 `read_buffer`、`write_buffer`、`read_obj`、`write_obj` 等方法。该 trait 的核心原语是 `translate_and_get_limit()`，与 `AddrSpace::translated_byte_buffer()`（`axaddrspace/src/address_space/mod.rs:197`）做同一件事——GPA 到 host 内存的翻译。

**问题**：`GuestMemoryAccessor` 包含泛型方法（`read_obj<V: Copy>`、`write_obj<V: Copy>`、`read_volatile<V: Copy>`、`write_volatile<V: Copy>`）。带泛型方法的 trait 不是 object-safe 的，不能做 `dyn GuestMemoryAccessor`，因此 `Arc<dyn GuestMemoryAccessor>` 编译不过。

**方案**：在 `axaddrspace` 中新增一个 object-safe 的子 trait `GuestMemoryBytes`，只包含非泛型方法：

```rust
pub trait GuestMemoryBytes: Send + Sync {
    fn read_buffer(&self, gpa: GuestPhysAddr, buf: &mut [u8]) -> AxResult<()>;
    fn write_buffer(&self, gpa: GuestPhysAddr, buf: &[u8]) -> AxResult<()>;
    fn translate_and_get_limit(&self, gpa: GuestPhysAddr) -> Option<(HostPhysAddr, usize)>;
}
```

`translate_and_get_limit` 是 virtio-blk nested DMA 所需——当前 `vm.rs` 中 descriptor chain 地址翻译通过 `address_space.translated_byte_buffer()` 做 GPA→HPA 翻译。只有 read/write_buffer 不够。

优先尝试泛型实现：`impl<H: PagingHandler + Send + Sync> GuestMemoryBytes for AddrSpace<H>`。如果 axaddrspace 的 trait bound 不允许，退回在 axvm 中为 `VmGuestMemory` 实现。

**guest_mem 所有权**：`address_space` 在 `AxVMInnerMut` 里，不是 Arc。设备需要长期持有引用，因此在 axvm 中新建 wrapper：

```rust
// axvm 里新建
pub struct VmGuestMemory {
    inner: Arc<Mutex<AxVMInnerMut>>,
}

impl GuestMemoryBytes for VmGuestMemory {
    fn read_buffer(&self, gpa: GuestPhysAddr, buf: &mut [u8]) -> AxResult<()> {
        let g = self.inner.lock();
        // 用 g.address_space.translated_byte_buffer(gpa, buf.len())
    }
    fn write_buffer(&self, gpa: GuestPhysAddr, buf: &[u8]) -> AxResult<()> {
        let g = self.inner.lock();
        // 用 g.address_space.translated_byte_buffer(gpa, buf.len())
    }
}
```

`AxVM` 创建 `Arc::new(VmGuestMemory { inner: self.inner_mut.clone() })`，传给设备。设备持有 `Arc<dyn GuestMemoryBytes>`。不存在死锁风险——`run_vcpu()` 调用设备 handler 时不持有 `inner_mut` lock（当前 `handle_fw_cfg_io_read` 就是在方法内部 lock `inner_mut` 的，`vm.rs:1447`）。

设备需要读写结构体时，在内部用 byte buffer + `from_le_bytes`/`to_le_bytes` 转换，不依赖泛型 trait object。

**依赖关系**：
```
x86_qemu_device → axaddrspace（GuestMemoryBytes trait）
axvm → axaddrspace（实现 GuestMemoryBytes trait，通过 VmGuestMemory wrapper）
```

设备不依赖 `axvm` 的 `PagingHandlerImpl` 具体类型，只依赖 `axaddrspace` 的 trait。参考 QEMU 的 `AddressSpace *dma_as`（`include/hw/nvram/fw_cfg.h:72`）和 Cloud Hypervisor 的 `Arc<dyn GuestMemoryAccess>` 模式。

### 5.4 I/O bitmap 从设备注册表自动生成

**决策**：`setup_io_bitmap()` 接收 `&AxVmDevices` 参数，遍历 `iter_port_dev()` 获取所有模拟设备的端口范围，自动设置 I/O bitmap。

**理由**：所有端口拦截都是为了设备——guest 访问一个端口时 VM exit，VMM 把该访问交给对应的设备处理。设备注册表持有所有已注册设备的信息，是端口拦截的唯一真相来源。设备通过 `add_port_dev()` 注册时声明自己的端口范围（通过 `BaseDeviceOps::address_range()` 方法，`axdevice_base/src/lib.rs:203`），`setup_io_bitmap()` 从注册表读取这些范围。这完成了 `dev` 分支遗留的 TODO（`vcpu.rs:435`）。`AxVmDevices` 已有 `iter_port_dev()` 方法（`device.rs`）可以遍历所有端口设备，不需要新增接口。

**效果**：添加新设备只需两步——实现 `BasePortDeviceOps` trait 和调用 `add_port_dev()` 注册。不需要修改 `vcpu.rs` 的 I/O bitmap 设置。

### 5.5 统一 `run_vcpu()` 的 I/O 分发

**决策**：`vm.rs` 的 `run_vcpu()` 中所有 I/O exit reason 统一走 `get_devices().handle_*()` 方法，删除内联的 if/else 分支。

**理由**：`dev` 分支的 `run_vcpu()`（`vm.rs:546`）对 MMIO、port I/O、sys reg 三种 exit 已经统一走 `get_devices().handle_port_read/write()` 分发。`uefi-develop-io` 分支在调用这个分发之前插入了一段 if/else 链（`vm.rs:758-822`），先检查 debugcon、fw_cfg、virtio-blk、ACPI PM 端口，绕过了设备注册表。原因是这些设备没有注册到 `AxVmDevices` 的端口设备表里。

重构后，这些设备注册到端口设备表里，if/else 链删掉，恢复 `dev` 分支的统一分发路径。`AxVmDevices::handle_port_read/write()`（`device.rs:472`）内部通过 `find_dev()` 线性扫描找到端口范围包含目标地址的设备，然后调用该设备的 `handle_read/write` 方法。参考 QEMU 的做法：所有端口访问都通过 `address_space_rw()` 统一分发，设备的具体处理逻辑在各自的 `MemoryRegionOps` 回调中。

### 5.6 QEMU 平台设备的注册方式

**决策**：QEMU 平台设备通过 TOML 配置的 `emu_devices` 字段声明，和 aarch64 的 GIC 设备使用同一套模式。不在 `axdevice` 代码中硬编码。

**理由**：aarch64 的 GIC 设备已经在 TOML 的 `emu_devices` 中声明（`emu_type = 0x20/0x21/0x22`），`init()` 中按 `emu_type` 匹配实例化。x86 的 QEMU 设备使用同样的模式，保持跨架构一致性。TOML 声明比硬编码更灵活——用户可以在配置文件中增减设备，不需要改代码；配置文件能直接看到 VM 有哪些设备，更透明。

**具体做法**：
1. 在 `axvmconfig` 的 `EmulatedDeviceType` 枚举中增加 x86 设备类型（0x40..0x43）：`Debugcon`、`FwCfg`、`QemuExit`、`OvmfVirtioBlkIo`
2. 在 `axdevice/src/device.rs` 的 `init()` 中增加对应的 match 分支，从 `x86_qemu_device` crate 引入设备并实例化
3. 在 TOML 配置文件中声明设备：

```toml
# ovmf-x86_64-qemu-smp1.toml
emu_devices = [
    ["debugcon",       0x402,  0x1,  0, 0x40, []],
    ["fw-cfg",         0x510,  0xC,  0, 0x41, []],
    ["qemu-exit",      0x604,  0x4,  0, 0x42, []],
    ["ovmf-virtio-blk",0x6000, 0x80, 0, 0x43, []],
]
```

emu_type 使用 0x40..0x43（`EmulatedDeviceType` 是 `#[repr(u8)]`，最大 0xFF）。端口范围在 TOML 中声明，设备构造函数从 config 读取。

ACPI PM（0x600..0x60F）不放入 emu_devices，保持穿透。原因：ACPI PM 当前是 host passthrough shim，不是真正模拟设备；且其端口范围与 QEMU exit（0x604）重叠——`find_dev()` 返回第一个匹配设备，如果 ACPI PM 排在前面，QemuExitDevice 永远不会被命中。

debugcon 和 qemu-exit 应出现在所有 x86 QEMU 配置中——当前 vcpu.rs 硬编码 0x604 对所有 x86 guest 生效，重构后如果 BIOS 配置不声明 qemu-exit，这些 guest 会丢失关机能力。

**guest_mem、shutdown_flag、fw_cfg_info 的传递**：这三个运行时依赖无法在 TOML 中声明。通过 `AxVmDevices::new(config, guest_mem, shutdown_flag, fw_cfg_info)` 传入。`fw_cfg_info` 是轻量 struct（定义在 axvmconfig），包含 memory_regions 和 cpu_count，用于 fw_cfg 构建 e820 表。不传 `AxVMConfig`（定义在 axvm，会导致 axdevice → axvm 循环依赖）。

### 5.7 未注册端口的处理策略

**决策**：`AxVmDevices` 的 `handle_port_read/write`（`device.rs:472`）在设备未找到时，不再调用 `panic_device_not_found()`（`device.rs:163`）。具体策略：
- **port read**：返回默认值（字节 0xFF、字 0xFFFF、双字 0xFFFFFFFF），与 QEMU 行为一致
- **port write**：忽略 + 打一条 warn 日志
- **不注入异常**：注入 `#GP` 需要修改 vCPU 状态，设备层不应有这个能力

**理由**：hypervisor 不应该因为 guest 访问了一个未注册的端口就崩溃。QEMU 对未注册端口返回全 1（`hw/core/unassigned-device.c`），guest 驱动通常能处理这个值。返回 `AxError` 不够——`run_vcpu()` 里的 `?` 传播会让 VM 错误退出，效果和 panic 类似，只是从崩溃变成了错误退出，没有真正解决问题。

---

## 6. 完整运行时流程

本节以 guest OVMF 访问 fw_cfg 端口为例，展示重构后从启动到 I/O 处理的完整流程。

### 6.1 阶段一：配置解析

AxVisor 启动时读取 TOML 配置文件 `ovmf-x86_64-qemu-smp1.toml`。`axvmconfig` crate 将 TOML 解析为 `AxVMCrateConfig` 结构体。该结构体包含三个段：`[base]` 段提供 VM 基本信息（id、cpu_num），`[kernel]` 段提供 UEFI 启动参数（OVMF 路径、内存布局），`[devices]` 段提供设备配置。

`[devices]` 段中的 `emu_devices` 字段声明了 QEMU 平台模拟设备（debugcon、fw_cfg、qemu_exit、ovmf-virtio-blk），每个设备指定名称、地址范围和 emu_type。ACPI PM 不在其中——保持穿透。`passthrough_devices` 字段包含四个穿透条目（PCI Low MMIO、PCI MMIO、IO APIC、HPET），这些地址范围将被标记为穿透，guest 访问时直接落到外层 QEMU。BIOS 启动配置的 `emu_devices` 不包含 fw_cfg 和 ovmf-virtio-blk。

`axvm` crate 将 `AxVMCrateConfig` 转换为运行时配置 `AxVMConfig`。该配置包含 `OvmfInfo`（OVMF 代码和变量区的 GPA 地址）和 `VMImageConfig`（内核、BIOS、OVMF 的加载地址）。

### 6.2 阶段二：设备注册

`AxVmDevices::new(config, guest_mem, shutdown_flag, fw_cfg_info)` 被调用，进入 `init()` 方法。该方法遍历 `emu_devices` 列表，按 `emu_type` 匹配设备类型并实例化。然后遍历 `passthrough_devices` 列表，将每个穿透条目的地址范围注册到穿透表中。

每个设备从 `x86_qemu_device` crate 引入，构造时使用 config 中的 `base_ipa` 和 `length` 作为端口范围，通过 `add_port_dev()` 注册。以 OVMF 配置为例：

- `OvmfDebugConDevice`（emu_type=0x40）注册端口 0x402
- `FwCfgDevice`（emu_type=0x41）注册端口 0x510 到 0x51C
- `QemuExitDevice`（emu_type=0x42）注册端口 0x604
- `OvmfVirtioBlkIoDevice`（emu_type=0x43）注册端口 0x6000 到 0x6080
- ACPI PM（0x600..0x60F）不注册，保持穿透

BIOS 启动配置的 `emu_devices` 不包含 FwCfg 和 OvmfVirtioBlkIo，因此不会注册这些设备。每个设备在注册时声明自己需要拦截的端口范围，这些端口范围随后会被用于 I/O bitmap 的自动配置。

### 6.3 阶段三：vCPU 创建与 I/O bitmap 配置

vCPU 在 `AxVmDevices` 之前创建（`vm.rs:411`），因此不能在创建时传入设备引用。方案：在 `AxVmDevices` 创建后（`vm.rs:508`），调用 `refresh_io_bitmap(devices)` 将注册的端口范围写入已创建的 vCPU 的 I/O bitmap 中。

配置完成后，I/O bitmap 中只有注册了模拟设备的端口才会触发 VM exit。未注册的端口保持 passthrough 状态，guest 访问时不会被 AxVisor 拦截。

### 6.4 阶段四：运行时 I/O 处理

OVMF 运行后，当它执行 `OUT` 指令访问端口 0x510（fw_cfg selector）时，CPU 查询 I/O bitmap，发现该端口需要拦截，触发 VM exit。`VmxVcpu::run()` 返回 `AxVCpuExitReason::IoWrite { port: 0x510, width: Word, data: 0x0005 }`。

`AxVM::run_vcpu()` 收到该 exit reason，调用 `self.get_devices().handle_port_write(port, width, data)`。`AxVmDevices` 的 `find_dev(0x510)` 方法在端口设备注册表中找到 `FwCfgDevice`，调用其 `handle_write()` 方法。`FwCfgDevice` 识别出 0x510 是 selector 端口，将 `data`（0x0005，即 `QEMU_FW_CFG_ITEM_SMP_CPU_COUNT`）存入内部状态，设置当前 item。处理完成后返回 `Ok(())`，`run_vcpu()` 继续运行 guest。

当 OVMF 随后执行 `IN` 指令读取端口 0x511（fw_cfg data）时，同样的流程再次触发。`FwCfgDevice` 的 `handle_read()` 方法从当前 item（SMP CPU count）中读取数据，返回值 1（单 vCPU）。该值被写入 guest 的 EAX 寄存器。

当 OVMF 执行 `REP INSB` 批量读取 fw_cfg data 时，CPU 触发 string I/O exit。`run_vcpu()` 调用 `get_devices().handle_string_read(port, guest_mem, dst_gpa, count)`。`FwCfgDevice` 的 `handle_string_read()` 方法通过 `GuestMemoryBytes` 引用将当前 item 的 `count` 字节写入 guest memory 目标地址。

当 OVMF 访问 PCI MMIO 地址 0x81000000 时，CPU 查询 EPT，发现该地址属于穿透范围，不触发 exit，访问直接落到外层 QEMU 的 PCI 设备上。外层 QEMU 处理后返回结果，guest 继续运行。

### 6.5 流程总结

```text
TOML 配置
  ↓ axvmconfig 解析
AxVMCrateConfig
  ↓ axvm 转换
AxVMConfig + AxVmDevices
  ↓ axdevice init()
  ├─ 注册穿透地址范围（PCI MMIO 等，来自 TOML）
  └─ 注册模拟设备（fw_cfg 等，来自 x86_qemu_device crate）
       ↓
     I/O bitmap 从注册表自动生成
       ↓
Guest I/O 访问 → CPU 查询 bitmap/EPT
  ├─ 穿透地址 → 直接落到外层 QEMU
  └─ 模拟端口 → VM exit → run_vcpu() → AxVmDevices 分发 → 具体设备处理
```

---

## 7. 详细实施计划

### Phase 1: trait 扩展和 object-safe 抽象

**文件**: `components/axdevice_base/src/lib.rs`、`components/axaddrspace/src/memory_accessor.rs`

**修改内容 A**：在 `axaddrspace` 中新增 `GuestMemoryBytes` object-safe 子 trait：

```rust
pub trait GuestMemoryBytes: Send + Sync {
    fn read_buffer(&self, gpa: GuestPhysAddr, buf: &mut [u8]) -> AxResult<()>;
    fn write_buffer(&self, gpa: GuestPhysAddr, buf: &[u8]) -> AxResult<()>;
    fn translate_and_get_limit(&self, gpa: GuestPhysAddr) -> Option<(HostPhysAddr, usize)>;
}
```

优先尝试泛型实现：`impl<H: PagingHandler + Send + Sync> GuestMemoryBytes for AddrSpace<H>`（用已有的 `translated_byte_buffer` 和 `translate_and_get_limit`）。如果 axaddrspace 的 trait bound 不允许，退回在 axvm 中为 `VmGuestMemory` 实现。

**修改内容 B**：给 `BaseDeviceOps` 增加两个默认方法：

```rust
fn handle_string_read(
    &self,
    _addr: R::Addr,
    _guest_mem: &dyn GuestMemoryBytes,
    _dst_gpa: GuestPhysAddr,
    _count: usize,
) -> AxResult<bool> {
    Ok(false)  // 默认不支持
}
fn handle_string_write(
    &self,
    _addr: R::Addr,
    _guest_mem: &dyn GuestMemoryBytes,
    _src_gpa: GuestPhysAddr,
    _count: usize,
) -> AxResult<bool> {
    Ok(false)  // 默认不支持
}
```

**影响范围**：零。默认方法对所有现有设备实现透明。

### Phase 2: 建立参数传递链路

**文件**: `components/axvm/src/vm.rs`、`components/axdevice/src/device.rs`

**修改内容**：
1. 在 axvm 中新建 `VmGuestMemory` wrapper（实现 `GuestMemoryBytes`，持有 `Arc<Mutex<AxVMInnerMut>>`）
2. `AxVM` 创建 `Arc<AtomicBool>` 作为 per-VM shutdown flag
3. 在 `axvmconfig` 中定义轻量平台信息 struct：`FwCfgPlatformInfo { memory_regions, cpu_count }`
4. `AxVmDevices::new()` 签名扩展为 `new(config, guest_mem, shutdown_flag, fw_cfg_info)`
5. `axvm/src/vm.rs` 的 VM 创建流程中，将 `guest_mem`、`shutdown_flag`、`fw_cfg_info`（从 AxVMConfig 提取）传入 `AxVmDevices::new()`
5. `run_vcpu()` 保留 shutdown flag 引用，循环末尾检查
6. `AxVmDevices::init()` 暂时不改注册逻辑——只接收新参数，注册逻辑在 Phase 4 改

**目的**：先打通数据流，确保设备注册所需的参数能从 VM 创建层流到设备注册层。后续搬出设备时可以直接使用这些参数。注册逻辑的变更集中在 Phase 4，不和本 Phase 重叠。

### Phase 3: 创建 `x86_qemu_device` crate，搬出设备

#### 3.1 创建 crate 骨架

**新文件**: `components/x86_qemu_device/Cargo.toml`, `src/lib.rs`

依赖：`axdevice_base`、`axaddrspace`、`ax_errno`、`spin`、`log`

#### 3.2 搬出 debugcon

**来源**: `components/axdevice/src/device.rs` 的 `OvmfDebugConDevice`（约 30 行）
**目标**: `components/x86_qemu_device/src/debugcon.rs`
**修改**: 实现 `handle_string_write` 方法，通过 `GuestMemoryBytes` 引用从 guest memory 读取字节并打印（支持 `rep outsb`）

#### 3.3 搬出 fw_cfg

**来源**: `components/axvm/src/vm.rs` 的 `FwCfgState`（约 155-310 行）、`handle_fw_cfg_io_read/write`（约 1442-1500 行）、`handle_fw_cfg_dma`（约 1500-1692 行）
**目标**: `components/x86_qemu_device/src/fw_cfg.rs`
**修改**:
- `FwCfgDevice` 结构体持有 `Mutex<FwCfgState>` + `Arc<dyn GuestMemoryBytes>`
- 实现 `BasePortDeviceOps`，端口范围 0x510 到 0x51C
- 实现 `handle_string_read`（支持 `rep insb` 读取 fw_cfg data）
- 从 `vm.rs` 删除所有 fw_cfg 相关常量、结构体、方法

#### 3.4 搬出 virtio-blk I/O

**来源**: `components/axvm/src/vm.rs` 的 `OvmfVirtioBlkIoState`（约 136-140 行）、`handle_ovmf_virtio_blk_io_read/write`（约 1174-1389 行）
**目标**: `components/x86_qemu_device/src/virtio_blk_io.rs`
**修改**:
- `OvmfVirtioBlkIoDevice` 持有 guest memory 引用（用于 descriptor chain 地址翻译）
- 实现 `BasePortDeviceOps`，端口范围 0x6000 到 0x6080
- 从 `vm.rs` 删除相关代码

#### 3.5 ACPI PM 保持穿透（不搬出）

ACPI PM（0x600..0x60F）当前是 host passthrough shim，不是真正模拟设备。且其端口范围与 QEMU exit（0x604）重叠。保持穿透，不放入 emu_devices，不搬出到 x86_qemu_device。从 `vm.rs` 中删除 `handle_acpi_pm_io_read/write` 方法和 `ACPI_PM_IO_*` 常量——穿透逻辑由 EPT 映射自动处理，不需要代码。

#### 3.6 创建 QEMU exit 设备

**来源**: `components/x86_vcpu/src/vmx/vcpu.rs` 的 0x604 端口处理（当前在 `run()` 方法中直接匹配端口返回 `SystemDown`）
**目标**: `components/x86_qemu_device/src/qemu_exit.rs`

**背景**：0x604 是 QEMU 的 `isa-debug-exit` 端口。QEMU 的实现（`hw/misc/debugexit.c`）在 guest 写入时直接调用 `exit(val)` 退出进程，因为 QEMU 是单 VM 进程模型。AxVisor 是多 VM 模型——一个 hypervisor 运行多个 VM，不能用 `exit()` 杀掉整个 hypervisor。因此需要一种机制只停一个 VM，不影响其他 VM 和 hypervisor 本身。

**方案**：用 per-VM 原子标志，不修改 `BaseDeviceOps` trait。

```rust
struct QemuExitDevice {
    shutdown_requested: Arc<AtomicBool>,
}

impl BasePortDeviceOps for QemuExitDevice {
    fn handle_write(&self, _addr: Port, width: AccessWidth, val: usize) -> AxResult {
        // 保持与当前 vcpu.rs 行为一致：只在 width == Word 且 magic 值匹配时关机
        if width == AccessWidth::Word && val as u32 == QEMU_EXIT_MAGIC {
            self.shutdown_requested.store(true, Ordering::Release);
        }
        Ok(())  // trait 返回类型不变
    }
}
```

**标志归属**：`Arc<AtomicBool>` 由 `AxVM` 在创建时分配，属于 VM 实例级别（不是全局静态）。注册 `QemuExitDevice` 时 clone 进去。`run_vcpu()` 检查同一个 `Arc<AtomicBool>`。这样多 VM 场景下，一个 guest 写 0x604 只停自己的 VM，不影响其他 VM。

```rust
// AxVM 创建时
let shutdown_flag = Arc::new(AtomicBool::new(false));
let qemu_exit = QemuExitDevice::new(shutdown_flag.clone());
devices.add_port_dev(Arc::new(qemu_exit));

// run_vcpu() 循环中
if shutdown_flag.load(Ordering::Acquire) {
    break AxVCpuExitReason::SystemDown;
}
```

**搬出语义的精确表述**：vcpu.rs 的 `run()` 方法仍然负责 VM exit 解码——0x604 端口的 `OUT` 指令触发 VM exit，`run()` 返回 `AxVCpuExitReason::IoWrite { port: 0x604, ... }`。这一步不变，因为 exit 解码是 vCPU 层的职责。搬出的是**语义判断**——vcpu.rs 不再匹配 `port == 0x604` 返回 `SystemDown`，而是让它走普通 `IoWrite` 路径。设备层的 `QemuExitDevice::handle_write()` 把这个 port write 解释为 shutdown flag。

**效果**：
- `BaseDeviceOps` trait 不改，所有现有设备零影响
- 0x604 是一个普通 port device，QEMU 知识全在 `x86_qemu_device` 里
- vCPU exit 解码层不包含 QEMU 语义判断（只做通用 exit 解码）
- 关机语义正确：per-VM flag 确保只有目标 VM 停止

#### 3.7 清理 `axdevice/src/device.rs`

**修改**: 将 `OvmfDebugConDevice` 从 `axdevice` 删除，改为从 `x86_qemu_device` 引入

### Phase 4: 设备注册统一化

**前提**：Phase 3 已完成设备搬出。本 Phase 注册设备。

**文件**: `components/axdevice/src/device.rs`、`components/axvmconfig/src/lib.rs`

**修改 `EmulatedDeviceType` 枚举**：增加 0x40..0x43（Debugcon、FwCfg、QemuExit、OvmfVirtioBlkIo）。同时更新 Display、from_usize()、Enumerable 和测试。ACPI PM 不加入。

**修改 `init()` 注册逻辑**：遍历 emu_configs，按 emu_type 匹配，从 x86_qemu_device 引入设备并实例化。设备构造函数接收 config 的 base_ipa/length 和运行时依赖。

**修改 TOML 配置**：emu_devices 声明 QEMU 设备。ACPI PM 不放入。

**更新测试**：`AxVmDevices::new(config, None, None, None)`。

### Phase 5: 清理 `vm.rs`

**前提**：Phase 4 已完成设备注册，设备表中有设备。本 Phase 删除 inline handler。

**文件**: `components/axvm/src/vm.rs`、`components/x86_vcpu/src/vmx/vcpu.rs`

**删除 vm.rs 中的 inline 代码**：
- `FwCfgState`、`OvmfVirtioBlkIoState` 结构体
- `handle_fw_cfg_*`、`handle_ovmf_virtio_blk_*`、`handle_acpi_pm_*`、`debugcon_write_bytes` 方法
- 所有 `FW_CFG_*`、`OVMF_VIRTIO_*`、`ACPI_PM_*` 常量
- `AxVMInnerMut` 中的 `fw_cfg`、`ovmf_virtio_blk` 字段

**简化 run_vcpu()**：
- IoRead/IoWrite：删除 if/else 链，统一走 get_devices().handle_port_read/write()
- IoStringRead/IoStringWrite：统一走 get_devices().handle_string_read/write()
- 循环末尾增加 shutdown_requested 标志检查

**修改 vcpu.rs**：删除 0x604 的 SystemDown 匹配逻辑。

**目标**：vm.rs 从约 1690 行缩减到约 800 行。

### Phase 6: I/O bitmap 自动联动

**文件**: `components/x86_vcpu/src/vmx/vcpu.rs`

**vCPU 生命周期约束**：vCPU 在 AxVmDevices 之前创建。方案：AxVmDevices 创建后调用 `refresh_io_bitmap(devices)` 刷新。

**修改**：
1. `setup_io_bitmap()` 删除硬编码端口范围
2. 新增 `refresh_io_bitmap(&mut self, devices: &AxVmDevices)` 方法
3. vm.rs 在 AxVmDevices::new() 之后、vcpu.run() 之前调用 refresh_io_bitmap()

**文件**: `components/axdevice/src/device.rs`、`components/axvmconfig/src/lib.rs`

**前提**：Phase 2 已完成 `AxVmDevices::new()` 签名扩展和参数传递。本 Phase 改 `init()` 的注册逻辑和 `EmulatedDeviceType` 枚举。

（此 Phase 内容已合并到 Phase 4）

### Phase 7: 回归验证

**验证**：
1. `cargo build --target x86_64-unknown-none` 通过
2. `cargo fmt --check` 通过
3. `cargo xtask clippy --package axdevice --package x86_qemu_device --package axvm` 通过
4. `make ovmf-run` — OVMF 走到 BDS、加载 BOOTX64.EFI
5. `arceos-x86_64-qemu-smp1.toml` 回归正常
6. `vm.rs` 行数 < 900

---

## 8. 涉及修改的文件清单

| 文件 | 修改类型 | Phase | 说明 |
|------|---------|-------|------|
| `components/axdevice_base/src/lib.rs` | 修改 | 1 | 扩展 trait 增加 string I/O 方法 |
| `components/axaddrspace/src/memory_accessor.rs` | 修改 | 1 | 新增 `GuestMemoryBytes` 子 trait 并为 `AddrSpace` 实现 |
| `components/axvm/src/vm.rs` | 修改 | 2, 5 | 新建 `VmGuestMemory` wrapper，扩展签名，删 inline handler |
| `components/axvmconfig/src/lib.rs` | 修改 | 2, 4 | 新增 `FwCfgPlatformInfo`；增加 EmulatedDeviceType 变体 0x40..0x43 |
| `components/axdevice/src/device.rs` | 修改 | 2, 6 | 扩展 `new()` 签名，修改 `init()` 注册逻辑，删除 debugcon |
| `components/axdevice/src/lib.rs` | 修改 | 6 | 导出新接口 |
| `components/x86_qemu_device/Cargo.toml` | 新建 | 3 | 新 crate 配置 |
| `components/x86_qemu_device/src/lib.rs` | 新建 | 3 | 模块入口 |
| `components/x86_qemu_device/src/debugcon.rs` | 新建 | 3 | 从 axdevice 搬出 |
| `components/x86_qemu_device/src/fw_cfg.rs` | 新建 | 3 | 从 vm.rs 搬出 |
| `components/x86_qemu_device/src/virtio_blk_io.rs` | 新建 | 3 | 从 vm.rs 搬出 |
| `components/axdevice/tests/test.rs` | 修改 | 4 | 适配 new(config, None, None, None) |
| `components/x86_qemu_device/src/qemu_exit.rs` | 新建 | 3 | 从 vcpu.rs 搬出 |
| `components/axvm/src/vm.rs` | 重构 | 4 | 删除约 500 行内联代码，简化 run_vcpu() |
| `components/axvm/Cargo.toml` | 修改 | 3 | 增加 x86_qemu_device 依赖 |
| `components/x86_vcpu/src/vmx/vcpu.rs` | 修改 | 5, 6 | 删除 0x604 SystemDown 匹配，新增 refresh_io_bitmap() |
| `components/x86_vcpu/Cargo.toml` | 修改 | 5 | 增加 axdevice 依赖 |
| `components/axvmconfig/src/lib.rs` | 修改 | 2, 4 | 新增 `FwCfgPlatformInfo`；增加 EmulatedDeviceType 变体 0x40..0x43 |
| `os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` | 修改 | 6 | 在 `emu_devices` 中声明 QEMU 设备 |

---

## 9. 验证方案

1. **编译验证**：`cargo build --target x86_64-unknown-none` 通过（axvm、axdevice、x86_qemu_device 均编译成功）
2. **功能验证**：`make ovmf-run` 启动 AxVisor + OVMF，确认 OVMF 能走到 BDS、加载 BOOTX64.EFI(这我自己的Makefile文件)
3. **回归验证**：非 UEFI 配置（`arceos-x86_64-qemu-smp1.toml`）仍然正常启动
4. **代码检查**：`vm.rs` 行数 < 900，`axdevice/src/device.rs` 不含设备实现代码

---

## 10. 认领顺序

| 顺序 | 任务 | Phase | 依赖 | 预估工作量 | 说明 |
|------|------|-------|------|-----------|------|
| 1 | 新增 GuestMemoryBytes 子 trait + 为 AddrSpace 实现 | 1 | 无 | 小 | object-safe 的 guest memory 抽象 |
| 2 | 扩展 BaseDeviceOps 增加 string I/O 方法 | 1 | 1 | 小 | 加两个默认方法，零影响 |
| 3 | 建立参数传递链路 | 2 | 1, 2 | 中 | VmGuestMemory wrapper，FwCfgPlatformInfo，AxVmDevices::new(config, guest_mem, shutdown_flag, fw_cfg_info) |
| 4 | 创建 x86_qemu_device crate 骨架 | 3.1 | 3 | 小 | Cargo.toml + lib.rs |
| 5 | 搬出 debugcon 到 x86_qemu_device | 3.2, 3.7 | 4 | 小 | 约 30 行，最简单的设备 |
| 6 | 搬出 fw_cfg 到 x86_qemu_device | 3.3 | 4 | 中 | 约 300 行，持有 GuestMemoryBytes 引用 |
| 7 | 搬出 virtio-blk I/O 到 x86_qemu_device | 3.4 | 4 | 中 | 约 200 行，含 DMA 地址翻译 |
| 8 | ACPI PM 保持穿透 | 3.5 | 4 | 小 | 从 vm.rs 删除 inline handler，穿透由 EPT 自动处理 |
| 9 | 创建 QemuExitDevice | 3.6 | 4 | 小 | 约 10 行，持有 per-VM shutdown flag |
| 10 | 设备注册统一化 | 4 | 5,6,7,9 | 中 | EmulatedDeviceType 增加 0x40..0x43，TOML 声明设备，init() 注册 |
| 11 | 清理 vm.rs，删除 inline handler | 5 | 10 | 中 | 先注册再删，确保设备表非空 |
| 12 | I/O bitmap 自动联动 | 6 | 11 | 小 | refresh_io_bitmap() 从设备表读取端口范围 |
| 13 | 回归验证 | 7 | 12 | 小 | BIOS 和 UEFI 配置分别验证 |

**关键路径**：1 → 2 → 3 → 4 → 5/6/7/9（可并行） → 10 → 11 → 12 → 13

Phase 1 到 Phase 5 是核心重构（trait → 参数传递 → 设备搬出 → 注册 → 清理），必须一起完成才能编译通过。Phase 6 到 Phase 7 是联动和验证。

---

## 11. 后续 roadmap 扩展点

本次重构完成后，后续 roadmap 模块的扩展方式：

| roadmap 模块 | 扩展位置 | 说明 | 参考 |
|-------------|---------|------|------|
| PCI 配置空间（模块 10） | `x86_qemu_device/src/pci_config.rs` | ECAM/PIO 端口设备，将替换 PCI MMIO 穿透 | QEMU `hw/pci/pci.c` 的配置空间读写逻辑；Cloud Hypervisor `vm-device/src/bus.rs` 的 BusDevice trait |
| vIOAPIC（中断链路） | `x86_qemu_device/src/ioapic.rs` | MMIO 设备，将替换 IO APIC 穿透 | QEMU `hw/intc/ioapic.c`；Cloud Hypervisor `devices/src/ioapic.rs` |
| i8259（中断链路） | `x86_qemu_device/src/i8259.rs` | port 设备 | QEMU `hw/intc/i8259.c` |
| virtio-blk device model（模块 11） | 新 crate `axdevice_virtio/` | 将替换 virtio-blk I/O 穿透和 nested DMA hack | Cloud Hypervisor `virtio-devices/src/block.rs` 和 `virtio-devices/src/transport/pci_device.rs`；QEMU `hw/block/virtio-blk.c` 和 `hw/virtio/virtio-pci.c` |
| virtio-net / virtio-console | `axdevice_virtio/` | 同上 | Cloud Hypervisor `virtio-devices/src/net.rs`、`virtio-devices/src/console.rs` |
| ACPI 表生成 | `x86_qemu_device/src/acpi_tables.rs` | 生成 RSDP/XSDT/FADT/MADT/MCFG | QEMU `hw/acpi/aml-build.c`（95KB，ACPI 表编译器）；Cloud Hypervisor `vmm/src/acpi.rs`（43KB） |
| MSI/MSI-X（中断链路） | 扩展 `BaseDeviceOps` 或 vLAPIC | 中期目标 | QEMU `hw/pci/msix.c` |

每个模块的扩展方式都是：在 `x86_qemu_device`（或新的 `axdevice_virtio`）crate 中实现 `BasePortDeviceOps` 或 `BaseMmioDeviceOps` trait，然后在 `axdevice/src/device.rs` 的 `init()` 中注册。I/O bitmap 和 `run_vcpu()` 分发不需要修改。

每个模拟设备的实现都意味着减少一项穿透依赖。当所有穿透设备都被模拟设备替换后，AxVisor 就可以在裸机上独立运行，不再依赖外层 QEMU。
