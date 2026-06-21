# axvisor-2026-notebooks

AxVisor x86_64 OVMF/UEFI guest bring-up 的工作笔记和调试记录。

代码在同工作区的 `tgoskits/`（AxVisor）和 `edk2/`（OVMF 固件）。本目录只放文档，不参与构建。

## 项目目标

让 AxVisor 能启动标准 OVMF 固件，沿 UEFI 路径加载 guest。过程中补齐 AxVisor 缺失的 x86 PC 平台基础设施：UEFI 配置链路、fw_cfg、MSR/APIC 虚拟化、PCI MMIO、legacy virtio-blk、ACPI 转发、PIT/PIC 中断模型。

当前已验证的完整链路：

```
AxVisor → OVMF_CODE/VARS → x86 reset vector → SEC/PEI/DXE/BDS
→ legacy virtio-blk → FAT32 ESP → /EFI/BOOT/BOOTX64.EFI
→ ArceOS UEFI helloworld
```

在此基础上已开始 Linux UEFI guest 启动尝试，当前卡在 timer/APIC 中断投递路径（详见 `develop/linux-uefi-timer-apic/`）。

## 目录结构

```
.
├── start.md                 运行手册（QEMU 命令、镜像布局、常见问题）
├── ALL.md                   总任务地图
├── develop/                 调试和实现的阶段记录（事实来源）
├── docs/                    背景资料（OVMF 启动流程、x86 手册索引）
├── issue/                   面向协作的压缩总结和进展报告
└── subagent_watch/          外部模型审计输出
```

## develop/

按时间顺序记录每个调试阶段：看到了什么日志或源码、判断是什么、改了哪里、验证结果如何。

| 目录/文件 | 内容 |
|---|---|
| `develop0.md` ~ `develop7-UEFI-ArceOS.md` | OVMF 从 reset vector 到 ArceOS helloworld 的逐步推进 |
| `develop8-fwcfg-extract.md`, `develop9-fwcfg-content-model.md` | fw_cfg 从 vm.rs 提取到 axdevice，内容模型整理 |
| `virtio-blk-nested-dma/`（10 份文档） | virtio-blk 从 vm.rs 特殊路径到 axdevice 设备模型的完整重构过程 |
| `uefi-acpi-passthrough/` | 外层 QEMU ACPI fw_cfg blob 转发到嵌套 guest |
| `linux-uefi-pf-diagnostics/` | Linux EFI stub 早期 #PF 诊断（已越过） |
| `linux-uefi-timer-apic/`（4 份文档） | Linux timer/APIC：PIT reset、LINT0 路由、PIC 8259、ISR stuck 修复、当前 inject 传递问题 |

阅读顺序建议：先读 `start.md` 了解怎么跑，再按 `develop0` 到 `develop7` 看 OVMF 主线，然后按需读 `virtio-blk-nested-dma/` 和 `linux-uefi-timer-apic/`。

## issue/

| 文件 | 内容 |
|---|---|
| `ovmf-infra-roadmap.md` | 14 个模块的基础设施清单，每个说明 OVMF 阶段、bring-up 角色、当前适配状态、代码位置 |
| `ovmf-progress-2026-05-06.md` | 近期进展总结（virtio-blk 重构、fw_cfg 提取、ACPI 转发、Linux 启动尝试） |
| `ing.md` | 早期欠账草稿，仅供参考 |
| `mod0/`, `mod1/` | 模块级详细报告 |

## subagent_watch/

外部模型的审计输出，按主题分组：

- **APIC/PIC 合同**：`linux-lvt0-write-path-audit.md`、`vmx-apic-write-path-contract.md`、`virtual-wire-extint-contract-matrix.md`、`intel-vmx-apic-access-contract.md`
- **PIT/timer 诊断**：`linux-check-timer-pit-pic-window.md`、`reference-pit-pic-extint-window-contract.md`、`axvisor-pit-poll-window-audit.md`
- **virtio-blk 协议**：`virtio-blk-audit-prompt1~4-*.md`、`mature-irq0-chain-comparison.md`
- **ACPI/IRQ 边界**：`acpi-platform-forwarding-evidence.md`、`uefi-irq-acpi-platform-boundary.md`

## docs/

| 文件 | 内容 |
|---|---|
| `OVMF-Boot-Overview.md` | OVMF 启动阶段概览 |
| `virtio-blk-device-model.md` | virtio-blk 设备模型参考 |
| `x86_64/inode.md` | Intel x86 SDM 页码索引，按需更新 |

## 工作区关系

```
<workspace>/
├── axvisor-2026-notebooks/     本目录（文档）
├── tgoskits/                   AxVisor / ArceOS 代码
│   └── references/             上游参考实现（QEMU、Cloud Hypervisor）
└── edk2/                       OVMF / UEFI 固件代码
```

## 使用约定

- `develop/` 是事实来源。写新总结或拆任务前，先读相关 develop 记录。
- `issue/` 是面向人的摘要，不作为判断依据。
- 每次有意义的调试推进，同步到 `develop/` 再切上下文。
- OVMF/EDK2 源码不做逻辑修改（debug 输出和 fw_cfg 读入除外）。
- 借鉴 `references/` 下的成熟实现，不自己发明大框架。
- 运行方式、镜像位置、预期日志变化时，同步更新 `start.md`。
