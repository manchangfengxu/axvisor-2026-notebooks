# rebase 后的 virtio-blk nested DMA 架构

本文记录 2026-06-14 重新检查并恢复 `tgoskits` 后的架构。旧的 `00-design.md`、`01-implementation.md`、`02-fix-hpa-hva.md` 是变基前写的，代码后来丢了；这份文档以当前 `uefi-develop` 为准。

当前分支关系：

- `tgoskits/dev` 等同 `upstream/dev`，当作纯上游基线。
- `tgoskits/uefi-develop` 比 `dev` 多 7 个提交，作者是 `manchangfengxu` 和 `l`。这些是我们的 UEFI bring-up 补丁，不算上游已经吸收。
- 当前工作目录在 `uefi-develop`。检查上游能力时对照 `dev`，实现修改落在 `uefi-develop`。

## 这次上游已经补好的部分

相比旧笔记里的代码状态，新的上游基础设施已经补了不少东西。

1. 配置链更完整

   文件：

   - `tgoskits/virtualization/axvmconfig/src/lib.rs`
   - `tgoskits/virtualization/axvm/src/config.rs`
   - `tgoskits/os/axvisor/src/config.rs`
   - `tgoskits/os/axvisor/src/images/mod.rs`

   已有字段和路径：

   - `VMKernelConfig::boot`
   - `VMKernelConfig::boot_protocol`
   - `VMKernelConfig::uefi_firmware_path`
   - `VMKernelConfig::ovmf_code_path`
   - `VMKernelConfig::ovmf_code_base`
   - `VMKernelConfig::ovmf_vars_path`
   - `VMKernelConfig::ovmf_vars_base`
   - `VMKernelConfig::reset_vector`
   - `OvmfInfo`
   - `ImageLoader::load_uefi_ovmf_images()`

   结论：不要再重做 UEFI 配置链。恢复 virtio-blk 时只补 `disk_path` 的消费路径。

2. guest memory 访问接口已经存在

   文件：

   - `tgoskits/virtualization/axaddrspace/src/memory_accessor.rs`

   关键接口：

   - `trait GuestMemoryAccessor`
   - `GuestMemoryAccessor::read_obj()`
   - `GuestMemoryAccessor::write_obj()`
   - `GuestMemoryAccessor::read_buffer()`
   - `GuestMemoryAccessor::write_buffer()`

   这个接口正好适合 virtqueue nested DMA。设备模型可以继续用 GPA，不需要把 descriptor table 里的地址改成 HPA。

   注意：`translate_and_get_limit()` 的返回值会被默认 `read_buffer()` 当成可解引用地址。AxVM 侧实现这个 trait 时，不能直接返回 HPA，必须返回 direct-map HVA 对应的数值。旧笔记 `02-fix-hpa-hva.md` 的判断仍然有效。

3. x86 平台设备比以前多

   文件：

   - `tgoskits/virtualization/axvm-types/src/lib.rs`
   - `tgoskits/virtualization/axdevice/src/device.rs`
   - `tgoskits/virtualization/x86_vlapic/src/vioapic.rs`
   - `tgoskits/virtualization/x86_vlapic/src/pit.rs`
   - `tgoskits/virtualization/x86_vlapic/src/serial.rs`
   - `tgoskits/virtualization/axvm/src/runtime/x86_irq.rs`

   已有类型和接线：

   - `EmulatedDeviceType::X86IoApic`
   - `EmulatedDeviceType::X86Pit`
   - `EmulatedDeviceType::Console`
   - `AxVmDevices::x86_ioapic_assert_gsi()`
   - `AxVmDevices::x86_ioapic_end_of_interrupt()`
   - `AxVmDevices::x86_pit_consume_irq0_if_due()`
   - `AxVmDevices::x86_serial_poll_irq()`
   - `runtime::x86_irq`

   结论：短期 virtio-blk 仍然可以先不注入中断，让 OVMF polling 跑通。后面要接 INTx 时，可以直接复用 `x86_ioapic_assert_gsi()` 这条路，不用从零写 vIOAPIC。

