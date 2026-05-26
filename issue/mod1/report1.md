# OVMF 早期启动推进报告

## 当前进展

已经确认：

- AxVisor 可以加载自编译 DEBUG OVMF；
- OVMF 已经进入 **PEI**；
- OVMF 的 DEBUG 输出已经能通过 AxVisor 打到宿主终端；
- OVMF 在 PEI 里的 `PlatformPei` 能通过最小 fw_cfg 读到必要信息；
- 当前新的停点已经从 fw_cfg 转到 PEI 内存初始化相关的 MTRR 断言。

`OVMF-Boot-Overview.md` 里对应的阶段判断是：**PEI**。已经走到 `PlatformPei` / `QemuFwCfgPei` 之后，进入 `PublishPeiMemory()` / `MemDetect` 相关路径。

---

## 做了什么

本轮工作可以分成两条线：

1. **让 OVMF 的 DEBUG 输出可见**
   - 目的：先把 OVMF 内部日志接出来，才能知道它到底跑到哪里。
2. **给 OVMF 提供最小 fw_cfg**
   - 目的：满足 PEI 阶段 `PlatformPei` 获取 `QEMU` 签名、CPU 数量、`etc/e820` 内存布局等信息的需求。

---

## 阶段判断：PEI

日志上稳定看到：

- `SecCoreStartupWithStack(...)`
- `Platform PEIM Loaded`
- `QemuFwCfgProbe: Supported 1, DMA 1`
- `QemuFwCfgInitCache Pass!!!`
- `PlatformMaxCpuCountInitialization: BootCpuCount=1 MaxCpuCount=1`
- `PlatformGetLowMemoryCB: LowMemory=0x4000000`
- `PublishPeiMemory: ...`

说明 OVMF 已经从 SEC 进入 PEI，并且 `PlatformPei` 已经开始拿 fw_cfg 提供的信息来做内存发现。

当前新停点是：

- `ASSERT MemDetect.c(1344): (MtrrSettings.MtrrDefType & 0x00000400) == 0`

这说明问题已经不在 fw_cfg，而是在 PEI 内存初始化阶段的 MTRR 初始状态。

---

## 第一条线：把 OVMF 的 DEBUG 输出接到 AxVisor

### 为什么要做

一开始只能看到 AxVisor 的 VMX exit 和异常现场，无法直接知道 OVMF 自己执行到了哪里。要继续往下定位，就必须让 OVMF 的 `DEBUG()` 输出能打印到宿主终端。

### OVMF 侧做了什么

位置：`edk2/OvmfPkg/Library/PlatformDebugLibIoPort/DebugLib.c`

这里把 debugcon 输出路径从 `IoWriteFifo8()` 改成了逐字节 `IoWrite8()`。

原因是：

- OVMF 默认 debug 端口是 `0x402`；
- 之前 `IoWriteFifo8()` 会生成 `rep outsb`；
- AxVisor 当前对这种 string/repeat I/O 的 VMX exit 还不完整；
- 所以先把 OVMF 的输出改成普通 byte I/O，避免卡在 unsupported I/O exit。

### AxVisor 侧做了什么

#### 1）拦截 debugcon 端口

