# AxVisor x86_64 OVMF/UEFI 启动手册

这份文档记录当前变基后的通用启动流程。目标是跑通下面这条链路：

```text
外层 QEMU
  -> AxVisor UEFI app
  -> AxVisor 创建 OVMF guest
  -> 嵌套 OVMF 从 legacy virtio-blk 读 GPT/FAT 盘
  -> 打开 \EFI\BOOT\BOOTX64.EFI
  -> 启动 ArceOS UEFI helloworld
```

当前验证重点不是 Linux EFI stub，而是先确认 OVMF、fw_cfg、debugcon、virtio-blk、BDS 磁盘启动这一段能稳定走通。

## 1. 工作区

下面用 `$WORKSPACE` 表示仓库根目录：

```bash
export WORKSPACE=~/work/axvisor-uefi
```

目录应包含：

```text
$WORKSPACE/tgoskits
$WORKSPACE/axvisor-2026-notebooks
$WORKSPACE/edk2
```

AxVisor 启动命令在这里执行：

```bash
cd "$WORKSPACE/tgoskits/os/axvisor"
```

不要在 `$WORKSPACE/tgoskits` 根目录跑旧的 `cargo xtask qemu`。当前可用的是 `os/axvisor` 下面的 xtask 入口。

## 2. QEMU 和 KVM

确认 QEMU：

```bash
which qemu-system-x86_64
qemu-system-x86_64 --version
```

确认 KVM 权限：

```bash
ls -l /dev/kvm
```

如果没有权限，QEMU 会报：

```text
qemu-system-x86_64: -accel kvm: Could not access KVM kernel module: Permission denied
```

当前 x86_64 嵌套虚拟化 smoke 默认走 KVM + VMX。没有 KVM 时不要继续看 OVMF 日志，先修宿主权限。

## 3. 三个镜像的关系

这里最容易搞错。现在不要把 UEFI 可启动盘直接当外层 `--rootfs`。

正确关系是：

```text
外层 QEMU 启动盘:
  tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img

AxVisor rootfs 内的 OVMF firmware:
  /guest/ovmf/OVMF_CODE.fd
  /guest/ovmf/OVMF_VARS.fd

AxVisor rootfs 内的嵌套 guest disk:
  /guest/disks/uefi-boot-test.img
```

`/guest/disks/uefi-boot-test.img` 才是嵌套 OVMF 通过 virtio-blk 看到的盘。它里面有 GPT、ESP 和 `\EFI\BOOT\BOOTX64.EFI`。

如果把 `$WORKSPACE/tmp/uefi-boot-test.img` 直接传给外层 `--rootfs`，外层 OVMF 会先从它的 `EFI/BOOT/BOOTX64.EFI` 启动。这样看到的是宿主 OVMF 直接加载 helloworld，不是 AxVisor 启动嵌套 OVMF。

## 4. VM 配置

主配置文件：

```text
$WORKSPACE/tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml
```

临时运行配置：

```text
$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml
```

如果临时配置不存在，先复制：

```bash
mkdir -p "$WORKSPACE/tgoskits/os/axvisor/tmp/configs"
cp "$WORKSPACE/tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml" \
  "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml"
```

当前 OVMF VM 配置里关键字段应是：

```toml
[kernel]
entry_point = 0xffff_fff0
boot_protocol = "uefi"
image_location = "fs"
kernel_path = "/guest/uefi/empty-kernel-placeholder.bin"
kernel_load_addr = 0x20_0000
enable_bios = true

ovmf_code_path = "/guest/ovmf/OVMF_CODE.fd"
ovmf_code_base = 0xffc8_4000
ovmf_vars_path = "/guest/ovmf/OVMF_VARS.fd"
ovmf_vars_base = 0xffc0_0000
reset_vector = 0xffff_fff0
disk_path = "/guest/disks/uefi-boot-test.img"
```

说明：

- `boot_protocol = "uefi"` 是当前上游配置模型识别的字段。
- `enable_bios = true` 在这里表示走固件类启动路径，不是只能启动 SeaBIOS。
- `ovmf_code_base = 0xffc8_4000` 对应当前 `OVMF_CODE.fd` 大小 `0x37c000`，使 reset vector 落在 `0xfffffff0`。
- `disk_path` 是 AxVisor rootfs 内路径，不是宿主机路径。

旧配置里的 `boot = "uefi"` / `enable_bios = false` 已经过时。会在配置解析时报：

