# Prompt 2: 成熟实现对照差距矩阵

## 1. 成熟实现共同点

### 1.1 Queue/Descriptor Layering

**共同结构**：QEMU 和 cloud-hypervisor 都将 virtqueue 操作收敛到独立的 queue 抽象层，设备核心不直接操作 desc table / avail ring / used ring 的原始字节。

- **QEMU**: `virtqueue_pop()` / `virtqueue_push()` / `virtio_notify()` 构成完整的 queue API。`virtio_blk_get_request()` 通过 `virtqueue_pop(vq, sizeof(VirtIOBlockReq))` 获取请求，`virtio_blk_req_complete()` 通过 `virtqueue_push()` + `virtio_notify()` 提交完成。设备代码永远不直接读写 descriptor 表或 avail/used ring 字节。 (`virtio-blk.c:172-178`, `virtio-blk.c:57-69`)
- **cloud-hypervisor**: `Queue` (from `virtio_queue` crate) 封装了完整的 split virtqueue 状态机。`queue.iter()` 返回 `DescriptorChain` 迭代器，`queue.add_used()` 提交完成，`queue.needs_notification()` 检查 event_idx，`queue.enable_notification()` 重新开启通知。设备 handler 通过 `queue.go_to_previous_position()` 回退以支持 rate limiter。 (`block.rs:231-237`, `block.rs:264-270`, `block.rs:386-399`)

**共同语义**：
- 都支持 `VIRTIO_RING_F_EVENT_IDX`：cloud-hypervisor 显式调用 `queue.set_event_idx(event_idx)` 并在处理完成后检查 `needs_notification()` (`block.rs:1100-1101`, `block.rs:386-399`)。QEMU 通过 `virtio_queue_get_notification()` / `virtio_queue_set_notification()` 控制抑制 (`virtio-blk.c:1021-1041`)。
- 都有 head index 重复检测：cloud-hypervisor 显式检查 `is_head_in_flight()` (`block.rs:209-215`)，QEMU 通过 `VirtIOBlockReq` 链表跟踪未完成请求 (`virtio-blk.c:1074-1097`)。
- 都有请求批处理/合并：QEMU 有 `MultiReqBuffer` + `virtio_blk_submit_multireq()` (`virtio-blk.c:221-341`)，cloud-hypervisor 有 batch request 提交 (`block.rs:314-383`)。

### 1.2 Device Core / Transport Separation

**共同结构**：设备核心（block I/O 语义）与传输层（PCI/MMIO/legacy I/O 端口）严格分离。

- **QEMU**: `virtio-blk.c` 是纯设备核心，通过 `virtio_blk_handle_output()` 回调被 VirtQueue 层调用。`virtio-blk-pci.c` 是完全独立的 PCI 绑定层，只负责 PCI BAR/MSI-X 映射和 `qdev_realize()`，通过 `VirtIOBlkPCI { VirtIOPCIProxy parent_obj; VirtIOBlock vdev; }` 组合设备核心。 (`virtio-blk-pci.c:36-39`, `virtio-blk-pci.c:49-64`)
- **cloud-hypervisor**: `Block` 结构体实现 `VirtioDevice` trait（设备核心），通过 `VirtioCommon` 与传输层解耦。`BlockEpollHandler` 持有 `Queue`（传输层抽象）而非 PCI 寄存器。传输层 `VirtioPciDevice` 在 `transport/mod.rs` 中定义 `VirtioTransport` trait。 (`block.rs:710-727`, `block.rs:1038-1040`, `transport/mod.rs:11-13`)

### 1.3 Guest Memory Access Boundary

**共同结构**：设备核心通过内存抽象层访问 guest 内存，不持有 VM/地址空间的直接引用。

