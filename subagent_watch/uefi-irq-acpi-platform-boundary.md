# nested OVMF IRQ/ACPI 平台信息边界证据表

---

# 问题 A

fw_cfg file directory 里是否存在 ACPI / table-loader / SMBIOS / IRQ platform info 相关表项？

- [证据] `tgoskits/virtualization/axdevice/src/fw_cfg.rs:136-157` — `FwCfgInner::configure()` — 此函数构建 fw_cfg 初始内容。只插入 3 个 `known_items`（signature、interface version、smp cpu count）和 1 个 `file_items`（`"etc/e820"`）。没有任何 ACPI/table-loader/SMBIOS/IRQ 相关 item。
- [证据] `tgoskits/virtualization/axdevice/src/fw_cfg.rs:151-155` — `file_items.push(FwCfgFileItem { name: "etc/e820", ... })` — `configure()` 唯一写入的 file item 是 `"etc/e820"`。
- [证据] `tgoskits/virtualization/axdevice/src/fw_cfg.rs:343-345` — `FwCfgDevice::add_file_item()` — 公开接口，可动态追加 file item。但全树搜索无任何非测试调用方。
- [证据] `tgoskits/virtualization/axvm/src/vm.rs:428-430` — `self.inner_const().devices.x86_fw_cfg_configure(&memory_regions, fw_cfg_cpu_count)` — AxVM 初始化时只调用 `configure()`，不调用 `add_file_item()`。
- [证据] `tgoskits/virtualization/axdevice/src/device.rs:558-560` — `AxVmDevices::x86_fw_cfg_configure()` — 直接转发到 `fw_cfg.configure(memory_regions, cpu_count)`，无额外追加。
- [证据] 全树 `grep add_file_item` 非测试命中为零（只在 `fw_cfg.rs:344` 定义和 `fw_cfg.rs:483` 测试中出现）。

**[结论] fw_cfg file directory 只含 `etc/e820`，不含任何 ACPI/MADT/RSDP/table-loader/SMBIOS/IRQ 路由表项。`add_file_item` 接口存在但无调用方。**

---

# 问题 B

UEFI 路径是否有意不提供 MP table / ACPI / IRQ platform info？证据在哪里？

- [证据] `tgoskits/os/axvisor/src/images/mod.rs:330-371` — `ImageLoader::load_uefi_ovmf_images()` — UEFI 路径只做两件事：(1) 加载 OVMF_CODE 到 `ovmf_code_gpa`，(2) 加载 OVMF_VARS 到 `ovmf_vars_gpa`。不加载 MP table、不加载 boot_params、不构建 ACPI 表、不调用 `x86_mptable::build()`。
- [证据] `tgoskits/os/axvisor/src/images/mod.rs:497-504` — `load_x86_linux_layout()` — MP table **仅**在此函数中构建和加载：`let mp_table = x86_mptable::build();` + `load_vm_image_from_memory(&mp_table, x86_mptable::MP_TABLE_GPA.into(), ...)`。此函数只被 Linux 直接启动路径调用。
- [证据] `tgoskits/os/axvisor/src/images/mod.rs:707-731` — `fs::load_vm_images_from_filesystem()` — UEFI 分支（行 723-730）调用 `load_uefi_ovmf_images()` + `load_virtio_blk_disk_from_filesystem()` 后直接 `return Ok(())`。不走 `load_x86_linux_images_from_filesystem()`，不走 `load_x86_linux_layout()`，不走 `build_x86_boot_params()`。
- [证据] `tgoskits/os/axvisor/src/images/mod.rs:495-496` — `let boot_params = self.build_x86_boot_params(header, layout, kernel)?;` — `boot_params`（含 e820 mmap、command line、reserved ranges）也只在 Linux 路径构建。UEFI 路径不构建。
- [证据] `tgoskits/os/axvisor/src/images/mod.rs:584-592` — `build_x86_boot_params()` 中 `builder.add_reserved_range(x86_mptable::reserved_range())` — MP table 的 reserved range 也只在 Linux 路径的 boot_params 中注册。
- [证据] `tgoskits/virtualization/axdevice/src/fw_cfg.rs:151-155` — `configure()` 中 `etc/e820` 是唯一 file item — UEFI 路径通过 fw_cfg 提供的平台信息只有 e820 内存映射和 CPU 拓扑。
- [证据] `tgoskits/os/axvisor/src/images/x86/mptable.rs` — `x86_mptable::build()` — 整个 mptable 模块只被 `load_x86_linux_layout()` 引用（`mod.rs:37` 的 `use x86::mptable as x86_mptable` 只在 Linux 路径使用）。

