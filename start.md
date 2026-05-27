# AxVisor x86_64 OVMF/UEFI 启动运行手册

本文档是一个可直接照着执行的运行说明，目标是让任何人在自己的机器上都能复现 AxVisor 的 x86_64 OVMF/UEFI 启动验证。

当前运行目标不是完整启动 Linux EFI 系统，而是验证以下最小链路：

```text
AxVisor 启动
  -> 读取 OVMF VM 配置
  -> 为低端 RAM 和高端 OVMF firmware window 建立 GPA -> HPA 映射
  -> 从 AxVisor rootfs 加载 OVMF_CODE / OVMF_VARS
  -> BSP vCPU 从 x86 reset vector 进入 OVMF
  -> 通过 PEI / DXE / BDS
  -> 在 BDS 中发现 PCI / virtio-blk 设备
  -> 通过 VM-Exit / debugcon 日志观察 OVMF 后续启动设备路径
```

当前默认推荐使用自己编译的 DEBUG OVMF。这样能直接看到 `AXOVMF` / `OVMF debugcon` 日志，也能更快把失败点推进到具体的 PEI/DXE/BDS 源码位置。
---

## 1. 前置条件

你需要有一个包含以下目录的工作区：

```text
<workspace>/axvisor-2026-notebooks
<workspace>/edk2
<workspace>/tgoskits
```

下面用 `<workspace>` 代表你的仓库根目录。请把它替换成你自己的实际路径。

建议先定义一个环境变量：

```bash
export WORKSPACE=/path/to/axvisor-uefi
# example:
export WORKSPACE=~/work/axvisor-uefi
```

后续命令都可以直接复制粘贴，只需要把 `$WORKSPACE` 换成你的实际路径即可。

---

## 2. 运行目录

所有 AxVisor 启动命令都在下面这个目录执行：

```bash
cd "$WORKSPACE/tgoskits/os/axvisor"
```

不要在仓库根目录执行 `cargo xtask qemu`。仓库根目录的 xtask 是 `tgoskits` 顶层工具，不提供 AxVisor 的 `qemu` 子命令。

---

## 3. QEMU 和 KVM

推荐使用可用的 QEMU 版本，并确保它在 `PATH` 中优先可见。

确认当前使用的 QEMU：

```bash
which qemu-system-x86_64
qemu-system-x86_64 --version
```

如果你有自己编译的 QEMU，请把它放到 `PATH` 前面，例如：

```bash
export PATH="/opt/qemu-10.2.1/bin:$PATH"
```

AxVisor 的 x86_64 OVMF 启动依赖 KVM 加速。请确认当前用户可以访问 `/dev/kvm`：

```bash
ls -l /dev/kvm
```

正常情况下会看到类似：

```text
crw-rw---- 1 root kvm ... /dev/kvm
```

如果没有权限，QEMU 会报类似：

```text
qemu-system-x86_64: -accel kvm: Could not access KVM kernel module: Permission denied
qemu-system-x86_64: -accel kvm: failed to initialize kvm: Permission denied
```

---

## 4. 准备运行配置

OVMF 启动使用三类配置文件：

```text
$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml
$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml
$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml
```

它们的作用分别是：

| 文件 | 作用 |
| --- | --- |
| `qemu-x86_64.toml` | AxVisor board/build 配置，包含 target、features、日志等级等。 |
| `qemu-x86_64-runtime.toml` | 外层 QEMU 启动配置，包含 KVM、内存、磁盘等参数。 |
| `ovmf-x86_64-qemu-smp1.toml` | Guest VM 配置，描述 OVMF 路径、加载地址、reset vector、内存布局等。 |

如果目录不存在，先创建：

```bash
mkdir -p "$WORKSPACE/tgoskits/os/axvisor/tmp/configs"
```

如果需要从模板重新生成临时 VM 配置：

```bash
cp "$WORKSPACE/tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml" \
  "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml"
```

当前 OVMF VM 配置中的关键字段应为：

```toml
[kernel]
entry_point = 0xffff_fff0
boot = "uefi"
image_location = "fs"
kernel_path = "/guest/uefi/empty-kernel-placeholder.bin"
kernel_load_addr = 0x20_0000
enable_bios = false

ovmf_code_path = "/guest/ovmf/OVMF_CODE.fd"
ovmf_code_base = 0xffc8_4000
ovmf_vars_path = "/guest/ovmf/OVMF_VARS.fd"
ovmf_vars_base = 0xffc0_0000
reset_vector = 0xffff_fff0

memory_regions = [
  [0x0000_0000, 0x0400_0000, 0x7, 0],
  [0xffc0_0000, 0x0040_0000, 0x7, 0],
]
```

