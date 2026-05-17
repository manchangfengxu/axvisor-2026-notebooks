# AxVisor x86_64 UEFI 客户机支持：先把 OVMF 接入并跑起来

## 核心判断

当前最可行、也最应该优先推进的目标，不是一次性补齐完整 PC 平台，而是先让 **OVMF 能作为 AxVisor 的 x86_64 客户机固件被正确加载、正确进入，并产生可观测输出**。

这个方向是可行的，而且非常适合作为第一阶段目标。原因是：

1. **OVMF 本身就是标准 UEFI 固件实现**。只要 AxVisor 能提供它最早期需要的复位入口、固件映射和基本执行环境，OVMF 就能开始运行。
2. **只要 OVMF 跑起来，后续问题会从“猜缺什么”变成“看它卡在哪里”**。OVMF 的串口日志、调试输出和 VM-Exit 记录可以直接告诉我们它正在访问哪些 I/O 端口、MMIO 区域、PCI 配置空间或 ACPI/fw_cfg 数据。
3. **完整 UEFI guest 启动链路太长，不适合一开始整体攻关**。ACPI、PCI、virtio-pci、vIOAPIC、MSI/MSI-X 都重要，但它们应该在 OVMF 能启动并报出具体缺失之后逐步补齐。

因此，第一阶段的准心应该从“让 Linux EFI guest 完整启动”前移到：

> 让 AxVisor 能以 UEFI boot mode 加载 OVMF，并让 vCPU 从 OVMF 的复位入口开始执行，至少能看到 OVMF 早期日志、断点、异常或明确的 VM-Exit 行为。

---

## 当前 AxVisor 的问题定位

当前 x86_64 客户机启动方式大致还是传统 BIOS/trampoline 模型：

- 配置中通过 `entry_point`、`bios_load_addr` 等字段把启动代码固定加载到低地址。
- 固件依赖 `rvm-bios.bin` 或 `axvm-bios.bin` 这类自定义 BIOS blob。
- 镜像加载逻辑主要认识“把某个二进制文件按 GPA 直接放进去”这一类模型。
- vCPU 初始状态更接近从低地址 trampoline 或传统实模式代码开始执行。
- x86 平台设备模型还很薄，尚未形成标准 PC/UEFI guest 所需的 ACPI、PCI、fw_cfg、pflash、virtio-pci 和中断链路。

不应该假设 AxVisor 已经是一个 QEMU 风格 PC 平台，然后直接讨论 OVMF 在 PEI/DXE 阶段如何枚举硬件。更实际的切入点是：

1. AxVisor 配置层能描述 UEFI 启动。
2. 镜像加载器能把 OVMF_CODE 和 OVMF_VARS 按固件语义放到客户机物理地址空间。
3. x86 vCPU 能从 OVMF 期望的复位入口开始执行。
4. AxVisor 能收集 OVMF 的早期输出或失败现场。

---

## 分阶段目标

### 阶段 0：建立 UEFI boot mode 配置模型

这是第一个要改的环节。

目前 AxVisor 的配置更像是：

```toml
entry_point = 0x8000
bios_path = "..."
bios_load_addr = 0x8000
```

这适合传统 BIOS/trampoline，但不适合 OVMF。需要新增一种明确的启动模式，例如：

```toml
boot = "uefi"

[uefi]
ovmf_code_path = "OVMF_CODE.fd"
ovmf_vars_path = "OVMF_VARS.fd"
ovmf_code_base = 0xfffc0000
ovmf_vars_base = 0xfff80000
```

字段名字可以后续根据现有 `axvmconfig` / `axvm::config` 风格调整，但语义要明确：

- `boot = "trampoline"`：保留当前传统启动方式。
- `boot = "uefi"`：启用 OVMF 启动路径。
- `OVMF_CODE`：固件代码区域，通常只读。
- `OVMF_VARS`：UEFI 变量存储区域，需要可写，后续应按 pflash/NVRAM 语义处理。
- 固件 GPA：不能继续用低地址 `0x8000`，而要放在 x86 复位向量可达的高地址区域。

第一阶段可以先不做复杂的 pflash 设备模型，但配置层必须先把 UEFI 和传统 BIOS 区分开，否则后续代码会不断在旧模型上打补丁。

