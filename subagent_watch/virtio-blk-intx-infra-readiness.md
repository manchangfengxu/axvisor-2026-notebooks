# virtio-blk INTx 接线分析

## 分析目标

判断 AxVisor 现有基础设施是否足够支持 legacy virtio-blk 走 INTx / vIOAPIC 路径。

---

## 1. 现有可复用基础设施

| 文件路径 | 符号 | 职责 | 是否足够支撑 virtio-blk INTx |
|---|---|---|---|
| `tgoskits/virtualization/axdevice/src/virtio_blk.rs` | `LegacyVirtioBlk` / `LegacyNotifyResult.should_raise_irq` | notify 后返回是否应触发 IRQ；已设 `isr_status` 字段 + `VIRTIO_ISR_QUEUE` | 足够。设备核心已完成 notify→ISR 状态转换 |
| `tgoskits/virtualization/axdevice/src/virtio_blk.rs` | `REG_ISR_STATUS = 0x13` / `LegacyVirtioBlk::handle_read` 读 ISR 后清零 | 读 ISR 寄存器触发自清除 | 足够。标准 legacy virtio ISR 语义 |
| `tgoskits/virtualization/axdevice/src/device.rs` | `AxVmDevices::x86_ioapic` 字段 | 持有 `Option<Arc<EmulatedIoApic>>` | 足够。但当前 OVMF VM 配置未初始化此字段 |
| `tgoskits/virtualization/axdevice/src/device.rs` | `AxVmDevices::x86_ioapic_assert_gsi(gsi) -> Option<IoApicInterrupt>` | 将 GSI 翻译为 guest vector 并路由 | 足够。已有完整 GSI→vector 翻译逻辑 |
| `tgoskits/virtualization/axdevice/src/device.rs` | `AxVmDevices::x86_ioapic_end_of_interrupt(vector)` | EOI 后检查 pending level 并触发 re-assert | 足够。level-triggered 语义已实现 |
| `tgoskits/virtualization/x86_vlapic/src/vioapic.rs` | `EmulatedIoApic::assert_gsi()` / `IoApicInterrupt` | 检查 mask/delivery mode，设置 remote-IRR，返回 vector + trigger mode | 足够。edge 和 level trigger 均支持 |
| `tgoskits/virtualization/x86_vlapic/src/vioapic.rs` | `EmulatedIoApic::new_default()` | 创建 GPA `0xFEC0_0000` 标准 IOAPIC | 足够。PC 标准地址 |
| `tgoskits/virtualization/axdevice/src/device.rs` | `AxVmDevices::init()` 对 `EmulatedDeviceType::X86IoApic` 的分支（行 373-393） | 根据配置创建 `EmulatedIoApic` 并加入 mmio 设备列表 | 足够。已存在初始化路径，只需配置里补一行 |
| `tgoskits/virtualization/axvm/src/vm.rs` | `AxVmGuestMemory` + `impl GuestMemoryAccessor` | 给设备模型提供 GPA→HVA 读写能力 | 足够。已在 virtio-blk adapter 中使用 |
| `tgoskits/virtualization/axvm/src/vm.rs` | `handle_ovmf_virtio_blk_io_write` (行 1206-1233) | adapter 已捕获 `LegacyNotifyResult.should_raise_irq` | 部分足够。捕获了结果，但只打日志，未注入 |
| `x86_vcpu` crate | `vcpu.inject_interrupt_with_trigger(vector, trigger_mode)` | 向 vCPU 注入中断 | 足够。已在 `x86_irq.rs` 中被 PIT/serial/passthrough 使用 |
| `axvm-types/src/lib.rs` | `EmulatedDeviceType::X86IoApic` | 配置枚举值 | 足够。可直接用于 `emu_devices` 配置 |

---

## 2. 最小接线路径

**第 1 步：改 VM 配置，补 `X86IoApic` emu_device**

文件：`tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`

复用：`AxVmDevices::init()` 已有 `EmulatedDeviceType::X86IoApic` 分支，会自动创建 `EmulatedIoApic` 并注册为 mmio 设备。

为什么不需要新接口：这条路径是已有配置链，只需在 `emu_devices` 数组里加一行 TOML。

