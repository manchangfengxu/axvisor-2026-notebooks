# Outer QEMU ACPI fw_cfg Forwarding

## Goal

让 nested OVMF / 后续 nested Linux 看到与当前 outer QEMU PCI / ECAM 视图同源的 ACPI 平台信息，不再停留在 `etc/e820` 单项输入。

## Boundary

- `outer QEMU fw_cfg` 是 ACPI 真源
  - `etc/acpi/tables`
  - `etc/acpi/rsdp`
  - `etc/table-loader`
- `os/axvisor`
  - 负责读取 outer fw_cfg 的三份原始 blob
  - 负责按需缓存
- `axdevice::fw_cfg`
  - 继续只负责 fw_cfg file item 行为
- `axvm`
  - 只负责把上层提供的 blob 注册进 nested fw_cfg
  - 不生成 ACPI，不解释 table-loader

## Scope

- 只为 `x86_64 + UEFI + OVMF` 打开这条路径
- 三件套必须完整成功
- 不做部分启用
- 非该路径 VM 保持当前行为

## Failure Policy

- 若当前 VM 走 `x86_64 + UEFI + OVMF` 路径，且 outer fw_cfg 三件套任一缺失或读取失败，则 VM 创建失败
- 不接受“缺一项也先启动”

## Lifecycle

1. `init_guest_vm()` 创建 VM、加载镜像
2. `vm.init()` 创建 nested `FwCfgDevice`
3. `os/axvisor` 按需读取并缓存 outer fw_cfg ACPI blobs
4. `os/axvisor` 通过窄胶水把三份 blob 注册进 nested fw_cfg
5. guest 启动后由 OVMF 按 QEMU 兼容路径消费三项 fw_cfg 文件

## Notes

- 当前 `PCI config / ECAM passthrough` 仍然保留
- 这次实现只补平台信息链，不接 virtio-blk INTx
