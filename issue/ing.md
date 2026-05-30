# OVMF 适配欠账清单

为了把 OVMF 从 reset vector 推进到 BDS + UEFI app 启动，有不少基础设施是临时绕过的。这些欠账不补，guest OS 跑不起来。

---

## 1. String/repeat I/O exit

**怎么绕过的：** AxVisor 不支持 `rep insb`/`rep outsb`，改了 OVMF 两处源码，把 FIFO 读写改成逐字节。debugcon 输出（`IoWriteFifo8` → `IoWrite8`）和 fw_cfg probe（`IoReadFifo8` → `IoRead8`）都是这么绕过去的。

**为什么能绕过：** 这两处是 OVMF 自己的 debug 和 probe 路径，改成逐字节不影响功能。

**在哪绕的：**
- `edk2/OvmfPkg/Library/PlatformDebugLibIoPort/DebugLib.c` — develop0 Step 6
- `edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` — develop2 Step 10

**正式要做的：** AxVisor 的 VMX/SVM exit handler 要能处理 string I/O exit（`is_string && is_repeat`），然后撤掉 OVMF 的两处 workaround。别的 UEFI 驱动如果用了 string I/O，现在会直接 panic。

---

## 2. pflash 语义

**怎么绕过的：** OVMF_CODE 和 OVMF_VARS 直接加载到固定 GPA（`0xFFC8_4000` 和 `0xFFC0_0000`），跳过了整个 pflash 设备模型。没有 CFI flash 命令协议，没有只读保护，没有写入语义。

**为什么能绕过：** OVMF 的 SEC/PEI 阶段不走 pflash 读取固件，它直接从内存地址执行。变量区的读写在早期阶段也不频繁。

**在哪绕的：** 没有正式代码改动，是 VM 启动时的镜像加载逻辑直接忽略了 pflash 语义。`axvm/src/vm.rs` 的 image loading 路径。

**正式要做的：**
- pflash 设备模型（CFI flash 命令协议）
- OVMF_CODE 只读保护（guest 写固件区要拒绝或忽略）
- OVMF_VARS 可写持久化（guest 写变量区要落盘）
- flash 命令/状态寄存器访问

不补这个，OVMF 每次启动丢失所有变量（boot order、Secure Boot 设置等）。

---

## 3. fw_cfg 设备模型

**怎么绕过的：** 实现了 PEI 阶段必需的最小 item 集合：signature、interface version、SMP CPU count、`etc/e820`、file directory。file directory 是写死的固定内容，DMA error 语义简化（`FW_CFG_DMA_CTL_ERROR` 只做了最小回写），不支持 write item。

**为什么能绕过：** OVMF PEI 的 `QemuFwCfgInitCache` 只读这几个 item，读完就过了。DMA 通路只需要能处理 read 就够。

**在哪绕的：**
- `axvm/src/vm.rs` `FwCfgState` / `handle_fw_cfg_dma()` — develop2 Step 9
- DMA 地址语义修正 — develop2 Step 9（`SwapBytes32` 还原）
- fw_cfg DMA 稳定性复核 — develop3 Step 3

**正式要做的：**
- file directory 要根据实际 guest 镜像动态生成
- DMA status error 语义要完整
- 要支持 fw_cfg write item
- DXE 阶段可能需要更多 `etc/` 文件项（ACPI table blob、SMBIOS、boot menu 等）

---

## 4. CPUID 虚拟化

**怎么绕过的：** 只改了 leaf 1。把 APIC ID、CPU 数量、x2APIC 能力、hypervisor bit、MCE 位处理了，但 topology、cache、extended 等 leaf 都没碰。多 vCPU 时每个 vCPU 需要不同的 APIC ID 和 topology，现在只支持单核。

**为什么能绕过：** OVMF PEI/DXE 阶段主要看 leaf 1，topology leaf 在单 CPU 场景下不是阻塞项。

**在哪绕的：**
- VMX：`x86_vcpu/src/vmx/vcpu.rs` `handle_cpuid()` — develop4 Step 1, develop5 Step 1
- SVM：`x86_vcpu/src/svm/vcpu.rs` — 同步

**正式要做的：**
- topology leaf `0x0b` / `0x1f`（x2APIC ID、CPU 拓扑）
- cache leaf `0x2` / `0x4`
- extended leaf `0x80000001`
- 每 vCPU 不同的 APIC ID 和 topology 信息

---

## 5. MTRR 虚拟化

**怎么绕过的：** 只给 `IA32_MTRR_DEF_TYPE (0x2ff)` 做了 shadow，固定返回 `0x0806`（WB, E=1, FE=0）。读写都在 VMX/SVM 内部完成，不走通用 MSR 链路。

