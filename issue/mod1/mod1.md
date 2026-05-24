# mod1：OVMF 早期启动推进总结

## 目标和边界

本阶段的目标不是一次性补齐完整 x86_64 UEFI 平台，而是让 AxVisor 能加载自编译 DEBUG OVMF，并沿着 OVMF 的早期启动路径逐步定位卡点。推进方式保持小步：先让日志可见，再根据日志判断 OVMF 处于哪个阶段，只对当前阶段暴露出的缺口做最小适配。

OVMF 的构建、写入 rootfs、启动命令和常见运行方式已经整理在 `axvisor-2026-notebooks/start.md`。本文只简要引用这些流程，不再重复展开构建手册。

## develop0：从早期 VMX 异常日志确认 OVMF 已被执行

`develop0.md` 主要是早期观察和定位记录，连续的纯日志推进：

- AxVisor 已经能够把 OVMF 镜像加载到 guest 物理地址，并从 reset vector 进入 guest。
- 通过增强 VMX exit、异常投递、guest RIP、IDT-vectoring、段寄存器等日志，确认问题不再是镜像不存在或构建失败。
- 当时看到的典型现场是 `guest_rip=0x82997a`，`IDT-vectoring` 显示 `#BR bound range exceeded`，随后又触发 `#GP general protection`，错误码指向 `GDT selector 0x10`。
- 这些日志说明 AxVisor 已经真正运行到了 OVMF 早期代码，但当时从 VMX 视角还无法判断 OVMF 内部具体执行到了 SEC、PEI 还是更后面的驱动。

因此 develop0 的核心产出是：确认启动链路已经越过“镜像加载/入口设置”问题，下一步必须让 OVMF 自己的 DEBUG 信息可见，才能继续缩小故障位置。

## develop1：围绕日志可见性定位到 PEI 的 QemuFwCfgPei

`develop1.md` 的主线确实主要是日志相关，但可以分成两类：一类是给 OVMF 内部路径加诊断日志，另一类是打通 OVMF DEBUG 输出到 AxVisor 终端的链路。第二类不是普通日志增强，而是为后续所有 OVMF 内部定位建立基础设施。

### 1. 使用 DEBUG OVMF 和内部日志确认进入 PEI

首先将自编译 DEBUG OVMF 放入 guest rootfs，重新运行 smoke test。此后不再只看到 AxVisor 的 VMX 异常视角，而是能看到 OVMF 自己打印的异常摘要，例如：

```text
!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!
RIP - 000000000084F404
Find image based on IP(0x84F404) ... PlatformPei.dll
```

结合反汇编和源码位置，`RIP` 落在 `OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` 的 `InternalQemuFwCfgDmaBytes()` 附近。这一步把阶段判断从 develop0 的“OVMF 早期异常”推进到明确的 **PEI**：已经进入 `PlatformPei.dll`，并且卡在 PEI 中的 QEMU fw_cfg 路径。

后续又在 `InternalQemuFwCfgDmaBytes()` 增加了 `AXOVMF` 诊断日志，用来打印：

- DMA 传输大小；
- guest buffer 地址；
- DMA control；
- `QEMU_FW_CFG_WORK_AREA`；
- 当前 item、offset、reading 状态。

这类 OVMF 内部日志的作用是缩小问题范围：不是 ACPI、PCI、virtio 等后续平台设备，也不是 DXE 阶段，而是 PEI 中 `PlatformPei` 读取 fw_cfg 时触发异常。

### 2. 打通 OVMF DEBUG 输出到 AxVisor 的 debugcon 链路

后来发现 OVMF 的 `DEBUG()` 输出默认写到 `PcdDebugIoPort`，在当前 OVMF 配置里是 I/O port `0x402`。AxVisor 起初没有把这个端口当作 debugcon 处理，因此 OVMF 内部新增的 `DEBUG_INFO` 不会自然显示到宿主终端。

为此做了两边的最小适配。

AxVisor 侧：

- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
  - 在 VMX I/O bitmap 中拦截 `0x402`。