4. virtio 类型枚举已经预留

   文件：

   - `tgoskits/virtualization/axvm-types/src/lib.rs`

   已有枚举：

   - `EmulatedDeviceType::VirtioBlk`
   - `EmulatedDeviceType::VirtioNet`
   - `EmulatedDeviceType::VirtioConsole`

   但这只是配置类型。上游还没有实现 virtio-blk 设备模型，也没有把它接进 `AxVmDevices::init()`。

## 本次恢复的部分

这些内容已经在当前工作区恢复。

1. AxVisor 自己的 virtio-blk 设备模型

   文件：

   - `tgoskits/virtualization/axdevice/src/virtio_blk.rs`

   关键符号：

   - `LegacyVirtioBlk`
   - `LegacyQueue`
   - `VirtioBlkRequest`
   - `Descriptor`

   当前实现是 legacy split queue，一个 queue，内存后端。它处理：

   - `VIRTIO_BLK_T_IN`
   - `VIRTIO_BLK_T_OUT`
   - `VIRTIO_BLK_T_GET_ID`

   旧的 descriptor 改写路径已经删除。现在 AxVisor 自己读写 virtqueue，不再把 guest descriptor 里的 GPA 改成 HPA。

2. `disk_path` 已经接入

   文件：

   - `tgoskits/os/axvisor/src/images/mod.rs`
   - `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`

   当前路径：

   - `kernel.disk_path = "/guest/disks/uefi-boot-test.img"`
   - `ImageLoader::load_virtio_blk_disk_from_filesystem()`
   - `fs::read_image_file()`
   - `AxVM::install_virtio_blk_disk_image()`
   - `LegacyVirtioBlk::install_disk_image()`

   raw disk 读进内存后交给设备模型。当前不会把写请求回写到 rootfs 文件。

3. VMX/SVM 的 OVMF 端口拦截补齐

   文件：

   - `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
   - `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs`

   VMX 已经拦截：

   - `0x402`
   - `0x510..0x512`
   - `0x514..0x51c`
   - `0x6000..0x6080`

   本次把 SVM 也补到同一组端口。否则 AMD/SVM 路径下 debugcon、fw_cfg 和 virtio-blk 都可能不会退出到 AxVM。

4. 设备框架还没有方向二需要的上下文接口

   现有接口：

   - `BaseDeviceOps::handle_read(addr, width)`
   - `BaseDeviceOps::handle_write(addr, width, val)`

   它没有 `GuestMemoryAccessor`，也没有 `IrqSink`。所以本阶段不要硬把 virtio-blk 注册成普通 `BasePortDeviceOps`。否则要提前改公共 trait，范围会变大，也容易和方向二冲突。

## 最终架构

本次只拆一小层。没有新建 `axvirtio_blk` crate，也没有把 virtio-blk 放回 `axvm/src/virtio_blk/`。

当前只用一个文件：

```text
tgoskits/virtualization/axdevice/src/virtio_blk.rs
```

当前功能不多，单文件更容易审。后面如果要加中断、配置空间细节、多队列，再拆出 `queue.rs` 和 `backend.rs`。

模块职责：

- `LegacyVirtioBlk` 处理 legacy I/O BAR `0x6000..0x607f`。
- `LegacyQueue` 保存 `queue_pfn`、`queue_size`、`last_avail_idx`，按 GPA 读取 descriptor table、avail ring、used ring。
- `VirtioBlkRequest` 负责解析 descriptor chain。
- `LegacyVirtioBlk::disk` 持有 `Vec<u8>`。读写请求只修改内存里的 raw disk，不回写 rootfs 文件。

AxVM 只保留胶水：

- `AxVMInnerMut::ovmf_virtio_blk: LegacyVirtioBlk`
- `struct AxVmGuestMemory`
- `impl GuestMemoryAccessor for AxVmGuestMemory`
- `AxVM::install_virtio_blk_disk_image()`
- `AxVM::handle_ovmf_virtio_blk_io_read()`
- `AxVM::handle_ovmf_virtio_blk_io_write()`

AxVM 侧 `handle_ovmf_virtio_blk_io_write()` 在 notify 时构造 `AxVmGuestMemory`，然后调用设备：

```text
OVMF out 0x6010
  -> AxVM::handle_ovmf_virtio_blk_io_write()
  -> LegacyVirtioBlk::handle_write(port, width, value, &guest_memory)
  -> LegacyQueue::pop_available()
  -> LegacyQueue::collect_chain()
  -> VirtioBlkRequest::from_chain()
  -> LegacyVirtioBlk::execute_read/write/get_id()
  -> GuestMemoryAccessor::write_buffer()
  -> LegacyQueue::publish_used()