---

### 阶段 1：加载和映射 OVMF_CODE / OVMF_VARS

这是 OVMF 能否“被唤醒”的关键。

x86 CPU 复位后从接近 4GiB 顶部的位置开始取指，典型复位向量是：

```text
0xfffffff0
```

OVMF_CODE.fd 通常需要被映射到 4GiB 顶端附近的固件 ROM 区域，例如常见布局是：

```text
OVMF_CODE: 0xfffc0000 .. 0xffffffff
```

实际地址和大小要以选用的 OVMF.fd 文件为准，不能硬编码成唯一值。配置层应该允许指定 base 和 size，加载器根据文件大小和地址范围做基本检查。

这一阶段 AxVisor 需要完成：

1. **把 OVMF_CODE 映射到客户机高地址固件区域**
   - 只读更符合固件代码语义。
   - 复位向量 `0xfffffff0` 必须落在该映射范围内。
   - 客户机从复位向量取到的第一条指令应该来自 OVMF_CODE。

2. **把 OVMF_VARS 映射为变量存储区域**
   - 这一区域后续应具有可写、持久化、pflash block 写入等语义。
   - 第一阶段可以先做最小可写映射，目标是让 OVMF 继续往前跑。
   - 后续再补严格的 pflash 行为。

3. **避免继续沿用低地址 BIOS 加载假设**
   - UEFI 路径不应该依赖 `bios_load_addr = 0x8000`。
   - `entry_point` 也不应该手动指到低地址 trampoline。
   - UEFI 模式下入口应由 x86 reset state 和固件映射共同决定。

这一阶段的验收标准不是进入 UEFI Shell，而是更基础：

- vCPU 能从 OVMF_CODE 覆盖的复位向量开始执行。
- 如果执行失败，能看到失败是非法指令、页表/EPT 映射错误、VM-Exit 未处理，还是访问了未实现 I/O 端口。

---

### 阶段 2：调整 x86 vCPU 初始状态

仅仅把 OVMF_CODE 放进内存还不够。AxVisor 还需要让 vCPU 以 OVMF 期望的 x86 reset 状态启动。

需要重点检查：

- `CS:IP` / `RIP` 是否能对应到 `0xfffffff0` 复位向量。
- 段寄存器、控制寄存器、EFER、CR0/CR4 初始值是否符合实模式复位语义。
- 当前是否依赖 unrestricted guest 直接在低地址运行 trampoline。
- 如果 AxVisor 当前强行设置 `entry_point`，UEFI 模式下应绕开这条逻辑。

这一阶段不要急着实现 ACPI 或 PCI。先确保 OVMF 的 SEC 阶段能开始执行。OVMF 的最早期代码会完成从 16 位实模式到 32 位保护模式，再进入后续 PEI/DXE 流程。如果这里都没有跑起来，后面的 fw_cfg、ACPI、virtio 都没有调试基础。

验收标准：

- 能确认 vCPU 第一段执行流进入 OVMF。
- 能捕获到 OVMF 早期执行过程中的 VM-Exit。
- 如果卡住，能知道卡在模式切换、MSR、CPUID、I/O 访问还是内存访问。

---

### 阶段 3：建立早期可观测性

这一步和阶段 1、2 同样重要。UEFI 启动调试最怕“黑屏无输出”。

第一阶段必须尽早建立至少一种可观测通道：

1. **串口输出**
   - 优先支持 legacy COM1，例如 I/O port `0x3f8`。
   - 让 OVMF 的 debug 或 console 输出能从 AxVisor 日志中看到。

2. **VM-Exit 日志**
   - 对未处理的 I/O port、MMIO、CPUID、MSR、EPT violation 打出清晰日志。
   - 日志中应包含 vCPU id、RIP、访问地址、访问宽度、读写方向、退出原因。

3. **OVMF debug 构建**
   - 如果使用 DEBUG 版 OVMF，串口日志会非常有价值。
   - 如果使用 RELEASE 版 OVMF，也至少要能看到串口或未处理 exit。

这一阶段的目标是让后续开发方式变成：