```text
boot_protocol requires enable_bios = true
```

## 5. 准备 OVMF firmware

当前默认使用 DEBUG OVMF：

```text
$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd
$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_VARS.fd
```

构建命令：

```bash
export EDK2_WORKSPACE="$WORKSPACE/edk2"
export PACKAGES_PATH="$EDK2_WORKSPACE"
export BUILD_DIR="$WORKSPACE/Build"
export EDK_TOOLS_PATH="$EDK2_WORKSPACE/BaseTools"
export CONF_PATH="$EDK2_WORKSPACE/Conf"
export PATH="$EDK2_WORKSPACE/BaseTools/BinWrappers/PosixLike:$EDK2_WORKSPACE/BaseTools/Bin/Linux-x86_64:$PATH"

cd "$EDK2_WORKSPACE"
. edksetup.sh
build -p OvmfPkg/OvmfPkgX64.dsc -a X64 -t GCC -b DEBUG -n 4 \
  --build-dir "$BUILD_DIR"
```

写入 AxVisor rootfs：

```bash
ROOTFS="$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"

debugfs -w -R 'rm /guest/ovmf/OVMF_CODE.fd' "$ROOTFS"
debugfs -w -R "write $WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd /guest/ovmf/OVMF_CODE.fd" "$ROOTFS"

debugfs -w -R 'rm /guest/ovmf/OVMF_VARS.fd' "$ROOTFS"
debugfs -w -R "write $WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_VARS.fd /guest/ovmf/OVMF_VARS.fd" "$ROOTFS"
```

如果 `rm` 报文件不存在，可以忽略，再执行 `write`。

可选核对：

```bash
tmpfile=$(mktemp)
debugfs -R "dump /guest/ovmf/OVMF_CODE.fd $tmpfile" "$ROOTFS" >/dev/null 2>&1
sha256sum "$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd" "$tmpfile"
rm -f "$tmpfile"
```

两个 hash 一致，说明 rootfs 里的 OVMF_CODE 已更新。

## 6. 准备嵌套 UEFI 启动盘

当前已验证的嵌套盘来源是：

```text
$WORKSPACE/tmp/nested-uefi-disk.img
```

它会被写入 AxVisor rootfs：

```text
/guest/disks/uefi-boot-test.img
```

写入命令：

```bash
ROOTFS="$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"

debugfs -w -R 'mkdir /guest/disks' "$ROOTFS"
debugfs -w -R 'rm /guest/disks/uefi-boot-test.img' "$ROOTFS"
debugfs -w -R "write $WORKSPACE/tmp/nested-uefi-disk.img /guest/disks/uefi-boot-test.img" "$ROOTFS"
```

如果目录或旧文件不存在，可以忽略对应错误。

检查 rootfs 内文件：

```bash
debugfs -R 'ls -l /guest/disks' "$ROOTFS"
```

应看到：

```text
uefi-boot-test.img  33554432
```

检查嵌套盘里的默认启动文件：

```bash
mdir -i "$WORKSPACE/tmp/nested-uefi-disk.img@@1048576" ::/EFI/BOOT
```

当前 `BOOTX64.EFI` 是 ArceOS UEFI helloworld，大小约 23 KiB。

## 7. 构建并写入 ArceOS UEFI payload

如果要更新 `BOOTX64.EFI`：

```bash
cd "$WORKSPACE/tgoskits"
cargo build -p ax-uefi-helloworld --target x86_64-unknown-uefi
```

确认产物类型：

```bash
file "$WORKSPACE/tgoskits/target/x86_64-unknown-uefi/debug/ax-uefi-helloworld.efi"
```

期望类似：

```text
PE32+ executable (EFI application) x86-64
```

写入嵌套盘 ESP：

```bash
mcopy -o -i "$WORKSPACE/tmp/nested-uefi-disk.img@@1048576" \
  "$WORKSPACE/tgoskits/target/x86_64-unknown-uefi/debug/ax-uefi-helloworld.efi" \
  ::/EFI/BOOT/BOOTX64.EFI
```

写完后记得重新把嵌套盘写入 AxVisor rootfs：

```bash
ROOTFS="$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"
debugfs -w -R 'rm /guest/disks/uefi-boot-test.img' "$ROOTFS"
debugfs -w -R "write $WORKSPACE/tmp/nested-uefi-disk.img /guest/disks/uefi-boot-test.img" "$ROOTFS"
```

## 8. 启动命令

进入运行目录：

