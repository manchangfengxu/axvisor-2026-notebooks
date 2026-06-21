# Prompt 1: CLAUDE.md 契约合规审计

审计时间: 2026-06-17
审计范围: virtio-blk 实现（`axdevice/src/virtio_blk.rs` + `axvm/src/vm.rs` adapter + `os/axvisor/src/images/mod.rs` loading）
审计基线: `uefi-develop` 分支，当前 smoke 已通过

---

## A. 契约检查

### 符合 CLAUDE.md 的地方

**1. 协议核心归属 axdevice**

- `LegacyVirtioBlk` 定义于 `axdevice/src/virtio_blk.rs:290`
- `LegacyQueue` 定义于 `axdevice/src/virtio_blk.rs:184`
- `VirtioBlkRequest` 定义于 `axdevice/src/virtio_blk.rs:84`
- `Descriptor` 定义于 `axdevice/src/virtio_blk.rs:70`
- `MemoryDisk`（block backend）定义于 `axdevice/src/virtio_blk.rs:298`
- 事实: 所有协议核心符号均在 `axdevice` crate 中，无 `axvm` 依赖。`virtio_blk.rs` 的 import 只有 `ax_errno`、`axaddrspace::GuestMemoryAccessor`、`axdevice_base::{AccessWidth, Port}`、`axvm_types::GuestPhysAddr`。

**2. 通过 GuestMemoryAccessor 访问 GPA descriptor chain**

- `virtio_blk.rs:230`: `fn read_desc<M: GuestMemoryAccessor>`
- `virtio_blk.rs:245`: `fn collect_chain<M: GuestMemoryAccessor>`
- `virtio_blk.rs:263`: `fn pop_available<M: GuestMemoryAccessor>`
- `virtio_blk.rs:275`: `fn publish_used<M: GuestMemoryAccessor>`
- `virtio_blk.rs:92`: `VirtioBlkRequest::from_chain<M: GuestMemoryAccessor>`
- `virtio_blk.rs:451`: `fn execute_read<M: GuestMemoryAccessor>`
- 事实: 所有 guest memory 访问均通过泛型 `M: GuestMemoryAccessor`，无 GPA-to-HPA 原地改写。旧的 `rewrite_ovmf_virtio_blk_descriptors()` 已删除（`03-final-architecture-after-rebase.md` 验证，`rg` 搜索无匹配）。

**3. AxVM 只保留 adapter glue**

`vm.rs` 中 virtio-blk 相关符号清单：
- `vm.rs:130`: `ovmf_virtio_blk: LegacyVirtioBlk` -- 持有设备实例
- `vm.rs:134`: `struct AxVmGuestMemory` -- 构造 GuestMemoryAccessor
- `vm.rs:1101`: `install_virtio_blk_disk_image()` -- 透传 disk bytes
- `vm.rs:1183`: `handle_ovmf_virtio_blk_io_read()` -- port I/O 分发
- `vm.rs:1206`: `handle_ovmf_virtio_blk_io_write()` -- port I/O 分发 + notify 日志

事实: `vm.rs` 中不存在以下逻辑：
- 无 virtqueue descriptor 解析（全在 `LegacyQueue`）
- 无 block request 解析（全在 `VirtioBlkRequest`）
- 无 disk sector 读写逻辑（全在 `MemoryDisk`）
- 无 feature negotiation 逻辑
- 无 ISR 管理逻辑（在 `LegacyVirtioBlk::isr_status`）

**4. LegacyNotifyResult / ISR 语义标准化完成**

- `virtio_blk.rs:37-41`: `LegacyNotifyResult` 包含 `used_any`、`published_used_count`、`should_raise_irq`
- `virtio_blk.rs:384`: `handle_write` 返回 `AxResult<LegacyNotifyResult>`
- `virtio_blk.rs:369-371`: `REG_ISR_STATUS` 读取后清零（`self.isr_status = 0`）
- `virtio_blk.rs:405-407`: notify 后设置 `self.isr_status |= VIRTIO_ISR_QUEUE`
- 事实: 符合 CLAUDE.md "Latest local standardization" 契约。

**5. 不注入 INTx**

- `vm.rs:1231-1236`: `should_raise_irq` 分支只输出 trace 日志，不调用 `x86_ioapic_assert_gsi()`
- 事实: 符合 CLAUDE.md "Do not continue virtio-blk INTx wiring for the OVMF DXE path" 契约。

**6. 不发明 direction-2 本地替代框架**

