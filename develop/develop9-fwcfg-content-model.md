# develop9 - fw_cfg 最终修改总览

## 简介

本轮把 OVMF 需要的 fw_cfg 从 `axvm/src/vm.rs` 的特化逻辑，收敛到 `axdevice/src/fw_cfg.rs` 的设备模型里。

最终边界是：

- 普通 port I/O 由 `FwCfgDevice: BaseDeviceOps<PortRange>` 接管。
- fw_cfg item 状态、file directory、DMA descriptor ABI 解析和 DMA 执行放在 `FwCfgDevice`。
- `AxVM` 不再解析 fw_cfg 协议，只提供 `AxVmGuestMemory` 作为 `GuestMemoryAccessor`，并在 I/O write 后触发 pending DMA。

## 借鉴 reference 的部分

参考：

- `references/cloud-hypervisor/devices/src/legacy/fw_cfg.rs`
- `references/cloud-hypervisor/docs/fw_cfg.md`

借鉴内容：

- typed content：用 `FwCfgContent::{Bytes, U32}` 代替裸 `Vec<u8>` item map。
- named file item：用 `FwCfgFileItem` 表示 named item，并由 `add_file_item()` 自动维护 `FW_CFG_FILE_DIR`。
- DMA ABI：用 `FwCfgDmaRequest::parse()` 集中解析 16 字节 big-endian descriptor。

没有照搬：

- 不引入 Cloud Hypervisor 的 bus/device/config/file-backed 模型。
- 不扩展 VM TOML schema。
- 不改变当前已 smoke 通过的 `etc/e820` selector，仍保留 `0x8000`。

## 修改定位

### `virtualization/axdevice/src/fw_cfg.rs`

新增/调整的类型和常量：

- `pub const FW_CFG_IO_DATA`
  - 导出 fw_cfg data port，供 `axvm` 的 string I/O 判断使用。
- `pub enum FwCfgContent`
  - `Bytes(Vec<u8>)`
  - `U32(u32)`
- `struct FwCfgFileItem`
  - `selector`
  - `name`
  - `content`
- `pub struct FwCfgDmaRequest`
  - `control`
  - `length`
  - `address`
  - `parse([u8; 16])`
- `struct FwCfgInner`
  - `selector`
  - `offset`
  - `dma_bytes`
  - `dma_bytes_written`
  - `pending_dma`
  - `known_items`
  - `file_items`

核心函数：

- `FwCfgInner::configure()`
  - 填充 signature、DMA feature、CPU count、默认 `etc/e820`。
- `FwCfgInner::add_file_item()`
  - 新增 named item，分配 selector，并重建 file directory。
- `FwCfgInner::allocate_file_selector()`
  - 从 `0x20` 起分配普通 named item selector。
- `FwCfgInner::rebuild_file_dir()`
  - 更新 `QEMU_FW_CFG_ITEM_FILE_DIR`。
- `FwCfgInner::build_file_dir()`
  - 生成 QEMU fw_cfg file directory entry。
- `FwCfgInner::selected_content()`
  - 在 known item 和 named file item 中查找当前 selector。
- `FwCfgInner::read_bytes()`
  - 按当前 selector/offset 读取，末尾 zero-fill。
- `FwCfgInner::write_bytes()`
  - 当前保留为 DMA WRITE no-op，行为等价于旧 `vm.rs` 里读出后丢弃。
- `FwCfgInner::write_dma_address_part()`
  - 累积 DMA address high/low dword，生成 `pending_dma`。
- `FwCfgInner::take_pending_dma()`
  - 取走 pending descriptor GPA。

对外设备接口：

- `FwCfgDevice::configure()`
- `FwCfgDevice::take_pending_dma()`
- `FwCfgDevice::read_string_bytes()`
- `FwCfgDevice::execute_dma<M: GuestMemoryAccessor>()`
  - 读 guest 中的 DMA descriptor。
  - 执行 `SELECT` / `READ` / `WRITE` / `SKIP`。
  - 写回 descriptor status。
  - guest memory 访问不持有 fw_cfg 内部锁。
- `FwCfgDevice::add_file_item()`
- `impl BaseDeviceOps<PortRange> for FwCfgDevice`
  - `address_range()`: `0x510..0x518+4`
  - `handle_read()`: data port `0x511`
  - `handle_write()`: selector port `0x510`、DMA address ports `0x514/0x518`

单测：

- `known_item_reads_reset_offset_on_select`
- `content_model_supports_bytes_and_u32`
- `selected_item_reads_data_and_zero_fills_after_end`
- `file_directory_lists_e820_file`
- `add_file_item_updates_directory_and_allocates_selectors`
- `dma_address_write_sets_one_pending_descriptor`
- `dma_request_parses_big_endian_descriptor`