```

当前恢复阶段仍然不注入中断。OVMF legacy virtio-blk 路径可以 polling，之前 smoke 已经验证过这点。下一步局部标准化阶段不应继续依赖 polling，应按下文 INTx 路径复用 `AxVmDevices::x86_ioapic_assert_gsi()`。

## 和方向二的兼容边界

方向二未来会把设备访问整理成类似 `BusAccess + DeviceContext + IrqSink` 的结构。当前上游还没有这个接口，所以方向一不要等它。

为了以后好迁移，本阶段保留这几个边界：

- `LegacyVirtioBlk` 不依赖 `AxVM`。
- `LegacyVirtioBlk` 只通过 `GuestMemoryAccessor` 访问 guest memory。
- `LegacyVirtioBlk` 不直接碰 vLAPIC、vIOAPIC 或 vCPU。
- `disk_path` 的读取放在 `os/axvisor/src/images/mod.rs`，设备只接收 `Vec<u8>`。
- `AxVM` 里的特殊 I/O 分发只当临时适配层，后续 `DeviceContext` 出来后可以删。

这样后续迁移只需要把：

```text
AxVM 手动构造 AxVmGuestMemory
```

替换成：

```text
DeviceContext::guest_memory
```

设备内部的 virtqueue、request、backend 逻辑不用重写。

## 下一步局部标准化设计

方向一继续推进，但不要继续把 OVMF smoke 需要的逻辑堆进 `axvm`。目标是在当前架构能承受的范围内，把设备核心写成标准 hypervisor 设备模型；方向二重构成熟后，只替换 VM 侧 glue，不重写 virtqueue、request、backend 和基础设备语义。

### 1. virtio-blk 核心保持设备化

当前正确边界：

- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - `LegacyVirtioBlk`
  - `LegacyQueue`
  - `VirtioBlkRequest`
  - `Descriptor`

这些符号不应该依赖 `AxVM`、vCPU、OVMF 或 VM-exit。它们只通过 `GuestMemoryAccessor` 读写 guest memory。

下一步完善点：

- `LegacyVirtioBlk::handle_write()` 在 `REG_QUEUE_NOTIFY` 后应返回是否发布了 used buffer，方便上层触发 INTx。
- `LegacyVirtioBlk` 增加 legacy ISR status 语义，至少支持 queue used bit。
- `VirtioBlkRequest` 继续只处理标准请求：`VIRTIO_BLK_T_IN`、`VIRTIO_BLK_T_OUT`、`VIRTIO_BLK_T_GET_ID`。
- `LegacyQueue` 继续保存 GPA 视角的 queue 状态，禁止恢复 descriptor GPA->HPA 原地改写。

后续如果文件变大，再拆为：

- `tgoskits/virtualization/axdevice/src/virtio/queue.rs`
  - `LegacyQueue`
  - `Descriptor`
- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - `LegacyVirtioBlk`
  - `VirtioBlkRequest`

现在不需要为了拆而拆。

### 2. AxVM 只保留临时 adapter

当前 adapter 定位：

- `tgoskits/virtualization/axvm/src/vm.rs`
  - `AxVMInnerMut::ovmf_virtio_blk`
  - `AxVmGuestMemory`
  - `AxVM::install_virtio_blk_disk_image()`
  - `AxVM::handle_ovmf_virtio_blk_io_read()`
  - `AxVM::handle_ovmf_virtio_blk_io_write()`

这些符号可以短期存在，但职责只能是：

- 持有设备实例。
- 把 VM exit 的 PIO 访问转给设备。
- 构造 `AxVmGuestMemory`。
- 在设备完成请求后接现有 x86 IRQ 注入路径。

不应该再加入：

- virtqueue descriptor 解析。
- block request 解析。
- disk sector 读写逻辑。
- OVMF 专用状态机。

方向二之后，以上 adapter 应替换为：

```text
DeviceContext::guest_memory
DeviceContext::irq_sink
BusRouter::pio_dispatch()
```

但 `LegacyVirtioBlk` 内部逻辑不应改。

### 3. INTx 应依托现有 vIOAPIC

不要长期依赖 OVMF polling。当前已有基础设施：

- `tgoskits/virtualization/axdevice/src/device.rs`
  - `AxVmDevices::x86_ioapic_assert_gsi()`
  - `AxVmDevices::x86_ioapic_end_of_interrupt()`
- `tgoskits/virtualization/axvm/src/runtime/x86_irq.rs`
  - `inject_pending_ioapic_irq_after_eoi()`
  - `inject_due_pit_irq0()`
  - `inject_pending_serial_irq()`
- `tgoskits/virtualization/x86_vlapic/src/vioapic.rs`
  - `EmulatedIoApic::assert_gsi()`
  - `EmulatedIoApic::end_of_interrupt()`

推荐最小路径：

```text
OVMF out 0x6010
  -> AxVM::handle_ovmf_virtio_blk_io_write()
  -> LegacyVirtioBlk::handle_write()
  -> LegacyQueue::publish_used()
  -> LegacyVirtioBlk sets ISR queue bit
  -> AxVmDevices::x86_ioapic_assert_gsi(virtio_blk_gsi)
  -> vcpu.inject_interrupt_with_trigger()