其中：

- `boot = "uefi"` 表示进入 UEFI/OVMF 加载链路。
- `reset_vector = 0xffff_fff0` 表示 x86 reset vector。
- `ovmf_code_base = 0xffc8_4000` 用于让 OVMF_CODE 文件末尾的 reset-vector 代码落在 `0xffff_fff0`。
- `ovmf_vars_base = 0xffc0_0000` 是 OVMF 变量区起始 GPA。
- `[0xffc0_0000, 0x0040_0000]` 是 4 MiB firmware window，覆盖 OVMF_VARS、OVMF_CODE 和 reset vector。

当前 BDS / PCI 枚举还需要低端 PCI MMIO passthrough，否则 OVMF 访问 framebuffer / BAR 时会在 `0x81000000` 附近触发 `NestedPageFault`：

```toml
passthrough_devices = [
  ["PCI Low MMIO", 0x8000_0000, 0x8000_0000, 0x6000_0000, 0x1],
  ["PCI MMIO", 0xe000_0000, 0xe000_0000, 0x1000_0000, 0x1],
  ["IO APIC", 0xfec0_0000, 0xfec0_0000, 0x1000, 0x1],
  ["Local APIC", 0xfee0_0000, 0xfee0_0000, 0x1000, 0x1],
  ["HPET", 0xfed0_0000, 0xfed0_0000, 0x1000, 0x1],
]
```

---

## 5. 准备 OVMF 镜像

当前配置中的 OVMF 路径是：

```text
/guest/ovmf/OVMF_CODE.fd
/guest/ovmf/OVMF_VARS.fd
```

这是 AxVisor rootfs 内部路径，不是宿主机路径。

运行时实际使用的 rootfs 镜像通常是：

```text
$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

### 5.1 默认来源：自编译 DEBUG OVMF

当前调试默认使用自己编译的 DEBUG OVMF：

```text
$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd
$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_VARS.fd
```

注意：当前可复现到 BDS / `VirtioBlkInit` 的镜像来自仓库根目录下的 `Build/`。不要和
`$WORKSPACE/edk2/Build/` 下的旧产物混用；旧 `OVMF_CODE.fd` 可能不包含
`QemuFwCfgPei.c` 中绕开 `rep insb` 的改动，会退回到
`VMX unsupported IO-Exit ... port: 0x511`。

原因：

```text
1. 可以在 OVMF 源码里加最小 DEBUG 日志，例如 AXOVMF。
2. 可以通过 AxVisor 的 0x402 debugcon 端口看到 OVMF 内部日志。
3. 能把失败点定位到 PlatformPei / QemuFwCfgPei 等具体阶段，而不是只看 VM-Exit 或 triple fault。
```

### 5.2 构建 DEBUG OVMF

在当前这棵 EDK2 树中，工具链 tag 使用 `GCC`，不是 `GCC5`。

```bash
export WORKSPACE=/home/ssdns/work/axvisor-uefi
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

构建成功后应生成：

```text
$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd
$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_VARS.fd
```

### 5.3 OVMF_CODE 大小和 reset vector

当前使用的 DEBUG OVMF_CODE 大小是 `0x37c000`，因此：

```text
ovmf_code_base = 0x1_0000_0000 - 0x37c000 = 0xffc8_4000
```

这样 OVMF_CODE 文件末尾的 reset-vector 代码才能落在：

```text
0xffff_fff0
```

如果以后 OVMF_CODE 文件大小变化，必须重新计算 `ovmf_code_base`，不要盲目沿用 `0xffc8_4000`。

### 5.4 写入 AxVisor rootfs

推荐使用 `debugfs -w` 直接更新 ext4 rootfs，不需要 sudo mount。

覆盖 `OVMF_CODE.fd`：

```bash
debugfs -w -R 'rm /guest/ovmf/OVMF_CODE.fd' \
  "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"

debugfs -w -R "write $WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd /guest/ovmf/OVMF_CODE.fd" \
  "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"
```

如果也需要覆盖 `OVMF_VARS.fd`：

```bash
debugfs -w -R 'rm /guest/ovmf/OVMF_VARS.fd' \
  "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"

debugfs -w -R "write  $WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_VARS.fd /guest/ovmf/OVMF_VARS.fd" \
  "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"
```

写入后建议 dump 出来核对 SHA256：

```bash
tmpfile=$(mktemp)
debugfs -R "dump /guest/ovmf/OVMF_CODE.fd $tmpfile" \
  "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img" >/dev/null 2>&1

sha256sum \
  "$WORKSPACE/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd" \
  "$tmpfile"

rm -f "$tmpfile"
```