---

**第 2 步：确认 passthrough_devices 里 `IO APIC` 行不会冲突**

文件：`tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` 行 69

复用：`AxVM::init()` 里 passthrough 设备映射逻辑。

原因：当前 passthrough_devices 有 `["IO APIC", 0xfec0_0000, ...]`，如果同时把 IOAPIC 设为 emu_device，会存在同一个 GPA 既做 passthrough 映射又做 mmio 拦截的冲突。要移除 passthrough_devices 里的 IO APIC 行，或者只保留 emu_device 版本。

---

**第 3 步：在 `AxVM::handle_ovmf_virtio_blk_io_write` 里注入 IRQ**

文件：`tgoskits/virtualization/axvm/src/vm.rs`，行 1230-1233 处

复用：`AxVmDevices::x86_ioapic_assert_gsi(gsi)` 已有。将 trace 日志行替换为实际调用 `assert_gsi` + `vcpu.inject_interrupt_with_trigger()`。

为什么不需要新接口：这段代码在已有的 AxVM adapter 函数里，不扩展公共 trait，不新建设备类型。

---

**第 4 步：确定 virtio-blk 使用的 GSI**

文件：`tgoskits/virtualization/axvm/src/vm.rs`

复用：可调常量 `VIRTIO_BLK_GSI`，默认候选值 18（来自 `tgoskits/os/axvisor/src/images/x86/mptable.rs` 行 150 的 `pci_intx_gsi(dev=3, pin=0) = 18` 声明）。但 GSI 18 与标准 q35 PIRQ 路由公式矛盾（slot 3 INTA 标准计算得 GSI 23），且 AxVisor 没有自己的 PCI / ACPI IRQ 路由模型；nested OVMF 虽可直接枚举 outer QEMU 的 PCI config space，但这条发现链没有导向 emulated IOAPIC，因此无法确信 OVMF 会 unmask 对应的 IOAPIC redirection entry。

为什么不需要新接口：当前没有 AxVisor 自己的 PCI configuration / BAR / IRQ 路由模型可复用到这个局部改动里。一个可调常量本来足够作为临时接线手段，但最新运行证据表明 OVMF 没有编程 emulated IOAPIC，所以这一步先停在证据收集。详见 `axvisor-2026-notebooks/subagent_watch/virtio-blk-gsi-evidence.md`。

---

**第 5 步：在 `x86_irq.rs` 增加非 passthrough 的 emulated IRQ 注入函数（或在 adapter 里直接注入）**

文件：`tgoskits/virtualization/axvm/src/runtime/x86_irq.rs` 或 `axvm/src/vm.rs`

复用：`EmulatedIoApic::assert_gsi()` + `vcpu.inject_interrupt_with_trigger()`，复用和 `inject_due_pit_irq0` 完全相同的 vCPU 注入调用。

为什么不需要新接口：这是在已有函数旁加一个窄的注入入口，不引入 `IrqSink` 或 `DeviceContext`。两种实现方式：
- 方式 A：直接在 `handle_ovmf_virtio_blk_io_write` 里调用（最窄，不改 x86_irq.rs）
- 方式 B：在 x86_irq.rs 里加一个 `inject_emulated_ioapic_irq_for_device` 函数（稍宽，但保持 IRQ 注入收口在一个文件）

方式 A 更符合当前 adapter 胶水定位。

---

**第 6 步：确认 `vcpu` 引用在 adapter 调用点可用**

文件：`tgoskits/virtualization/axvm/src/vm.rs`，`run_vcpu` 里的 `IoWrite` 分支（行 679-711）

复用：`vcpu` 引用已在 `run_vcpu` 闭包参数里，可直接传给注入调用。

原因：不需要额外的 vCPU 引用查找机制。

---

**第 7 步：测试 edge-triggered 路径**

文件：无代码修改，只验证

复用：OVMF 默认将 IOAPIC redirection entry 配置为 edge-triggered（unmasked，delivery mode fixed）。`EmulatedIoApic::assert_gsi()` 对 edge-triggered 不设 remote-IRR，`IoApicInterrupt.level_triggered = false`，直接注入。

---

## 3. 冲突和风险