> 启动 OVMF → 看日志或 VM-Exit → 确认它缺哪个平台接口 → 补最小实现 → 再启动。

这比一开始凭经验完整实现 fw_cfg、ACPI、PCI 和 virtio 更稳。

---

### 阶段 4：最小 fw_cfg 支持

当 OVMF 能进入 PEI/DXE 后，很可能会访问 QEMU 风格的 `fw_cfg`。这时再开始补 fw_cfg。

fw_cfg 的作用是让固件从 hypervisor 获取平台信息，例如：

- 内存布局。
- CPU 数量和拓扑。
- ACPI 表位置或 ACPI table blob。
- 启动相关配置。
- kernel/initrd/cmdline 等 QEMU direct boot 信息。

第一版 fw_cfg 不需要追求完整，只需要按 OVMF 实际访问路径补最小集合。开发方法应该是：

1. 运行 OVMF。
2. 记录它访问了哪些 fw_cfg selector 或文件名。
3. 只实现当前启动路径需要的 key。
4. 再运行，观察下一个缺口。

这一阶段的目标是让 OVMF 不再因为完全没有 fw_cfg 而早早失败。

---

### 阶段 5：最小 ACPI 和 PCI 平台骨架

当 OVMF 已经能稳定进入 DXE 阶段后，再补标准 PC 平台发现路径。

需要粗略完成：

- RSDP。
- XSDT 或 RSDT。
- FADT。
- MADT。
- MCFG。
- 最小 PCI host bridge。
- PCI 配置空间访问机制，可以先选 ECAM 或 PIO config port 中更容易接入 AxVisor 的一种。

这部分的目标不是一开始就完美模拟 QEMU，而是让 OVMF 能识别“这是一个有 PCI 根桥的 PC 平台”。

---

### 阶段 6：virtio-pci block 与启动设备

有了 PCI root bridge 后，再把 virtio-block 通过 virtio-pci 暴露给 OVMF。

需要补：

- virtio-pci capability 或 legacy virtio-pci 设备模型。
- block backend。
- INTx 或 MSI/MSI-X 中断路径。
- OVMF 内置 VirtioBlkDxe 能识别并绑定设备。
- BDS 阶段能在块设备上找到 EFI System Partition。

这一阶段的验收目标可以从低到高分三步：

1. OVMF 能枚举到 virtio-pci block 设备。
2. UEFI Shell 中能看到块设备或文件系统。
3. 能加载 `\EFI\BOOT\BOOTX64.EFI` 或 Linux EFI stub。

---

### 阶段 7：完整启动 Linux EFI guest

最后才是完整操作系统启动。

这一阶段需要继续补齐：

- 更完整的 ACPI 表。
- LAPIC / IOAPIC / i8259 关系。
- INTx、EOI、IPI、timer。
- MSI/MSI-X。
- virtio-net。
- virtio-console。
- HPET 或其他时钟设备。
- Linux 启动后的各种 VM-Exit 处理。

此时目标才是：

> 从 OVMF 进入 BDS，找到 virtio-block 上的 EFI 启动项，加载 Linux EFI stub，调用 `ExitBootServices()`，最终进入 Linux kernel。

---

## OVMF 启动流程与 AxVisor 责任边界

下面按 OVMF 时间线重新整理，但把重点放在 AxVisor 每个阶段真正需要提供什么。

### 0. Reset Vector：从 `0xfffffff0` 开始

OVMF 被唤醒的第一步是 x86 复位向量。

AxVisor 需要保证：

- `0xfffffff0` 能取到 OVMF_CODE 中的跳转指令。
- vCPU 初始状态符合 x86 reset 语义。
- UEFI 模式不再强行跳到低地址 BIOS/trampoline。

如果这一步失败，后续所有 UEFI 平台设备都还用不上。

### 1. SEC：早期汇编和模式切换

OVMF 会从 16 位实模式开始执行，逐步建立临时执行环境，并进入后续 PEI。

AxVisor 需要保证：

- 基本指令执行、控制寄存器访问、MSR/CPUID 行为不会立即阻断。
- VM-Exit 日志足够清晰，能看出 OVMF 访问了什么。

### 2. PEI：开始需要平台信息

