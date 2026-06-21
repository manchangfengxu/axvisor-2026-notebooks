# Outer QEMU ACPI fw_cfg Forwarding Implementation Plan

## Steps

1. 在 `os/axvisor` 增加 outer fw_cfg ACPI reader
2. 先为 file directory 解析与 ACPI 三件套提取补单元测试
3. 在 `axdevice::AxVmDevices` 增加最小 fw_cfg 文件注册入口
4. 在 `init_guest_vm()` 的 `vm.init()` 之后接入按需缓存 + 严格注册
5. 运行针对性测试与一次 x86 UEFI smoke