- `virtio_blk.rs` 中无 `DeviceContext`、`BusRouter`、`IrqSink`
- `vm.rs` 中无此类
- `05-model-first-device-core.md` 明确声明 "Not Changed: 没有引入本地 `DeviceContext`、`IrqSink`、`BusRouter`"
- 事实: 符合。

**7. fw_cfg 已完全提取到 axdevice**

- `axdevice/src/fw_cfg.rs` 拥有 `FwCfgDevice`、`FwCfgInner`、`FwCfgItem`、`FwCfgContent`、`FwCfgFileDirectory`
- `axvm/src/vm.rs` 只提供 guest-memory access 和 narrow I/O glue
- 事实: 符合。

### 偏离或违反 CLAUDE.md 的地方

**偏离 1: `inner_mut` 锁在 queue processing 期间一直持有**

- `vm.rs:1216`: `let mut g = self.inner_mut.lock();`
- `vm.rs:1220`: `let notify = g.ovmf_virtio_blk.handle_write(port, width, val, &mem)?;`
- `handle_write` 内部调用 `process_queue` -> `pop_available` -> `collect_chain` -> `execute_request` -> `execute_read/write/get_id`，所有 guest memory 拷贝和 descriptor chain 遍历都在锁持有期间完成。

CLAUDE.md 契约原文:
> "virtio-blk locking rule: avoid holding the broad `AxVM::inner_mut` lock across protocol-heavy queue processing or guest-memory copies."

事实: 锁跨 protocol-heavy queue processing 和 guest-memory copies 被持有。

但 CLAUDE.md 也允许:
> "If the current infrastructure forces a temporary adapter, keep the lock scope explicit and document it"

`vm.rs:136-137` 的 `AxVmGuestMemory` 注释声明意图解耦:
> "Raw pointer keeps emulated-device DMA from holding AxVMInnerMut while it performs guest-memory reads/writes."

推论: 注释与实际行为不符。raw pointer 允许 `AxVmGuestMemory` 独立于锁的生命周期，但 `g.ovmf_virtio_blk.handle_write(...)` 仍然借用 `g`，所以锁在实际执行期间一直被持有。raw pointer 在当前代码中只在 fw_cfg DMA 路径（`vm.rs:709`）实现了"不持锁"的意图，virtio-blk 路径没有实现。

严重性: 中。当前 OVMF polling 路径单 vCPU，影响有限。但 SMP 下或大 I/O 时会成为瓶颈。

**偏离 2: VirtioBlk 未注册为 AxVmDevices 第一类设备**

- `device.rs:428`: `EmulatedDeviceType::VirtioBlk` 落入 `_ => { warn!(...) }` catch-all 分支
- `ovmf-x86_64-qemu-smp1.toml:68`: `emu_devices = []`
- `LegacyVirtioBlk` 不实现 `BasePortDeviceOps`（签名不兼容: 缺 `GuestMemoryAccessor` 参数，返回类型不同，需要 `&mut self`）
- `vm.rs:654-694`: virtio-blk I/O 通过 special-case `if` 分支绕过 `AxVmDevices::handle_port_read/write` 标准分发路径

CLAUDE.md 契约原文:
> "Current virtio-blk placement note: `virtio_blk.rs` is currently a device core plus AxVM adapter split, not yet a fully integrated `AxVmDevices` device. Keep that distinction explicit until AxVisor's existing infrastructure can carry guest-memory and IRQ needs without inventing a partial local direction-2 framework."

推论: 这是 CLAUDE.md 明确允许的当前状态，不是违反。但它是当前架构与方向二之间最大的结构性差距。

### 尤其回答

**是否把可复用协议逻辑放在了 axdevice?**
是。所有协议逻辑（LegacyVirtioBlk, LegacyQueue, VirtioBlkRequest, Descriptor, MemoryDisk, LegacyNotifyResult, RequestCompletion）均在 `axdevice/src/virtio_blk.rs`。设备核心通过泛型 `GuestMemoryAccessor` 访问 guest memory，不依赖 AxVM。事实。

**是否把 axvm 只保留为 VM 上下文 / guest-memory / dispatch glue?**
是。`vm.rs` 中的 virtio-blk 符号只有 adapter glue（持有实例、构造 accessor、port I/O 分发、disk 透传）。无协议逻辑泄漏。事实。

**是否为了标准化而偷偷跨进了方向二边界?**
没有。没有发明 `DeviceContext`、`BusRouter`、`IrqSink`。没有修改 `BaseDeviceOps` trait。没有把 virtio-blk 硬塞进 `AxVmDevices`。`05-model-first-device-core.md` 明确声明了这些"Not Changed"项。事实。

---

## B. 基础设施利用度检查

### 当前已经利用到的现有基础设施

