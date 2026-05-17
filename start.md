# AxVisor x86_64 OVMF/UEFI 启动运行手册

本文档用于说明如何在当前仓库中运行 AxVisor x86_64 OVMF/UEFI 启动验证。

当前运行目标不是完整启动 Linux EFI 系统，而是验证以下最小链路：

```text
AxVisor 启动
  -> 读取 OVMF VM 配置
  -> 为低端 RAM 和高端 OVMF firmware window 建立 GPA -> HPA 映射
  -> 从 AxVisor rootfs 加载 OVMF_CODE / OVMF_VARS
  -> BSP vCPU 从 x86 reset vector 进入 OVMF
  -> 通过 VM-Exit / fault 日志观察 OVMF 后续执行状态
```

---

## 1. 运行目录

所有命令都应在 AxVisor 目录下执行：

```bash
cd /home/ssdns/code/tgoskits/os/axvisor
```

不要在仓库根目录执行 `cargo xtask qemu`。仓库根目录的 xtask 是 tgoskits 顶层工具，不提供 AxVisor 的 `qemu` 子命令。

---

## 2. QEMU 环境

推荐使用本地编译的 QEMU 10.2.1：

```bash
export PATH="/opt/qemu-10.2.1/bin:$PATH"
```

确认当前使用的 QEMU：

```bash
which qemu-system-x86_64
qemu-system-x86_64 --version
```

如果系统中还有 apt 安装的 QEMU 8.x，需要确保 `/opt/qemu-10.2.1/bin` 位于 `PATH` 前面。

---

## 3. KVM 权限要求

当前 x86_64 AxVisor 运行依赖外层 QEMU KVM 加速。

确认当前用户是否可以访问 `/dev/kvm`：

```bash
ls -l /dev/kvm
```

正常情况下 `/dev/kvm` 类似：

```text
crw-rw---- 1 root kvm ... /dev/kvm
```

当前用户需要属于 `kvm` 组，或者用具备 KVM 权限的方式运行。否则会出现：

```text
qemu-system-x86_64: -accel kvm: Could not access KVM kernel module: Permission denied
qemu-system-x86_64: -accel kvm: failed to initialize kvm: Permission denied
```

---

## 4. 准备运行配置

当前 OVMF 启动使用三类配置文件：

```text
tmp/configs/qemu-x86_64.toml
tmp/configs/qemu-x86_64-runtime.toml
tmp/configs/ovmf-x86_64-qemu-smp1.toml
```

含义如下：

| 文件 | 作用 |
| --- | --- |
| `tmp/configs/qemu-x86_64.toml` | AxVisor board/build 配置，包含 target、features、日志等级等。 |
| `tmp/configs/qemu-x86_64-runtime.toml` | 外层 QEMU 启动配置，包含 KVM、内存、磁盘等参数。 |
| `tmp/configs/ovmf-x86_64-qemu-smp1.toml` | Guest VM 配置，描述 OVMF 路径、加载地址、reset vector、内存布局等。 |

如果 `tmp/configs` 不存在，先创建：

```bash
mkdir -p tmp/configs
```

如果需要从仓库模板重新生成临时 VM 配置：

```bash
cp configs/vms/ovmf-x86_64-qemu-smp1.toml tmp/configs/ovmf-x86_64-qemu-smp1.toml
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

---

## 5. 准备 OVMF 镜像

当前配置中的 OVMF 路径是：

```text
/guest/ovmf/OVMF_CODE.fd
/guest/ovmf/OVMF_VARS.fd
```

这是 AxVisor rootfs 内部路径，不是宿主机路径。

宿主机上的 OVMF 文件通常位于：

```text
/usr/share/OVMF/OVMF_CODE_4M.fd
/usr/share/OVMF/OVMF_VARS_4M.fd
```

需要把它们复制进 AxVisor 使用的 rootfs 镜像中。

当前运行时实际使用的 rootfs 镜像路径是 monorepo 工作区根目录下的：

```text
/home/ssdns/code/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