**冲突 1：`interrupt_mode = "passthrough"` 与 `x86_irq.rs` 全局检查**

文件：`tgoskits/virtualization/axvm/src/runtime/x86_irq.rs`，`inject_due_pit_irq0` 行 38，`inject_pending_serial_irq` 行 63

符号：`vm.interrupt_mode() != VMInterruptMode::Passthrough` guard

现状：`x86_irq.rs` 中所有 IRQ 注入函数都守着 `interrupt_mode == Passthrough`，但 OVMF VM 配置正是 `interrupt_mode = "passthrough"`。如果直接复用这些函数，条件会通过。

风险：当前 guard 里的逻辑是"把 host 物理 IOAPIC 中断转发给 guest"，不是"从 emulated IOAPIC 注入"。如果在 `handle_ovmf_virtio_blk_io_write` 里直接调用 `AxVmDevices::x86_ioapic_assert_gsi()`，绕过 x86_irq.rs，guard 不会阻拦。风险可控。

---

**冲突 2：`emu_devices = []` 与 `x86_ioapic` 未初始化**

文件：`tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` 行 62

符号：`emu_devices = []`

现状：`AxVmDevices::new()` 里 x86 路径会自动创建 `fw_cfg` 和 `OvmfDebugCon`（行 199-203），但 `x86_ioapic` 只在 `emu_configs` 含 `EmulatedDeviceType::X86IoApic` 时才初始化（行 373-393）。配置为空意味着 `x86_ioapic = None`，`x86_ioapic_assert_gsi()` 直接返回 `None`。

风险：不改配置就接 INTx，注入会静默失败。这是最大阻塞点。

---

**冲突 3：passthrough_devices 里 IO APIC 与 emu_device 里 X86IoApic 的 GPA 冲突**

文件：`tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` 行 69

符号：`["IO APIC", 0xfec0_0000, 0xfec0_0000, 0x1000, 0x1]`

现状：`0xFEC0_0000` 在 passthrough_devices 里做 GPA→HPA 线性映射，guest IOAPIC MMIO 访问直接穿透到 host 物理 IOAPIC。如果同时把 `0xFEC0_0000` 注册为 emu_device mmio 设备，会产生同一个 GPA 既被 stage-2 直通映射又被设备模拟覆盖的未定义行为。

风险：必须二选一。选择 emulated IOAPIC 就必须从 passthrough_devices 移除 IO APIC 行。

---

**冲突 4：`runtime/x86_irq.rs` 不是通用设备 IRQ 框架**

文件：`tgoskits/virtualization/axvm/src/runtime/x86_irq.rs`

符号：整个文件

现状：函数硬编码了 PIT（GSI 0）、COM1（GSI 4）的注入逻辑，以及 host IOAPIC 中断转发逻辑。没有通用的 `device_request_irq(gsi)` 接口。现有函数和 virtio-blk IRQ 没有直接关联。

风险：如果要在 x86_irq.rs 里加通用路径，范围会扩大；如果直接在 adapter 里注入，范围最小，但和 PIT/serial 的收口方式不一致。建议先走 adapter 直接注入。

---

**冲突 5：passthrough_devices 里的 IOAPIC MMIO 映射会绕过 emulated IOAPIC**

文件：`tgoskits/virtualization/axvm/src/vm.rs`，`init()` 行 347-357

符号：`pt_dev_region` + `map_linear(GUEST→HOST, DEVICE|READ|WRITE|USER)`

现状：passthrough 映射在 stage-2 page table 里建立后，guest 对 `0xFEC0_0000` 的 MMIO 访问不会再触发 vmexit，直接穿透到 host。这意味着 OVMF 目前看到的 IOAPIC 是 host 物理 IOAPIC，不是 AxVisor 的 `EmulatedIoApic`。如果 OVMF 配置了 host IOAPIC 的 redirection entry，注入的是 host vector，不是 guest vector。

风险：必须移除 IOAPIC passthrough 行，切换到 emulated IOAPIC，才能让 guest IOAPIC redirection entry 控制中断路由。

---

**冲突 6：`LegacyNotifyResult` 当前只在 adapter 里打印日志**

文件：`tgoskits/virtualization/axvm/src/vm.rs`，行 1230-1233