| 基础设施 | 位置 | 利用方式 | 证据 |
|---|---|---|---|
| `GuestMemoryAccessor` | `axaddrspace/src/memory_accessor.rs` | 设备核心通过泛型 `M: GuestMemoryAccessor` 访问 GPA | `virtio_blk.rs:4` import, `virtio_blk.rs:92,230,245,263,275,451,471,489` 使用 |
| `Port` / `AccessWidth` | `axdevice_base` | 端口编号和访问宽度 | `virtio_blk.rs:5` import |
| `GuestPhysAddr` | `axvm-types` | GPA 类型 | `virtio_blk.rs:6` import |
| `EmulatedDeviceType::VirtioBlk` | `axvm-types/src/lib.rs` | 枚举占位（未被 `init()` 消费） | `device.rs:428` catch-all |
| `x86_ioapic_assert_gsi()` | `axdevice/src/device.rs:526` | 存在但 virtio-blk 未连接 | `03-final-architecture-after-rebase.md:79` |

### 明明已有但还没吃上的基础设施

**1. `BasePortDeviceOps` / `AxVmDevices::handle_port_read/write` 标准分发**

- `device.rs:686-703`: `handle_port_read/write` 已实现完整的 port device 查找和分发
- `device.rs:497-499`: `add_port_dev` 已实现 port device 注册
- virtio-blk 完全绕过这条路径，通过 `vm.rs:654-694` 的 special-case if 分支处理

推论: 这是被 `BaseDeviceOps` trait 签名限制（无 `GuestMemoryAccessor` 参数）阻塞的，不是没有尝试利用。

**2. `EmulatedDeviceType::VirtioBlk` 枚举值**

- `axvm-types/src/lib.rs` 已预留 `VirtioBlk` 枚举
- `device.rs:428` 的 `init()` match 不处理它，直接 warn 并跳过
- 事实: 枚举预留了但消费端没有实现

**3. `x86_ioapic_assert_gsi()` + `inject_pending_ioapic_irq_after_eoi()`**

- `device.rs:526-530`: `x86_ioapic_assert_gsi` 已实现
- `vm.rs` 运行时有 `x86_irq` 模块提供 IRQ 注入路径
- virtio-blk 当前不使用（by design，polling path）

推论: 这条路径已准备好，等待 OVMF 切换到中断驱动模式时接入。

### 不能吃、吃了会越界的基础设施

**1. `BaseDeviceOps` trait 的 `handle_write` 签名**

- 签名: `fn handle_write(&self, addr: R::Addr, width: AccessWidth, val: usize) -> AxResult`
- virtio-blk 需要: `fn handle_write(&mut self, port: Port, width: AccessWidth, value: usize, mem: &M) -> AxResult<LegacyNotifyResult>`
- 差异: 需要 `&mut self`、需要 `GuestMemoryAccessor` 参数、返回 `LegacyNotifyResult`

如果修改 `BaseDeviceOps` 来适配 virtio-blk，会同时影响所有其他设备（IOAPIC、PIT、Serial、fw_cfg、DebugCon），并且会触及 direction-2 的公共 trait 设计。这是 CLAUDE.md 明确禁止的:
> "Direction 2 compatibility rule: do not invent a partial `DeviceContext`, `BusRouter`, or `IrqSink` in direction 1."

**2. `AxVmDevices` 中的 `x86_fw_cfg` 字段模式**

- fw_cfg 能作为 `AxVmDevices` 第一类设备，是因为它只通过 port I/O（无 guest memory 参数）访问
- virtio-blk 的 notify 路径需要 guest memory access（queue 处理），这个需求无法用当前 `BasePortDeviceOps` 表达
- 如果为 virtio-blk 在 `AxVmDevices` 中加一个 `Option<Arc<LegacyVirtioBlk>>` 字段，会复制 fw_cfg 的模式但无法通过标准 dispatch 路径访问，造成不一致

---

## C. 具体问题清单（按严重性排序）

### 问题 1: `inner_mut` 锁跨 queue processing 持有

**问题**: `vm.rs:1216-1220` 中 `self.inner_mut.lock()` 在 `handle_write` 期间一直持有，覆盖了整个 `process_queue` -> descriptor chain 遍历 -> guest memory 拷贝。

**证据**:
- `vm.rs:1216`: `let mut g = self.inner_mut.lock();`
- `vm.rs:1220`: `let notify = g.ovmf_virtio_blk.handle_write(port, width, val, &mem)?;`
- `virtio_blk.rs:415-426`: `process_queue` 包含循环遍历多个 request
- `vm.rs:136-137`: `AxVmGuestMemory` 注释声称 "keeps emulated-device DMA from holding AxVMInnerMut"，但 virtio-blk 路径未实现此解耦