- **QEMU**: 通过 `QEMUIOVector` + `iov_from_buf()` / `iov_to_buf()` / `iov_size()` 操作 guest 内存。`VirtQueueElement` 中的 `in_sg[]` / `out_sg[]` scatter-gather 数组由 VirtQueue 层填充，设备核心使用 iovec 抽象，不涉及 GPA/HPA 转换。 (`virtio-blk.c:830-863`)
- **cloud-hypervisor**: 通过 `GuestMemoryAtomic<GuestMemoryMmap>` + `mem.memory().read_obj()` / `mem.memory().write_obj()` 访问 guest 内存。`Request` 的 `complete_async()` 方法接收 `&GuestMemoryMmap` 参数而非设备级引用。(`block.rs:153-156`, `block.rs:555-566`)

### 1.4 Feature Negotiation / Config Space

**共同结构**：设备核心暴露完整的 feature 集合和 `virtio_blk_config` 配置空间。

- **QEMU**: `virtio_blk_get_features()` 返回包含 `VIRTIO_BLK_F_SEG_MAX | GEOMETRY | TOPOLOGY | BLK_SIZE | WCE | RO | MQ | DISCARD | WRITE_ZEROES | FLUSH` 的 feature 集合。`virtio_blk_update_config()` 填充完整 `struct virtio_blk_config` 包含 geometry、topology、discard/write-zeroes 参数。 (`virtio-blk.c:1276-1307`, `virtio-blk.c:1175-1264`)
- **cloud-hypervisor**: `Block::new()` 初始化 `avail_features` 包含 `VIRTIO_F_VERSION_1 | FLUSH | CONFIG_WCE | BLK_SIZE | TOPOLOGY | SEG_MAX | EVENT_IDX | INDIRECT_DESC | WRITE_ZEROES | DISCARD | MQ | RO | ACCESS_PLATFORM`。配置 `VirtioBlockConfig` 包含 topology、seg_max、discard/write-zeroes 限制。 (`block.rs:780-858`)

### 1.5 ISR / Notify / Used Ring Semantics

**共同结构**：ISR 状态位按 spec 定义实现，used ring 写入后通过中断通知 guest。

- **QEMU**: `virtio_blk_req_complete()` 先写 status 字节到 guest memory，然后调用 `virtqueue_push()` 更新 used ring，最后 `virtio_notify()` 发送中断。 (`virtio-blk.c:57-69`)
- **cloud-hypervisor**: 先 `mem.write_obj(status, request.status_addr())`，再 `queue.add_used()` 更新 used ring，最后 `self.signal_used_queue()` 通过 `interrupt_cb.trigger(VirtioInterruptType::Queue)` 发送中断。 (`block.rs:555-566`)

### 1.6 Error Handling

**共同结构**：对每种错误类型返回对应的 virtio-blk status code。

- **QEMU**: `VIRTIO_BLK_S_IOERR` 用于 I/O 错误，`VIRTIO_BLK_S_UNSUPP` 用于不支持的命令，`VIRTIO_BLK_S_ZONE_*` 用于 zoned device 特定错误。每种错误路径都正确调用 `virtio_blk_req_complete()` 后释放 req。 (`virtio-blk.c:87-96`, `virtio-blk.c:1000-1013`)
- **cloud-hypervisor**: `check_request()` 返回 `ExecuteError::ReadOnly` 处理 RO 违规。`Request` 的 `execute_async()` 返回 `VIRTIO_BLK_S_IOERR`。队列损坏/重复 head index 触发 `NEEDS_RESET`。 (`block.rs:178-204`, `block.rs:402-419`)

---

## 2. 当前 AxVisor 与这些共同点的差距矩阵