### `virtualization/axdevice/src/device.rs`

新增字段：

- `AxVmDevices::x86_fw_cfg`

初始化：

- `AxVmDevices::new()`
  - 创建 `Arc<FwCfgDevice>`
  - 保存到 `x86_fw_cfg`
  - 通过 `add_port_dev()` 注册到统一 port device dispatch

新增 wrapper：

- `AxVmDevices::x86_fw_cfg_configure()`
- `AxVmDevices::x86_fw_cfg_read_string_bytes()`
- `AxVmDevices::x86_fw_cfg_execute_pending_dma()`
  - 合并 `take_pending_dma()` 和 `execute_dma()`。
  - 没有 pending DMA 时直接返回 `Ok(())`。

### `virtualization/axdevice/src/lib.rs`

新增：

- `pub mod fw_cfg`

### `virtualization/axvm-types/src/lib.rs`

新增设备类型：

- `EmulatedDeviceType::X86FwCfg`

用途：

- `FwCfgDevice::emu_type()` 不再伪装成 console 或其它设备。

### `virtualization/axvm/src/vm.rs`

新增/调整：

- `use axdevice::fw_cfg::FW_CFG_IO_DATA`
- `struct AxVmGuestMemory`
  - 持有 `*const AddrSpace<HostPagingHandler>`。
  - 实现 `GuestMemoryAccessor`。
  - 目的：让设备 DMA 使用现有 guest memory 访问接口。
- `AxVM::init()`
  - memory regions 确定后调用 `devices.x86_fw_cfg_configure()`。
- `IoWrite` 路径
  - 普通 port write 仍走 `AxVmDevices::handle_port_write()`。
  - write 后调用 `x86_fw_cfg_execute_pending_dma(&mem)`。
  - `inner_mut` 锁只用于取得 `address_space` 指针，不跨 fw_cfg DMA 执行。
- `IoStringRead`
  - `port == FW_CFG_IO_DATA` 时调用 `x86_fw_cfg_read_string_bytes()`。

移除/替代的旧职责：

- `FwCfgState`
- `handle_fw_cfg_io_read()`
- `handle_fw_cfg_io_write()`
- `handle_fw_cfg_dma()` / `execute_fw_cfg_dma()` 中的手写 descriptor 解析
- `vm.rs` 内的 fw_cfg item 构造、E820 构造、file directory 构造

## 锁边界

`FwCfgDevice::execute_dma()` 的锁粒度：

- 读 DMA descriptor：不持有 fw_cfg inner lock。
- `SELECT`：短暂持有 fw_cfg inner lock。
- `READ`：短暂持有 fw_cfg inner lock 取出 `Vec<u8>`，随后释放锁，再写 guest memory。
- `WRITE`：先读 guest memory，再短暂持有 fw_cfg inner lock 调 `write_bytes()`。
- `SKIP`：短暂持有 fw_cfg inner lock。
- 写 DMA status：不持有 fw_cfg inner lock。

`AxVM` 的锁粒度：

- `IoWrite` 后只短暂持有 `inner_mut`，取得 `address_space` 指针。
- `x86_fw_cfg_execute_pending_dma()` 执行期间不持有 `inner_mut`。

这避免了 VM 大锁覆盖 guest memory 读写和 fw_cfg 设备状态操作。

## 当前保留限制

- fw_cfg DMA WRITE 仍是 no-op，保持旧行为；当前 OVMF smoke 不依赖 WRITE mutation。
- 没有实现 TOML 配置驱动的 fw_cfg named item。
- 没有实现 file-backed content。
- 没有引入 PCI bus / PCI BAR / MSI-X；这不属于 fw_cfg 修改范围。

## 验证

本轮已验证：

```bash
cargo fmt --package axdevice --package axvm --package axvm-types
cargo test -p axdevice fw_cfg -- --nocapture  # 7 passed
cargo test -p axdevice --lib                  # 17 passed
cargo test -p axvm --lib                      # passed
cargo clippy -p axdevice --all-targets --all-features -- -D warnings
cargo clippy -p axvm --all-targets -- -D warnings
cargo clippy -p axvm-types --all-targets --all-features -- -D warnings
cargo check -p axdevice -p axvm
cargo run --manifest-path xtask/Cargo.toml -- axvisor build \
  --config os/axvisor/configs/board/qemu-x86_64.toml \
  --vmconfigs os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml
```

smoke run 仍按 `axvisor-2026-notebooks/start.md`。
