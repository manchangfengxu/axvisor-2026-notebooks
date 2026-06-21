# virtio-blk model-first device-core cleanup

## Scope

本轮只整理 legacy virtio-blk 设备核心和 AxVM 适配边界，不引入新的通用设备接口，不接 INTx。

## Modified Symbols

- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - `LegacyNotifyResult`
    - 用结构化结果替换原来的 bare `bool` queue completion 结果，给 AxVM 一个稳定的结果消费点。
  - `LegacyVirtioBlk::handle_write`
    - 保持 port register dispatch 入口，`REG_QUEUE_NOTIFY` 返回 `LegacyNotifyResult`，其它寄存器写返回 idle 结果。
  - `LegacyVirtioBlk::process_queue`
    - 从单个 `bool` 改成聚合 `LegacyNotifyResult`，统一统计本次 notify 是否发布 used buffer、数量多少、是否应触发 IRQ。
  - `VirtioBlkRequest::validate_data_descriptors`
    - 把 data descriptor 的存在性和方向校验从执行路径里收口出来。
  - `RequestCompletion`
    - 把 guest-visible completion 语义收成 `status + used_len`，供 queue 发布和 status 写回统一消费。
  - `LegacyVirtioBlk::execute_request`
    - 先做请求校验，再按 request kind 分发执行；校验失败返回 `ioerr` completion，unsupported 返回 `unsupp` completion。
  - `LegacyVirtioBlk::execute_read`
    - 不再直接写 status，只返回 `RequestCompletion::ok(...)` 或 `RequestCompletion::ioerr()`。
  - `LegacyVirtioBlk::execute_write`
    - 不再直接写 status，只返回 `RequestCompletion::ok(1)` 或 `RequestCompletion::ioerr()`。
  - `LegacyVirtioBlk::execute_get_id`
    - 假设 data descriptor 已经过校验，不再在执行时跳过只读 descriptor。

- `tgoskits/virtualization/axvm/src/vm.rs`
  - `AxVM::handle_ovmf_virtio_blk_io_write`
    - 继续只做 OVMF port-I/O 适配和 `AxVmGuestMemory` 构造，日志里消费 `LegacyNotifyResult`，不重新解释 virtio 协议。

## Behavior Summary

- `REG_QUEUE_NOTIFY` 现在返回结构化 notify 结果，而不是只有“是否完成过请求”这一个 bit。
- `status` 写回和 `used ring` 发布都由 `process_queue` 基于 `RequestCompletion` 统一处理。
- 新增的 guest-visible 校验点：
  - read/write/GET_ID 请求不能缺少 data descriptor。
  - read/GET_ID 的 data descriptor 必须可写。
  - write 的 data descriptor 必须可读。

## Not Changed

- 没有改 `BaseDeviceOps`。
- 没有引入本地 `DeviceContext`、`IrqSink`、`BusRouter`。
- 没有把 legacy virtio-blk 硬注册成当前 `AxVmDevices` 的普通 port device。
- 没有接 `X86IoApic` / INTx。
