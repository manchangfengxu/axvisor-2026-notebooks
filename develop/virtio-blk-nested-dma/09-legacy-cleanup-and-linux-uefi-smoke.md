# 09 Legacy 清理与 Linux UEFI smoke

## Scope

这一步做两件事：

1. 把最近一轮 legacy virtio-blk 清理补记到 `develop/`。
2. 重新验证一次 `OVMF -> legacy virtio-blk -> Linux EFI stub`，确认当前失败点是不是还停在历史上的 `Failed to decompress kernel`。

不展开 modern virtio / virtio-pci，不引入 direction-2 本地替代框架。

## Legacy 清理补记

### 变更点

- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`
  - `GET_ID` 返回的设备标识补成 NUL 结尾，和 legacy virtio-blk 设备常见行为对齐。
  - 新增单测 `get_id_writes_a_null_terminated_serial`。

- `tgoskits/virtualization/axvm/src/vm.rs`
  - 把 `ovmf_virtio_blk` 从 `AxVM::inner_mut` 里拆出来，改成独立 `Mutex<LegacyVirtioBlk>`。
  - queue 处理和 guest memory copy 不再长期持有 `inner_mut` 这把大锁。

### 边界判断

- 这轮仍然是 **legacy-only**。
- 设备核心还在 `axdevice::virtio_blk`，AxVM 仍然保留 adapter glue。
- 还没有把它正规接入 `AxVmDevices` 的 first-class port device 路径。
- 也没有补 modern capability、virtio-pci config space、MSI/MSI-X、event idx。

### 验证

- `cargo test -p axdevice --lib`
- `cargo test -p axvm --lib`
- `cargo check -p axvisor --target x86_64-unknown-none --features fs,svm`
- `cargo check -p axvisor --target x86_64-unknown-none --features fs,vmx`

## Linux UEFI smoke

### 现状先确认

- 当前仓库里的 `tgoskits/os/axvisor/configs/vms/qemu/x86_64/linux-uefi-smp1.toml` 不能直接当作这条验证链的现成入口。
- 原因不是配置语法错，而是当前 `load_vm_images_from_filesystem()` 在 `boot_protocol = "uefi"` 且存在 OVMF firmware path 时，会只装载：
  - `OVMF_CODE` / `OVMF_VARS`
  - 可选 `disk_path`
  - 然后直接返回
- 也就是说这条 UEFI 路径当前真正可靠的是“**OVMF 从磁盘文件系统里加载 EFI payload**”，不是“AxVisor 预加载 Linux kernel_path / ramdisk_path 后交给 OVMF”。

### 本次实际 smoke 资产

- 外层 rootfs：
  - `tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img`
- 新建 nested disk：
  - `tmp/linux-uefi-smoke.img`
  - 写回到 `/guest/disks/linux-uefi-smoke.img`
- nested disk 内容：
  - `\EFI\BOOT\BOOTX64.EFI` = `tmp/BOOTX64-shell-edk2.efi`
  - `\vmlinuz.efi` = 从 rootfs 提取的 `/guest/linux/linux-qemu`
    - `file` 识别为：
      - `Linux kernel x86 boot executable bzImage, version 7.0.0-gdd6c438c3e64-dirty`
  - `\initramfs.cpio` = `tgoskits/os/axvisor/scripts/build_x86_64_linux_initramfs.sh` 生成的最小 initramfs
  - `\startup.nsh` = 自动执行 `FS0:\vmlinuz.efi ... initrd=\initramfs.cpio`

- 临时 VM 配置：
  - `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-linux-smoke.toml`

### 观察到的推进

- OVMF 成功从 nested disk 打开 `\EFI\BOOT\BOOTX64.EFI`
- EFI Shell 成功自动执行 `startup.nsh`
- OVMF 成功多次打开 `\vmlinuz.efi`
- OVMF 成功打开 `initramfs.cpio`

关键日志：

- `FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success`
- `Shell> FS0:\vmlinuz.efi ... initrd=\initramfs.cpio ...`
- `FSOpen: Open '\vmlinuz.efi' Success`
- `FSOpen: Open 'initramfs.cpio' Success`

### 当前新失败点

这次已经 **不是** 历史上的：

- `EFI stub: ERROR: Failed to decompress kernel`

而是推进到了 Linux 更靠后的早期阶段，然后 guest 自己反复触发 `#PF`：

- `guest_rip = 0xffffffff834d2400`
- `vector_name = #PF page fault`
- `qualification = 0xffff88800e276000`
- `rdi = 0xffff88800e276000`
- `rbx = r15 = 0x0e276000`

从寄存器看，这更像是 Linux 早期代码在访问某个 `phys=0x0e276000` 对应的 direct-map 虚拟地址时，guest 页表里没有把它映出来；已经不是“磁盘没读到”或“EFI stub 没接住 initrd”这一类前置问题。

## 当前结论

- 现有 legacy virtio-blk **足够支撑 OVMF 读盘并把 Linux EFI payload 拉起来**。
- 但它还 **不够支撑我们宣称 Linux guest 已跑通**。
- 当前主阻塞点已经从 legacy virtio-blk 协议面，推进到了 **Linux 早期运行时 / 平台信息 / 内存映射** 这一层。

## Next

下一步先不要把问题重新解释成“只差 modern virtio”。

更值得做的是：

1. 固定这条可复现的 Linux UEFI smoke 命令和镜像资产。
2. 结合 `guest_rip=0xffffffff834d2400`、`rdi=ffff88800e276000`、`rbx=0xe276000` 去查：
   - 这是不是 initrd / early memblock / EFI memory map 相关路径。
3. 再决定下一轮是查：
   - Linux 版本差异
   - EFI memory map / ACPI 平台信息
   - 还是更底层的 x86 guest 页表/异常上下文问题
