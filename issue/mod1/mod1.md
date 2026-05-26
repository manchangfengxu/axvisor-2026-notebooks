# mod1：OVMF 早期启动推进总结

## 目标和边界

这一阶段没有打算一次补完 x86_64 UEFI 平台。目标更小：先确认 AxVisor 能把自编译 DEBUG OVMF 拉起来，再沿着 OVMF 的早期启动路径一点点往前推。做法也尽量保守：日志先可见，靠日志判断 OVMF 走到哪一段，然后只补当前卡点暴露出来的最小缺口。

OVMF 的构建、写入 rootfs、启动命令和常见运行方式已经放在 `axvisor-2026-notebooks/start.md`。这里不再重复构建手册，只在需要时引用运行结果。

## develop0：先确认 OVMF 真的跑起来了

`develop0.md` 基本是早期日志和现场定位。连续的日志细节可以压成一条主线：

- AxVisor 已经能把 OVMF 镜像加载到 guest 物理地址，并从 reset vector 进入 guest。
- 后面补了 VMX exit、异常投递、guest RIP、IDT-vectoring、段寄存器等日志，确认问题不是镜像不存在，也不是 OVMF 没进 guest。
- 当时比较典型的现场是 `guest_rip=0x82997a`，`IDT-vectoring` 显示 `#BR bound range exceeded`，随后又触发 `#GP general protection`，错误码指向 `GDT selector 0x10`。
- 这些信息只能说明 AxVisor 已经执行到了 OVMF 早期代码。至于 OVMF 内部到底在 SEC、PEI，还是更后面的模块里，当时光靠 VMX 视角还看不出来。

所以 develop0 的结论很朴素：启动链路已经越过“镜像加载 / 入口设置”这一关。下一步必须让 OVMF 自己的 DEBUG 日志出来，否则只能继续猜。

## develop1：把 OVMF 日志打出来，并定位到 PEI 的 QemuFwCfgPei

`develop1.md` 主要围绕日志展开，但里面其实有两件事。第一件是在 OVMF 里加诊断日志，看清楚它走到了哪里；第二件更关键，是把 OVMF 的 `DEBUG()` 输出接到 AxVisor 终端上。后者不是普通“多打印几行”，而是后面继续定位 OVMF 内部问题的基础设施。

### 1. 用 DEBUG OVMF 和内部日志确认进入 PEI

先把自编译 DEBUG OVMF 放进 guest rootfs，再跑 smoke test。这之后不再只有 AxVisor 侧的 VMX 异常日志，OVMF 自己也开始打印异常摘要，例如：

```text
!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!
RIP - 000000000084F404
Find image based on IP(0x84F404) ... PlatformPei.dll
```

结合反汇编和源码，`RIP` 落在 `OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` 的 `InternalQemuFwCfgDmaBytes()` 附近。这一步把判断从“OVMF 早期异常”收窄到了 **PEI**：已经进入 `PlatformPei.dll`，并卡在 PEI 里的 QEMU fw_cfg 路径。

随后又在 `InternalQemuFwCfgDmaBytes()` 加了 `AXOVMF` 日志，打印这些信息：

- DMA 传输大小；
- guest buffer 地址；
- DMA control；
- `QEMU_FW_CFG_WORK_AREA`；
- 当前 item、offset、reading 状态。

这些日志的用处很直接：排除 ACPI、PCI、virtio 这类后面才会碰到的平台设备，也排除 DXE。问题就在 PEI 的 `PlatformPei` 读取 fw_cfg 这条路径上。

### 2. 接通 OVMF DEBUG 到 AxVisor 的 debugcon 链路

后来确认，当前 OVMF 配置里的 `DEBUG()` 输出默认写到 `PcdDebugIoPort`，也就是 I/O port `0x402`。AxVisor 一开始没有把这个端口当 debugcon 处理，所以 OVMF 里新加的 `DEBUG_INFO` 并不会自动出现在宿主终端。

这里做了两边的最小改动。

AxVisor 侧：

- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`
  - 在 VMX I/O bitmap 中拦截 `0x402`。
- `tgoskits/components/axdevice/src/device.rs`
  - 新增一个很小的 `OvmfDebugConDevice`。
  - `IoRead8(0x402)` 返回 Bochs debug port magic `0xE9`，让 OVMF 的 debug port probe 通过。
  - `IoWrite8(0x402)` 按字节收集，遇到换行后用 AxVisor 日志打印成 `OVMF debugcon: ...`。

OVMF 侧：

- `edk2/OvmfPkg/Library/PlatformDebugLibIoPort/DebugLib.c`
  - 把 debugcon 输出路径里的 `IoWriteFifo8()` 改成逐字节 `IoWrite8()`。

OVMF 侧这处改动是被 AxVisor 当前能力逼出来的。`IoWriteFifo8()` 会生成 `rep outsb`，而 AxVisor 当时还不支持 string/repeat I/O exit，运行时会停在：

```text
VMX unsupported IO-Exit: ... is_string: true, is_repeat: true, port: 0x402
```

所以这里没有去补完整字符串 I/O，只是为了先把 debug 输出链路打通，把 OVMF 的 DEBUG 写端改成普通逐字节端口写。

链路打通后，可以看到类似日志：

```text
OVMF debugcon: SecCoreStartupWithStack(...)
OVMF debugcon: Platform PEIM Loaded
OVMF debugcon: QemuFwCfgProbe: Supported 1, DMA 1
```

也就是说，OVMF DEBUG 字节从 guest 写到 I/O port `0x402`，触发 VM-exit，再由 AxVisor 的模拟 debugcon 设备按行打印到宿主终端。

### 3. develop1 最后的定位

debugcon 可见后，`AXOVMF` 日志给出了第一次 fw_cfg DMA 请求：

```text
AXOVMF InternalQemuFwCfgDmaBytes: size=0x4 buffer=81DF28 control=0x2 workarea=815931 item=0x0 offset=0x0 reading=0
```

随后 OVMF 仍然抛 `#BR`，位置还在 `PlatformPei.dll`。这时卡点已经比较清楚了：OVMF 判断 fw_cfg 存在，也认为 DMA 可用，然后在第一次通过 fw_cfg DMA control/address port 读数据时失败。

因此 develop1 的结论是：系统已经进入 PEI，问题落在 `PlatformPei` 的 `QemuFwCfgPei` 路径。继续盲目加日志意义不大，下一步应该给 AxVisor 补一个 PEI 阶段够用的最小 fw_cfg 行为。

## develop2：补最小 fw_cfg，让 PlatformPei 拿到内存信息

`develop2.md` 前半部分和 develop1 有不少重叠，主要是 DEBUG OVMF、debugcon、`QemuFwCfgPei` 定位这些上下文。真正的新进展在 Step 9 和 Step 10：AxVisor 开始实现最小 fw_cfg，目标是满足 PEI 阶段 `PlatformPei` 读取 QEMU 签名、接口能力、CPU 数量和内存布局的需求。

这里可以分两条线看：

1. AxVisor 按 fw_cfg 格式构造当前阶段需要的内容；
2. OVMF 当前这条代码路径要能通过 AxVisor 读到这些内容。

### 1. 为什么只做最小 fw_cfg

当前卡点在 PEI 的 `PlatformPei`，还没到 DXE，也没走到完整设备枚举。这个阶段 `PlatformPei` 主要需要知道：

- fw_cfg 是否存在；
- fw_cfg 是否支持 DMA；
- guest 有多少 CPU；
- guest 低端内存范围是多少；
- 能不能通过 `etc/e820` 拿到内存布局。

所以这一步不做完整 QEMU fw_cfg 设备模型，也不提前补 ACPI、PCI、virtio 后面可能需要的全部 fw_cfg 文件。只构造 OVMF PEI 当前实际读取的那一小组 item。

### 2. 按 fw_cfg 格式构造最小内容