- `tgoskits/components/axdevice/src/device.rs`
  - 新增最小 `OvmfDebugConDevice`。
  - `IoRead8(0x402)` 返回 Bochs debug port magic `0xE9`，让 OVMF 的 debug port probe 通过。
  - `IoWrite8(0x402)` 按字节收集，按行通过 AxVisor 日志输出为 `OVMF debugcon: ...`。

OVMF 侧：

- `edk2/OvmfPkg/Library/PlatformDebugLibIoPort/DebugLib.c`
  - 将 debugcon 输出路径中的 `IoWriteFifo8()` 改为逐字节 `IoWrite8()`。

这个 OVMF 侧改动的原因是 AxVisor 当前 VMX I/O exit 对 string/repeat I/O 支持不足。`IoWriteFifo8()` 会生成 `rep outsb`，运行时会停在：

```text
VMX unsupported IO-Exit: ... is_string: true, is_repeat: true, port: 0x402
```

因此这里没有尝试实现完整字符串 I/O，而是只为了 debug 输出链路，把 OVMF DEBUG 写端改成普通逐字节端口写。

打通后可以看到类似日志：

```text
OVMF debugcon: SecCoreStartupWithStack(...)
OVMF debugcon: Platform PEIM Loaded
OVMF debugcon: QemuFwCfgProbe: Supported 1, DMA 1
```

这说明 OVMF DEBUG 输出已经能从 guest 通过 I/O port `0x402` 触发 VM-exit，再由 AxVisor 模拟设备打印到宿主终端。

### 3. develop1 的最终定位

在 debugcon 可见后，新增的 `AXOVMF` 日志明确显示首次 fw_cfg DMA 请求：

```text
AXOVMF InternalQemuFwCfgDmaBytes: size=0x4 buffer=81DF28 control=0x2 workarea=815931 item=0x0 offset=0x0 reading=0
```

随后 OVMF 抛出 `#BR`，位置仍在 `PlatformPei.dll`。这说明当时的真实卡点是：OVMF 判断 fw_cfg 存在，并且认为 DMA 访问可用，然后在对 fw_cfg DMA control/address port 发起第一次读请求时失败。

因此 develop1 的结论是：已经进入 PEI，问题具体落在 `PlatformPei` 的 `QemuFwCfgPei` 路径；下一步不应继续盲目加日志，而应给 AxVisor 补一个足够 PEI 使用的最小 fw_cfg 行为。

## develop2：实现最小 fw_cfg，让 PlatformPei 获得内存信息

`develop2.md` 前半部分与 develop1 有重叠，主要继承了 DEBUG OVMF、debugcon、`QemuFwCfgPei` 定位等上下文。真正的新进展集中在 Step 9 和 Step 10：AxVisor 开始实现最小 fw_cfg，目标是满足 PEI 阶段 `PlatformPei` 对 QEMU 签名、接口能力、CPU 数量和内存布局的读取需求。

这里可以按两条线理解：一条是“按标准构造当前阶段需要的 fw_cfg 内容”，另一条是“让 OVMF 当前代码路径能通过 AxVisor 读到这些内容”。

### 1. 为什么只做最小 fw_cfg

当前卡点位于 PEI 的 `PlatformPei`，不是 DXE，也不是完整设备枚举阶段。`PlatformPei` 此时主要需要知道：

- fw_cfg 是否存在；
- fw_cfg 是否支持 DMA；
- guest 有多少 CPU；
- guest 低端内存范围是多少；
- 是否能通过 `etc/e820` 读取内存布局。

因此本阶段不实现完整 QEMU fw_cfg 设备模型，也不提前补 ACPI、PCI、virtio 所需的全部 fw_cfg 文件。只构造 OVMF PEI 当前实际读取的最小集合。

### 2. 按 fw_cfg 格式构造最小内容

AxVisor 侧在 `tgoskits/components/axvm/src/vm.rs` 中新增最小 fw_cfg 状态和 item 表，核心端口和 item 包括：

- selector port：`0x510`；
- data port：`0x511`；
- DMA address port：`0x514` 和 `0x518`；
- signature item：`0x0000`，内容为 `QEMU`；
- interface version item：`0x0001`，包含 `FW_CFG_F_DMA`；
- SMP CPU count item：`0x0005`；
- file directory item：`0x0019`；
- `etc/e820` 文件 item：当前使用 `0x8000`。