```

注意：当前 `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` 里 `emu_devices = []`，还没有把 `X86IoApic` 接入这个 OVMF VM。补 INTx 前要先让配置和设备模型一致。

### 4. PCI 先定义边界，不抢完整实现

当前 `LEGACY_BLK_IO_BASE = 0x6000` 能让 OVMF smoke 跑通，但它不是完整 virtio-pci 发现路径。

短期可接受定位：

- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - `LEGACY_BLK_IO_BASE`
  - `LEGACY_BLK_IO_SIZE`

把它称为 legacy virtio-blk PIO transport，不要称为完整 virtio-pci。

更标准的下一步是最小 PCI config：

- 一个 root bus。
- 一个 virtio-blk function。
- 一个 I/O BAR 指向 `0x6000..0x607f`。
- 一个 INTx pin/line。

这部分应尽量独立于 `LegacyVirtioBlk` 核心。未来方向二做 `BusRouter` 或 PCI bus 时，只替换 config/BAR 注册路径。

### 5. fw_cfg 和 ACPI PM 不继续留在 vm.rs

当前还在 `axvm` 中：

- `tgoskits/virtualization/axvm/src/vm.rs`
  - `FwCfgState`
  - `AxVM::handle_fw_cfg_io_read()`
  - `AxVM::handle_fw_cfg_io_write()`
  - `AxVM::handle_fw_cfg_dma()`
  - `AxVM::handle_acpi_pm_io_read()`
  - `AxVM::handle_acpi_pm_io_write()`

下一步应整理成 x86 platform device：

- `fw_cfg` 作为 QEMU fw_cfg port/DMA 设备，继续支持 selector、data port、DMA read、file directory、`etc/e820`。
- `acpi_pm` 先实现最小 PM1_CNT/PM1_STS/PM timer 语义，不继续 host port passthrough。
- `debugcon` 保持普通 port device，避免同时在 `AxVmDevices` 和 `AxVM::run()` 中重复特判 `0x402`。

方向二之后，这些设备应接入统一 bus；但设备寄存器语义本身不应重写。

### 6. 不在本阶段引入的东西

本阶段不要提前实现：

- 通用 `DeviceContext`。
- 通用 `BusAccess` / `BusRouter`。
- 通用 `IrqSink`。
- async worker / eventfd。
- MSI / MSI-X。
- packed queue。
- multi-queue。
- modern virtio-pci capability。

这些要么属于方向二，要么属于 Linux EFI 完整启动后续阶段。当前只保证 OVMF bring-up 所需路径按标准设备语义收敛。

## 可以借鉴的现成实现

不要自己死磕复杂 virtio 代码。当前只做 legacy split queue 的最小 block 设备，可以借鉴两处：

- `references/cloud-hypervisor/virtio-devices/src/block.rs`
- `references/cloud-hypervisor/vm-virtio/src/queue.rs`

Cloud Hypervisor 的实现比我们当前需要的大很多。可借鉴的是边界：

- block 设备自己解析 request。
- queue 逻辑和 backend I/O 分开。
- 设备通过 guest memory 抽象读写 descriptor、data buffer 和 used ring。

不需要照搬的内容：

- async I/O
- epoll worker
- rate limiter
- migration
- multi-queue
- packed queue
- MSI/MSI-X
- discard/write zeroes

当前只需要：

- legacy split queue
- 一个 queue
- read/write/get-id
- polling
- memory backend

## 验收点

恢复后先跑小闭环，不要一次追 Linux EFI。

代码检查：

```bash
rg "rewrite desc|rewrite_ovmf|translated_queue_pfn|translate_ovmf_virtio_blk_queue_pfn|OvmfVirtioBlkIoState" tgoskits/virtualization tgoskits/os
cargo fmt --check --package axdevice --package axvm --package axvisor
cargo test -p axdevice virtio_blk -- --nocapture
cargo test -p axvm --lib
cargo run --manifest-path xtask/Cargo.toml -- axvisor build --config os/axvisor/configs/board/qemu-x86_64.toml --vmconfigs os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml
```

smoke 目标：

```text
Loading virtio-blk disk image from /guest/disks/uefi-boot-test.img
VirtioBlkInit: LbaSize=0x200[B]
FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success
ArceOS UEFI helloworld
```

不应该出现：

```text
Unhandled #PF
rewrite desc
translated_queue_pfn
```

## 本次文档修改记录

新增文件：

- `axvisor-2026-notebooks/develop/virtio-blk-nested-dma/03-final-architecture-after-rebase.md`

## 本次代码修改记录

以下记录按文件和变量名定位，方便以后追 diff。

- `tgoskits/virtualization/axdevice/Cargo.toml`
  - 新增依赖 `axaddrspace`
- `tgoskits/virtualization/axdevice/src/lib.rs`
  - 新增导出 `pub mod virtio_blk`
- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - 新增 `LegacyVirtioBlk`
  - 新增 `LegacyQueue`
  - 新增 `VirtioBlkRequest`
  - 新增 `Descriptor`
  - 新增端口常量 `LEGACY_BLK_IO_BASE`、`LEGACY_BLK_IO_SIZE`
  - 新增单元测试 `reports_capacity_from_installed_disk_image`
  - 新增单元测试 `read_request_copies_sector_into_guest_buffer_and_publishes_used_ring`
  - 新增单元测试 `write_request_updates_backend_and_reports_only_status_byte_used`
- `tgoskits/virtualization/axvm/src/vm.rs`
  - `AxVMInnerMut::ovmf_virtio_blk` 改为 `LegacyVirtioBlk`
  - 新增 `AxVmGuestMemory`
  - 新增 `impl GuestMemoryAccessor for AxVmGuestMemory`
  - 新增 `AxVM::install_virtio_blk_disk_image()`
  - `handle_ovmf_virtio_blk_io_read()` 改为调用 `LegacyVirtioBlk::handle_read()`
  - `handle_ovmf_virtio_blk_io_write()` 改为调用 `LegacyVirtioBlk::handle_write()`
  - `IoStringRead` 对 `FW_CFG_IO_DATA` 使用 `FwCfgState::read_bytes()`
  - `IoStringRead` 和 `IoStringWrite` 按 `count * width.size()` 计算 guest buffer 字节数
  - 删除旧符号 `OvmfVirtioBlkIoState`
  - 删除旧函数 `translate_ovmf_virtio_blk_queue_pfn()`
  - 删除旧函数 `rewrite_ovmf_virtio_blk_descriptors()`
  - 删除旧函数 `dump_ovmf_virtio_blk_queue()`
- `tgoskits/os/axvisor/src/images/mod.rs`
  - 新增 `ImageLoader::load_virtio_blk_disk_from_filesystem()`
  - `fs::read_image_file()` 改为 `pub(crate)`
  - OVMF 分支消费 `loader.config.kernel.disk_path`
  - 普通 filesystem image 分支也消费 `loader.config.kernel.disk_path`
- `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`
  - 新增 `kernel.disk_path`
- `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs`
  - 新增 `OVMF_DEBUGCON_PORT`
  - 新增 `FW_CFG_IO_BASE`
  - 新增 `FW_CFG_DMA_IO_BASE`
  - 新增 `OVMF_VIRTIO_BLK_IO_BASE`
  - `setup_io_bitmap()` 补齐 debugcon、fw_cfg、fw_cfg DMA、virtio-blk 端口拦截
  - `SvmExitCode::IOIO` 对 string I/O 生成 `AxVCpuExitReason::IoStringRead`
  - `SvmExitCode::IOIO` 对 string I/O 生成 `AxVCpuExitReason::IoStringWrite`

本次检查过的上游关键代码位置：

- `tgoskits/virtualization/axaddrspace/src/memory_accessor.rs`
  - `GuestMemoryAccessor`
- `tgoskits/virtualization/axvm-types/src/lib.rs`
  - `EmulatedDeviceType::VirtioBlk`
  - `EmulatedDeviceType::X86IoApic`
  - `EmulatedDeviceType::X86Pit`
- `tgoskits/virtualization/axdevice/src/device.rs`
  - `AxVmDevices`
  - `AxVmDevices::x86_ioapic_assert_gsi()`
  - `AxVmDevices::x86_ioapic_end_of_interrupt()`
  - `BaseDeviceOps` 分发路径
- `tgoskits/virtualization/axvm/src/vm.rs`
  - `OvmfVirtioBlkIoState`
  - `handle_ovmf_virtio_blk_io_read()`
  - `handle_ovmf_virtio_blk_io_write()`
  - `rewrite_ovmf_virtio_blk_descriptors()`
  - `handle_fw_cfg_io_read()`
  - `handle_fw_cfg_io_write()`
- `tgoskits/os/axvisor/src/images/mod.rs`
  - `ImageLoader::load_uefi_ovmf_images()`
  - `VMKernelConfig::disk_path`
- `references/cloud-hypervisor/virtio-devices/src/block.rs`
  - block request 和 backend 边界
- `references/cloud-hypervisor/vm-virtio/src/queue.rs`
  - split queue 测试和内存布局

## 本次验证记录

最终跑过：

```bash
cargo test -p axdevice virtio_blk -- --nocapture
cargo test -p axvm --lib
cargo test -p x86_vcpu --lib
cargo fmt --check --package axdevice --package axvm --package axvisor --package x86_vcpu
cargo run --manifest-path xtask/Cargo.toml -- axvisor build --config os/axvisor/configs/board/qemu-x86_64.toml --vmconfigs os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml
```

结果：

- `axdevice virtio_blk`：3 passed
- `axvm --lib`：编译通过，0 tests
- `x86_vcpu --lib`：61 passed
- `cargo fmt --check`：通过
- AxVisor OVMF 配置构建：通过，后端自动选择 `vmx`，最终验证时间 2026-06-14 13:39 CST

另外跑过旧 hack 搜索：

```bash
rg "rewrite desc|rewrite_ovmf|translated_queue_pfn|translate_ovmf_virtio_blk_queue_pfn|OvmfVirtioBlkIoState" tgoskits/virtualization tgoskits/os
```

没有匹配项。

## 2026-06-14 真实启动后的修订

这次不只跑 build，已经跑了外层 QEMU + AxVisor + 嵌套 OVMF。

启动时不能把 `tmp/uefi-boot-test.img` 直接作为外层 `--rootfs`。这个镜像本身有 `EFI/BOOT/BOOTX64.EFI`，外层 OVMF 会先从它启动，结果看到的是宿主 OVMF 直接加载 helloworld，不是 AxVisor 启动嵌套 OVMF。

正确关系是：

- 外层 QEMU rootfs：`tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img`
- rootfs 内嵌套盘：`/guest/disks/uefi-boot-test.img`
- VM 配置字段：`kernel.disk_path = "/guest/disks/uefi-boot-test.img"`

本次为了 smoke，把 `tmp/nested-uefi-disk.img` 写入了 Alpine rootfs：

```bash
debugfs -w -R 'mkdir /guest/disks' tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
debugfs -w -R 'write tmp/nested-uefi-disk.img /guest/disks/uefi-boot-test.img' \
  tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

