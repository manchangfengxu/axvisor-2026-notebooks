# nested OVMF ACPI 平台信息转交证据表

---

# Q1

OVMF / QEMU 之间，ACPI 通过 fw_cfg 的精确 ABI 是什么？

- [证据] `references/qemu/include/hw/acpi/aml-build.h:11-14` — 宏定义 — QEMU 暴露三个核心 ACPI fw_cfg file item：`"etc/acpi/tables"`（拼接表 blob）、`"etc/acpi/rsdp"`（RSDP）、`"etc/table-loader"`（loader 命令脚本）。
- [证据] `references/qemu/hw/i386/acpi-build.c:2207-2240` — `acpi_build()` 末尾 — 注册 fw_cfg items：行 2209 注册 `etc/acpi/tables`，行 2214 注册 `etc/table-loader`，行 2240 注册 `etc/acpi/rsdp`。
- [证据] `edk2/OvmfPkg/Include/IndustryStandard/QemuLoader.h:20-25` — `QEMU_LOADER_COMMAND_TYPE` 枚举 — table-loader 包含四种命令：`Allocate`（下载 fw_cfg file 到分配的内存）、`AddPointer`（修补相对指针为绝对地址）、`AddChecksum`（重算校验和）、`WritePointer`（写绝对地址到可写 fw_cfg file）。
- [证据] `edk2/OvmfPkg/Include/IndustryStandard/QemuLoader.h:37-41` — `QEMU_LOADER_ALLOCATE` 结构体 — 每条 Allocate 命令指定 file name、alignment、zone（High 或 FSeg）。
- [证据] `edk2/OvmfPkg/Include/IndustryStandard/QemuLoader.h:50-55` — `QEMU_LOADER_ADD_POINTER` 结构体 — 指定 PointerFile、PointeeFile、PointerOffset、PointerSize。用于修补 RSDP→RSDT/XSDT 指针、RSDT→各表指针、XSDT→各表指针。
- [证据] `edk2/OvmfPkg/Include/IndustryStandard/QemuLoader.h:93-102` — `QEMU_LOADER_ENTRY` 结构体 — 每条命令 128 字节，含 Type (u32) + 命令体 + padding[124]。
- [证据] `edk2/OvmfPkg/Library/AcpiPlatformLib/QemuFwCfgAcpi.c:1099-1143` — `InstallQemuFwCfgTables()` — OVMF 入口函数。行 1121 先用 `QemuFwCfgFindFile("etc/table-loader", ...)` 查找 loader 脚本。找不到就直接返回错误。
- [证据] `edk2/OvmfPkg/Library/AcpiPlatformLib/QemuFwCfgAcpi.c:1136-1143` — 读取 `etc/table-loader` 全部内容到 `LoaderStart`，然后逐条执行命令。
- [证据] `references/qemu/hw/acpi/aml-build.c:1810-1871` — `build_rsdp()` — RSDP 中的 `RsdtAddress` 和 `XsdtAddress` 初始为 0（占位符），后续由 table-loader 的 AddPointer 命令修补。
- [证据] `references/qemu/hw/acpi/aml-build.c:1877-1899` — `build_rsdt()` — RSDT 中的表指针也是占位符，由 AddPointer 修补。
- [证据] `references/qemu/hw/acpi/aml-build.c:1905-1927` — `build_xsdt()` — XSDT 同理。
- [证据] `edk2/OvmfPkg/AcpiPlatformDxe/EntryPoint.c:39-57` — `OnRootBridgesConnected()` — ACPI 表安装触发点：PCI root bridge 枚举完成后回调 `InstallAcpiTables()`。
- [证据] `edk2/OvmfPkg/AcpiPlatformDxe/AcpiPlatform.c:30-45` — `InstallAcpiTables()` — 对 QEMU 平台调用 `InstallQemuFwCfgTables()`。

