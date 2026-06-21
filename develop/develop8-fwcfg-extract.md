# develop8 — fw_cfg 设备提取

## 动机

当前 `axvm/src/vm.rs` 里 `FwCfgState` 的全部逻辑（约 130 行）和 `handle_fw_cfg_io_read/write/dma`（约 85 行）都是硬编码在 VM 层内的。上游 dev 分支已有 `BaseDeviceOps<PortRange>`、`AxVmDevices` 统一调度、`GuestMemoryAccessor`，具备把 port I/O 设备状态从 vm.rs 拿出去的条件。

fw_cfg 比 virtio-blk 更适合做第一个提取模板：纯 port I/O 设备，不涉及 block backend 和 virtqueue。

## 关键设计决策：DMA 不塞进 BaseDeviceOps

`BaseDeviceOps::handle_write(addr, width, val)` 传不了 guest memory。fw_cfg DMA 需要读 guest 里的 16 字节 descriptor，再把数据写回 guest。因此：

- **普通 I/O（selector 0x510, data 0x511, DMA-address 0x514/0x518）** → 进 `FwCfgDevice` 实现 `BaseDeviceOps<PortRange>`，由 `AxVmDevices` 统一 dispatch。
- **DMA 执行 + String I/O** → 留 thin adapter 在 vm.rs。DMA address 写完 second dword 后设 `pending_dma`，vm.rs 取走、用 `GuestMemoryAccessor` 读写 guest memory，select/read/skip 通过 `AxVmDevices` 公开的方法回调 FwCfgDevice。

这和当前 `LegacyVirtioBlk` adapter 收敛方向一致。

## 配置时机

`FwCfgDevice` 在 `AxVmDevices::new()` 时无条件创建并注册 port range（类似 `OvmfDebugConDevice`）。此时 items 为空。`AxVM::init()` 在 memory_regions 确定后调用 `devices.x86_fw_cfg_configure()` 填充 items。OVMF 到 PEI 才访问 fw_cfg，时机满足。

## 文件变更

### 新增

| 文件 | 说明 |
|---|---|
| `virtualization/axdevice/src/fw_cfg.rs` | `FwCfgDevice` + `FwCfgInner`，~225 行 |

### 修改

| 文件 | 变更 |
|---|---|
| `virtualization/axdevice/src/lib.rs` | 加 `mod fw_cfg` |
| `virtualization/axdevice/src/device.rs` | `AxVmDevices` 加 `x86_fw_cfg` 字段 + 在 `new()` 中注册 + 5 个 helper 方法（`x86_fw_cfg_configure`, `x86_fw_cfg_take_pending_dma`, `x86_fw_cfg_read_string_bytes`, `x86_fw_cfg_select`, `x86_fw_cfg_skip_bytes`） |
| `virtualization/axvm/src/vm.rs` | 删除 `FwCfgState` (~100 行) + `handle_fw_cfg_io_read/write/dma` (~85 行) + 旧常量 (~25 行)；新增 thin `execute_fw_cfg_dma` (~35 行)；IoRead/IoWrite 删除 fw_cfg 分支；IoWrite 后加 `take_pending_dma` 检查；IoStringRead 0x511 改用 `devices.x86_fw_cfg_read_string_bytes` |

### 附带修复

| 文件 | 说明 |
|---|---|
| `virtualization/axdevice/src/virtio_blk.rs` | 修复预存的 merge conflict artifact：`disk_offset()` 中重复的 `if offset > self.disk.len()` / `self.disk.as_slice().len()` |

## 架构关系

```
FwCfgDevice (axdevice/src/fw_cfg.rs)
  ├── Mutex<FwCfgInner>      // selector, offset, items, dma_bytes, pending_dma
  ├── impl BaseDeviceOps<PortRange>  // 0x510..0x51C, handle_read/write
  └── pub methods: configure, take_pending_dma, read_string_bytes, select, skip_bytes

AxVmDevices (axdevice/src/device.rs)
  ├── x86_fw_cfg: Option<Arc<FwCfgDevice>>
  └── x86_fw_cfg_*() wrappers  // 统一接口，不暴露内部 Arc

AxVM (axvm/src/vm.rs)
  ├── run_vcpu: IoRead/IoWrite → 统一 dispatch（不再特殊处理 fw_cfg ports）
  ├── run_vcpu: IoWrite 后 → take_pending_dma → execute_fw_cfg_dma
  ├── run_vcpu: IoStringRead 0x511 → read_string_bytes → write_guest_bytes
  └── init(): x86_fw_cfg_configure(memory_regions, cpu_count)
```

## 验证

- `cargo check -p axdevice -p axvm` 通过
- `cargo test -p axdevice --lib` 全部 10 个测试通过
- `cargo test -p axvm --lib` 通过
- `cargo fmt --package axdevice --package axvm` 通过
- `cargo xtask axvisor build --config .../qemu-x86_64.toml --vmconfigs .../ovmf-x86_64-qemu-smp1.toml` release build 通过
- smoke run（按 start.md）：待验证