| 维度 | AxVisor 现状 | mature reference 做法 | 差距类型 | 是否必须现在修 | 修到什么程度才不越界 |
|------|-------------|----------------------|---------|--------------|-------------------|
| **Queue 独立性** | `LegacyQueue` 是 `axdevice/src/virtio_blk.rs` 内的私有 struct，手动计算 desc table / avail ring / used ring 偏移，手动读写 GPA 字节 | QEMU 有完整的 `VirtQueue` 层 (`virtqueue_pop/push/notify`)；cloud-hypervisor 有 `virtio_queue::Queue` crate | 设备边界/协议语义 | 否 — 当前 `LegacyQueue` 功能正确且 unit test 覆盖 | 不需要移到独立 crate；保持 `LegacyQueue` 在 `axdevice` 内即可。如果 queue 逻辑未来需要被多个设备类型复用（virtio-net 等），才需要提取到共享模块 |
| **Descriptor chain 重复检测** | `collect_chain()` 仅检查链长度不超过 `self.size`，不检查 head index 是否重复 | cloud-hypervisor: `is_head_in_flight()` 显式检查重复 head；QEMU: `VirtIOBlockReq` 链表跟踪 | 协议语义 | 否 — OVMF 是单线程单 vCPU 驾动，不会复用 head | 不需要增加 inflight tracking。如果多 vCPU 驱动场景出现，再添加 head-in-flight 检查 |
| **Event idx 支持** | `LegacyVirtioBlk` 不识别 `VIRTIO_RING_F_EVENT_IDX`，available ring flags 字段被忽略 | QEMU: `virtio_queue_get_notification()` / `virtio_queue_set_notification()`；cloud-hypervisor: `queue.set_event_idx()` + `needs_notification()` | 协议语义 | 否 — 当前 OVMF DXE 路径 polling based，不依赖 event idx | 如果需要支持多队列或非 polling 驱动，event idx 是必要的。当前不修不会阻塞 OVMF bring-up |
| **Device / Transport 分离** | `LegacyVirtioBlk` 同时包含设备核心（request handling）和传输层（I/O port register dispatch）；在 `axvm/src/vm.rs` 中通过 `handle_ovmf_virtio_blk_io_read/write` 手动路由端口 | QEMU: `virtio-blk.c`（核心）与 `virtio-blk-pci.c`（传输）严格分离；cloud-hypervisor: `Block` 实现 `VirtioDevice` trait，`VirtioPciDevice` 负责 PCI 绑定 | 设备边界 | 否 — 当前是 legacy transport 且 port I/O 路由在 `axvm` 层 | 保持当前 split：`LegacyVirtioBlk` 在 `axdevice` 处理 request，`axvm` 的 `handle_ovmf_virtio_blk_io_*` 保持端口路由 glue。不要现在拆出独立 transport 层 — 没有新的 transport 消费者 |
| **Guest memory 访问边界** | `LegacyVirtioBlk::handle_write<M: GuestMemoryAccessor>` 泛型接受 `GuestMemoryAccessor` trait；`axvm` 通过 `AxVmGuestMemory` (裸指针包装) 提供实现 | QEMU: iovec scatter-gather + `iov_from_buf/iov_to_buf`；cloud-hypervisor: `GuestMemoryAtomic<GuestMemoryMmap>` | 设备边界 | 否 — 当前设计已经正确：设备核心通过 trait 抽象访问 guest 内存，不直接持有 `AxVMInnerMut` | `GuestMemoryAccessor` trait 定义正确。当前裸指针 `*const AddrSpace` 是已知的临时方案，方向 2 会替换。不要现在改进这个边界 |
| **ISR 状态语义** | `isr_status` 按 `REG_ISR_STATUS` 端口读取后清零（正确）；ISR 在 `handle_write` 中由设备核心设置；`LegacyNotifyResult::should_raise_irq` 传递给 `axvm` 但当前不注入 INTx | QEMU: ISR 由 VirtQueue 层在 `virtio_notify()` 中设置；cloud-hypervisor: `interrupt_cb.trigger()` 由设备 handler 调用 | 协议语义 | 否 — OVMF polling based，ISR 行为正确 | 保持当前 ISR 行为。`should_raise_irq` signal 被 `axvm` 保留为 observation log 是正确的设计选择。不要现在添加 INTx 注入 |
| **Used ring 写入语义** | `publish_used()` 手动计算 used ring 偏移，写入 `used_elem { id, len }`，然后递增 `idx`。`used_len` 对 read request 是 data bytes + 1 (status)，对 write 是 1 | QEMU: `virtqueue_push()` 更新 used ring；cloud-hypervisor: `queue.add_used(mem, desc_index, len)` | 协语义 | 否 — 当前 used ring 写入与 spec 一致 | 保持不变。used_len 计算正确：read 返回 data + 1 (status byte)，write 返回 1 (status byte only) |
| **多队列支持** | 单队列 (`DEFAULT_QUEUE_SIZE = 8`)，不支持 `VIRTIO_BLK_F_MQ` | QEMU: `num_queues` 可配置，支持 `VIRTIO_BLK_F_MQ`；cloud-hypervisor: `num_queues` 参数化，每个 queue 独立线程 | 协议语义 | 否 — OVMF 使用单队列 | 如果未来需要多队列，需要扩展 `LegacyQueue` 为 `Vec<LegacyQueue>`。当前不需要 |
| **Feature negotiation** | `handle_read(REG_DEVICE_FEATURES)` 始终返回 0；不暴露任何 feature bits | QEMU: 返回 `SEG_MAX | GEOMETRY | TOPOLOGY | BLK_SIZE | WCE | RO | MQ` 等；cloud-hypervisor: 返回 `VERSION_1 | FLUSH | CONFIG_WCE | BLK_SIZE | TOPOLOGY | SEG_MAX | EVENT_IDX | INDIRECT_DESC` | 协议语义 | 否 — OVMF legacy driver 不依赖 feature negotiation（它默认 legacy mode） | 保持返回 0。如果需要支持 modern virtio 或 feature negotiation，需要定义完整的 feature bitmap |
| **Config space** | 只暴露 `capacity_low/high` (sector count) 和 `queue_size`；无 `blk_size`、`seg_max`、`topology` 等 | QEMU: 完整 `struct virtio_blk_config`；cloud-hypervisor: `VirtioBlockConfig` 包含 topology、seg_max、write_zeroes/discard 限制 | 协议语义 | 否 — OVMF legacy driver 只读 capacity | 保持当前最小 config space。如果未来 OVMF 驱动检查 blk_size 或 topology，需要扩展 config space |
| **FLUSH 命令** | `execute_request()` 对 `VIRTIO_BLK_T_FLUSH` 返回 `VIRTIO_BLK_S_UNSUPP`（不支持） | QEMU: `virtio_blk_handle_flush()` 调用 `blk_aio_flush()`；cloud-hypervisor: 通过 `VIRTIO_BLK_F_FLUSH` feature 通告 | 协议语义 | 否 — OVMF bring-up 不需要 flush | 如果 guest OS 发送 flush 需要支持时，在 `MemoryDisk` 上添加 `sync()` 语义 |
| **WRITE_ZEROES / DISCARD** | 不支持，未暴露相关 features | QEMU: 支持 `VIRTIO_BLK_T_WRITE_ZEROES` 和 `VIRTIO_BLK_T_DISCARD`；cloud-hypervisor: `VIRTIO_BLK_F_WRITE_ZEROES | VIRTIO_BLK_F_DISCARD` | 协议语义 | 否 | 不需要支持。如果 guest OS 发送这些命令，当前返回 `VIRTIO_BLK_S_UNSUPP` 是正确的 |
| **request 批处理/合并** | `process_queue()` 逐个处理 avail ring 中的请求，不做合并 | QEMU: `MultiReqBuffer` + `multireq_compare()` + `virtio_blk_submit_multireq()` 合并相邻 sector 请求；cloud-hypervisor: batch request 提交 | 协议语义/测试 | 否 — 单队列 polling 场景下合并优化无意义 | 如果需要性能优化，可以在 `process_queue()` 中添加请求批处理 |
| **锁粒度** | `handle_ovmf_virtio_blk_io_write` 持有 `self.inner_mut.lock()` 整个 `process_queue` 期间 | QEMU: `rq_lock` 保护 request 链表，`blk_aio_*` 异步完成后释放；cloud-hypervisor: 每个 queue 独立线程持有独立 `Queue` | 设备边界 | 否 — 单 vCPU 场景，锁竞争不发生 | 保持当前锁粒度。如果多 vCPU 并发 I/O 场景出现，需要拆分锁 |
| **Backend 抽象** | `MemoryDisk` 是内嵌 `Vec<u8>`，`read_at/write_at` 直接操作字节数组 | QEMU: `BlockBackend` + `blk_aio_preadv/pwritev` 异步 I/O；cloud-hypervisor: `Box<dyn AsyncIo>` trait object | 设备边界 | 否 — bring-up 阶段 in-memory disk 足够 | 保持 `MemoryDisk`。如果需要 file-backed disk，可以引入 `BlockBackend` trait 但不要现在做 |
| **Block backend async I/O** | 同步 `process_queue()` → `execute_request()` → `MemoryDisk::read_at/write_at` | QEMU: `blk_aio_preadv/pwritev` + coroutine；cloud-hypervisor: `execute_async()` + completion event loop | 协议语义 | 否 — 同步 I/O 对 bring-up 正确 | 保持同步模型。异步 I/O 需要 direction 2 的 event loop |
| **测试覆盖** | `axdevice` 内有 `#[cfg(test)]` 模块：capacity、read request、write request、idle notify、ISR clear、error handling 共 7 个 unit tests | QEMU: 有 `tests/unit/` 中的 virtio-blk 单元测试；cloud-hypervisor: `vm-virtio/src/queue.rs` 中有 `VirtQueue` testing 模块 (`queue.rs:11-250`) | 测试 | 否 — 当前 unit test 覆盖了核心路径 | 保持当前测试集。如果添加新功能（event idx、多队列等），需要对应增加测试 |