**[结论] QEMU 通过三个 fw_cfg file item 提供 ACPI：`etc/acpi/tables`（所有表拼接成一个 blob）、`etc/acpi/rsdp`（RSDP 单独存放）、`etc/table-loader`（命令脚本）。table-loader 是一个命令数组，每条 128 字节，指示 OVMF 如何分配内存、修补指针、重算校验和。不能直接透传原始 ACPI 表——RSDP/RSDT/XSDT 中的表指针是占位符（初始为 0），必须由 table-loader 命令修补为 OVMF 实际分配的物理地址。checksum 也必须在指针修补后重算。**

---

# Q2

AxVisor 当前代码树里，是否已经存在获取 outer QEMU ACPI 的现成来源？

- [证据] `tgoskits/components/someboot/src/efi_stub/mod.rs:330-361` — `find_acpi_rsdp()` — ArceOS host 启动时从 UEFI config tables 中查找 ACPI 2.0/1.0 RSDP GUID，找到后调用 `set_rsdp(addr)` 存入 `static mut RSDP`。
- [证据] `tgoskits/components/someboot/src/acpi/mod.rs:16` — `static mut RSDP: usize = 0` — RSDP 物理地址全局存储。
- [证据] `tgoskits/components/someboot/src/acpi/mod.rs:38` — `pub fn rsdp_addr_phys() -> Option<usize>` — 公开接口，返回 host RSDP 物理地址。
- [证据] `tgoskits/platforms/somehal/src/driver.rs:10-15` — `rdrive_setup()` — 用 `someboot::rsdp_addr_phys()` 获取 RSDP，创建 `AcpiRoot::new(rsdp, phys_to_virt)`，传给 `rdrive::init(Platform::Acpi(...))`。
- [证据] `tgoskits/drivers/rdrive/src/probe/acpi.rs:50-53` — `AcpiRoot { rsdp: usize, phys_to_virt: fn(usize) -> *mut u8 }` — ACPI 解析入口结构体。
- [证据] `tgoskits/drivers/rdrive/src/probe/acpi.rs:1042` — `AcpiTables::from_rsdp(self.handler(), self.rsdp)` — 使用 `acpi` crate 解析 RSDP → XSDT → 所有表。
- [证据] `tgoskits/drivers/rdrive/src/probe/acpi.rs:1325-1336` — `read_interrupt_routing()` — 从 MADT 提取 IOAPIC 条目（id、address、gsi_base、redirection_entries）。
- [证据] `tgoskits/drivers/rdrive/src/probe/acpi.rs:1309-1323` — `read_pci_config_regions()` — 从 MCFG 提取 PCI ECAM region 列表（base_address、bus range）。
- [证据] `tgoskits/platforms/somehal/src/arch/x86_64/mod.rs:109-116` — `X86IoApic::new(info)` — 用 ACPI 解析出的 `info.address` 初始化物理 IOAPIC 硬件。行 125 打印日志：`"ACPI IOAPIC initialized: id={} base={:#x} gsi_base={} entries={}"`。
- [证据] `tgoskits/drivers/rdrive/src/probe/acpi.rs:1337-1352` — ISA IRQ override 解析 — 从 MADT 提取 interrupt_source_overrides（ISA IRQ → GSI 映射、触发模式、极性）。

**[结论] AxVisor host 运行时可以完整读取 outer QEMU 的 ACPI 表。入口链路：`someboot::find_acpi_rsdp()` → `set_rsdp()` → `someboot::rsdp_addr_phys()` → `somehal::rdrive_setup()` → `rdrive::probe::acpi::AcpiRoot` → `AcpiTables::from_rsdp()`。解析结果包括 MADT（IOAPIC 地址、GSI base、ISA IRQ override）、MCFG（ECAM region）、CPU 拓扑。`acpi` crate 提供完整的 RSDP/XSDT/FADT/MADT/MCFG 解析。但当前这些数据只用于 host 自身的驱动初始化，没有任何代码将它们转交给 nested guest。**

---

# Q3

outer QEMU 的 ACPI 描述，与 nested guest 当前可见平台是否一致？