**为什么能绕过：** OVMF PEI 的 `MemDetect.c` 只检查 `MtrrDefType` 的 FE 位，过了这关就能继续。

**在哪绕的：** `x86_vcpu/src/vmx/vcpu.rs` `builtin_vmexit_handler()` — develop3

**正式要做的：**
- Fixed MTRR（`0x250`~`0x268`）
- Variable MTRR（`0x200`~`0x20f`, `0x280`~`0x29f`）
- 现在 OVMF 后续读这些会直接拿到 host 值，内存类型可能对不上

---

## 6. APIC / 中断链路

**怎么绕过的：** 做了最小的 xAPIC MMIO 虚拟化：VMX 开了 `USE_TPR_SHADOW` 和 `VIRTUALIZE_APIC`，guest LAPIC GPA 映射到 APIC-access backing page，APIC write exit 能分发到 vLAPIC。但 read exit 没做全，APIC timer 没虚拟化，EOI 没串到外部中断链路。单 vCPU 的 broadcast `AllExcludingSelf` 做 no-op。vIOAPIC、i8259、INTx 路由、MSI/MSI-X 全没做。

**为什么能绕过：** OVMF PEI 的 CpuMpPei 只需要读写 LAPIC 基本寄存器（APIC ID、SVR 等），不依赖 timer 和外部中断。BDS 阶段的 virtio-blk 走 polling，不依赖完成中断。

**在哪绕的：**
- vLAPIC 基础：`x86_vlapic/src/vlapic.rs` — develop4 Step 1
- xAPIC MMIO：`x86_vcpu/src/vmx/vcpu.rs` + `x86_vlapic/src/lib.rs` — develop5 Step 3
- LAPIC GPA EPT 映射：`axvm/src/vm.rs` — develop5 Step 5

**正式要做的：**
- APIC timer（periodic/one-shot/TSC-deadline）—— guest 内核的调度 tick 靠这个
- EOI 处理串到 vIOAPIC
- vIOAPIC（`0xFEC0_0000` MMIO、Redirection Table）
- i8259 PIC（端口 `0x20`/`0x21`/`0xA0`/`0xA1`）
- INTx → vIOAPIC 中断路由
- MSI/MSI-X 支持
- APIC read exit 完整处理
- 多 vCPU：active vCPU mask、INIT/SIPI、AP wakeup

没有这些，guest 内核基本跑不起来（timer 没有、中断送不进去）。

---

## 7. x2APIC MSR 处理

**怎么绕过的：** x2APIC MSR 范围 `0x800..0x83f` 已经在 MSR bitmap 里拦截，但 vLAPIC 对这些 MSR 的处理不完整。当前 smp1 场景下 OVMF 在 xAPIC 模式，没走到 x2APIC，所以没暴露。develop4 Step 2 参考了 crosvm 的实现，只做了代码拉取，没有正式实现。

**为什么能绕过：** 单 CPU 场景下 OVMF 不需要开启 x2APIC 模式。

**在哪绕的：**
- MSR bitmap 拦截：`x86_vcpu/src/vmx/vcpu.rs` `setup_msr_bitmap()` — develop4 Step 1
- 参考拉取：`/tmp/crosvm-ref/devices/src/irqchip/apic.rs` — develop4 Step 2

**正式要做的：** ICR `0x830`、TPR `0x808`、EOI `0x80B`、LDR `0x80D`、SelfIPI `0x83F` 等 MSR 的完整处理。多 vCPU 场景下 OVMF 可能会开 x2APIC。

---

## 8. MpInitLib 单 CPU 绕开

**怎么绕过的：** 改了 OVMF 的 `MpInitLib` 三个函数：`WakeUpAP()` 单 CPU broadcast 时直接跳过、`GetBspNumber()` APIC ID 不匹配时 fallback 到 BSP 0、`MpInitLibInitialize()` 单 CPU 时覆盖 CPU info。这是 OVMF 侧的 workaround，不是 AxVisor 侧的 SMP 支持。

**为什么能绕过：** smp1 配置下只有一个 vCPU，OVMF 的 AP wakeup 流程会一直等 AP 响应，永远等不到。

**在哪绕的：**
- `edk2/UefiCpuPkg/Library/MpInitLib/MpLib.c` — develop4 Step 4
- `edk2/UefiCpuPkg/Library/BaseXApicX2ApicLib/BaseXApicX2ApicLib.c` — develop4 Step 4（IPI 日志）