---

## 3. 明确指出三件事

### 3.1 哪些内容应该继续留在 axdevice

以下内容的归属已经正确，不需要移动：

- **`LegacyVirtioBlk` 设备核心**：request parsing（`VirtioBlkRequest::from_chain`）、validation（`validate_data_descriptors`）、I/O execution（`execute_read/write/get_id`）、used ring publication（`publish_used`）——这些是纯 virtio-blk 协议行为，与 VM 上下文无关。 (证据: `virtio_blk.rs:83-527`)
- **`LegacyQueue` virtqueue 操作**：desc table 读取、avail ring pop、used ring push——这些是纯 virtqueue 协议操作。 (证据: `virtio_blk.rs:183-287`)
- **`LegacyNotifyResult`**：device core 通知 VMM "有完成项需要关注" 的信号结构——这是设备边界信号，不是 VM 行为。 (证据: `virtio_blk.rs:36-67`)
- **ISR 状态管理**：`isr_status` 在设备核心读取端口 0x13 时清零，写操作设置 `VIRTIO_ISR_QUEUE`——这是 virtio spec 行为。 (证据: `virtio_blk.rs:369-374`, `virtio_blk.rs:405-408`)
- **`MemoryDisk` 后端**：当前 in-memory backend 的 read/write 语义是设备核心的一部分。 (证据: `virtio_blk.rs:297-333`)