- [证据] `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml` passthrough_devices — `["PCI MMIO", 0xe000_0000, 0xe000_0000, 0x1000_0000, 0x1]` — ECAM GPA=HPA=0xE0000000, 256MB。Q35 默认 ECAM 基址就是 0xE0000000。**一致。**
- [证据] 同文件 emu_devices — `["X86 IO APIC", 0xfec0_0000, 0x1000, 0, 0x23, []]` — emulated IOAPIC 在 GPA 0xFEC00000，GSI base 0，24 entries（0x23=35=type ID）。Q35 MADT 描述 IOAPIC 在 0xFEC00000, GSI base 0, 24 entries。**地址和参数一致，但 guest 看不到 MADT 来知道这些。**
- [证据] 同文件 passthrough_devices — `["HPET", 0xfed0_0000, 0xfed0_0000, 0x1000, 0x1]` — HPET GPA=HPA=0xFED00000。Q35 HPET 标准地址 0xFED00000。**一致。**
- [证据] 同文件 memory_regions — `[0x0000_0000, 0x0400_0000, 0x7, 0]` — 64 MB RAM。fw_cfg `etc/e820` 由 `fw_cfg.rs:388-399` `build_e820()` 构建，只包含 memory_regions 中 < 4GB 的区域。**与 Q35 默认 RAM 大小不一致**（Q35 默认 128MB-3GB+），但这是 AxVisor 的 guest 配置选择，不是 bug。
- [证据] `tgoskits/virtualization/axdevice/src/fw_cfg.rs:136-157` — `FwCfgInner::configure()` — fw_cfg 只含 signature、interface version、smp cpu count、`etc/e820`。**没有 `etc/acpi/tables`、`etc/acpi/rsdp`、`etc/table-loader`。Guest 看不到任何 ACPI 表。**
- [证据] 同文件 `build_e820()` 行 397 — `append_u32_le(&mut e820, 1)` — e820 条目 type 固定为 1（RAM）。不包含 MMIO reserved 区域（ECAM、IOAPIC、HPET、PCI Low MMIO）。
- [证据] AxVisor 代码树中无 DSDT / _PRT / PIRQ 生成代码（全树 grep 无命中）。

**[结论] 硬件地址映射层面一致：ECAM (0xE0000000)、IOAPIC (0xFEC00000)、HPET (0xFED00000)、PCI Low MMIO (0x80000000) 的 GPA 与 Q35 标准一致。但 ACPI 表层面完全缺失：nested guest 没有 MADT（不知道 IOAPIC 存在）、没有 MCFG（不知道 ECAM 地址）、没有 FADT/DSDT（不知道 PM timer、PCI IRQ 路由）、没有 RSDP（不知道 ACPI 表在哪）。fw_cfg `etc/e820` 的 type 全部为 RAM，不描述 MMIO reserved 区域。不一致项：(1) 无 ACPI 表，(2) e820 不含 MMIO reserved，(3) RAM 大小可配置但与 Q35 默认不同。**

---

# Q4

以"后续跑 nested Linux"为目标，最小需要哪些 ACPI 表？

- [证据] `edk2/OvmfPkg/Library/AcpiPlatformLib/QemuFwCfgAcpi.c:1121-1124` — `InstallQemuFwCfgTables` 先找 `etc/table-loader`，找不到直接返回错误 — **如果 nested OVMF 找不到 table-loader，就不会安装任何 ACPI 表。** 这是 OVMF 的硬性依赖。
- [证据] `references/qemu/hw/i386/acpi-build.c:1971-2079` — QEMU 生成的表列表 — FACS、DSDT、FADT、MADT、HPET、MCFG、SRAT、SLIT、SSDT 等。不是所有都是 Linux 启动必需。
- [证据] `tgoskits/drivers/rdrive/src/probe/acpi.rs:1325-1336` — `read_interrupt_routing()` 依赖 MADT — 没有 MADT，host 的 IOAPIC 发现和 ISA IRQ override 解析失败。
- [证据] `tgoskits/drivers/rdrive/src/probe/acpi.rs:1309-1323` — `read_pci_config_regions()` 依赖 MCFG — 没有 MCFG，PCI ECAM region 发现失败（但 legacy 0xCF8/0xCFC 仍可用）。

