# TASK
我们的总任务是给axvisor添加整个OVMF客户机启动链条。
## 详细说明：
### 当前状态
现在 Axvisor 里的 x86_64 客户机只能以传统 BIOS 方式启动。具体来说：

配置中使用 entry_point 和 bios_load_addr 将启动代码固定加载到 0x8000 这样的低地址位置。
启动依赖我们自己的 rvm-bios.bin 或 axvm-bios.bin 固件，而不是行业标准的 OVMF/UEFI 固件。
镜像加载器只认识“按物理地址直接放置内核、BIOS、内存盘、设备树”这一种方式，缺少对 OVMF_CODE（UEFI 代码）、OVMF_VARS（UEFI 变量存储）、pflash、fw_cfg 以及 UEFI 变量区域等语义的支持。
vCPU 初始状态也被设定为实模式 / trampoline 风格，依靠“不受限客户机”能力直接在低地址执行启动代码。
设备模型非常单薄：x86 的 emu_devices 列为空，IOAPIC/LAPIC/HPET 基本直接穿透给客户机，但客户机一侧看不到 ACPI 表、PCI 根桥、virtio-pci 设备，也缺少虚拟 IOAPIC 和 MSI 中断路由。
一句话总结：当前缺的不是某个小功能，而是一整套能让标准 UEFI 客户机找到硬件并启动的“PC 平台骨架”。

### 目标
让 Axvisor 能够启动标准的 x86_64 UEFI 客户机：

使用 OVMF / EDK2 作为客户机固件，从固件入口自然进入 UEFI 环境。
通过 ACPI 表、PCI 总线、virtio 设备等标准 PC 发现路径，逐步启动 Linux EFI stub、UEFI 应用或其他 UEFI 感知的操作系统。
最小可行目标包括：

OVMF_CODE 与 OVMF_VARS 的加载和地址映射
fw_cfg（QEMU 风格的固件配置接口）
基本 ACPI 表（最少要包含 RSDP、XSDT/RSDT、FADT、MADT、MCFG）
一个最小可用的 PCI 主桥和配置空间（ECAM 或 PIO 方式）
virtio-block 设备（通过 virtio-pci 暴露），能用来枚举磁盘并启动 OS
后续接入 virtio-net 和 virtio-console
基本中断链路：vLAPIC、vIOAPIC、i8259、INTx，后续再完善 MSI/MSI-X
### 技术实现路线
扩展启动配置模型 将启动方式从单一的 bios_path / bios_load_addr 扩展为可配置的 boot = "trampoline"（传统 BIOS 方式）或 boot = "uefi"。配置项需要显式描述 OVMF 的代码、变量存储、pflash 属性，以及固件在客户机物理地址空间中的位置。这部分改动会落在 axvmconfig、axvm::config 以及镜像加载器上。
补齐 UEFI 固件所需的“平台设备” 先实现一个 QEMU 兼容的 fw_cfg 设备，用以向固件传递启动信息。接着生成并暴露最小 ACPI 表集合：RSDP、XSDT/RSDT、FADT、MADT、MCFG。之后根据需要逐步加入 HPET、SPCR 以及 DSDT/SSDT 等更完整的表。
补齐 PC 风格的客户机设备模型 实现最小 PCI 主桥和配置空间，优先支持 virtio-pci block 设备，让 UEFI 固件能发现磁盘并从中启动。随后再加入 virtio-net 和 virtio-console 支持，构成一套基本可用的输入输出环境。
完善 x86 中断链路 短期：利用 vIOAPIC + INTx 让 virtio-pci 的基本中断跑通。 中期：完善 vLAPIC 的 EOI、IPI、timer 以及 MSI/MSI-X 支持。目前 x86_vcpu 中已经有 pending event 和 vLAPIC 雏形，但还未能串成完整的 IRQ 数据通路，需要把它们连起来并填补缺失逻辑。
建立验证闭环 新建一个专门的 x86_64 UEFI 客户机配置，并编写对应的 QEMU 冒烟测试。验证路径从 OVMF 的 UEFI Shell 开始，逐步到通过 Linux EFI stub 和 virtio-block 启动完整内核。目前 qemu-x86_64.toml 中仍标记为 uefi = false，我们需要将其升级为真正的 UEFI 启动测试。

# 当前进展
我们已经完成了最小ovmf加载客户机路径的适配,但是只做到了运行arceos helloword镜像后开始停止.之前的修复完善面向ovmf的输出日志,面向结果的最小修复.基本堆在了axvm模块里,现在我们开始进行之前的完善.
参考`axvisor-2026-notebooks/issue/ovmf-infra-roadmap.md`,这是在没有开始完善任何基础设施之前写的.
note: 方向二对于我们的当下开发必不可少,但是方向二不是我们组在进行的,且无法认为可以基于别人方向二设计的框架.同时axvisor现有基础设施并非是完善的.我们不应该为axvisor补全难度较大的基础设施,而是在现有基础设施上实现一个相对标准的为ovmf 客户机加载而需要完善的基础设施.但是如果需要适配ovmf加载而让axvisor完善的一些基础设施工程量不大,则可以借鉴其他项目,依托axvisor现有架构进行完善.`references`里有cloud-hypervisor,和qemu