上游配置模型也变了。旧笔记里的 `boot = "uefi"` / `enable_bios = false` 会在 `axvmconfig` 阶段失败：

```text
boot_protocol requires enable_bios = true
```

现在的 OVMF 配置需要按上游字段写：

- `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`
  - `kernel.boot_protocol = "uefi"`
  - `kernel.enable_bios = true`

这里的 `enable_bios` 名字有历史包袱。对当前 `axvmconfig::VMKernelConfig::validate_boot_config()` 来说，它表示走固件类启动路径，不是说只能加载 SeaBIOS。

还修了一个上游变基后暴露出的重复映射：

- `tgoskits/virtualization/axvm/src/vm.rs`
  - 删除 `AxVM::init()` 中旧的 `inner_mut.address_space.map_linear(GuestPhysAddr::from(0xfee0_0000), crate::vcpu::EmulatedLocalApic::virtual_apic_access_addr(), ...)`
  - 保留后面的上游 VMX 条件映射：`X86_APIC_ACCESS_GPA` + `x86_apic_access_page_addr()`

不删旧块时，VM 创建 vCPU 后会失败：

```text
Mapping error: AlreadyExists
VM[1] setup failed: AxErrorKind::AlreadyExists
```

原因不是 virtio-blk，也不是 OVMF 固件窗口，而是同一个 APIC access GPA `0xfee0_0000` 被映射了两次。

顺手清了一个 warning：

- `tgoskits/virtualization/axvm/src/vcpu.rs`
  - 删除未使用的 `pub use x86_vcpu::EmulatedLocalApic`

这次真实启动看到的关键日志：

```text
Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000
Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000
Loading virtio-blk disk image from /guest/disks/uefi-boot-test.img (33554432 bytes)
OVMF debugcon:  BlockSize : 512
OVMF debugcon:  LastBlock : FFFF
OVMF debugcon:  Valid primary and Valid backup partition table
OVMF debugcon: FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success
ArceOS UEFI helloworld
ArceOS UEFI shell-stage boot OK
```

这说明方向一的最小闭环已经走通：嵌套 OVMF 通过 AxVisor 内的 legacy virtio-blk 设备模型读到了 GPT/FAT 盘，并启动了 ESP 里的 UEFI 应用。