**[结论] UEFI 路径是有意不提供 MP table / boot_params / IRQ 路由信息的。MP table 构建和加载是 Linux 直接启动路径的专属逻辑。UEFI 路径假设 OVMF 自己通过 fw_cfg（`etc/e820`）和 PCI config space 枚举获取平台信息，但 AxVisor 的 fw_cfg 不提供 ACPI 表（MADT/RSDP/_PRT 等），OVMF 在 nested 环境下没有 IRQ 路由信息来源。**

---

# 问题 C

setup_io_bitmap() 目前是否继续放行 PCI config 相关端口访问？如果是，具体放行了哪些端口，意味着什么边界？

- [证据] `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs:385-406` — `SvmVcpu::setup_io_bitmap()` — 显式拦截的端口清单：
  - `0x604`（QEMU_EXIT_PORT，2 字节）
  - `0x40-0x43`（X86_PIT_PORT_BASE，4 端口）
  - `0x61`（PIT_SPEAKER_PORT）
  - `0x3f8-0x3ff`（COM1，8 端口，条件拦截）
  - `0x402`（OVMF_DEBUGCON_PORT，1 端口）
  - `0x510-0x511`（FW_CFG_IO_BASE，2 端口）
  - `0x514-0x51b`（FW_CFG_DMA_IO_BASE，8 端口）
  - `0x6000-0x607f`（OVMF_VIRTIO_BLK_IO_BASE，0x80 端口）
- [证据] `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:481-510` — `VmxVcpu::setup_io_bitmap()` — 与 SVM 完全对称的端口清单，同样不含 0xCF8/0xCFC。
- [证据] `tgoskits/virtualization/x86_vcpu/src/vmx/structs.rs:65-69` — `IOBitmap::passthrough_all()` — 分配两页全零 I/O bitmap。全零 = 所有端口默认不拦截（passthrough）。
- [证据] `tgoskits/virtualization/x86_vcpu/src/svm/structs.rs:51-57` — `IOPm::passthrough_all()` — 三页全零 IOPm。全零 = 所有端口默认不拦截。
- [证据] 全树 `grep "0xcf8\|0xcfc\|0xCF8\|0xCFC\|PCI_CONFIG_ADDRESS\|CONFIG_DATA"` 在 `x86_vcpu/src/` 无命中 — 0xCF8/0xCFC 从未出现在任何 I/O bitmap 拦截代码中。
- [证据] `edk2/OvmfPkg/Library/DxePciLibI440FxQ35/PciLib.c:35-36` — `mRunningOnQ35 = (PcdGet16(PcdOvmfHostBridgePciDevId) == INTEL_Q35_MCH_DEVICE_ID)` — OVMF 自动检测 Q35（读 bus 0 dev 0 func 0 的 vendor/device ID）。Outer QEMU 是 q35（`tgoskits/os/axvisor/configs/qemu/qemu-x86_64.toml:6`：`machine = "q35"`），device ID 0x29C0，所以 OVMF 进入 Q35 模式。
- [证据] `edk2/OvmfPkg/Library/DxePciLibI440FxQ35/PciLib.c:92-95` — Q35 模式下 `PciRead8` dispatch 到 `PciExpressRead8`，使用 ECAM MMIO（0xE0000000 起）而非 0xCF8/0xCFC。但 ECAM MMIO 在 passthrough 范围内（`PCI MMIO` passthrough 0xE0000000 起 256MB）。
- [证据] `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml` — passthrough_devices 含 `["PCI MMIO", 0xe000_0000, 0xe000_0000, 0x1000_0000, 0x1]` — ECAM 窗口被 stage-2 线性映射到 host。

