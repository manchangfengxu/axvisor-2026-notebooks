# Outer fw_cfg ACPI forwarding implementation

## Code changes

- `tgoskits/virtualization/axdevice/src/fw_cfg.rs`
  - `FW_CFG_IO_SELECTOR`
  - `QEMU_FW_CFG_ITEM_FILE_DIR`
  - `FwCfgFileDirEntry`
  - `parse_file_directory`
  - 作用：把 QEMU fw_cfg file directory 的解析留在 `axdevice::fw_cfg`，供 AxVisor 侧读取 outer fw_cfg 目录时复用。

- `tgoskits/virtualization/axdevice/src/device.rs`
  - `AxVmDevices::x86_fw_cfg_add_bytes_file`
  - 作用：提供窄入口，把上层拿到的 raw bytes 注册成 nested fw_cfg file item。

- `tgoskits/os/axvisor/src/x86_fw_cfg.rs`
  - `OuterQemuAcpiFwCfgBlobs`
  - `cached_outer_qemu_acpi_fw_cfg_blobs`
  - `read_outer_qemu_acpi_fw_cfg_blobs`
  - `collect_required_acpi_blobs`
  - `read_outer_fw_cfg_file_directory`
  - `MAX_FW_CFG_FILE_DIR_ENTRIES`
  - 作用：在 `os/axvisor` 读取 outer QEMU fw_cfg，并严格要求三项 ACPI blob 同时存在：
    - `etc/acpi/tables`
    - `etc/acpi/rsdp`
    - `etc/table-loader`

- `tgoskits/os/axvisor/src/config.rs`
  - `x86_uefi_ovmf_fw_cfg_forwarding_required`
  - `install_outer_qemu_acpi_fw_cfg`
  - `init_guest_vm`
  - 作用：只在 `x86_64 + UEFI + OVMF` 路径、`vm.init()` 之后安装 outer ACPI fw_cfg blobs。失败则 VM 创建失败，不做部分启用。

- `tgoskits/os/axvisor/src/main.rs`
  - `mod x86_fw_cfg`
  - 作用：接入 x86 outer fw_cfg reader。

- `tgoskits/os/axvisor/Cargo.toml`
  - `axdevice`
  - `x86`
  - 作用：`os/axvisor` 读取 outer fw_cfg 需要复用 `axdevice::fw_cfg` 常量/解析函数，并访问 x86 PIO。

- `tgoskits/Cargo.lock`
  - 作用：记录 `os/axvisor` 新增 workspace dependency 后的锁文件变化。

- `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs`
  - `use crate::host`
  - 作用：修复已有 SVM CPUID 路径的漏 import，解锁 `cargo check -p axvisor --features fs,svm`。

- `tgoskits/os/axvisor/src/shell/command/base.rs`
  - `unsupported_io_error`
  - `entry_file_name`
  - `entry_file_type`
  - `current_dir_text`
  - 作用：修复 shell 文件命令在 `x86_64-unknown-none` 与动态 `std-compat` 运行目标之间的 API 差异，避免真实 smoke 在构建阶段失败。

- `tgoskits/os/axvisor/src/shell/command/mod.rs`
  - `prompt_string`
  - 作用：同上，当前目录显示同时兼容 `ax_std` 的 `String` 和 host `std` 的 `PathBuf`。

## Verification

- `cargo test -p axdevice --lib`
  - 结果：通过，21 tests。

- `cargo check -p x86_vcpu --target x86_64-unknown-none --no-default-features --features svm`
  - 结果：通过。

- `cargo check -p axvm --target x86_64-unknown-none --no-default-features --features svm`
  - 结果：通过。

- `cargo check -p axvisor --target x86_64-unknown-none --features fs,svm`
  - 结果：通过。

- `cargo check -p axvisor --target x86_64-unknown-none --features fs,vmx`
  - 结果：通过。

- `timeout 45 cargo xtask qemu ...`
  - 结果：跑到 nested OVMF boot 目标点。
  - 关键输出：
    - `Loaded outer QEMU ACPI fw_cfg blobs: tables=131072 bytes, rsdp=20 bytes, table-loader=4096 bytes`
    - `VM[...] forwarding outer QEMU ACPI fw_cfg blobs`
    - `FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success`
    - `ArceOS UEFI shell-stage boot OK`
  - `timeout` 最终返回 124，表示 45 秒超时杀掉 QEMU，不表示 guest boot 失败。

## Notes

- 这轮没有接 virtio-blk INTx。
- 这轮没有生成 ACPI，也没有解释 `etc/table-loader`。
- ACPI blob 的来源仍是 outer QEMU fw_cfg，nested OVMF 按 QEMU 兼容 fw_cfg ABI 消费。