### 3.2 哪些内容可以暂时留在 axvm 作为 replaceable glue

以下内容是当前方向 1 的已知 glue，等 direction 2 基础设施到位后可以替换：

- **`AxVMInnerMut::ovmf_virtio_blk`**：设备实例的所有权在 `AxVMInnerMut` 中——等 `AxVmDevices` 能直接持有 port device 时可以移出。 (证据: `vm.rs:130-131`)
- **`AxVmGuestMemory`**：裸指针包装 `AddrSpace`——等 direction 2 提供 guest-memory trait 实现时可以替换。 (证据: `vm.rs:133-149`)
- **`handle_ovmf_virtio_blk_io_read/write`**：端口路由 glue（检查端口范围 → 委托给 `LegacyVirtioBlk`）——等 `AxVmDevices` 能将 `LegacyVirtioBlk` 注册为 `BasePortDeviceOps` 时可以消除。 (证据: `vm.rs:1182-1239`)
- **`install_virtio_blk_disk_image`**：disk image 安装入口——等设备实例归 `AxVmDevices` 管理时可以移出。 (证据: `vm.rs:1100-1106`)

### 3.3 哪些内容绝对不能现在为了"更标准"而做

以下改动会被 CLAUDE.md 中的规则判定为越界：

- **不要将 `LegacyQueue` 提取为独立 crate**：当前 queue 逻辑只被 `LegacyVirtioBlk` 使用。提取为独立 crate 需要新建基础设施，属于 direction 2 工作，违反"不要启动本地 direction-2 框架替代品"规则。
- **不要引入 `virtio_queue` crate**：cloud-hypervisor 使用的 `virtio_queue::Queue` 需要 `std` 或特定 `no_std` 配置，且接口与当前 `LegacyQueue` 不兼容。引入它需要重写整个 queue 路径，属于 direction 2 工作。
- **不要为 `LegacyVirtioBlk` 添加 PCI/modern transport 支持**：当前 `EmulatedDeviceType` 中已有 `VirtioBlk` 占位，但完整的 virtio-pci 实现需要 PCI config space、MSI-X、modern queue layout——这些都是 direction 2 范围。
- **不要添加异步 I/O backend**：引入 `AsyncIo` trait + completion event loop 需要 direction 2 的 event loop 基础设施。
- **不要添加 feature negotiation / config space 扩展**：`blk_size`、`topology`、`seg_max` 等 config space 字段需要 `VIRTIO_BLK_F_BLK_SIZE` 等 feature bits 协商，当前 OVMF 不需要。
- **不要添加 event_idx 支持**：需要修改 `LegacyQueue` 的 avail ring flags 检查和 `publish_used()` 的 used ring event_idx 写入，属于协议语义变更，当前 OVMF polling 路径不需要。
- **不要将 virtio-blk 注册为 `BasePortDeviceOps`**：当前端口路由 glue 是稳定的临时方案。将 `LegacyVirtioBlk` 注册为 `BasePortDeviceOps` 需要它实现 `BaseDeviceOps<PortRange>` trait，但 `handle_write` 的 `&mut self` 要求与 `Arc<dyn BaseDeviceOps>` 不兼容——这需要先修改 trait 或引入内部可变性，属于 direction 2 工作。