**为什么是"非标准"而不是单纯未完成**: 这不是缺失功能，而是一个已知的架构偏差。`AxVmGuestMemory` 的 raw pointer 设计意图是解耦 guest memory access 和锁持有，但 `g.ovmf_virtio_blk.handle_write()` 仍然借用 `g`，使 raw pointer 的解耦意图在 virtio-blk 路径上落空。fw_cfg DMA 路径（`vm.rs:705-710`）正确实现了"先释放锁、再用 raw pointer 访问 memory"的模式，但 virtio-blk 没有。

**修正时必须守住的边界**: 不能为此发明 `DeviceContext` 或修改 `BaseDeviceOps`。修正方向应是：在 `vm.rs` adapter 层将锁拆分为"获取 address_space raw pointer"和"调用设备核心"两步，参照 fw_cfg DMA 路径的已有模式。

### 问题 2: virtio-blk 绕过 AxVmDevices 标准 port 分发

**问题**: `vm.rs:654-694` 中 virtio-blk 通过 `handle_ovmf_virtio_blk_io_read/write` special-case 绕过了 `AxVmDevices::handle_port_read/write`（`device.rs:686-703`）。如果 guest 访问 `0x6000-0x607f` 范围内的端口但该端口未被 virtio-blk special-case 捕获（例如写入 `0x6020`），代码会 fall through 到 `self.get_devices().handle_port_read/write`，而 `AxVmDevices` 中没有注册任何覆盖该范围的 port device，会导致 `panic_device_not_found`（`device.rs:165-176`）。

**证据**:
- `vm.rs:683-688`: `if self.handle_ovmf_virtio_blk_io_write(port, width, data as usize)? { // 成功由 ovmf_virtio_blk 处理 }`
- `vm.rs:691-693`: fall through 到 `self.get_devices().handle_port_write(port, width, data as usize)?`
- `device.rs:696-703`: `handle_port_write` 中 `find_port_dev` 找不到设备会 panic
- `device.rs:428`: `EmulatedDeviceType::VirtioBlk` 在 `init()` 中落入 warn catch-all，不注册任何 port device

**为什么是"非标准"而不是单纯未完成**: 这是 virtio-blk 未注册为 `AxVmDevices` 第一 class 设备的直接后果。CLAUDE.md 允许这种 split，但 split 意味着 adapter 层必须完美覆盖设备的端口范围。当前 `LEGACY_BLK_IO_BASE..LEGACY_BLK_IO_BASE+LEGACY_BLK_IO_SIZE` 的 range check 在 `handle_ovmf_virtio_blk_io_write` 的第一行（`vm.rs:1212`），确保了未命中范围会返回 `false`。但如果 VM 配置中的 `passthrough_devices` 也覆盖了 `0x6000` 附近区域，可能产生歧义。

**修正时必须守住的边界**: 不能为了修复分发路径而发明 `BusRouter`。当前的 special-case 模式在方向二之前是被允许的临时 adapter。

### 问题 3: `LegacyVirtioBlk::handle_read` 需要 `&mut self`

**问题**: `virtio_blk.rs:357`: `pub fn handle_read(&mut self, ...)` 需要可变借用，因为 `REG_ISR_STATUS` 读取后要清零（`virtio_blk.rs:369-371`: `self.isr_status = 0`）。这导致 `LegacyVirtioBlk` 无法实现 `BasePortDeviceOps`（其 `handle_read` 签名要求 `&self`）。

**证据**:
- `virtio_blk.rs:357`: `pub fn handle_read(&mut self, port: Port, width: AccessWidth) -> AxResult<usize>`
- `virtio_blk.rs:369-371`: `(REG_ISR_STATUS, AccessWidth::Byte) => { let value = self.isr_status; self.isr_status = 0; Ok(value as usize) }`
- `axdevice_base/src/lib.rs:223`: `fn handle_read(&self, addr: R::Addr, width: AccessWidth) -> AxResult<usize>` -- `&self`

**为什么是"非标准"而不是单纯未完成**: 这不是遗漏，而是 virtio 协议的 ISR clear-on-read 语义与 `BaseDeviceOps` 不可变借用设计之间的根本矛盾。其他设备（fw_cfg、debugcon）不需要状态修改，所以 `&self` 足够。但 virtio ISR 的 read-clear 行为是 virtio 规范要求的。