符号：`trace!("[OVMF-VIRTIO-BLK-IO] queue completed; INTx injection is not wired yet")`

现状：注入判断已捕获，但注入代码缺失。这是最小改动点。

---

## 4. 明确禁止项

1. 不引入 `DeviceContext`、`IrqSink`、`BusRouter` 接口。这些属于方向二。

2. 不把 `LegacyVirtioBlk` 注册成 `AxVmDevices` 的通用 `BasePortDeviceOps`。当前 `BaseDeviceOps::handle_write` 没有 `GuestMemoryAccessor` 参数，硬注册会迫使改公共 trait。

3. 不在 AxVisor 本地实现 MSI / MSI-X 支持。INTx 是当前 legacy transport 的正确路径。

4. 不在 `x86_irq.rs` 里引入通用 `DeviceIrqSink` 路由表。adapter 直接注入是正确边界。

5. 不恢复 descriptor GPA→HPA 改写路径。`GuestMemoryAccessor` 已足够。

6. 不改 `LegacyVirtioBlk` 内部的 `should_raise_irq` 判断逻辑。当前语义（至少发布一个 used buffer）就是 legacy virtio 的正确 IRQ 触发条件。

7. 不在本地实现 PCI config space 或 BAR 分配。GSI 用可调常量，启动后通过 IOAPIC 写日志验证。

---

## 5. 参考实现对照

### A. 可直接借鉴的行为

| 参考文件 + 符号 | 本地对应 + 符号 | 为什么现在能用 |
|---|---|---|
| QEMU `virtio_blk_req_complete` 行 57-69：`virtqueue_push` + `stb_p(&req->in->status, status)` + `virtio_notify` | `axdevice/src/virtio_blk.rs` `LegacyVirtioBlk::process_queue` + `write_status` + `LegacyNotifyResult.should_raise_irq` | AxVisor 已实现相同语义：publish used ring + 写 status byte + 判断是否应触发 IRQ。无需额外借鉴 |
| QEMU `virtio_split_should_notify` 行 2612-2632：检查 `VRING_AVAIL_F_NO_INTERRUPT` 标志，若未设则始终 notify | `axdevice/src/virtio_blk.rs` `LegacyNotifyResult::completed_one()` 始终设 `should_raise_irq = true` | AxVisor 当前不检查 `VRING_AVAIL_F_NO_INTERRUPT`，对 legacy 传输这是正确简化。OVMF legacy virtio-blk 没有使用 suppress interrupt 机制 |
| QEMU `virtio_blk_req_complete` → `virtio_notify` → `virtio_irq` → ISR 置位 + MSI/INTx 触发 | `axdevice/src/virtio_blk.rs` `isr_status |= VIRTIO_ISR_QUEUE` + `axvm/vm.rs` adapter 捕获 `should_raise_irq` | AxVisor 已有 ISR 置位，只差最后一步 vIOAPIC 注入 |
| QEMU `virtio_set_isr(vdev, 0x1)` (queue bit) 和 `virtio_set_isr(vdev, 0x3)` (config+queue) | `axdevice/src/virtio_blk.rs` `VIRTIO_ISR_QUEUE = 1` + `isr_status |= VIRTIO_ISR_QUEUE` | 完全一致的 ISR 语义，`REG_ISR_STATUS` 读取后清零也已实现 |
| Cloud Hypervisor `signal_used_queue` 行 585-587：`interrupt_cb.trigger(VirtioInterruptType::Queue(queue_index))` | `axvm/src/vm.rs` adapter `handle_ovmf_virtio_blk_io_write` + `AxVmDevices::x86_ioapic_assert_gsi()` | CH 用 eventfd+callback，AxVisor 用同步 VM-exit 路径。本质相同：设备完成→触发中断。AxVisor 路径更简单，不需要 eventfd 抽象 |

### B. 不能直接借鉴的行为

