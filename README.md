# axvisor-2026-notebooks

这个仓库是 AxVisor x86_64 OVMF/UEFI 适配的工作笔记。它不负责构建 AxVisor，也不保存 EDK2 源码。真正的代码仓库在同一工作区里的 `tgoskits/` 和 `edk2/`。

当前主线是让 AxVisor 能启动标准 OVMF 固件，并沿着 UEFI 路径加载最小 guest。这个过程会补到一批原来在 AxVisor 里不完整的 PC 平台基础设施，包括 UEFI 配置链路、OVMF 镜像加载、fw_cfg、MSR/APIC 虚拟化、PCI MMIO、virtio-blk、ACPI PM I/O 和 UEFI payload。

## 当前状态

已经验证的最小链路是：

```text
AxVisor
  -> OVMF_CODE / OVMF_VARS
  -> x86 reset vector
  -> OVMF SEC / PEI / DXE / BDS
  -> PCI / virtio-blk
  -> FAT32 ESP
  -> /EFI/BOOT/BOOTX64.EFI
  -> ArceOS UEFI helloworld
```

这说明 OVMF 已经能越过早期固件阶段，进入 BDS，从 virtio-blk 上读取 ESP，并加载一个 PE32+ UEFI 应用。它还不是完整 Linux EFI 启动，也不是完整 PC 平台模型。

当前最新工作在 `develop/develop7-UEFI-ArceOS.md`。

## 先读什么

| 目的 | 文档 |
|---|---|
| 跑当前 smoke 流程 | `start.md` |
| 了解总任务和仓库使用方式 | `ALL.md` |
| 看 OVMF 启动阶段 | `docs/OVMF-Boot-Overview.md` |
| 看 TOML 到 VM 的配置消费链路 | `develop/config-chain.md` |
| 看团队模块认领和欠账整理 | `issue/ovmf-infra-roadmap.md` |
| 看最新进展 | `develop/develop7-UEFI-ArceOS.md` |

`issue/ing.md` 只能当作历史草稿参考。判断事实、写新总结或拆任务时，以 `develop/` 里的阶段记录和当前代码为准。

## 目录说明

```text
.
├── ALL.md                         总任务和文档地图
├── start.md                       当前可执行的 OVMF/UEFI 运行手册
├── develop/                       按时间推进的调试和实现记录
├── docs/                          背景资料、启动流程和规划材料
└── issue/                         面向协作和 issue/report 的压缩总结
```

`develop/` 是事实来源。每个阶段记录应该说明看到了什么日志或源码逻辑、判断是什么、改了哪里、验证结果是什么。

`docs/` 更偏背景材料。这里的文档可以解释 OVMF 阶段、UEFI 路径、x86 架构资料和早期规划，但不一定代表最新实现状态。

`issue/` 用来放给团队协作看的高层整理。这里的内容应该能帮助成员认领模块、定位当前污染代码和理解基础设施目标，不应该变成过细的实现方案。

## develop 阅读顺序

| 顺序 | 文档 | 主题 |
|---|---|---|
| 0 | `develop/develop0.md` | reset vector、早期异常、DEBUG OVMF |
| 1 | `develop/develop1.md` | debugcon、fw_cfg 早期定位 |
| 2 | `develop/develop2.md` | 最小 fw_cfg、string I/O workaround |
| 3 | `develop/develop3.md` | MTRR / MSR 最小虚拟化 |
| 4 | `develop/develop4-msr.md` | APIC_BASE、x2APIC、DXE/BDS 停点 |
| 5 | `develop/develop5.md` | CPUID guest 视图、xAPIC MMIO、AP startup |
| 6 | `develop/develop6-BDS.md` | BDS、virtio-blk、ESP、UEFI app 加载 |
| 7 | `develop/develop7-UEFI-ArceOS.md` | ArceOS UEFI app 和后续 runtime 接入边界 |

`develop/config-chain.md` 不属于阶段推进，但它对理解 OVMF 加载很重要。改 TOML 字段、镜像加载、vCPU 初始入口前，先读它。

## 当前边界

- 默认验证目标是 `smp1`。
- 当前 smoke 目标是 ArceOS UEFI helloworld，不是 Linux EFI。
- OVMF/EDK2 逻辑代码原则上不应为了 AxVisor 适配而修改。已有例外主要是 debug 输出和 fw_cfg 读取路径上的诊断性改动。
- 现在有些代码是为了最小跑通写的 workaround。整理到上游 PR 前，需要先分清它是基础设施雏形、临时透传、诊断日志，还是应该删除的污染代码。
- 上游合入标准不在这个 notebook 仓库里定义。这里负责记录事实、说明目标、给出定位和协作上下文。

## 维护约定

- 新推进的调试过程写进 `develop/`，按阶段追加，不要只写结论。
- 运行方式、镜像位置、预期日志发生变化时，同步更新 `start.md`。
- 面向团队认领的总结写进 `issue/`，保持高层，不替认领者决定具体实现。
- 总结当前实现时标注来源，例如 `develop4 Step 6`。
- 不把 `issue/ing.md` 当事实来源。
- 写文档时优先用当前代码路径和可复现日志，不写猜测成分。

## 工作区关系

默认工作区大致是：

```text
<workspace>/
├── axvisor-2026-notebooks/        本仓库
├── tgoskits/                      AxVisor / ArceOS 代码
└── edk2/                          OVMF / UEFI 固件代码
```

运行 AxVisor 时优先看 `start.md`。修改 AxVisor 代码前，先确认对应阶段记录里写的最新停点；如果日志已经变化，先更新判断，再动代码。