# 文档指南
- axvisor-2026/start.md下是如何启动axvisor，可以根据需要进行修改
- docs/OVMF-Boot-Overview.md 最权威的OVMF启动流程
- docs/x86_64 虚拟化官方架构
- docs/inode 官方知识索引
- self.md，我手动总结的最简洁的规划，包括一定的原理解释，能反映目前进行到哪一步
- config-chain.md 目前需要理解，反应从toml->配置链路->axvisor了解的关键地方，也是OVMF加载关键（有时效性）
- develop文件夹,描述当前进展,当前修改
- /issue/mod0.md 目前需要理解为了让OVMF开始启动的所有修改路线（有时效性）
- /issue/ing.md 是还需要继续进行对mod0的补充（规划不一定对）
- /issue/ovmf-infra-roadmap.md OVMF适配查缺补漏文档
- OVMF-go.md 是目前的详细规划由ai生成，self.md可以反应我检验到哪一步
- OVMF-all.md 是OVMF大致模块规划

# skill
- /doc: 把当前的对话内容整理成文档（注意规范，不要以对话的形式）


方向二(其他组的课题选择):
方向二：设备与中断框架重构
当前状态
目前的设备与中断框架可以工作，但属于“拼接式”设计，扩展和维护成本较高：

AxVmDevices 内部将设备按 MMIO、SysReg、Port 三类分别放进 Vec，访问时通过线性查找分发，效率和控制力都有限。
添加一种新设备需要改动中心化的 EmulatedDeviceType 大枚举，并在各处补充 match 分支，这意味着“加一个设备”很容易触碰核心代码。
中断注入的实现路径随架构散落在不同角落：AArch64 直接操作 VGIC / GICH / ICH 寄存器，RISC‑V 通过查找 vPLIC 设备并写 pending 位，LoongArch 写 CSR / GCSR，x86 则在 vCPU 里自行排队 pending event。VM 层面的通用接口（如 inject_interrupt_to_cpus）还挂着 TODO，vLAPIC 也有大量路径未真正实现。
错误处理比较生硬：设备查找未命中就会 panic，很多地方还存在 unwrap、expect、todo!() 和 unimplemented!()，整体鲁棒性不足。
目标
将设备模型和中断模型从具体 VM 实现与架构特判中解耦出来，形成一套 “资源注册 + 总线路由 + IRQ 路由 + 后端抽象” 的统一框架：

每个设备都能清晰声明自己的资源和能力：哪些 MMIO 区域、哪些 PIO 端口、哪些系统寄存器、占用哪些 IRQ 线、是否支持 MSI/MSI‑X、DMA、PCI BAR，以及 reset/suspend/resume 行为等。
注册时统一进行资源冲突检查（地址范围重叠、总线类型匹配、IRQ 资源合法性、架构支持等）。
VM exit 处理只抽象为统一的总线访问（BusAccess），不再直接关心“到哪个设备的哪张表里去查”。
中断方面，提供 VM 级的 InterruptRouter / IrqSink 抽象：设备只发出 raise / lower / pulse / msi / eoi 等语义操作，路由器再根据架构分发给 vLAPIC/vIOAPIC、VGIC、vPLIC/AIA、LoongArch 虚拟中断控制器。如此一来，设备后端再也不需要直接调用特定架构 API，也不会通过“写另一个设备的 MMIO”来间接注入中断。
技术实现路线
定义统一的概念层 先抽象出基本类型和接口：DeviceId、Resource、BusKind、BusAccess、BusResponse、DeviceError、IrqLine、IrqTarget、InterruptControllerOps 等。原有的 BaseDeviceOps<R> 先通过适配器接入，降低一次性迁移的风险。
升级设备容器为 DeviceRegistry + BusRouter 用带索引的注册表替代原来的 AxVmDevices<Vec>，按总线类型建立查找结构，并在注册时检测地址冲突。设备创建从集中式 match 大枚举改为工厂/注册机制，以后再新增 virtio、串口、IOAPIC、AIA/IMSIC、LoongArch 中断控制器等设备时，就不再需要修改核心容器代码。
统一 VM exit 分发 将 MmioRead/Write、IoRead/Write、SysRegRead/Write 统一转换为总线事务（bus transaction），由路由器统一返回结果或结构化错误。未命中、非法访问宽度、对只读寄存器的写操作等都不再直接 panic，而是返回可处理的错误信息。
引入 VM 级 IRQ 路由器 各设备后端改为持有 IrqSink，不再直接触碰架构相关代码。RISC‑V 的 vPLIC 仍然可以通过显式 set_pending 来工作，但这一操作只发生在路由器内部。AArch64 的 VGIC 通过封装 LR（List Register）注入，与设备解耦。x86 一侧则整理出清晰的 vLAPIC、vIOAPIC、MSI 路由路径。LoongArch 先用 CSR 后端作为过渡适配。
借助大模型辅助设计（但代码必须扎实） 我们可以利用大模型系统性比较 QEMU/KVM、crosvm、Firecracker、ACRN 等项目的设备与中断抽象，辅助生成 trait 草案、配置 schema、负例测试、迁移清单和能力矩阵。最终落地时，所有接口仍必须是强类型 API，并通过单元测试和冒烟测试充分验证，不依赖“模型直接生成的代码”。
补齐测试矩阵 围绕重构中容易出错的环节建立密集测试：地址范围冲突检查、总线未命中、访问宽度违规、只读寄存器写入保护、sysreg/port 范围越界、vPLIC claim/complete 流程、VGIC LR 满时的行为、vLAPIC timer/IPI/EOI 正确性、SMP IPI、timer tick、直通设备与仿真设备范围重叠防护等。 通过这套测试，不仅能保证重构后架构更清晰优雅，也能为后续支持 x86_64 UEFI 客户机这种设备拓扑更复杂的场景打下可靠基础。