| 参考文件 + 符号 | 为什么 AxVisor 当前承不住 | 如果硬搬会踩到的契约问题 |
|---|---|---|
| QEMU `virtio_notify` → `k->notify(qbus->parent, vector)`：通知路径走 PCI bus → virtio-bus → virtio-device 层级 | AxVisor 没有 PCI bus 抽象，没有 `virtio-bus` 层，没有 `notify` 回调链 | 违反契约：不允许新造通用 BusRouter / DeviceContext |
| QEMU `vring_get_used_event()` / `VRING_AVAIL_F_NO_INTERRUPT`：used_event 机制 | AxVisor 的 `LegacyQueue` 不跟踪 `used_event` 字段，也不读 `avail_ring` 的 flags 字段 | 当前 `LegacyNotifyResult` 始终 notify，对 legacy 是正确的。引入 used_event 要改 `LegacyQueue::publish_used` 和 `LegacyQueue::pop_available`，属于打包队列升级，不是当前目标 |
| QEMU `virtio_irqfd` / `msix_notify`：MSI-X + irqfd 异步触发 | AxVisor 没有 MSI-X 基础设施，没有 irqfd，没有 eventfd 主机侧机制 | 违反契约：不允许在本地实现 MSI/MSI-X |
| CH `VirtioInterrupt::trigger(VirtioInterruptType::Queue)`：通用 VirtioInterrupt trait | AxVisor 没有 `VirtioInterrupt` trait，没有 eventfd 回调注册机制 | 违反契约：不允许新造通用 IrqSink 或 interrupt callback 框架 |
| CH `EpollHelper` + `QUEUE_AVAIL_EVENT`：epoll 驱动的异步 worker | AxVisor 的设备模型是同步 VM-exit 路径，没有 epoll worker | 违反契约：不允许引入 async worker。当前架构同步足够 |

---

## 6. 结论

**现有基础设施在结构上足够，但当前运行时证据不允许接线。**

具体差距清单：

1. **配置阻塞**：`emu_devices = []`，需加 `X86IoApic` emu_device（改一行 TOML，复用 `AxVmDevices::init()` 已有路径）
2. **GPA 冲突**：passthrough_devices 里 `["IO APIC", 0xfec0_0000, ...]` 需移除（否则 emulated IOAPIC 被 passthrough 映射覆盖）
3. **注入缺失**：`axvm/src/vm.rs` 行 1230-1233 的 trace 需替换为实际 `x86_ioapic_assert_gsi(gsi)` + `vcpu.inject_interrupt_with_trigger()` 调用
4. **GSI 定义**：旧候选 GSI = 18（`mptable.rs` 行 150 的 `pci_intx_gsi(dev=3, pin=0)`）已经不足以支撑接线。最新真实运行里，emulated IOAPIC 已初始化、`LegacyNotifyResult.should_raise_irq` 已大量触发，但 `EmulatedIoApic::write_selected_register()` 没有看到任何 IOREDTBL 写入，说明 nested OVMF 没有 unmask 任何 GSI。详见 `virtio-blk-gsi-evidence.md`。

以上四项都在已有架构边界内，但当前阻塞已经从“基础设施不够”变成“运行时证据表明 guest 没有编程 vIOAPIC”。在这个证据变化之前，不应该继续做 INTx 接线实现。

**运行时补充**：

1. `tmp/configs/ovmf-x86_64-qemu-smp1.toml` 切到 emulated IOAPIC 后，日志中出现 `x86 IO APIC initialized with base GPA 0xfec00000 and length 0x1000`。
2. 同一次运行里，`AxVM::handle_ovmf_virtio_blk_io_write()` 多次打印 `irq=true`，说明设备完成路径已经走到“应该触发中断”的阶段。
3. 但完整日志没有任何 `vIOAPIC IOREGSEL write ...`、`vIOAPIC IOWIN read ...`、`vIOAPIC IOWIN write ...`，更没有 `vIOAPIC redirection entry ...`，说明 OVMF 根本没碰 emulated IOAPIC。
4. OVMF 仍然从 `PciRoot(0x0)/Pci(0x1,0x0)` 启动 `\EFI\BOOT\BOOTX64.EFI`，说明它看见的是 outer QEMU 的 PCI 设备，而不是 AxVisor 自己定义的 IRQ 路由。

---

## 本次文档修改记录

新增文件：

- `axvisor-2026-notebooks/subagent_watch/virtio-blk-intx-infra-readiness.md`