---

## 4. 最小标准化目标

以下是当前 bring-up 阶段值得做的最小标准化改动，不改变架构方向：

### 4.1 无需改动（现状正确）

当前 AxVisor legacy virtio-blk 在以下维度已经与成熟实现的核心语义对齐：

- Descriptor chain 解析和 validation（方向正确，test 覆盖）
- Used ring 写入语义（`used_len` 计算正确）
- ISR 读取清零行为（spec compliant）
- Guest memory 访问通过 `GuestMemoryAccessor` trait 抽象（边界正确）
- Device core 在 `axdevice`，VM glue 在 `axvm`（分层正确）

### 4.2 短期可选改进（不改变架构方向）

1. **`LegacyVirtioBlk` 中 `execute_read` 的 `used_len` 计算**：当前 read 返回 `used_len + 1`（data + status byte），write 返回 `1`。这两个都正确，但可以添加注释说明 why +1（virtio-blk spec 要求 used.len 包含 status byte）。这是文档改进，不是代码改动。

2. **`collect_chain()` 添加 loop detection 计数器**：当前用 `for _ in 0..self.size` 保护无限循环，这是正确的。可以考虑添加 `VIRTQ_DESC_F_NEXT` 检查时同时验证 `next < self.size`，但这已经在 `read_desc` 的 index range check 中覆盖。

3. **`handle_ovmf_virtio_blk_io_read` 中的 `info!` 日志**：当前每个 I/O read 都打 `info!` 级别日志 (`vm.rs:1198-1201`)。在 bring-up 阶段有用，但应标记为临时诊断日志。这不是功能差距，是清理项。

### 4.3 不在当前阶段做的

- Feature negotiation
- Config space 扩展（blk_size / topology / seg_max）
- Event idx 支持
- 多队列
- 异步 I/O backend
- PCI/modern transport
- head-in-flight 重复检测
- 请求批处理/合并