```bash
cd "$WORKSPACE/tgoskits/os/axvisor"
```

推荐显式传入 Alpine rootfs：

```bash
timeout 45 cargo xtask qemu \
  --config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml" \
  --qemu-config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml" \
  --vmconfigs "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml" \
  --rootfs "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"
```

`timeout 45` 退出码通常是 `124`，这不一定表示失败。只要成功日志已经出现，说明 smoke 已跑到目标点。QEMU 会继续停在 UEFI payload 的输出界面，所以需要 timeout 或手动退出。

## 9. 预期日志

应看到 AxVisor 加载 OVMF：

```text
Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000
Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000
```

应看到 AxVisor 加载嵌套 virtio-blk 后端盘：

```text
Loading virtio-blk disk image from /guest/disks/uefi-boot-test.img (33554432 bytes)
```

应看到 OVMF 通过 virtio-blk 读到 GPT/FAT：

```text
OVMF debugcon:  BlockSize : 512
OVMF debugcon:  LastBlock : FFFF
OVMF debugcon:  Valid primary and Valid backup partition table
```

最终应看到：

```text
OVMF debugcon: FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success
ArceOS UEFI helloworld
ArceOS UEFI shell-stage boot OK
```

这几个日志同时出现，说明方向一的最小闭环已经跑通。

## 10. 常见问题

### 外层 OVMF 直接启动 helloworld

现象：

```text
BdsDxe: starting Boot0002 "UEFI Misc Device"
ArceOS UEFI helloworld
```

但没有 AxVisor banner，也没有 `Loading OVMF_CODE image...`。

原因通常是把 UEFI 可启动盘直接传给了外层 `--rootfs`。外层 OVMF 抢先启动了这个盘里的 `EFI/BOOT/BOOTX64.EFI`。

修正：外层 `--rootfs` 使用 Alpine ext4 rootfs：

```text
$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

嵌套启动盘放进 rootfs 内的：

```text
/guest/disks/uefi-boot-test.img
```

### VM 配置解析失败

现象：

```text
boot_protocol requires enable_bios = true
```

修正：

```toml
boot_protocol = "uefi"
enable_bios = true
```

不要再用旧的：

```toml
boot = "uefi"
enable_bios = false
```

### VM setup 出现 AlreadyExists

现象：

```text
Mapping error: AlreadyExists
VM[1] setup failed: AxErrorKind::AlreadyExists
```

本次变基后的已知原因是 APIC access page `0xfee00000` 被重复映射。当前代码应只保留上游 VMX 条件映射：

```text
X86_APIC_ACCESS_GPA + x86_apic_access_page_addr()
```

不要恢复旧的：

```text
GuestPhysAddr::from(0xfee0_0000)
crate::vcpu::EmulatedLocalApic::virtual_apic_access_addr()
```

### 找不到 OVMF 文件

现象：

```text
Failed to open /guest/ovmf/OVMF_CODE.fd
```

原因是 AxVisor 在自己的 rootfs 里找路径，不是在宿主机文件系统里找。把 OVMF_CODE / OVMF_VARS 写入 Alpine rootfs 的 `/guest/ovmf/`。

### 找不到嵌套盘

现象：

```text
Failed to open /guest/disks/uefi-boot-test.img
```

把 `$WORKSPACE/tmp/nested-uefi-disk.img` 写入 rootfs 内的 `/guest/disks/uefi-boot-test.img`。

### debugcon 没输出

变基后 AxVisor 自身作为 UEFI app 启动，日志通道和早期裸机路径不同。当前 smoke 已经能在 QEMU 控制台看到 `OVMF debugcon:`，如果看不到，先确认：

- 使用的是 DEBUG OVMF。
- OVMF_CODE 已写入当前实际传给 QEMU 的 rootfs。
- 启动命令使用的是 `os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml`。

## 11. 快速验证命令

代码侧：

```bash
cd "$WORKSPACE/tgoskits"
cargo test -p axdevice virtio_blk -- --nocapture
cargo test -p axvm --lib
cargo test -p x86_vcpu --lib
cargo fmt --check --package axdevice --package axvm --package axvisor --package x86_vcpu
git diff --check
```

旧 hack 不应再出现：

```bash
rg "rewrite desc|rewrite_ovmf|translated_queue_pfn|translate_ovmf_virtio_blk_queue_pfn|OvmfVirtioBlkIoState" \
  virtualization os
```

这条命令无输出才是预期结果。