位置：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`

这里把 `0x402` 加进了 I/O bitmap 拦截范围，让 guest 对 debugcon 的访问会 VM-exit 到 AxVisor。

#### 2）实现最小 debugcon 设备

位置：`tgoskits/components/axdevice/src/device.rs`

新增 `OvmfDebugConDevice`，核心行为是：

- `IoRead8(0x402)` 返回 `0xE9`，让 OVMF 的 debug port probe 通过；
- `IoWrite8(0x402)` 按行聚合输出到宿主日志；
- 输出前缀统一为 `OVMF debugcon:`。

### 这条线的结果

打通后可以直接看到 OVMF 内部日志，例如：

- `SecCoreStartupWithStack(...)`
- `Platform PEIM Loaded`
- `QemuFwCfgProbe: Supported 1, DMA 1`

这一步的价值不只是“能打印日志”，而是给后续所有 OVMF 内部定位建立了可见性基础。

---

## 第二条线：给 OVMF 做最小 fw_cfg 适配

### 最小适配

当前卡点在 PEI 的 `PlatformPei`，它并不需要完整平台模型，只需要 fw_cfg 提供一小组必要信息：

- fw_cfg 是否存在；
- 是否支持 DMA；
- CPU 数量；
- `etc/e820` 内存布局；
- file directory 能让 OVMF 找到这些条目。


---

## 2.1 AxVisor 侧：构造最小 fw_cfg

位置：`tgoskits/components/axvm/src/vm.rs`

这里新增了最小 `FwCfgState` 和 item 构造逻辑。

### 端口

- selector：`0x510`
- data：`0x511`
- DMA address：`0x514` 和 `0x518`

### 条目

- `0x0000`：signature，内容是 `QEMU`
- `0x0001`：interface version，带 `FW_CFG_F_DMA`
- `0x0005`：SMP CPU count
- `0x0019`：file directory
- `0x8000`：`etc/e820`

### 这几个条目的作用

- `QEMU`：证明 fw_cfg 存在；
- `FW_CFG_F_DMA`：告诉 OVMF 这里支持 DMA；
- CPU count：让 `PlatformPei` 能初始化平台 CPU 相关信息；
- `etc/e820`：给 `PlatformPei` 提供内存布局；
- file directory：让 OVMF 能通过名字找到 `etc/e820`。

### 这里是怎么构造的

`etc/e820` 不是随便写死的，而是根据 AxVisor 的 guest memory region 生成，保证 PEI 阶段读到的是当前 VM 的实际内存布局。

---

## 2.2 OVMF 如何读取 fw_cfg

这里要分两层看：普通读取和 DMA 读取。

### 普通读取路径

位置：`edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c`

`QemuFwCfgProbe()` 会先：

1. 选中 signature item；
2. 读取 `QEMU`；
3. 再选中 interface version；
4. 读取 feature bits；
5. 判断是否支持 DMA。

之后 `QemuFwCfgInitCache()` 会继续把必要条目缓存起来，包括 file directory 和 `etc/e820`。

### DMA 读取路径

同样在：`edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c`

`InternalQemuFwCfgDmaBytes()` 会构造 `FW_CFG_DMA_ACCESS` descriptor，写入：

- `Control`
- `Length`
- `Address`

然后通过 `FW_CFG_IO_DMA_ADDRESS` 写出 descriptor 地址，让 fw_cfg 按 descriptor 执行读写。

### 这条路径的关键点

OVMF 会对 descriptor 字段做大小端转换，然后通过 I/O port 写出。AxVisor 这边必须按这个语义：

- 读 guest memory 里的 descriptor；
- 解释 control / length / address；
- 执行 read / write / skip / select；
- 再把 status 写回 descriptor。

---

## 2.3 AxVisor 侧还做了什么

### 1）端口拦截范围扩大

位置：`tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`

把 fw_cfg 相关端口也加进了 I/O bitmap：

- `0x510..0x511`
- `0x514..0x51b`

这样 OVMF 访问 fw_cfg 时会退出到 AxVisor 的设备模拟路径。

### 2）fw_cfg 状态机

位置：`tgoskits/components/axvm/src/vm.rs`

实现了：

- 当前 selector 记录；
- 当前 offset 记录；
- DMA 地址分两次 32 位写入后合成；
- 读取 guest descriptor；
- 执行 DMA read/write/skip/select；
- 写回 descriptor status。

---

## 2.4 一个小意外：fw_cfg probe 的 string I/O

第一次把最小 fw_cfg 接上后，新的停点不是原来的 DMA `#BR`，而是 OVMF 在 `QemuFwCfgProbe()` 里对 `0x511` 使用了 `IoReadFifo8()`，生成 `rep insb`。

AxVisor 当前还不支持这个 string/repeat I/O exit，所以先停在：

- `VMX unsupported IO-Exit: ... port: 0x511`

这里没有扩大 AxVisor 的 string I/O 支持，而是继续采用最小改动原则，改 OVMF 的 probe 代码。

位置：`edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c`

把：

- `IoReadFifo8(FW_CFG_IO_DATA, ...)`

改成了逐字节：

- `IoRead8(FW_CFG_IO_DATA)`

这样就绕过了当前 VMX 对 `rep insb` 的限制，让 PEI 里的 fw_cfg 继续往下走。

---

## 最终效果

现在已经能看到：

- `QemuFwCfgProbe: Supported 1, DMA 1`
- `QemuFwCfgInitCache Pass!!!`
- `Cache FwCfg Name: QemuFwCfgSignature ...`
- `Cache FwCfg Name: QemuFwCfgInterfaceVersion ...`
- `Cache FwCfg Name: QemuFwCfgFileDri ...`
- `Cache FwCfg Name: etc/e820 ...`
- `PlatformMaxCpuCountInitialization: BootCpuCount=1 MaxCpuCount=1`
- `PlatformGetLowMemoryCB: LowMemory=0x4000000`
- `PublishPeiMemory: ...`

这说明：

- `PlatformPei` 已经拿到了 fw_cfg 必要信息；
- PEI 的内存发现路径已经被推进；
- 当前失败点已经不是 fw_cfg，而是 MTRR 初始状态断言。

---

## 现在的新停点

当前停在：

- `ASSERT MemDetect.c(1344): (MtrrSettings.MtrrDefType & 0x00000400) == 0`

这说明下一步要看的不是 fw_cfg，而是 OVMF SEC / PEI 对 MTRR 的假设，以及 AxVisor 当前对相关 MSR 状态的暴露方式。

---

## 核心结论

- **日志线**：先把 OVMF DEBUG 接出来，才知道跑到了 PEI 和 `QemuFwCfgPei`。
- **fw_cfg 线**：只做最小适配，不做完整设备模型，先满足 `PlatformPei` 读 `QEMU` / CPU count / `etc/e820`。
- **当前阶段**：还是 **PEI**，但已经从 fw_cfg 进入到 `PublishPeiMemory` / `MemDetect`。
- **当前阻塞**：MTRR 断言，不是 fw_cfg。