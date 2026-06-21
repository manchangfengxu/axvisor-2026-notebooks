# Prompt 3: 当前阶段标准化设计结论

## A. 一句话判断

"当前 virtio-blk 最主要的不标准之处是 **设备核心无法接入 `AxVmDevices` 的通用 port dispatch 路径，导致 I/O 路由硬编码在 `axvm/src/vm.rs` 的 `run_vcpu()` match 分支里**"

具体表现：`LegacyVirtioBlk` 不实现 `BaseDeviceOps<PortRange>`，不在 `AxVmDevices::init()` 中注册，而是由 `AxVM::handle_ovmf_virtio_blk_io_read/write()` 手动检查端口范围后直接调用。这和 `FwCfgDevice`、`OvmfDebugConDevice`、`EmulatedPit`、`EmulatedSerialPort` 的注册-分发模式不一致。

根本原因不是设计选择，而是基础设施缺口：`BaseDeviceOps::handle_write(&self, ...)` 签名是共享引用，而 virtio-blk 的 `process_queue` 需要 `&mut self` 来修改 `LegacyQueue::last_avail_idx`；`BaseDeviceOps` 不携带 `GuestMemoryAccessor`，而 virtio-blk 的 descriptor chain 解析和数据搬运全部依赖它。这两点在方向二重构 `BaseDeviceOps` 之前无法通过常规手段弥合。

## B. 当前阶段推荐方案

### 设备核心边界

**结论**：`LegacyVirtioBlk`、`LegacyQueue`、`VirtioBlkRequest`、`Descriptor`、`LegacyNotifyResult`、`RequestCompletion` 保留在 `tgoskits/virtualization/axdevice/src/virtio_blk.rs`。

**理由**：
- 设备核心已经通过 `GuestMemoryAccessor` 访问 guest memory，不依赖 `AxVM` 类型（`virtio_blk.rs` 的 `use` 只有 `axaddrspace::GuestMemoryAccessor`、`axdevice_base::{AccessWidth, Port}`、`axvm_types::GuestPhysAddr`）。
- `LegacyVirtioBlk::handle_write()` 返回 `LegacyNotifyResult`，由调用方决定是否注入中断，设备核心本身不碰 vLAPIC/vIOAPIC。
- `LegacyVirtioBlk` 不引用 `AxVM`、`AxVCpu`、`OvmfVirtioBlkIoState`、`vm.rs` 中任何符号。
- 这符合 CLAUDE.md 约定："`virtio_blk.rs` 应 owns legacy virtio-blk protocol core"。

**不应再往 `virtio_blk.rs` 加的东西**：
- 不加 `BaseDeviceOps<PortRange>` impl。原因：当前 trait 签名是 `&self`，virtio-blk queue 处理需要 `&mut self`；强行 `impl` 会要求把整个 `LegacyQueue` 改为内部可变性，代价是污染 queue 逻辑的正确性保证。等方向二提供带 `GuestMemoryAccessor` + 可变语义的 device context 后自然解决。
- 不加 block backend trait 抽象。当前 `MemoryDisk` 是唯一后端，且只服务于 OVMF smoke。抽象 backend 接口属于后续扩展，不是当前阶段必须。
- 不加 PCI config space 仿真。`LEGACY_BLK_IO_BASE` 命名已经是 "legacy PIO transport"，不伪装 virtio-pci。

**为什么不违反边界**：CLAUDE.md 明确说 "如果当前 AxVisor 基础设施不能把设备作为 first-class `AxVmDevices` 设备承载，保持 device core in `axdevice` + replaceable `AxVM` adapter glue 的稳定拆分"。当前 `BaseDeviceOps` 缺少 `&mut self` 和 `GuestMemoryAccessor`，正是这个条件。

### transport / adapter 边界

**结论**：`axvm/src/vm.rs` 中的以下符号作为临时 adapter 继续存在：

- `AxVMInnerMut::ovmf_virtio_blk: LegacyVirtioBlk`
- `AxVM::install_virtio_blk_disk_image()`
- `AxVM::handle_ovmf_virtio_blk_io_read()`
- `AxVM::handle_ovmf_virtio_blk_io_write()`