不要把 OVMF 复制到下面这个路径：

```text
/home/ssdns/code/tgoskits/os/axvisor/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

原因是 QEMU 配置中的：

```text
file=${workspace}/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

这里的 `${workspace}` 指 monorepo 根目录 `/home/ssdns/code/tgoskits`，不是 `/home/ssdns/code/tgoskits/os/axvisor`。

挂载并复制：

```bash
cd /home/ssdns/code/tgoskits

sudo mkdir -p /tmp/axvisor-rootfs
sudo mount -o loop tmp/axbuild/rootfs/rootfs-x86_64-alpine.img /tmp/axvisor-rootfs

sudo mkdir -p /tmp/axvisor-rootfs/guest/ovmf
sudo cp /usr/share/OVMF/OVMF_CODE_4M.fd /tmp/axvisor-rootfs/guest/ovmf/OVMF_CODE.fd
sudo cp /usr/share/OVMF/OVMF_VARS_4M.fd /tmp/axvisor-rootfs/guest/ovmf/OVMF_VARS.fd

sync
sudo umount /tmp/axvisor-rootfs
```

复制完成后，AxVisor 运行时才能打开：

```text
/guest/ovmf/OVMF_CODE.fd
/guest/ovmf/OVMF_VARS.fd
```

如果没有复制到 rootfs，而是直接在 VM 配置中写宿主机路径，例如 `/usr/share/OVMF/...`，AxVisor 会在自己的 rootfs 中查找该路径，并报 `NotFound`。

---

## 6. OVMF_CODE 和 OVMF_VARS 的含义

OVMF 通常拆成两个 pflash 镜像：

| 文件 | 含义 | 典型属性 |
| --- | --- | --- |
| `OVMF_CODE.fd` | UEFI 固件代码 | 只读 |
| `OVMF_VARS.fd` | UEFI 变量存储区 | 可写 |

UEFI 固件运行时需要保存变量，例如：

```text
BootOrder
Boot#### 启动项
Secure Boot 变量
UEFI boot manager 配置
```

因此真实 QEMU/OVMF 启动通常不是只给一个 OVMF 文件，而是类似：

```bash
-drive if=pflash,format=raw,readonly=on,file=OVMF_CODE.fd
-drive if=pflash,format=raw,file=OVMF_VARS.fd
```

当前 AxVisor 还没有完整模拟 pflash 设备行为，只是先把两个 fd 文件按 OVMF 期望的高端地址布局加载进 guest memory，用于验证 OVMF 能否从 reset vector 开始执行。

---

## 7. 启动命令

在 `/home/ssdns/code/tgoskits/os/axvisor` 目录下执行：

```bash
cargo xtask qemu \
  --config /home/ssdns/code/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml \
  --qemu-config /home/ssdns/code/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml \
  --vmconfigs /home/ssdns/code/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml
```

如果需要避免命令长时间卡住，可以加 `timeout`：

```bash
timeout 240 cargo xtask qemu \
  --config /home/ssdns/code/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml \
  --qemu-config /home/ssdns/code/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml \
  --vmconfigs /home/ssdns/code/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml
```

---
## 8. 预期关键日志

### 8.1 OVMF 镜像加载成功

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

### 8.2 reset-vector VMCS 设置生效

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

### 8.3 OVMF 进入后续阶段

如果后续出现类似状态：

```text
guest_rip: 0x82997a
cr0: 0x80000023
cr3: 0x800000
cr4: 0x2660
cs: 0x38
```

这不是额外手写的“保护模式提示日志”，而是 VM-Exit 时 dump 出来的 guest VMCS 状态。

其中 `cr0 = 0x80000023` 可以看出：

```text
CR0.PE = 1
CR0.PG = 1
```

这表示 guest 已经开启保护模式和分页。因此可以判断 OVMF 已经不再停留在 reset-vector 附近，而是运行到了更后续的阶段。