到 PEI 后，OVMF 会逐渐需要内存布局、CPU 信息和固件配置数据。

AxVisor 后续需要提供：

- 最小 fw_cfg。
- 合理的 guest RAM 布局。
- 必要的 CPUID/MSR 支持。

但这不是第一行代码要改的地方。只有 OVMF 已经能走到这里，fw_cfg 才成为明确 blocker。

### 3. DXE：枚举平台设备

DXE 阶段会加载大量 UEFI 驱动，开始处理 ACPI、PCI、virtio 等平台设备。

AxVisor 后续需要提供：

- ACPI 表。
- PCI host bridge。
- PCI 配置空间。
- virtio-pci 设备。
- 中断控制器和中断路由。

这属于第二大阶段，不应该和最早期 OVMF 接入混在一起。

### 4. BDS：选择启动设备

BDS 阶段会根据 UEFI 变量和可用设备寻找启动项。

AxVisor 后续需要提供：

- 可写的 OVMF_VARS / pflash 语义。
- 可枚举的块设备。
- 能读取 EFI System Partition 的 virtio-block 路径。

### 5. ExitBootServices：交权给 OS

当 Linux EFI stub 或 bootloader 调用 `ExitBootServices()` 后，OVMF 将控制权交给 guest OS。

此后 AxVisor 的重点转为支撑 Linux kernel 的正常运行，包括中断、时钟、virtio、页表相关 VM-Exit 等。

---

## 第一阶段建议的具体工作清单

第一阶段建议只做以下事情，不要扩散：

1. 在配置模型中加入 `boot = "uefi"`。
2. 为 UEFI 模式增加 OVMF_CODE / OVMF_VARS 配置项。
3. 修改镜像加载器，把 OVMF_CODE 放到高地址固件区域。
4. 修改镜像加载器，把 OVMF_VARS 放到独立可写区域。
5. 修改 x86 vCPU 初始化逻辑，让 UEFI 模式从 reset vector 进入。
6. 增加或确认 COM1 串口输出路径。
7. 增加未处理 VM-Exit 的详细日志。
8. 新增一个最小 x86_64 UEFI guest 配置文件。
9. 建立一个最小冒烟测试：启动 OVMF，并检查是否出现 OVMF 串口输出或明确的 OVMF 相关 VM-Exit。

第一阶段暂时不要求：

- 启动 Linux。
- 进入 UEFI Shell。
- 完整 fw_cfg。
- 完整 ACPI。
- PCI root bridge。
- virtio-block。
- MSI/MSI-X。

如果能进入 UEFI Shell，当然很好；但它不应该是第一阶段的最低验收线。最低验收线应该是“确认 OVMF 已经真的开始执行，并且失败点可观测”。

---

## 推荐的开发闭环

后续开发应采用小步验证：

```text
接入一个最小能力
        ↓
启动 OVMF
        ↓
查看串口日志 / VM-Exit / fault 信息
        ↓
确认下一个缺失的平台接口
        ↓
补最小实现
        ↓
再次启动
```

这样可以避免一开始实现大量可能不符合 OVMF 实际访问路径的设备模型。

最终完整路线可以概括为：

```text
OVMF 能取指
  → OVMF 早期代码能执行
  → 有串口或 VM-Exit 可观测性
  → 补 fw_cfg
  → 补 ACPI
  → 补 PCI root bridge
  → 补 virtio-pci block
  → 进入 UEFI Shell / 找到 EFI 启动项
  → 启动 Linux EFI stub
  → Linux kernel 正常运行
```

---

## 总结

这件事可行，但第一阶段目标必须收敛。

对 AxVisor 来说，现在最重要的不是马上实现完整 UEFI PC 平台，而是先把 OVMF 当作真正的 x86 固件接进来：让它位于正确的高地址固件区域，让 vCPU 从正确的复位向量开始执行，并让我们能看到它运行到哪里、因为什么停下。

一旦 OVMF 能启动并产生可观测输出，后续的 fw_cfg、ACPI、PCI、virtio 和中断链路就可以按照它暴露出来的实际缺口逐步补齐。这样推进风险更低，也更符合 AxVisor 当前的源码状态。
