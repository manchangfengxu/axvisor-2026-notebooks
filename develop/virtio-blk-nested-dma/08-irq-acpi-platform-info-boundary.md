# IRQ / ACPI 平台信息边界

## 结论

- `fw_cfg` 当前只提供 `etc/e820`，没有 ACPI / MADT / RSDP / SMBIOS / IRQ 路由相关 file item。
- UEFI 路径显式只装载 `OVMF_CODE` / `OVMF_VARS`，没有像 Linux 路径那样补 MP table。
- `setup_io_bitmap()` 只拦截少数端口，没有拦截 `0xCF8/0xCFC`；Q35 下 OVMF 使用的 ECAM 访问也落在当前 passthrough 范围内。
- 这条 `PCI config / ECAM passthrough` 边界对当前 `OVMF polling` 启动里程碑仍然成立，但它本身不提供 nested guest 需要的 IRQ / ACPI 平台信息。

## 证据

- `tgoskits/virtualization/axdevice/src/fw_cfg.rs:136-185`
  - `FwCfgInner::configure`
  - `FwCfgInner::add_file_item`
  - `FwCfgInner::rebuild_file_dir`
  - 只初始化 `QEMU` / `DMA` / CPU 数 / `etc/e820`。
- `tgoskits/virtualization/axdevice/src/fw_cfg.rs:291-345`
  - `FwCfgDevice::configure`
  - `FwCfgDevice::add_file_item`
  - 生产路径只通过 `configure(memory_regions, cpu_count)` 进入；`add_file_item()` 接口存在，但当前树里没有生产调用方。
- `tgoskits/virtualization/axvm/src/vm.rs:419-430`
  - `AxVM` 只把 `memory_regions` 和 `cpu_count` 交给 `x86_fw_cfg_configure()`
- `tgoskits/os/axvisor/src/images/mod.rs:331-371`
  - `load_uefi_ovmf_images`
  - 只加载 `OVMF_CODE` / `OVMF_VARS`
- `tgoskits/os/axvisor/src/images/mod.rs:495-505`
  - `load_x86_linux_layout`
  - `build_x86_boot_params`
  - Linux 路径额外装载 `mp_table`
- `tgoskits/os/axvisor/src/images/mod.rs:707-730`
  - `load_vm_images_from_filesystem`
  - UEFI 分支只调用 `load_uefi_ovmf_images()`，再加载磁盘镜像后直接返回
- `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs:385-406`
- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:481-510`
  - `setup_io_bitmap()`
  - 只拦截 exit port、PIT、COM1、debugcon、fw_cfg、virtio-blk 端口；未拦截 `0xCF8/0xCFC`
- `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml:68-72`
  - `passthrough_devices`
  - 当前 `PCI MMIO` 范围覆盖 `0xe000_0000` 起的 Q35 PCI config MMIO / ECAM 窗口

## 未闭合

- `PCI MMIO` / ECAM 是否应继续保留为 passthrough，仍要结合后续 IRQ/ACPI 平台方案一起定。

## 下一步

先不要写 virtio-blk INTx。先决定 nested guest 的 IRQ / ACPI 平台信息来源，再决定是否继续保持 `OVMF polling + outer QEMU PCI passthrough`，还是开始补最小平台信息链。