**必需表（缺一个 Linux 都起不来或功能严重受损）：**

| 表 | 为什么必需 | 缺失后果 |
|---|---|---|
| RSDP | ACPI 入口指针，所有其他表的发现依赖它 | Linux 不识别 ACPI，回退到 MP table 或无法启动 |
| XSDT/RSDT | 索引所有其他表 | 同上 |
| FADT | PM timer 地址、SCI 中断、reset 寄存器、指向 FACS 和 DSDT | Linux 无法做 PM timer calibration，无法处理 ACPI 事件 |
| FACS | ACPI 睡眠状态唤醒向量、Hardware Reduced 标志 | S3/S4 不工作，但 Linux 可以在没有 FACS 的情况下启动（graceful degradation） |
| MADT | IOAPIC 地址/GSI base、CPU APIC ID 列表、ISA IRQ override | Linux 不知道 IOAPIC 在哪，不知道有哪些 CPU，PCI IRQ 完全不工作 |
| DSDT | PCI _PRT（IRQ 路由）、系统设备描述、ACPI namespace | PCI 设备无法获得 IRQ 路由，中断不工作 |
| MCFG | PCI ECAM base address | Linux 回退到 legacy 0xCF8 config access（功能可用但性能差、不支持 PCIe extended config） |

**可后补表：**

| 表 | 为什么可以后补 | 缺失影响 |
|---|---|---|
| HPET | Linux 可以用 PIT/ACPI PM timer 代替 | 没有高精度 timer，但启动不受阻 |
| SRAT/SLIT | NUMA topology | Linux 假设 UMA，单 socket 场景无影响 |
| SSDT | 额外设备描述 | 取决于具体设备需求 |

**关键约束：不能直接透传原始 ACPI 表。** table-loader 命令（Allocate + AddPointer + AddChecksum）必须被执行。三条路径：
1. 把 QEMU 的 `etc/table-loader` + `etc/acpi/tables` + `etc/acpi/rsdp` 三个 blob 原样放入 AxVisor fw_cfg——OVMF 自己执行 table-loader。
2. AxVisor 自己从 host ACPI 解析结果重建 ACPI 表（使用 `acpi` crate 已有的 MADT/MCFG/FADT 解析能力），绕过 table-loader。
3. 在 AxVisor 内部实现一个 table-loader 执行器，读取 QEMU 的 loader 脚本并在 host 侧完成分配和修补，把结果作为已完成的表放入 fw_cfg。

**[结论] nested Linux 最小需要：RSDP + XSDT + FADT + FACS + MADT + DSDT + MCFG。MADT 和 DSDT 是 PCI/IRQ/IOAPIC 的硬性依赖——没有 MADT 就不知道 IOAPIC，没有 DSDT/_PRT 就没有 PCI IRQ 路由。MCFG 影响 PCI config access 方式但不阻塞启动。HPET/SRAT/SLIT 可后补。最短路径是把 QEMU 的三个 fw_cfg ACPI blob 原样透传（需要 AxVisor fw_cfg 的 `add_file_item` 接口，已存在但无调用方）。**

---

# 风险点

1. **table-loader 修补依赖 fw_cfg file name 精确匹配**：QEMU 的 table-loader 命令中 `Allocate` 引用的 file name（如 `"etc/acpi/tables"`、`"etc/acpi/rsdp"`）必须与 fw_cfg file directory 中注册的 name 完全一致。如果 AxVisor 用不同的 name 注册这些 blob，table-loader 会找不到文件而失败。