`etc/e820` 根据 AxVisor 的 guest memory region 生成，提供 PEI 阶段需要的 RAM 范围信息。file directory 中登记 `etc/e820` 的名称、selector 和大小，使 OVMF 可以通过标准 `QemuFwCfgFindFile("etc/e820", ...)` 找到它。

这里的大小端处理需要区分两类数据：

- 普通 fw_cfg item 被 OVMF 直接读入本地整数，AxVisor 按 OVMF 当前读取方式提供字节即可，例如 interface version 保持让 OVMF 能直接判断 `FW_CFG_F_DMA`。
- fw_cfg file directory 和 DMA descriptor 中有 QEMU/fw_cfg 约定的大端字段，AxVisor 需要按对应格式构造和解析。

### 3. OVMF 如何读取 fw_cfg

OVMF 的读取路径大致是：

1. `QemuFwCfgProbe()` 先选择 signature item，读取 `QEMU`。
2. 再选择 interface version item，读取 feature bits，判断 `FW_CFG_F_DMA`。
3. 若 fw_cfg 可用，`PlatformPei` 后续通过 fw_cfg cache 初始化读取固定 item 和 file directory。
4. `QemuFwCfgFindFile("etc/e820", ...)` 从 file directory 中找到 `etc/e820` 对应的 selector 和大小。
5. OVMF select `etc/e820` item，并读取 e820 entries。
6. `PlatformPei` 根据 e820 和 CPU count 继续进行内存发现和 `PublishPeiMemory()`。

在 PEI 的 DMA 路径中，OVMF 使用 `FW_CFG_DMA_ACCESS` descriptor：

```c
typedef struct {
  UINT32 Control;
  UINT32 Length;
  UINT64 Address;
} FW_CFG_DMA_ACCESS;
```

OVMF 会对 descriptor 字段做 `SwapBytes32()` / `SwapBytes64()`，再通过 `IoWrite32(FW_CFG_IO_DMA_ADDRESS, ...)` 和 `IoWrite32(FW_CFG_IO_DMA_ADDRESS + 4, ...)` 写出 descriptor 地址。AxVisor 必须按这个语义还原 descriptor GPA，读取 guest memory 中的 descriptor，再执行 read / write / skip / select，最后把 status 写回 descriptor 的 control 字段。

### 4. AxVisor 侧实现了什么

主要改动集中在 `tgoskits/components/axvm/src/vm.rs`：

- 新增 `FwCfgState`，保存当前 selector、offset、DMA address、DMA address 写入中间状态和 item 表。
- 新增最小 item 构造逻辑：`QEMU` signature、DMA feature、CPU count、`etc/e820`、file directory。
- 新增普通端口读写处理：
  - 写 `0x510` 选择 item；
  - 读 `0x511` 从当前 item 的 offset 返回数据。
- 新增 DMA address 端口处理：
  - 接收 `0x514` / `0x518` 两次 32 位写；
  - 按 OVMF `SwapBytes32()` 的实际写法还原 64 位 descriptor 地址。
- 新增 DMA 执行逻辑：
  - 从 guest memory 读取 `FW_CFG_DMA_ACCESS`；
  - 按 control bits 执行 select / read / write / skip；
  - 对 read，把 item 数据写入 guest buffer；
  - 最后把 status 写回 descriptor。

同时在 `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs` 中继续扩大 I/O bitmap 拦截范围：

- 保留 `0x402` debugcon；
- 新增 `0x510..0x511`；
- 新增 `0x514..0x51b`。

这样 OVMF 的 fw_cfg 端口访问会触发 VM-exit，并进入 AxVisor 的模拟处理路径。

### 5. Step 10 的小意外：fw_cfg probe 的 string I/O

第一次加入最小 fw_cfg 后，旧的 DMA `#BR` 没有直接复现，先暴露出另一个更靠前的问题：OVMF 在 `QemuFwCfgProbe()` 中对 data port `0x511` 使用了 `IoReadFifo8()`，生成 `rep insb`，而 AxVisor 当前仍不支持 string/repeat I/O exit。

运行时停在：

```text
VMX unsupported IO-Exit: ... is_in: true, is_string: true, is_repeat: true, port: 0x511
```