**理由**：
- 这些函数的唯一职责是：(1) 端口范围检查、(2) 构造 `AxVmGuestMemory`、(3) 调用设备核心、(4) 消费 `LegacyNotifyResult`。
- 它们不重新解释 virtio 协议，不解析 descriptor，不处理 block I/O。
- `run_vcpu()` 的 match 分支中 `handle_ovmf_virtio_blk_io_read/write` 被调用，返回 `Option<usize>` / `bool`，和 `handle_acpi_pm_io_read/write` 模式完全一致。

**不应再往 adapter 加的东西**：
- 不加 virtqueue 状态管理（`queue_pfn`、`last_avail_idx`）。
- 不加 descriptor chain 解析。
- 不加 block request 构造或 sector 读写。
- 不加 OVMF 专用状态机或 OVMF 配置空间模拟。
- 不加中断注入。当前 `should_raise_irq` 只做 `trace!` 日志记录，OVMF polling 路径不依赖 INTx。CLAUDE.md 明确禁止在当前 DXE 路径接 INTx。

**为什么不违反边界**：adapter 只是 "持有设备实例 + 把 VM-exit PIO 访问转给设备 + 构造 `AxVmGuestMemory`"。方向二 `DeviceContext::guest_memory` 出来后，这些函数整体替换，不需要改动设备核心。

### guest memory / backend 边界

**结论**：`AxVmGuestMemory`（在 `axvm/src/vm.rs` 中）继续作为 `GuestMemoryAccessor` 的唯一 AxVM 侧实现。设备核心只通过 `GuestMemoryAccessor` trait 访问 guest memory。

**当前实现的正确性**：
- `AxVmGuestMemory::translate_and_get_limit()` 返回 `PhysAddr` 数值对应 direct-map HVA（不是 HPA），然后 `read_buffer()`/`write_buffer()` 通过 `core::ptr::copy_nonoverlapping` 操作。这符合 `03-final-architecture-after-rebase.md` 中 "不能直接返回 HPA，必须返回 direct-map HVA 对应数值" 的约束。
- `address_space` 通过裸指针持有，避免在 DMA 期间持有 `AxVMInnerMut` 锁。当前注释已记录这个设计选择。

**MemoryDisk 保持现状**：
- `LegacyVirtioBlk` 内嵌 `MemoryDisk`（`Vec<u8>`），raw disk 在 `ImageLoader` 中读入内存后交给设备模型。
- 不做写回 rootfs（当前 OVMF smoke 只读 GPT/FAT，不需要写回）。
- 不抽象为 `BlockBackend` trait。理由：当前只有这一个后端，抽象接口无消费者；后续加 file-backend 或 virtio-blk 写回时再引入。

**为什么不违反边界**：`GuestMemoryAccessor` 是 AxVisor 已有的 `axaddrspace` 基础设施，不是 direction-2 替代框架。`MemoryDisk` 是设备内部实现细节，不影响设备核心与 VM 之间的接口契约。

### IRQ / ISR / notify 边界

**结论**：ISR status 和 notify result 信号由设备核心产生，中断注入决策由 adapter（当前）或 direction-2 `IrqSink`（未来）消费。当前阶段不注入 INTx。

**当前状态**：
- `LegacyVirtioBlk::handle_write()` 在 `REG_QUEUE_NOTIFY` 后：(1) 调用 `process_queue()`、(2) 若 `should_raise_irq` 则设置 `self.isr_status |= VIRTIO_ISR_QUEUE`、(3) 返回 `LegacyNotifyResult`。
- `LegacyVirtioBlk::handle_read()` 对 `REG_ISR_STATUS (0x13)`：返回当前 `isr_status` 后清零。这实现了 legacy virtio 规范的 "reading ISR clears it" 语义。
- `AxVM::handle_ovmf_virtio_blk_io_write()` 消费 `LegacyNotifyResult`，当 `should_raise_irq` 为 true 时只做 `trace!` 日志。不调用 `x86_ioapic_assert_gsi()`。