AxVisor 侧在 `tgoskits/components/axvm/src/vm.rs` 里新增最小 fw_cfg 状态和 item 表。当前用到的端口和 item 是：

- selector port：`0x510`；
- data port：`0x511`；
- DMA address port：`0x514` 和 `0x518`；
- signature item：`0x0000`，内容为 `QEMU`；
- interface version item：`0x0001`，包含 `FW_CFG_F_DMA`；
- SMP CPU count item：`0x0005`；
- file directory item：`0x0019`；
- `etc/e820` 文件 item：当前使用 `0x8000`。

`etc/e820` 根据 AxVisor 的 guest memory region 生成，给 PEI 阶段提供 RAM 范围。file directory 里登记 `etc/e820` 的名称、selector 和大小，这样 OVMF 才能通过标准的 `QemuFwCfgFindFile("etc/e820", ...)` 找到它。

大小端这里容易踩坑，需要分开看：

- 普通 fw_cfg item 会被 OVMF 直接读进本地整数。AxVisor 只要按 OVMF 当前读取方式提供字节即可，比如 interface version 要让 OVMF 直接判断出 `FW_CFG_F_DMA`。
- fw_cfg file directory 和 DMA descriptor 使用 QEMU/fw_cfg 约定的大端字段，AxVisor 需要按这个格式构造和解析。

### 3. OVMF 是怎么读 fw_cfg 的

当前路径大致是这样：

1. `QemuFwCfgProbe()` 先选择 signature item，读出 `QEMU`。
2. 再选择 interface version item，读取 feature bits，判断 `FW_CFG_F_DMA`。
3. 如果 fw_cfg 可用，`PlatformPei` 后面会通过 fw_cfg cache 初始化读取固定 item 和 file directory。
4. `QemuFwCfgFindFile("etc/e820", ...)` 从 file directory 里找到 `etc/e820` 的 selector 和大小。
5. OVMF 选择 `etc/e820` item，读取 e820 entries。
6. `PlatformPei` 根据 e820 和 CPU count 继续做内存发现，并调用 `PublishPeiMemory()`。

PEI 的 DMA 路径使用这个 descriptor：

```c
typedef struct {
  UINT32 Control;
  UINT32 Length;
  UINT64 Address;
} FW_CFG_DMA_ACCESS;
```

OVMF 会先对 descriptor 字段做 `SwapBytes32()` / `SwapBytes64()`，再通过 `IoWrite32(FW_CFG_IO_DMA_ADDRESS, ...)` 和 `IoWrite32(FW_CFG_IO_DMA_ADDRESS + 4, ...)` 写出 descriptor 地址。AxVisor 这边必须按同样的语义还原 descriptor GPA，读 guest memory 里的 descriptor，执行 read / write / skip / select，最后把 status 写回 descriptor 的 control 字段。

### 4. AxVisor 侧具体补了什么

主要改动在 `tgoskits/components/axvm/src/vm.rs`：

- 新增 `FwCfgState`，保存当前 selector、offset、DMA address、DMA address 写入中间状态和 item 表。
- 新增最小 item 构造逻辑：`QEMU` signature、DMA feature、CPU count、`etc/e820`、file directory。
- 新增普通端口读写：
  - 写 `0x510` 选择 item；
  - 读 `0x511`，从当前 item 的 offset 返回数据。
- 新增 DMA address 端口处理：
  - 接收 `0x514` / `0x518` 两次 32 位写；
  - 按 OVMF `SwapBytes32()` 的实际写法还原 64 位 descriptor 地址。
- 新增 DMA 执行逻辑：
  - 从 guest memory 读取 `FW_CFG_DMA_ACCESS`；
  - 按 control bits 执行 select / read / write / skip；
  - read 时把 item 数据写入 guest buffer；
  - 最后把 status 写回 descriptor。

同时，`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs` 里继续扩大 I/O bitmap 拦截范围：

- 保留 `0x402` debugcon；
- 新增 `0x510..0x511`；
- 新增 `0x514..0x51b`。