**[结论] PCI config 端口（0xCF8/0xCFC）完全不被拦截，passthrough 到 host。此外，Q35 模式下 OVMF 使用 ECAM MMIO（0xE0000000 起 256MB），该范围也被 stage-2 线性映射到 host。因此 nested OVMF 的 PCI config 枚举（无论是 legacy CF8 还是 ECAM）全部穿透到 outer QEMU，看到的是 outer QEMU 的 q35 PCI 总线设备（host bridge 0x29C0、virtio-blk 0x1001、LPC 0x2918 等）。AxVisor 没有任何 PCI config space emulation 层。**

---

# 未闭合点

- **OVMF 读取 `PcdOvmfHostBridgePciDevId` 的时机和结果**：默认值为 0（`OvmfPkgX64.dsc:713`），PEI 阶段通过读 PCI config space bus 0 dev 0 自动检测（`PlatformPei/Platform.c:178`）。在 nested 环境下读到的是 outer QEMU 的 0x29C0。但没有直接日志证据证明 nested OVMF 确实进入了 Q35 模式——需要在运行时确认。
- **fw_cfg `etc/e820` 的内容是否被 nested OVMF 实际使用**：OVMF 的 fw_cfg DXE 驱动会尝试读取 `etc/e820`，但不确定 nested OVMF 在没有 ACPI 表的情况下是否仍然信任 fw_cfg e820 作为唯一内存信息来源。需要确认 OVMF 的内存发现路径。
- **ECAM MMIO 基址是否完全匹配**：AxVisor passthrough 映射 0xE0000000 起 256MB，但 OVMF 会编程 MCH PCIEXBAR 寄存器（`PlatformInitLib/Platform.c:278-316`）来启用 ECAM 窗口。如果 OVMF 编程的 PCIEXBAR 基址与 passthrough 映射不一致（比如不是 0xE0000000），ECAM 访问会穿透到错误的 host 地址。没有直接日志证据证明两者一致。
- **`add_file_item` 接口虽存在但无调用方**：这意味着如果要给 nested OVMF 提供 ACPI 表，需要在 AxVM 初始化路径中新增调用。但当前代码没有预留这个扩展点的调用代码。

---

# 下一步最小调查动作

1. **确认 OVMF 的 PCIEXBAR 编程值**：在 `edk2/OvmfPkg/Library/PlatformInitLib/Platform.c:278-316` 的 `PciExBarInitialization()` 中，找到 Q35 模式下 PCIEXBAR 的默认基址。如果基址不是 0xE0000000，AxVisor 的 ECAM passthrough 映射地址需要调整。文件：`edk2/OvmfPkg/Library/PlatformInitLib/Platform.c`，函数 `PciExBarInitialization`。

2. **确认 OVMF 的内存发现路径**：在 `edk2/OvmfPkg/` 中搜索 OVMF 的 `ScanOrAdd820Entry` 或 `PlatformScanOrAdd820Entry` 或 fw_cfg e820 读取路径，确认 nested OVMF 在没有 ACPI 表的情况下如何获取完整内存布局。文件：`edk2/OvmfPkg/PlatformPei/` 和 `edk2/OvmfPkg/Library/PlatformInitLib/`。

3. **确认 fw_cfg 的 CPU 拓扑 item 是否被 nested OVMF 实际读取**：在 AxVisor fw_cfg 的 `select()` 和 `read_bytes()` 中加日志，记录 OVMF 请求了哪些 fw_cfg selector 和 file name。这能闭合"OVMF 从 AxVisor fw_cfg 获取了哪些平台信息"的证据链。文件：`tgoskits/virtualization/axdevice/src/fw_cfg.rs`，函数 `FwCfgInner::select()`（行 203）和 `FwCfgInner::read_bytes()`（行 218）。