**基础设施已就绪但不接线**：
- `AxVmDevices::x86_ioapic_assert_gsi(gsi)` 已存在。
- `runtime::x86_irq::inject_pending_ioapic_irq_after_eoi()` 已存在。
- `EmulatedIoApic::assert_gsi()` 已存在。
- 但 `ovmf-x86_64-qemu-smp1.toml` 的 `emu_devices = []` 没有注册 `X86IoApic`。
- OVMF DXE virtio-blk 驱动轮询 `Used.Idx`，不依赖中断。

**为什么不违反边界**：设备核心正确产出了 ISR status 和 notify result 信号，这是设备模型的标准行为。不注入 INTx 是基于 OVMF 轮询行为的事实判断，不是设计遗漏。接入 INTx 的时机是用户明确要求或 OVMF 路径变化时。

## C. 明确列出应该落在哪些文件

### `tgoskits/virtualization/axdevice/src/virtio_blk.rs`

**应该保留**：
- `LegacyVirtioBlk` 及其所有内部状态（`queue: LegacyQueue`、`disk: MemoryDisk`、`status: u8`、`isr_status: u8`）
- `LegacyQueue`（`size`、`pfn`、`last_avail_idx` 及所有 GPA 操作方法）
- `VirtioBlkRequest`（`from_chain()`、`validate_data_descriptors()`）
- `Descriptor`（`writable()`）
- `LegacyNotifyResult`（`idle()`、`completed_one()`、`merge()`）
- `RequestCompletion`（`ok()`、`ioerr()`、`unsupp()`）
- `MemoryDisk`（`new()`、`capacity_sectors()`、`read_at()`、`write_at()`）
- 所有端口寄存器常量（`REG_*`、`LEGACY_BLK_IO_BASE`、`LEGACY_BLK_IO_SIZE`）
- 所有 virtio 协议常量（`VIRTIO_BLK_T_*`、`VIRTIO_BLK_S_*`、`VIRTQ_DESC_F_*`）
- `LegacyVirtioBlk::handle_read()`、`handle_write()`、`owns_port()`、`install_disk_image()`
- 所有现有单元测试

**不该再承载**：
- `BaseDeviceOps<PortRange>` impl（原因：`&self` 签名与 queue mutability 冲突）
- PCI config space 模拟
- block backend trait 抽象
- 中断注入逻辑（不调用 `ioapic_assert_gsi`）

**如果继续往里塞，会违反哪条 CLAUDE 契约**：
- 加 PCI config space：违反 "当前只做 legacy PIO transport，不伪装 virtio-pci"（`03-final-architecture-after-rebase.md` 末尾 "PCI 先定义边界，不抢完整实现"）
- 加 `BaseDeviceOps` impl：违背 `03-final-architecture-after-rebase.md` 中 "不要硬把 virtio-blk 注册成普通 `BasePortDeviceOps`，否则要提前改公共 trait，范围会变大，也容易和方向二冲突" 的设计判断
- 加中断注入：违反 "Do not continue virtio-blk INTx wiring for the OVMF DXE path unless the user explicitly reopens it"（CLAUDE.md）

### `tgoskits/virtualization/axvm/src/vm.rs`

**应该保留**：
- `AxVMInnerMut::ovmf_virtio_blk: LegacyVirtioBlk`（字段声明和初始化）
- `AxVmGuestMemory`（struct + `impl GuestMemoryAccessor`）
- `AxVM::install_virtio_blk_disk_image()`（pub fn，被 `os/axvisor/src/images/mod.rs` 调用）
- `AxVM::handle_ovmf_virtio_blk_io_read()`（private fn）
- `AxVM::handle_ovmf_virtio_blk_io_write()`（private fn）
- `run_vcpu()` match 分支中对上述两个函数的调用

**不该再承载**：
- virtqueue 描述符解析或 `queue_pfn` / `last_avail_idx` 管理
- block request 构造或 sector 读写逻辑
- OVMF 专用 virtio 状态机（旧的 `OvmfVirtioBlkIoState` 已删除）
- 中断注入调用（当前 `should_raise_irq` 只做 trace 日志）
- 任何新增的 virtio 协议常量或类型