这样 OVMF 的 fw_cfg 端口访问会触发 VM-exit，然后进入 AxVisor 的模拟路径。

### 5. Step 10 的小意外：fw_cfg probe 也用了 string I/O

第一次加完最小 fw_cfg 后，旧的 DMA `#BR` 没有立刻复现。先暴露出来的是另一个更靠前的问题：OVMF 在 `QemuFwCfgProbe()` 里对 data port `0x511` 使用了 `IoReadFifo8()`，生成 `rep insb`。AxVisor 当时仍然不支持 string/repeat I/O exit。

运行时停在：

```text
VMX unsupported IO-Exit: ... is_in: true, is_string: true, is_repeat: true, port: 0x511
```

处理办法延续 debugcon 那次的思路：不在 AxVisor 里补完整 string I/O，而是在 `edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` 的 `QemuFwCfgProbe()` 中，把 signature 和 interface version 的 `IoReadFifo8(FW_CFG_IO_DATA, ...)` 改成逐字节 `IoRead8(FW_CFG_IO_DATA)`。

这还是 OVMF 侧的临时适配。目的只是绕过 AxVisor 当前 VMX string I/O 缺口，让 PEI fw_cfg 路径继续往前走，看看下一个真实问题是什么。

### 6. develop2 的验证结果

修改后重新构建 DEBUG OVMF，写入 rootfs，再跑 smoke test，已经能看到 fw_cfg cache 成功：

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

这说明 Step 9/10 的目标已经完成：`PlatformPei` 能从 AxVisor 的最小 fw_cfg 里拿到 `QEMU` signature、DMA capability、CPU count 和 e820 内存信息。develop1 中 `QemuFwCfgPei` 的 DMA 卡点已经跨过去了。

新的停点变成 MTRR 断言：

```text
OVMF debugcon: ASSERT MemDetect.c(1344): (MtrrSettings.MtrrDefType & 0x00000400) == 0
!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!
RIP - 0000000000849106
Find image based on IP(0x849106) ... PlatformPei.dll
```

也就是说，当前问题已经从 fw_cfg 推进到 PEI 内存发布 / RAM 初始化里的 MTRR 初始状态。

## 当前阶段判断

整理 develop0、develop1、develop2 后，目前进度是：

1. OVMF 镜像加载和 reset vector 进入已经成立。
2. OVMF DEBUG 输出已经能通过 AxVisor 的 `0x402` debugcon 设备打印出来。
3. OVMF 已确认进入 PEI，并执行到 `PlatformPei`。
4. PEI 阶段需要的最小 fw_cfg 已经补上：signature、interface version、CPU count、file directory、`etc/e820`、DMA descriptor 处理。
5. 已跨过 `QemuFwCfgInitCache`，说明 `PlatformPei` 能拿到当前阶段需要的 fw_cfg 内存信息。
6. 新的阻塞点是 `MemDetect.c` 对 `IA32_MTRR_DEF_TYPE` 状态的断言，不再是 fw_cfg item 缺失。

## 后续边界

下一步不应该继续扩 fw_cfg item，也不应该直接跳到 ACPI / PCI / virtio 这类完整平台模型。更小的方向是查 OVMF SEC/PEI 对 MTRR 的假设，以及 AxVisor 当前怎么暴露 MSR/MTRR：

- OVMF `SecMtrrSetup()` 会把 MTRR default type 设成 write-back，并打开 enable bit。
- 现在的断言说明，进入 PEI 前 `IA32_MTRR_DEF_TYPE` 的 fixed MTRR enable bit 可能已经是 1。
- AxVisor 可能还在让 guest 直接看到 host/current MTRR 状态，也可能还没有给 OVMF 提供符合早期假设的虚拟 MTRR 初值。

所以后续最小动作是定位并处理 `IA32_MTRR_DEF_TYPE` / MTRR MSR。不要继续补完整 fw_cfg 设备模型。如果后面发现确实需要完整 MTRR 虚拟化，再把它作为新的边界单独记录。