两个 hash 一致，才说明 AxVisor rootfs 中的 `OVMF_CODE.fd` 已经更新成功。

### 5.5 备用来源：系统 OVMF

宿主机上的系统 OVMF 通常位于：

```text
/usr/share/OVMF/OVMF_CODE_4M.fd
/usr/share/OVMF/OVMF_VARS_4M.fd
```

它们可以作为备用来源，但当前调试不推荐默认使用，因为系统 OVMF 没有本项目添加的 `AXOVMF` 日志，也不一定和当前 `ovmf_code_base` 假设一致。

如果临时使用系统 OVMF，也必须先确认 `OVMF_CODE_4M.fd` 的实际大小，并据此检查 VM 配置中的 `ovmf_code_base`。

复制完成后，AxVisor 运行时才能打开：

```text
/guest/ovmf/OVMF_CODE.fd
/guest/ovmf/OVMF_VARS.fd
```

如果你没有把文件复制到 rootfs，而是直接在 VM 配置中写宿主机路径，例如 `/usr/share/OVMF/...`，AxVisor 会在自己的 rootfs 中查找该路径，然后报 `NotFound`。

---

## 6. 启动命令

在 `tgoskits/os/axvisor` 目录下执行：

```bash
cargo xtask qemu \
  --config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml" \
  --qemu-config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml" \
  --vmconfigs "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml"
```

如果需要避免命令长时间卡住，可以加 `timeout`：

```bash
timeout 240 cargo xtask qemu \
  --config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml" \
  --qemu-config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml" \
  --vmconfigs "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml"
```

---

## 7. 预期关键日志

### 7.1 OVMF 镜像加载成功

应能看到类似日志：

```text
Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000
Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000
```

这说明：

```text
AxVisor 已经从 rootfs 找到 OVMF 文件
OVMF_CODE / OVMF_VARS 已写入对应 guest physical address
```

### 7.2 reset-vector VMCS 设置生效

应能看到类似日志：

```text
VMX guest entry setup: entry=0xfffffff0, reset_vector=true, cs=0xf000, cs_base=0xffff0000, rip=0xfff0
```

这说明：

```text
BSP entry 被识别为 x86 reset vector
VMCS 中写入了接近 x86 reset state 的 CS/RIP
CS.base + RIP = 0xffff0000 + 0xfff0 = 0xfffffff0
```

### 7.3 OVMF debugcon 输出可见

如果自编译 DEBUG OVMF 和 AxVisor 的 debugcon 链路都正常，应能看到类似日志：

```text
OVMF debugcon: SecCoreStartupWithStack(...)
OVMF debugcon: Platform PEIM Loaded
OVMF debugcon: QemuFwCfgProbe: Supported 1, DMA 1
```

这说明：

```text
OVMF 的 PcdDebugIoPort=0x402 已经通过 AxVisor 设备层接出
OVMF 内部 DEBUG() 能被宿主机看到
```

### 7.4 PEI 阶段进入 QemuFwCfgPei

早期 PEI 验证日志是：

```text
OVMF debugcon: AXOVMF InternalQemuFwCfgDmaBytes: size=0x4 buffer=... control=0x2 ...
```

历史上如果随后出现：

```text
!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!
Find image based on IP(0x848FA8) ... PlatformPei.dll
```

则说明已经进入 **PEI**，并且 failure site 落在 `PlatformPei.efi` 的 `QemuFwCfg` DMA 路径，而不是 reset-vector 附近。当前代码已跨过该阶段。

### 7.5 DXE / BDS 当前关键日志

当前应能继续看到 DXE 和 BDS 日志：

```text
OVMF debugcon: Loading driver at ... PciHostBridgeDxe.efi
OVMF debugcon: Loading driver at ... QemuFwCfgAcpiPlatform.efi
OVMF debugcon: Loading driver at ... BdsDxe.efi
OVMF debugcon: Loading driver at ... PciBusDxe.efi
OVMF debugcon: Loading driver at ... VirtioPciDeviceDxe.efi
OVMF debugcon: FrameBufferBase: 0x80000000, FrameBufferSize: 0x3E8000
OVMF debugcon: PlatformBootManagerAfterConsole
OVMF debugcon: Found Mass Storage device: PciRoot(0x0)/Pci(0x3,0x0)
OVMF debugcon: VirtioBlkInit: LbaSize=0x200[B] NumBlocks=0x200000[Lba]
OVMF debugcon: BlockSize : 512
OVMF debugcon: LastBlock : 1FFFFF
```

这说明 OVMF 已经进入 **BDS**，并且能发现 PCI display、串口控制台和 virtio-blk 磁盘。

---

## 8. 常见错误和处理