**如果继续往里塞，会违反哪条 CLAUDE 契约**：
- 加 virtqueue/request 逻辑：违反 "Keep new virtqueue, request parsing, feature, ISR, and backend logic out of `axvm`"（CLAUDE.md）
- 加中断注入：违反 "Do not continue virtio-blk INTx wiring for the OVMF DXE path unless the user explicitly reopens it"（CLAUDE.md）
- 加 OVMF 状态机：违反 "不要继续把 OVMF smoke 需要的逻辑堆进 `axvm`"（`03-final-architecture-after-rebase.md`）

### `tgoskits/os/axvisor/src/images/mod.rs`

**应该保留**：
- `ImageLoader::load_virtio_blk_disk_from_filesystem()`
- `fs::read_image_file()`（pub(crate)）
- OVMF 分支消费 `loader.config.kernel.disk_path` 的路径
- 调用 `AxVM::install_virtio_blk_disk_image()`

**不该再承载**：
- 任何 virtio 协议逻辑
- disk 格式解析（GPT/FAT 等由 OVMF guest 侧处理）

### `tgoskits/virtualization/axdevice/src/device.rs`

**应该保留**：
- `AxVmDevices` 中已有的 x86 设备槽位（`x86_ioapic`、`x86_pit`、`x86_serial`、`x86_fw_cfg`）
- 所有 `x86_ioapic_assert_gsi()` / `x86_ioapic_end_of_interrupt()` 方法

**不该再承载**：
- virtio-blk 的 port device 注册（当前不满足 `BaseDeviceOps` 签名要求）
- 任何新增的 virtio 相关字段或方法

### `tgoskits/virtualization/axdevice_base/src/lib.rs`

**应该保留**：
- `BaseDeviceOps<R>` trait 定义（当前签名不变）
- `BasePortDeviceOps` trait alias

**不该再承载**：
- 任何为 virtio-blk 临时修改 `BaseDeviceOps` 签名的尝试。这属于方向二范畴。

## D. 明确列出当前阶段不做什么

### 1. direction-2 本地替代框架

不引入任何以下类型或模式：
- `DeviceContext`、`BusAccess`、`BusRouter`、`IrqSink`
- 任何带有 `guest_memory` 字段的新 struct
- 任何 "部分 bus dispatch" 实现

**依据**：CLAUDE.md "Do not pre-build a local substitute for that future framework"；`03-final-architecture-after-rebase.md` "do not invent a partial `DeviceContext`, `BusRouter`, or `IrqSink` in direction 1"；`ALL.md` "方向二不是我们组在进行的，且无法认为可以基于别人方向二设计的框架"。

### 2. 完整 virtio-pci modern

不实现：
- PCI config space ECAM 模拟
- virtio PCI capability 报告
- MSI-X 向量配置
- modern queue layout（packed ring）
- `VIRTIO_F_VERSION_1` 协商

**依据**：`03-final-architecture-after-rebase.md` "PCI 先定义边界，不抢完整实现"；CLAUDE.md 当前只支持 "legacy virtio-blk PIO transport"。

### 3. 新 PCI 框架

不引入：
- PCI bus 枚举逻辑
- PCI BAR 分配器
- PCI 配置空间 read/write 分发
- interrupt line/pin 自动路由

**依据**：当前 `LEGACY_BLK_IO_BASE = 0x6000` 是硬编码 legacy PIO transport，不是 PCI 设备发现路径。引入 PCI 框架需要方向二的 `BusRouter` 支撑，当前不具备。

### 4. 完整 IRQ 重构

不实现：
- virtio-blk INTx 注入（除非用户明确要求）
- MSI/MSI-X 中断路径
- `VirtioInterrupt` trait（cloud-hypervisor 风格的中断回调抽象）
- interrupt controller 与设备之间的通用 `IrqSink` 接口

**依据**：CLAUDE.md "Do not continue virtio-blk INTx wiring for the OVMF DXE path unless the user explicitly reopens it"；`05-model-first-device-core.md` "没有接 `X86IoApic` / INTx"；当前 OVMF DXE virtio-blk 驱动轮询 `Used.Idx`。

### 5. 其他明确不做