2. **RSDP 必须在 FSEG 内存（0xF0000-0xFFFFF）**：table-loader 的 `Allocate` 命令指定 `Zone = QemuLoaderAllocFSeg`，要求 RSDP 被分配到 FSEG 区域。AxVisor 的 nested guest 内存布局必须包含 FSEG 可用内存。当前 memory_regions 配置中 64MB RAM 起始于 0x0，FSEG 在范围内，但需要确认 0xF0000-0xFFFFF 没有被其他用途占用。

3. **e820 缺少 MMIO reserved 条目**：当前 `build_e820()` 只生成 type=1（RAM）条目。Linux 内核会用 e820 来避免将 RAM 分配到 MMIO 区域。如果 e820 不描述 MMIO reserved 区域（ECAM 0xE0000000、IOAPIC 0xFEC00000、HPET 0xFED00000），Linux 可能将 RAM 分配到这些地址导致冲突。

4. **DSDT 是 QEMU q35 特定的**：DSDT 中的 `_PRT` 方法、设备描述、PCI root bridge 定义都绑定到 q35 硬件拓扑。如果 AxVisor 的 passthrough 设备拓扑与 q35 不完全匹配（比如缺少某个设备或地址不同），DSDT 可能描述错误的平台。

5. **nested Linux 的 initrd/cmdline 不在 ACPI 范围内**：ACPI 表只解决平台发现问题。nested Linux 还需要 rootfs、initrd、kernel cmdline 等通过 boot_params 或 bootloader 协议传递的信息。

6. **`acpi` crate 的 MADT 解析能力已存在但输出格式是 `AcpiIoApic` / `AcpiRouting`**：从 host ACPI 解析结果重建 guest MADT 需要将这些结构转换为 ACPI binary table format。当前代码树中没有这个转换逻辑。

7. **fw_cfg `add_file_item` 接口存在但无调用方**：`FwCfgDevice::add_file_item()` (`fw_cfg.rs:343-345`) 是公开接口，但全树搜索无非测试调用。需要在 AxVM 初始化路径中新增调用来注册 ACPI blob。

---

# 未闭合点

- **QEMU table-loader 命令中的 file name 是否包含 `"etc/acpi/tables"` 还是更细粒度的子表名？** 需要读 `references/qemu/hw/acpi/bios-linker-loader.c` 确认 Allocate 命令引用的具体 file name。如果 QEMU 把所有表拼成一个 blob，table-loader 只引用 `"etc/acpi/tables"` 和 `"etc/acpi/rsdp"`，透传路径更简单。

- **nested OVMF 的 `PcdOvmfHostBridgePciDevId` 自动检测结果**：OVMF PEI 读 bus 0 dev 0 的 vendor ID 来判断 Q35 vs i440FX。nested 环境下读到的是 outer QEMU 的 0x29C0（Q35）。但这个检测是否在 ACPI 表安装之前完成？如果 ACPI 表中的 DSDT 引用了 Q35 特定寄存器，但 OVMF 没有正确识别 Q35 模式，会导致不一致。

- **AxVisor 的 host ACPI 解析结果能否直接重建完整的 Q35 DSDT？** `acpi` crate 解析 MADT/MCFG/FADT 没有问题，但 DSDT 是 AML bytecode，需要 `acpi` crate 的 AML 解释器（`rdrive` 中已使用 `aml` crate）来读取 namespace，但没有反向生成 AML 的能力。如果走"从 host 解析重建"路径，DSDT 可能需要硬编码 q35 模板。

- **`etc/system-states` fw_cfg item 是否被 nested OVMF 使用？** QEMU 还暴露了 `"etc/system-states"`（`references/qemu/hw/acpi/core.c:650`），描述 S3/S4/S5 状态支持。如果 nested Linux 依赖这个 item 来决定 ACPI 睡眠状态，也需要透传。

- **FACS 表的内存分配要求**：FACS 必须在 32-bit 可寻址内存内且 64 字节对齐。table-loader 的 Allocate 命令会指定 alignment 和 zone，AxVisor 的 guest 内存分配器必须满足这些约束。
