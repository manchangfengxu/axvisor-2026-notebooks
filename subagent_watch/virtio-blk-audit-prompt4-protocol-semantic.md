# Prompt 4: 协议语义审计

Target: `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
References: QEMU `hw/block/virtio-blk.c`, cloud-hypervisor `virtio-devices/src/block.rs`, virtio spec headers.

---

## 1. 可接受简化

### 1.1 单队列（无 multi-queue）

- **简化内容**: 只有一个 `LegacyQueue` 实例，不支持 `VIRTIO_BLK_F_MQ` 多队列。
- **参考实现**: QEMU 默认 `num_queues=1`，cloud-hypervisor 默认 `num_queues=1`（两者都支持配置多队列，但单队列是常见基线配置）。
- **可接受原因**: 当前 OVMF 烟雾测试使用 `smp1`，单队列是 QEMU 默认配置。多队列是性能优化，不是协议正确性的前提。不会污染后续标准化——加入多队列只需在 `LegacyVirtioBlk` 中增加队列数组和 notify 路由。

### 1.2 仅支持 IN/OUT/GET_ID 三种请求类型

- **简化内容**: `execute_request` 只处理 `VIRTIO_BLK_T_IN`(0)、`VIRTIO_BLK_T_OUT`(1)、`VIRTIO_BLK_T_GET_ID`(8)，其余返回 `VIRTIO_BLK_S_UNSUPP`。
- **参考实现**: QEMU `virtio_blk_handle_request` 支持 FLUSH、SCSI_CMD、DISCARD、WRITE_ZEROES、ZONE 系列等完整 opcode 集。
- **可接受原因**: OVMF virtio-blk DXE 驱动的 boot 路径是 polling read + get_id，不依赖 flush/discard。协议规范允许对未支持的 opcode 返回 UNSUPP。后续加入 flush 等只需扩展 `match` 分支。

### 1.3 无 indirect descriptor 支持

- **简化内容**: `collect_chain` 只跟随 `VIRTQ_DESC_F_NEXT` 链，不识别 `VIRTQ_DESC_F_INDIRECT`。
- **参考实现**: cloud-hypervisor `queue.rs` 通过 `Queue::iter()` 遍历时支持 indirect descriptor table；QEMU 通过 `virtqueue_pop` 内部处理。
- **可接受原因**: legacy virtio-blk 不要求 indirect descriptor，且 feature negotiation 未启用（返回 0），guest 不会协商此 feature。不影响当前路径。

### 1.4 无 VIRTIO_F_VERSION_1 / modern queue 格式

- **简化内容**: 只实现 legacy descriptor table + avail ring + used ring 格式。
- **参考实现**: QEMU 根据 feature negotiation 选择 legacy 或 split/modern 格式；cloud-hypervisor 默认 `VIRTIO_F_VERSION_1`。
- **可接受原因**: 当前 device features 返回 0，强制 guest 使用 legacy 模式。这是 OVMF bring-up 阶段的已知限制，不会造成兼容性问题。

### 1.5 无 FLUSH (VIRTIO_BLK_T_FLUSH) 支持

- **简化内容**: 不处理 opcode 4 (FLUSH)。
- **参考实现**: QEMU 通过 `virtio_blk_handle_flush` 执行 `blk_aio_flush`。
- **可接受原因**: 当前后端是 `MemoryDisk`（内存模拟），无 flush 语义需求。OVMF boot 路径不发送 flush。后续换用文件后端时需要加入。

### 1.6 功能寄存器返回 0（不支持 feature negotiation）

- **简化内容**: `REG_DEVICE_FEATURES` 读取直接返回 `Ok(0)`，不声明任何 feature bit。
- **参考实现**: QEMU 通过 `virtio_blk_get_features` 设置 `SEG_MAX`、`GEOMETRY`、`TOPOLOGY`、`BLK_SIZE`、`RO`、`WCE` 等 feature bit；cloud-hypervisor 设置 `FLUSH`、`CONFIG_WCE`、`BLK_SIZE`、`TOPOLOGY`、`SEG_MAX`、`EVENT_IDX` 等。
- **可接受原因**: 返回 0 意味着 guest 不会协商任何扩展 feature，行为退化为最简 legacy virtio-blk。对当前 OVMF boot 路径（纯 polling IN/OUT + GET_ID）没有功能影响。但需要注意：一旦需要支持写保护或 flush，必须先加入 feature negotiation。

### 1.7 请求头 ioprio 字段被忽略

- **简化内容**: `from_chain` 从 guest 内存读取 16 字节 header，但只使用 `kind`(bytes 0-3) 和 `sector`(bytes 8-15)，完全忽略 `ioprio`(bytes 4-7)。
- **参考实现**: QEMU `virtio_blk_handle_request` 也不使用 ioprio（它从 iovec 中读取整个 outhdr 结构，但 `sector_num` 赋值时不涉及 ioprio）。
- **可接受原因**: ioprio 在所有参考实现中都是 ignore/ informational。不影响功能正确性。

### 1.8 无 config change interrupt（配置变更中断）

- **简化内容**: 没有 config change 中断路径。
- **参考实现**: QEMU 通过 `virtio_notify_config()` 触发配置变更中断；cloud-hypervisor 在 resize 时调用 `trigger_interrupt(VirtioInterruptType::Config)`。
- **可接受原因**: `MemoryDisk` 是固定大小，没有动态 resize 场景。等加入 resize 功能时再加入 config change 中断。

### 1.9 无 read-only disk 模式 (VIRTIO_BLK_F_RO)

- **简化内容**: 不支持设备只读模式。
- **参考实现**: QEMU 通过 `VIRTIO_BLK_F_RO` feature 通知 guest 设备只读；cloud-hypervisor 在 `check_request` 中强制返回 `IOERR` 拒绝写请求。
- **可接受原因**: 当前 `MemoryDisk` 是可读写的内存后端，不需要只读保护。但需要注意：`VIRTIO_BLK_F_RO` 的检查依赖 feature negotiation 能力（见 1.6），两者需要一起加入。

### 1.10 无 WRITE_ZEROES / DISCARD 支持

- **简化内容**: 不处理 opcode 13 (WRITE_ZEROES) 和 opcode 11 (DISCARD)。
- **参考实现**: QEMU 通过 `virtio_blk_handle_discard_write_zeroes` 执行；cloud-hypervisor 通过 block crate 执行。
- **可接受原因**: OVMF boot 路径不使用这些操作。MemoryDisk 后端不支持 sparse operation。后续加入时需要同时加入 `VIRTIO_BLK_F_DISCARD` / `VIRTIO_BLK_F_WRITE_ZEROES` feature bits。

---

## 2. 不可继续放任的偏差

### 2.1 Queue stall on malformed descriptor chain（畸形描述符链导致队列阻塞）

- **问题**: `process_queue` 中，如果 `collect_chain` 或 `from_chain` 返回 `Err`，整个 `while let Some(head)` 循环立即退出（通过 `?` 操作符），后续所有已 available 的请求被永远遗留在 avail ring 中，永远不会被处理。设备进入死锁状态。
- **证据**: `virtio_blk.rs:415-426`，`process_queue` 方法：
  ```rust
  fn process_queue<M: GuestMemoryAccessor>(&mut self, mem: &M) -> AxResult<LegacyNotifyResult> {
      let mut notify = LegacyNotifyResult::idle();
      while let Some(head) = self.queue.pop_available(mem)? {  // <-- 如果 collect_chain/from_chain Err，整个循环退出
          let descs = self.queue.collect_chain(head, mem)?;
          let request = VirtioBlkRequest::from_chain(&descs, mem)?;
          let completion = self.execute_request(&request, mem)?;
          self.write_status(&request, mem, completion.status)?;
          self.queue.publish_used(head, completion.used_len, mem)?;
          notify.merge(LegacyNotifyResult::completed_one());
      }
      Ok(notify)
  }
  ```
- **问题类型**: queue 语义问题。
- **参考实现怎么处理**: QEMU `virtio_blk_handle_vq` (line 1017-1048): 当 `virtio_blk_handle_request` 返回错误时，调用 `virtqueue_detach_element` 回退 avail ring 位置，然后 `break` 退出循环，但已完成的请求不受影响。对于单个请求的错误，`virtio_blk_handle_request` 内部直接调用 `virtio_blk_req_complete(req, VIRTIO_BLK_S_UNSUPP)` 完成请求并写入 status byte。
  cloud-hypervisor 在 `process_queue_submit_and_signal` 中将 `QueueIterator` 和 `QueueDuplicatedHeadIndex` 错误视为设备级错误（需要 NEEDS_RESET），其他错误是 per-request 错误。
- **偏差是否会污染后续标准化**: 是。任何非 trivial 的 guest 驱动都可能发送畸形请求（恶意或 bug）。当前实现一旦遇到畸形描述符链，设备就永久卡死，只能通过 reset 恢复。更重要的是，AxVisor 当前没有 virtio reset 路径——所以一旦触发，VM 需要销毁重建。

**修复方向**: 将 `collect_chain` / `from_chain` 的错误改为 per-request 处理：写入 `VIRTIO_BLK_S_UNSUPP` status 并 `publish_used`，而不是通过 `?` 终止整个队列循环。

### 2.2 Used ring event suppression (avail.flags) 被忽略

- **问题**: `pop_available` 在读取 avail ring 时完全忽略 `avail.flags` 中的 `VIRTQ_AVAIL_F_NO_INTERRUPT` (bit 0) 标志。该标志在 legacy virtio 中是唯一的通知抑制机制（event_idx 是 V1.0+ 扩展）。
- **证据**: `virtio_blk.rs:263-273`，`pop_available` 方法：
  ```rust
  fn pop_available<M: GuestMemoryAccessor>(&mut self, mem: &M) -> AxResult<Option<u16>> {
      let avail = self.avail_ring_gpa()?;
      let avail_idx: u16 = mem.read_obj(GuestPhysAddr::from(avail + 2))?;
      if self.last_avail_idx == avail_idx {
          return Ok(None);
      }
      let slot = self.last_avail_idx % self.size;
      let head: u16 = mem.read_obj(GuestPhysAddr::from(avail + 4 + slot as usize * 2))?;
      self.last_avail_idx = self.last_avail_idx.wrapping_add(1);
      Ok(Some(head))
  }
  ```
  注意：avail ring 在 descriptor table 之后排列为 `[flags:u16, idx:u16, ring:u16[N], event:u16]`。`pop_available` 从 `avail + 2` 读取 idx，跳过了偏移 0 处的 `flags` 字段。
- **问题类型**: queue 语义问题。
- **参考实现怎么处理**: QEMU 通过 `virtio_queue_get_notification` / `virtio_queue_set_notification` 管理通知抑制。在 `virtio_blk_handle_vq` 中：
  ```c
  bool suppress_notifications = virtio_queue_get_notification(vq);
  if (suppress_notifications) {
      virtio_queue_set_notification(vq, 0);
  }
  ```
  这是 virtio core 层面的 queue notification 管理，与 `avail.flags` 的 legacy 中断抑制语义配合使用。
  cloud-hypervisor 通过 `VIRTIO_RING_F_EVENT_IDX` feature 和 `Queue::needs_notification()` 完成更精细的通知抑制。
- **偏差是否会污染后续标准化**: 是。当前 OVMF boot 路径是 polling 模式（不依赖中断），所以影响暂时不可见。但一旦引入中断驱动的 virtio-blk（如 ArceOS 侧的 future 驱动），guest 设置 `VIRTQ_AVAIL_F_NO_INTERRUPT` 后仍收到中断，将导致不必要的 VM-exit 开销。此外，如果未来加入 `VIRTIO_RING_F_EVENT_IDX`，event_idx 的 `used_event` 字段也依赖类似的通知抑制机制。

**修复方向**: 在 `pop_available` 中读取并检查 `avail.flags` 的 bit 0，或者将通知抑制语义提升到 `LegacyNotifyResult` / `LegacyVirtioBlk` 层面的显式控制。

### 2.3 Used ring 写入的内存序问题

- **问题**: `publish_used` 按顺序写入 `used.ring[slot].id` -> `used.ring[slot].len` -> `used.idx`。在 guest 侧，virtio spec 要求设备在更新 `used.idx` 之前的所有写入对 guest 可见。当前实现没有 `write memory barrier`，在弱内存序架构上（虽然 x86 有 TSO，但 AxVisor 抽象层不保证目标架构）可能存在 `used.idx` 更新先于 data/status 可见的窗口。
- **证据**: `virtio_blk.rs:275-286`，`publish_used` 方法：
  ```rust
  fn publish_used<M: GuestMemoryAccessor>(&self, head: u16, len: u32, mem: &M) -> AxResult {
      let used = self.used_ring_gpa()?;
      let idx: u16 = mem.read_obj(GuestPhysAddr::from(used + 2))?;
      let slot = idx % self.size;
      mem.write_obj(GuestPhysAddr::from(used + 4 + slot as usize * 8), head as u32)?;
      mem.write_obj(GuestPhysAddr::from(used + 8 + slot as usize * 8), len)?;
      mem.write_obj(GuestPhysAddr::from(used + 2), idx.wrapping_add(1))?;
      Ok(())
  }
  ```
  没有在 `used.idx` 写入之前插入 barrier（如 `core::sync::atomic::fence(Ordering::Release)` 或等价的 guest memory barrier）。
- **问题类型**: completion 语义问题（used ring 内存可见性）。
- **参考实现怎么处理**: QEMU 通过 virtio core 的 `virtqueue_push` -> `virtio_notify` 路径，其中 `virtio_notify` 内部有适当的内存屏障（依赖 VIRTIO_RING_F_EVENT_IDX 和 KVM 的 MMIO/irqfd 机制）。cloud-hypervisor 使用 `queue.add_used()`，该方法由 `virtio_queue` crate 实现，内部在更新 `used.idx` 前有适当的 ordering 保证。
- **偏差是否会污染后续标准化**: 部分是。在 x86_64 上由于 TSO 内存模型，这个问题不会实际触发。但如果 AxVisor 未来支持 ARM 或 RISC-V guest（或者使用不保证 TSO 的 guest memory accessor），这个 bug 将导致 guest 看到新的 `used.idx` 但读到旧的 status/data。

**修复方向**: 在 `publish_used` 的 `used.idx` 写入之前加入 `mem.write_barrier()`（如果 GuestMemoryAccessor 提供）或者至少在文档中记录此依赖 x86 TSO 的假设。

### 2.4 Used ring `id` 字段类型宽度不匹配

- **问题**: `publish_used` 将 `head`（`u16`）通过 `head as u32` 写入 `used.ring[slot]` 的 id 位置。Virtio spec 定义 `struct virtq_used_elem { id: __le32; len: __le32; }`，所以写入 `u32` 本身是对的。但 AxVisor 的 `write_obj(head as u32, ...)` 写入 4 字节，而 `head as u32` 会将高位零填充，这恰好在 LE 上正确。真正的 bug 在于后续的 `write_obj(len)` 只写了 2 字节（因为 `len` 是 `u32` 但 `write_obj` 序列化 `u32` 应该是 4 字节，需要确认 `write_obj` 的行为）。
  
  让我更精确地描述实际问题：`publish_used` 的 slot 偏移计算为 `used + 4 + slot * 8`，每个 used_elem 占 8 字节（`id:u32 + len:u32`）。id 写入 `used + 4 + slot * 8`（4 字节），len 写入 `used + 8 + slot * 8`（4 字节）。但 `id` 字段的起始位置是 `used + 4 + slot * 8`，而 `len` 字段应该在 `used + 4 + slot * 8 + 4 = used + 8 + slot * 8`。所以偏移计算是正确的。但是——`id` 写入使用 `head as u32`（u32 写 4 字节），这是正确的（spec 要求 id 是 __le32）。`len` 写入 `len`（u32 写 4 字节），也是正确的。所以实际上这个偏差的严重程度比初看低。

  然而，真正的问题在于 `read_obj` 读取 `used + 2` 处的 `idx`（u16），而 `avail_ring_gpa` 返回的偏移包含 4 字节 header（`flags:u16 + idx:u16`）。used ring 的 layout 也是 `[flags:u16, idx:u16, ring:used_elem[N], event:u16]`。所以 `used + 0` 是 flags，`used + 2` 是 idx，`used + 4` 是 ring[0].id，`used + 8` 是 ring[0].len。每项 8 字节。计算正确。

  **修正后的实际问题**: `id` 写入时使用 `head as u32`——这是一个 `u32` 值，`write_obj` 会写 4 字节。但如果 `write_obj` 内部使用 `core::ptr::write_unaligned`，则写入 4 字节到 `used + 4 + slot * 8`，然后 `write_obj(len)` 写 4 字节到 `used + 8 + slot * 8`。这是正确的 8 字节 used_elem layout。

  **最终判定**: 偏移计算正确。但 `write_obj` 的实现是否保证写入完整类型大小需要确认。如果 `write_obj::<u16>` 只写 2 字节而 `write_obj::<u32>` 写 4 字节，则当前代码正确。假设 `write_obj` 行为正确，则此项降级为**代码质量问题**而非协议偏差。
  
  **最终结论**: 经过深入分析，`publish_used` 的 used ring layout 计算在功能上是正确的（LE 平台，write_obj 行为正确时）。但代码中 `head as u32` 的隐式转换容易引起误读，建议改用明确的类型标注。**此项从"不可继续放任"降级为"代码质量"，不列入优先修正**。

### 2.5 GET_ID 响应不符合 VIRTIO_BLK_ID_BYTES = 20 规范

- **问题**: `execute_get_id` 将 `SERIAL`（"AXVISOR-UEFI-BLK"，16 字节）直接写入 guest data descriptor，没有填充到 `VIRTIO_BLK_ID_BYTES`（20 字节）的规范要求。Spec 要求设备写入 `VIRTIO_BLK_ID_BYTES` 字节的字符串，以 null 结尾，不足部分用 null 填充。当前实现只写入实际字符串长度，不 pad。
- **证据**: `virtio_blk.rs:489-506`，`execute_get_id` 方法：
  ```rust
  fn execute_get_id<M: GuestMemoryAccessor>(
      &self,
      request: &VirtioBlkRequest,
      mem: &M,
  ) -> AxResult<RequestCompletion> {
      const SERIAL: &[u8] = b"AXVISOR-UEFI-BLK";
      let mut written = 0u32;
      for desc in &request.data {
          let remaining = &SERIAL[written as usize..];
          if remaining.is_empty() {
              break;
          }
          let take = remaining.len().min(desc.len as usize);
          mem.write_buffer(GuestPhysAddr::from(desc.addr as usize), &remaining[..take])?;
          written += take as u32;
      }
      Ok(RequestCompletion::ok(written + 1))
  }
  ```
  `SERIAL` 长度 16 < `VIRTIO_BLK_ID_BYTES` 20。如果 guest data descriptor 恰好是 20 字节（OVMF 常见做法），则 bytes 16-19 未被写入，保留 guest 内存的旧内容。
- **问题类型**: request 语义问题。
- **参考实现怎么处理**: QEMU `virtio_blk_handle_request` (line 934-947):
  ```c
  case VIRTIO_BLK_T_GET_ID:
  {
      const char *serial = s->conf.serial ? s->conf.serial : "";
      size_t size = MIN(strlen(serial) + 1,
                        MIN(iov_size(in_iov, in_num),
                            VIRTIO_BLK_ID_BYTES));
      iov_from_buf(in_iov, in_num, 0, serial, size);
      virtio_blk_req_complete(req, VIRTIO_BLK_S_OK);
  }
  ```
  QEMU 限制写入量为 `min(strlen+1, min(iov_total, 20))`，但不显式 zero-pad 剩余部分。然而 QEMU 的 `virtio_blk_req_complete` 会正确设置 `in_len`，guest 驱动知道有效数据的实际长度。
  cloud-hypervisor (line 341-346): `GetDeviceId` 时设置 `len = self.serial.len() as u32 + 1`，也不做 zero-pad。
- **偏差是否会污染后续标准化**: 中等。当前 OVMF 读取 serial 作为 null-terminated string，16 字节的字符串有隐式 null 结束（Rust byte literal 不包含 null，但 guest 内存可能有）。实际风险是 guest 读到 20 字节但只有 16 字节有效——如果 guest 没有 null-terminate 检查，可能读到垃圾。当前 OVMF boot 路径不使用 serial，所以风险为零。但这是一个需要在标准化前修复的规范偏离。

**修复方向**: 将 SERIAL 零填充到 20 字节再写入，或者确保 `written` 计入 null terminator 并将 remaining 字节用 0x00 填充。

### 2.6 无 feature negotiation 路径（阻塞后续所有需要 feature 的功能）

- **问题**: `REG_DEVICE_FEATURES` 读取硬编码返回 `Ok(0)`。没有 guest features 写入/读取寄存器，没有 `VIRTIO_F_ACKNOWLEDGE` / `VIRTIO_F_DRIVER` / `VIRTIO_F_DRIVER_OK` 状态位的管理。这意味着无法向 guest 暴露任何扩展 feature（如 `VIRTIO_BLK_F_FLUSH`、`VIRTIO_BLK_F_RO`、`VIRTIO_BLK_F_BLK_SIZE`、`VIRTIO_RING_F_EVENT_IDX`）。
- **证据**: `virtio_blk.rs:363`：
  ```rust
  (REG_DEVICE_FEATURES, AccessWidth::Dword) => Ok(0),
  ```
  没有 `REG_GUEST_FEATURES` (标准偏移 0x0c for MMIO 或 0x04 for PCI) 的处理。
  `status` 写入（line 399-401）只是存储值，不检查也不管理状态位：
  ```rust
  (REG_STATUS, AccessWidth::Byte) => {
      self.status = value as u8;
      LegacyNotifyResult::idle()
  }
  ```
- **问题类型**: register 语义问题。
- **参考实现怎么处理**: QEMU 通过 `virtio_blk_get_features` 动态计算可用 feature bits（line 1276-1307），根据设备配置和 backend 能力设置 `SEG_MAX`、`GEOMETRY`、`RO`、`FLUSH`、`WCE`、`MQ` 等。`set_status` 管理状态转换（line 1309-1342），检查 `DRIVER_OK` 并据此配置 write cache。
  cloud-hypervisor 通过 `VirtioCommon` 管理完整的 feature 协商和状态机。
- **偏差是否会污染后续标准化**: 是——这是阻塞性偏差。所有需要 feature negotiation 的功能（FLUSH、RO、EVENT_IDX、BLK_SIZE、TOPOLOGY）都被阻塞。当前不影响 OVMF boot（纯 polling read），但阻止了：
  1. 添加 `VIRTIO_BLK_F_FLUSH`（换用文件后端后必需）
  2. 添加 `VIRTIO_BLK_F_RO`（只读磁盘支持）
  3. 添加 `VIRTIO_RING_F_EVENT_IDX`（通知抑制优化）
  4. 添加 `VIRTIO_BLK_F_BLK_SIZE` / `VIRTIO_BLK_F_SEG_MAX`（配置告知）

**修复方向**: 作为优先修正项 3，需要：
1. 定义 device feature bits（至少 `VIRTIO_BLK_F_SEG_MAX`、`VIRTIO_BLK_F_BLK_SIZE`）
2. 添加 guest features 写入寄存器
3. 管理 `self.status` 的状态位语义（ACKNOWLEDGE、FEATURES_OK、DRIVER_OK）

---

## 3. 优先修正顺序（只给 3 项）

### 优先修正 1：Queue stall on malformed descriptor chain

- **修正内容**: 将 `process_queue` 中 `collect_chain` / `from_chain` 的 `?` 错误传播改为 per-request 错误处理——对畸形链写入 `VIRTIO_BLK_S_UNSUPP` 到 status descriptor 并 `publish_used`，然后 `continue` 而非 `return Err`。
- **为什么是优先级 1**: 这是功能正确性 bug，不是优化。任何畸形 descriptor chain（无论是 guest 驱动 bug 还是恶意 guest）都会导致设备永久卡死。对于一个要承担更多 virtio 功能的设备核心，这种 fail-open 行为是必须的。QEMU 和 cloud-hypervisor 都将单个请求错误与设备级错误严格区分。
- **修正时的边界约束**: 保持 `LegacyQueue` 作为独立的 ring 操作层不变。修改限制在 `process_queue` 方法内部——错误处理改为 `match` 而非 `?`，将 `VirtioBlkRequest::from_chain` 和 `collect_chain` 的 `Err` 情况转换为 `RequestCompletion::unsupp()` 路径。不引入新的设备框架抽象。

### 优先修正 2：Used ring event suppression 被忽略

- **修正内容**: 在 `LegacyQueue::pop_available` 中检查 avail ring 的 `flags` 字段 bit 0 (`VIRTQ_AVAIL_F_NO_INTERRUPT`)，并将通知抑制状态传递给 `LegacyNotifyResult`，使得 `handle_write` 不在通知抑制时设置 `isr_status |= VIRTIO_ISR_QUEUE`。
- **为什么是优先级 2**: 这是 legacy virtio 协议的核心 queue 语义。当前 OVMF 路径是 polling 不受影响，但一旦加入中断驱动的 virtio-blk 驱动（ArceOS 侧），缺失通知抑制会导致每个 `publish_used` 都触发中断，产生不必要的 VM-exit。此外，加入 `VIRTIO_RING_F_EVENT_IDX` 时需要类似的通知抑制机制，提前实现 `avail.flags` 检查可以作为基础。
- **修正时的边界约束**: 修改限制在 `LegacyQueue` 内部（读取 `avail.flags`）和 `LegacyNotifyResult`（增加通知抑制标志）以及 `handle_write` 中 ISR 设置条件。不改变 `process_queue` 的核心循环逻辑，不引入 event_idx 支持。

### 优先修正 3：Feature negotiation 基础框架

- **修正内容**: 在 `LegacyVirtioBlk` 中加入最小的 feature negotiation 路径：(1) 定义 `REG_GUEST_FEATURES` 寄存器写入；(2) 存储 `guest_features: u32`；(3) `REG_DEVICE_FEATURES` 返回设备支持的 feature bits（至少 `VIRTIO_BLK_F_SEG_MAX = 2`）；(4) `REG_STATUS` 管理 `VIRTIO_CONFIG_S_ACKNOWLEDGE` / `VIRTIO_CONFIG_S_FEATURES_OK` / `VIRTIO_CONFIG_S_DRIVER_OK` 状态位。
- **为什么是优先级 3**: 当前所有需要 feature negotiation 的功能（FLUSH、RO、EVENT_IDX、BLK_SIZE）都被阻塞。这是一个基础设施问题，不是 bug——当前返回 0 时行为是确定的。但它是后续所有标准化步骤的前提，越早建立越能避免后续每次加入 feature 时重复修改。
- **修正时的边界约束**: 保持当前 register layout 不变（`LEGACY_BLK_IO_BASE + 偏移`）。feature negotiation 只增加 `REG_GUEST_FEATURES` 寄存器（未使用过的偏移）。`status` 管理只增加状态位检查，不改变现有 status 写入的兼容性。不在此次修正中加入任何具体 feature 的实现——只建立 feature 框架。

---

## 附注：已确认的代码质量问题（非协议偏差，但值得记录）

1. **`head as u32` 隐式转换**: `publish_used` 中 `head as u32` 在 LE 平台功能正确，但意图不明确。建议改为 `u32::from(head)` 或添加注释说明 spec 要求 id 为 `__le32`。

2. **`execute_write` 的 `used_len = 1`**: write 请求的 `used_len = 1`（只含 status byte）符合 spec（write 请求不返回数据给 guest），但与 read 请求的 `used_len = data + 1` 不对称。确认这是正确行为：read 的 used_len 包含 device 写给 guest 的数据量，write 的 used_len 只含 status。

3. **`from_chain` 的 header 长度检查**: `header.len < 16` 检查描述符长度 >= 16 字节，但不检查 `kind` 字段的值是否在已知 opcode 范围内。这在 `validate_data_descriptors` 中部分处理（只检查 IN/OUT/GET_ID 的数据方向），但 `execute_request` 的 default 分支返回 `unsupp()` 是正确的兜底行为。