- 不拆分 `virtio_blk.rs` 为 `virtio/queue.rs` + `virtio_blk.rs`（当前文件 960 行，不需要为了拆而拆）
- 不抽象 `BlockBackend` trait（当前只有一个 `MemoryDisk` 后端）
- 不做 disk 写回 rootfs（OVMF smoke 只读）
- 不做 multi-queue
- 不做 async I/O / epoll worker
- 不做 rate limiter
- 不做 migration / snapshot
- 不做 discard / write zeroes
- 不做 `VIRTIO_F_NOTIFY_ON_EMPTY` / `VIRTIO_F_RING_EVENT_IDX` feature 协商（当前 `REG_DEVICE_FEATURES` 返回 0，OVMF 不要求这些 feature）

## E. 最小测试矩阵

以下测试用于验证当前标准化状态的正确性，不覆盖未来扩展。

### 单元测试（已有，需保持通过）

| 测试名 | 验证点 |
|---|---|
| `reports_capacity_from_installed_disk_image` | capacity low/high 寄存器返回正确值 |
| `read_request_copies_sector_into_guest_buffer_and_publishes_used_ring` | T_IN 请求：sector 数据搬运到 guest buffer，used ring 正确发布 |
| `write_request_updates_backend_and_reports_only_status_byte_used` | T_OUT 请求：guest buffer 数据写入 disk backend，used_len=1 |
| `notify_without_available_request_returns_idle_notify_result` | 无 pending request 时 notify 返回 idle |
| `completed_request_reports_notify_result_and_sets_queue_isr` | request 完成后 ISR queue bit 置位，read ISR 清零 |
| `read_request_without_data_descriptors_reports_ioerr` | 缺少 data descriptor 的 T_IN 返回 IOERR |
| `get_id_with_readonly_data_descriptor_reports_ioerr` | T_GET_ID 的 data descriptor 不可写时返回 IOERR |

### 编译检查（已有，需保持通过）

| 命令 | 验证点 |
|---|---|
| `cargo test -p axdevice --lib` | virtio_blk 单元测试 + axdevice 其他 lib 测试 |
| `cargo check -p axvm --target x86_64-unknown-none --no-default-features --features svm` | axvm 编译不引入新依赖 |
| `cargo check -p axvisor --target x86_64-unknown-none --features fs,svm` | axvisor SVM 构建 |
| `cargo check -p axvisor --target x86_64-unknown-none --features fs,vmx` | axvisor VMX 构建 |

### 代码卫生检查（已有，需保持通过）

| 检查 | 命令 |
|---|---|
| 无旧 hack 残留 | `rg "rewrite desc\|rewrite_ovmf\|translated_queue_pfn\|OvmfVirtioBlkIoState" tgoskits/virtualization tgoskits/os` |
| 格式 | `cargo fmt --check --package axdevice --package axvm --package axvisor` |

### Smoke 测试（已有，需保持通过）

| 信号 | 含义 |
|---|---|
| `Loading virtio-blk disk image from /guest/disks/uefi-boot-test.img` | disk image 加载成功 |
| `FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success` | OVMF 通过 virtio-blk 读到 FAT32 ESP |
| `ArceOS UEFI helloworld` | UEFI app 从 virtio-blk 启动 |
| `ArceOS UEFI shell-stage boot OK` | shell-stage 完成 |
| 无 `Unhandled #PF` | guest memory 访问路径无页错误 |

### 新增验证点（不写代码，仅作为 review checklist）

以下项用于审查任何对 virtio-blk 相关文件的修改：

- [ ] `virtio_blk.rs` 的 `use` 中无 `axvm` crate 引用
- [ ] `virtio_blk.rs` 中无 `AxVM`、`AxVCpu`、`AxVmGuestMemory` 类型引用
- [ ] `axvm/src/vm.rs` 中 virtio-blk 相关函数不超过 5 个（`install_disk_image` + `io_read` + `io_write` + `run_vcpu` 中两处调用）
- [ ] `axvm/src/vm.rs` 中 virtio-blk 相关代码不超过 80 行
- [ ] `LegacyNotifyResult::should_raise_irq` 在 `vm.rs` 中只被 trace/log 消费，不被 `x86_ioapic_assert_gsi()` 消费
- [ ] `process_queue()` / `handle_write()` 的返回值链路完整：`LegacyNotifyResult` -> adapter trace -> 无中断注入