### 8.1 在错误目录运行 xtask

错误现象：

```text
error: unrecognized subcommand 'qemu'
Usage: tg-xtask <COMMAND>
```

原因：

```text
当前目录不是 $WORKSPACE/tgoskits/os/axvisor。
```

处理：

```bash
cd "$WORKSPACE/tgoskits/os/axvisor"
```

然后重新运行 `cargo xtask qemu`。

### 8.2 KVM 权限不足

错误现象：

```text
qemu-system-x86_64: -accel kvm: Could not access KVM kernel module: Permission denied
```

原因：

```text
当前用户没有访问 /dev/kvm 的权限。
```

处理：

```text
使用具备 KVM 权限的用户运行，或把当前用户加入 kvm 组后重新登录。
```

### 8.3 OVMF 文件找不到

错误现象：

```text
Failed to open /guest/ovmf/OVMF_CODE.fd
```

或：

```text
Failed to open /usr/share/OVMF/OVMF_CODE_4M.fd
```

原因：

```text
AxVisor 看到的是自己的 rootfs，不是宿主机根目录。
```

处理：

```text
把自编译 DEBUG OVMF 或备用系统 OVMF 复制进 AxVisor rootfs 的 /guest/ovmf/。
VM 配置中继续使用 /guest/ovmf/OVMF_CODE.fd 和 /guest/ovmf/OVMF_VARS.fd。
```

### 8.4 placeholder kernel 文件找不到

错误现象：

```text
Failed to open /guest/uefi/empty-kernel-placeholder.bin
```

原因：

```text
UEFI 模式不应该继续读取 Linux kernel header。
```

如果仍出现该错误，说明当前运行的代码不是最新构建，或 VM 配置中的 `boot = "uefi"` 没有生效。

### 8.5 OVMF 在 debugcon 输出前后出现 unsupported I/O

如果看到类似：

```text
VMX unsupported IO-Exit: VmxIoExitInfo { access_size: 0x1, is_in: false, is_string: true, is_repeat: true, port: 0x402 }
```

说明 OVMF 的 debugcon 正在走字符串 I/O，而 AxVisor 的 VMX 层还不支持这类 I/O 退出。

当前已采用的最小处理是修改 OVMF 的 `DebugLib.c`，让 debugcon 逐字节写出，避免 `rep outsb`。

### 8.6 OVMF 在 PEI 中触发 #BR

历史上观察到的 failure 是：

```text
OVMF debugcon: AXOVMF InternalQemuFwCfgDmaBytes: size=0x4 buffer=... control=0x2 ...
!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!
Find image based on IP(0x848FA8) ... PlatformPei.dll
```

这说明不是 reset-vector 问题，而是 PEI 中 `QemuFwCfgPei` 的 DMA 读路径需要继续定位。当前代码已经跨过该问题。

### 8.7 BDS 访问 0x81000000 时 NestedPageFault

错误现象：

```text
VM[1] run VCpu[0] unhandled vmexit: NestedPageFault { addr: GPA:0x81000000, access_flags: READ }
```

原因：

```text
OVMF 的 PCI root bridge 宣告低端 MMIO window 为 0x80000000 - 0xDFFFFFFF。
如果 VM 配置没有映射该窗口，BDS 连接 PCI display / BAR 时会访问未映射 GPA。
```

处理：

```text
在 ovmf-x86_64-qemu-smp1.toml 的 passthrough_devices 中加入：
["PCI Low MMIO", 0x8000_0000, 0x8000_0000, 0x6000_0000, 0x1]
```

---

## 9. 当前运行状态的判断标准

当前阶段的成功标准不是启动 Linux EFI。

当前阶段的成功标准是：

```text
1. AxVisor 能创建 OVMF VM。
2. OVMF_CODE / OVMF_VARS 能从 rootfs 加载到预期 GPA。
3. BSP vCPU 能以 reset-vector 语义进入 OVMF。
4. OVMF debugcon 日志能够通过 AxVisor 看到。
5. OVMF 能跨过 PEI 和 DXE，进入 BDS。
6. BDS 能发现 PCI / virtio-blk 磁盘，并输出 `VirtioBlkInit` / `BlockSize` / `LastBlock`。
7. 能通过 VM-Exit / exception / debugcon 日志定位 BDS 启动设备路径的下一阶段缺失内容。
```

如果日志已经显示 `Found Mass Storage device: PciRoot(0x0)/Pci(0x3,0x0)` 和 `VirtioBlkInit`，说明当前最小 OVMF 启动链路已经推进到 BDS。后续工作应转向确认 BDS 是否发起磁盘读、是否找到 EFI boot 文件、以及当前 guest 介质是否实际可启动。