---

## 9. 常见错误和处理

### 9.1 在仓库根目录运行 xtask

错误现象：

```text
error: unrecognized subcommand 'qemu'
Usage: tg-xtask <COMMAND>
```

原因：

```text
当前目录是 tgoskits 仓库根目录，调用的是顶层 tg-xtask。
```

处理：

```bash
cd /home/ssdns/code/tgoskits/os/axvisor
```

然后重新运行 `cargo xtask qemu`。

### 9.2 KVM 权限不足

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

### 9.3 OVMF 文件找不到

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
把宿主机 /usr/share/OVMF 下的 OVMF_CODE_4M.fd 和 OVMF_VARS_4M.fd 复制进 AxVisor rootfs 的 /guest/ovmf/。
VM 配置中继续使用 /guest/ovmf/OVMF_CODE.fd 和 /guest/ovmf/OVMF_VARS.fd。
```

需要特别确认复制的是运行时实际使用的 rootfs：

```text
/home/ssdns/code/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

不是：

```text
/home/ssdns/code/tgoskits/os/axvisor/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

### 9.4 placeholder kernel 文件找不到

错误现象：

```text
Failed to open /guest/uefi/empty-kernel-placeholder.bin
```

原因：

```text
UEFI 模式不应该继续读取 Linux kernel header。
```

当前代码已经通过 `boot = "uefi"` 分支跳过该读取。如果仍出现该错误，说明当前运行的代码不是最新构建，或 VM 配置中的 `boot = "uefi"` 没有生效。

### 9.5 OVMF 在 reset-vector 附近 #UD

错误现象：

```text
VMX exception/NMI VM-Exit
vector_name=#UD invalid opcode
guest_rip 接近 0xfff0 / 0xfff4
```

可能原因：

```text
OVMF_CODE 没有放到正确 base，导致 0xffff_fff0 处不是 OVMF reset-vector 指令。
```

处理：

```text
确认 OVMF_CODE 文件大小，并确认 ovmf_code_base 使文件末尾 - 0x10 正好落到 0xffff_fff0。
当前 OVMF_CODE_4M.fd 大小为 0x37c000，因此 ovmf_code_base 使用 0xffc8_4000。
```

### 9.6 OVMF 运行后 triple fault

当前已观察到 OVMF 能运行到后续阶段后触发 triple fault。

下一步应重点观察以下日志：

```text
VMX exception/NMI VM-Exit
VMX exception raw fields
VMX guest state
VMX guest CS
```

或：

```text
VMX triple fault VM-Exit
VMX triple fault raw fields
VMX triple fault guest state
VMX triple fault guest CS
```

如果先捕获到异常，根据 vector 判断：

| 异常 | 可能方向 |
| --- | --- |
| `#PF` | GPA 映射、页表访问、MMIO 缺失。 |
| `#GP` | 段、控制寄存器、MSR、权限或 CPU 状态问题。 |
| `#UD` | 指令或 CPUID/MSR 暴露能力不匹配。 |
| `#DF` | 前一个异常处理失败，需反推 IDT/GDT/栈/异常注入路径。 |

---

## 10. 当前运行状态的判断标准

当前阶段的成功标准不是进入 UEFI shell，也不是启动 Linux EFI。

当前阶段的成功标准是：

```text
1. AxVisor 能创建 OVMF VM。
2. OVMF_CODE / OVMF_VARS 能从 rootfs 加载到预期 GPA。
3. BSP vCPU 能以 reset-vector 语义进入 OVMF。
4. OVMF 不再在 reset-vector 附近立刻 #UD。
5. 能通过 VM-Exit / exception / triple fault 日志定位 OVMF 下一阶段缺失内容。
```

如果日志已经显示 OVMF 运行到保护模式 / 分页模式后的地址，再触发 triple fault，则说明当前最小启动链路已经初步跑通，后续工作应转向定位 triple fault 前的第一个异常。 