**正式要做的：** AxVisor 侧实现多 vCPU AP startup 支持（INIT/SIPI 处理、AP wakeup 等待、active vCPU mask），然后撤掉 OVMF 的三个 workaround。

---

## 9. PCI 设备模型

**怎么绕过的：** PCI MMIO 是纯直通。没有 ECAM、没有 PCI config space 拦截/模拟、没有 PCI 主桥模拟。低端 MMIO（`0x8000_0000..0xDFFF_FFFF`）和高端 MMIO（`0xE000_0000..0xEFFF_FFFF`）都是 EPT 直接映射。

**为什么能绕过：** OVMF BDS 阶段的 PCI 设备枚举走的是外层 QEMU 的真实 PCI config space，AxVisor 只需要让 MMIO 访问能穿过 EPT 就行。

**在哪绕的：**
- 低端 MMIO passthrough 配置 — develop5 Step 5
- 高端 MMIO passthrough 配置 — develop5 Step 5

**正式要做的：**
- ECAM（PCIe configuration space MMIO，通常在 `0xE000_0000`）
- PCI config space 拦截/模拟
- PCI 主桥（host bridge）模拟
- PCI 中断路由（INTx → vIOAPIC）

---

## 10. virtio-blk nested-DMA

**怎么绕过的：** 拦截 virtio-blk legacy I/O BAR `0x6000..0x607f`，queue PFN 写入时做 GPA→HPA 翻译，queue notify 前原地改写 descriptor table 里的 `desc.addr`（GPA→HPA），最多遍历 8 个 descriptor。能跑，但会改 guest 内存。

**为什么能绕过：** nested virtualization 下，L2 guest 的 queue GPA 对外层 QEMU 没有意义。原地改写是最小侵入的方式，不需要 AxVisor 自己维护 virtqueue 状态。

**在哪绕的：**
- `axvm/src/vm.rs` `OvmfVirtioBlkIoState` / `rewrite_ovmf_virtio_blk_descriptors()` — develop6 Step 2
- I/O 拦截从 VMX 层移到 AxVM 层 — develop6 Step 2

**正式要做的：**
- virtqueue 代理（AxVisor 自己维护 avail/used ring，不改 guest 内存）
- request 拆包和后端抽象
- modern virtio 支持
- 完成中断注入（依赖 vIOAPIC 或 APIC timer）
- net、console 等其他 virtio 设备

---

## 11. ACPI PM I/O passthrough

**怎么绕过的：** 端口 `0x600`~`0x60F` 直接转发到宿主端口，不是 device model。OVMF 读 PM1_CNT 拿到宿主的 S0 状态，写 PM1_CNT 设宿主的 SCI_EN。

**为什么能绕过：** OVMF BDS 阶段只做一次 PM1_CNT 读写来检测/使能 ACPI，不做 PM timer 轮询。

**在哪绕的：** `axvm/src/vm.rs` `handle_acpi_pm_io_read/write()` — develop6 Step 3

**正式要做的：**
- ACPI PM device model（不依赖宿主端口）
- PM timer（`0x608`）
- ACPI SCI 中断注入

---

## 12. UEFI 可启动镜像

**怎么绕过的：** 手动创建了 64MB GPT 盘（`tmp/uefi-boot-test.img`），用 `losetup` + mount 放入 Shell.efi 和自定义 UEFI app。`--rootfs` 参数绕过 ostool 默认 Alpine ext4 镜像。OVMF_VARS 不持久化。

**为什么能绕过：** 只需要验证 OVMF → virtio-blk → PE32+ 的链路能通，不需要自动化的镜像构建。

**在哪绕的：** develop6 Step 4-5

**正式要做的：**
- 镜像构建自动化
- OVMF_VARS 持久化（依赖 pflash）
- Linux EFI stub 解压失败要排查（可能是 SSE/AVX 暴露或 DMA 数据完整性）

---

## 13. ArceOS UEFI 适配

**怎么绕过的：** 目前只做了 ArceOS 目录内的 UEFI helloworld 示例（`os/arceos/examples/uefi-helloworld/`），验证 PE32+ 构建能跑。还没接入 ArceOS runtime。

**为什么能绕过：** helloworld 足以验证 OVMF → virtio-blk → PE32+ 加载 → 执行 的完整链路。

**在哪绕的：** develop7 Step 1-2

**正式要做的：**
- linker script 适配（ELF → PE32+）
- UEFI entry（`efi_main`）接 `rust_main`
- UEFI memory map → multiboot-info 转换（`mem.rs`）
- `ExitBootServices` 处理
- ArceOS 自己的 console/allocator 接管
- 多 vCPU SMP