处理方式延续 debugcon 的最小策略：不在 AxVisor 中实现完整 string I/O，而是在 `edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` 的 `QemuFwCfgProbe()` 中，将 signature 和 interface version 的 `IoReadFifo8(FW_CFG_IO_DATA, ...)` 改为逐字节 `IoRead8(FW_CFG_IO_DATA)`。

这属于 OVMF 侧临时适配，目的只是绕过 AxVisor 当前 VMX string I/O 缺口，让 PEI fw_cfg 路径继续向前暴露真实问题。

### 6. develop2 的验证结果

修改后重新构建 DEBUG OVMF、写入 rootfs 并运行 smoke test，已经看到 fw_cfg cache 成功：

```text
OVMF debugcon: QemuFwCfgProbe: Supported 1, DMA 1
OVMF debugcon: QemuFwCfgInitCache Pass!!!
OVMF debugcon: Cache FwCfg Name: QemuFwCfgSignature Item:0x0 Size: 0x4
OVMF debugcon: Cache FwCfg Name: QemuFwCfgInterfaceVersion Item:0x1 Size: 0x4
OVMF debugcon: Cache FwCfg Name: QemuFwCfgFileDri Item:0x19 Size: 0x44
OVMF debugcon: Cache FwCfg Name: etc/e820 Item:0x8000 Size: 0x28
OVMF debugcon: PlatformMaxCpuCountInitialization: BootCpuCount=1 MaxCpuCount=1
OVMF debugcon: PlatformGetLowMemoryCB: LowMemory=0x4000000
OVMF debugcon: PublishPeiMemory: ...
```

这说明 Step 9/10 的目标已经达成：`PlatformPei` 能通过 AxVisor 的最小 fw_cfg 获取 `QEMU` signature、DMA capability、CPU count 和 e820 内存信息，并且已经跨过 develop1 中的 `QemuFwCfgPei` DMA 卡点。

新的停点变成 MTRR 相关断言：

```text
OVMF debugcon: ASSERT MemDetect.c(1344): (MtrrSettings.MtrrDefType & 0x00000400) == 0
!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!
RIP - 0000000000849106
Find image based on IP(0x849106) ... PlatformPei.dll
```

因此当前问题已经从 fw_cfg 推进到 PEI 内存发布 / RAM 初始化路径中的 MTRR 初始状态。

## 当前阶段判断

综合三份 develop，目前进度是：

1. OVMF 镜像加载和 reset vector 进入已经成立。
2. OVMF DEBUG 输出已经能通过 AxVisor 的 `0x402` debugcon 设备打印出来。
3. 已确认 OVMF 进入 PEI，并执行到 `PlatformPei`。
4. 已为 PEI 阶段实现最小 fw_cfg：signature、interface version、CPU count、file directory、`etc/e820`、DMA descriptor 处理。
5. 已跨过 `QemuFwCfgInitCache`，说明 `PlatformPei` 能拿到当前阶段需要的 fw_cfg 内存信息。
6. 当前新的阻塞点是 `MemDetect.c` 对 `IA32_MTRR_DEF_TYPE` 状态的断言，不再是 fw_cfg item 缺失。

## 后续边界

下一步不应继续扩展 fw_cfg item，也不应直接进入 ACPI / PCI / virtio 等大平台模型。更小的后续方向是检查 OVMF SEC/PEI 的 MTRR 假设和 AxVisor 当前 MSR/MTRR 暴露方式：

- OVMF `SecMtrrSetup()` 会设置 MTRR default type 为 write-back 并打开 enable bit。
- 当前断言失败点说明 `IA32_MTRR_DEF_TYPE` 的 fixed MTRR enable bit 可能在进入 PEI 前已经为 1。
- AxVisor 可能仍在让 guest 直接看到 host/current MTRR 状态，或者没有为 OVMF 提供符合其早期假设的虚拟 MTRR 初始值。

因此后续最小动作应是定位并处理 `IA32_MTRR_DEF_TYPE` / MTRR MSR，而不是继续补完整 fw_cfg 设备模型。如果发现需要完整 MTRR 虚拟化，也应作为新的边界单独记录。