**修正时必须守住的边界**: 不能为了适配而把 ISR clear 逻辑移到 adapter 层（这会把协议语义泄漏到 axvm）。如果将来要把 virtio-blk 接入 `BasePortDeviceOps`，需要在 direction-2 中讨论是否给 `handle_read` 加 `&mut self` 或用内部可变性（`Cell`/`AtomicU8`）包装 `isr_status`。

### 问题 4: disk 加载全量读入内存，无 write-back

**问题**: `images/mod.rs:624-632`: `load_virtio_blk_disk_from_filesystem` 将整个 disk image 读入 `Vec<u8>`，交给 `LegacyVirtioBlk::install_disk_image`。`MemoryDisk`（`virtio_blk.rs:298`）是纯内存后端，写请求只修改内存副本，不回写 rootfs 文件。

**证据**:
- `images/mod.rs:625`: `let disk = fs::read_image_file(disk_path)?;`
- `images/mod.rs:631`: `self.vm.install_virtio_blk_disk_image(disk);`
- `03-final-architecture-after-rebase.md:179`: "raw disk 读进内存后交给设备模型。当前不会把写请求回写到 rootfs 文件。"

**为什么是"非标准"而不是单纯未完成**: 这是 bring-up 阶段的有意简化。OVMF boot path 只需要 read（读 GPT、FAT、BOOTX64.EFI），不需要 write-back。但如果后续要支持 OS 安装或 runtime write，这会成为功能缺口。

**修正时必须守住的边界**: 不能在 direction-1 阶段引入 async I/O 或 file backend。CLAUDE.md 明确禁止 "async worker / eventfd"（`03-final-architecture-after-rebase.md:378`）。

### 问题 5: `vm.rs` 中 I/O dispatch 的 debug 级别 `info!` 日志

**问题**: `vm.rs:1198-1202` 和 `vm.rs:1222-1229` 在每次 port I/O 读写时输出 `info!` 级别日志。virtio-blk 的 notify 路径在 OVMF boot 期间可能触发数百次。

**证据**:
- `vm.rs:1198`: `info!("[OVMF-VIRTIO-BLK-IO] in port={:#x} ...")`
- `vm.rs:1222`: `info!("[OVMF-VIRTIO-BLK-IO] out port={:#x} ...")`

**为什么是"非标准"而不是单纯未完成**: 这是 bring-up 阶段的取证日志。`07-ovmf-polling-path-cleanup.md` 已经做过一轮取证日志回退，但这两个 `info!` 保留了下来。对于生产代码应降级为 `trace!` 或 `debug!`。

**修正时必须守住的边界**: 纯日志级别调整，不涉及架构边界。

---

## D. 结论

**当前最大问题是设备边界。**

证据:

1. `LegacyVirtioBlk` 无法实现 `BasePortDeviceOps`（签名不兼容: 缺 `GuestMemoryAccessor`、需 `&mut self`、返回 `LegacyNotifyResult`），导致它不是 `AxVmDevices` 第一类设备。`device.rs:428` 中 `EmulatedDeviceType::VirtioBlk` 落入 warn catch-all。

2. virtio-blk 的 port I/O 通过 `vm.rs:654-694` 的 special-case if 分支绕过 `AxVmDevices::handle_port_read/write` 标准分发路径。这条 special-case 是 adapter 层的临时方案，不是基础设施的一部分。

3. 锁持有问题（问题 1）的根因也是设备边界: 如果 virtio-blk 是 `AxVmDevices` 的第一 class port device，标准 dispatch 路径可以在适当时机管理锁范围；但现在 adapter 层必须自己管理锁，而 `g.ovmf_virtio_blk.handle_write()` 的借用关系使锁解耦变得困难。

4. CLAUDE.md 对此有明确的定位: "`virtio_blk.rs` is currently a device core plus AxVM adapter split, not yet a fully integrated `AxVmDevices` device." 以及 "if current AxVisor infrastructure cannot yet carry a device as a first-class `AxVmDevices` device, keep a stable split."

协议语义（LegacyNotifyResult、ISR、RequestCompletion、validate_data_descriptors）是干净的，代码组织（axdevice core vs axvm adapter vs os/axvisor loading）也是清晰的。阻塞点在于 `BaseDeviceOps` trait 无法表达 virtio-blk 的 `GuestMemoryAccessor` 需求，这不是当前 workstream 应修复的（会触及 direction-2 公共 trait），而是方向二重构时需要解决的结构性问题。

在方向二之前，当前的 device-core + adapter split 是 CLAUDE.md 允许的正确状态。需要守住的边界是: 不为此发明部分方向二框架，不修改 `BaseDeviceOps`，不在 `AxVmDevices` 中加无法通过标准 dispatch 访问的 virtio-blk 字段。
