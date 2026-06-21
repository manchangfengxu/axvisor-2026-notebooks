# Linux UEFI timer/APIC 阶段 handoff

## 一句话状态

这条验证链已经越过：

- `OVMF -> legacy virtio-blk -> BOOTX64.EFI`
- `EFI stub: Failed to decompress kernel`
- 第一阶段 Linux early `#PF`

当前主阻塞已经收敛成一个更具体的矛盾：

> Linux 在 timer fallback 里确实写了 `APIC_LVT0`，但 AxVisor 没观察到对应的 `LINT0` 写路径；因此 `PIT IRQ0` 到期后，`GSI2` 和 `LINT0` 两条注入路都没真正建立起来。

## 这一步已经落实的推进

### 1. PIT reset 语义已修

对照 `references/qemu/hw/timer/i8254_common.c::pit_reset_common()` 后，已经把
`tgoskits/virtualization/x86_vlapic/src/pit.rs` 的 reset 语义补成 PC 兼容状态：

- channel 0 reset 后立即运行
- mode 3
- divisor `0x10000`

这一步不是猜测，已经有新单测和新 smoke 日志支撑。

### 2. 当前 panic 不再是“PIT 根本没动”

最新日志已经能稳定看到：

- `x86 PIT IRQ0 due ...`
- `vIOAPIC GSI2 is still masked while the guest waits for IRQ0`

所以当前不是：

- `PIT` 不工作
- 没有 poll 点
- 完全没有回到 VMM

而是 **guest timer route 没立起来**。

### 3. 窄版 LINT0 fallback 已经补上，但也没有路

这一阶段补了很窄的 `LINT0` 语义解码和 fallback：

- `Fixed { vector }`
- `ExtINT`
- `masked -> None`

然后在 `axvm` 里让 `PIT IRQ0` 在 `GSI2` 无路时，进一步尝试 `LAPIC LINT0`。

结果日志明确表明：

- `x86 PIT IRQ0 due but neither guest GSI2 nor LAPIC LINT0 had an injectable route`

也就是说，这不是“只差一条 GSI2 glue”的问题。

## 当前最重要的事实链

### 事实 A：Linux 确实写了 `APIC_LVT0`

`axvisor-2026-notebooks/subagent_watch/linux-lvt0-write-path-audit.md`
已经核死这条链。

Linux `check_timer()` 会沿着：

- 8259A through IO-APIC
- Virtual Wire IRQ
- ExtINT IRQ

依次尝试，并对 `APIC_LVT0` 做 6 次写入。关键值包括：

- `0x00030`：Fixed, unmasked, vector `0x30`
- `0x00700`：ExtINT, unmasked

这些写不是 x2APIC MSR，而是：

- **xAPIC MMIO 写 `0xfee00350`**

### 事实 B：AxVisor 当前完全没看到这些写

本地已经在 `write_lvt(LvtLint0)` 路径加了日志：

- `"[VLAPIC] LINT0 write: raw=... route=..."`

但最新 smoke 日志里这条日志一次都没有。

同时 `vmx-apic-write-path-contract.md` 也指出：

- 没有任何 `APIC_ACCESS` 的 `LinearDataWrite` 命中 `LVT0`
- 唯一一次 `APIC_ACCESS` 出现在 panic 之后，而且是读，不是写

### 事实 C：ExtINT 最小语义不是当前第一阻塞点

`virtual-wire-extint-contract-matrix.md` 已经对照过：

- ExtINT 需要 PIC ack + vector from PIC
- Fixed Virtual Wire 才是直接拿 LVT 向量

但 AxVisor 当前这部分并不是空白：

- `PIC read_irq_vector()` 已经带 ack
- `ExtINT` fallback 已经从 PIC 读 vector

所以现在最先要解决的，不是“再讲一遍 ExtINT 语义”，而是：

- **为什么 Linux 已经写了 `LVT0`，AxVisor 却完全没观察到**

## 当前代码面已经落地的东西

### `tgoskits/virtualization/x86_vlapic/src/pit.rs`

- `PitState::new(now_ns)`
- `EmulatedPit::new()`
- `channel0_is_live_after_reset()`

作用：

- 修 PIT channel 0 reset 语义
- 加回归测试

### `tgoskits/virtualization/x86_vlapic/src/vlapic.rs`

- `LegacyPicLint0Route`
- `lint0_route_from_lvt()`
- `VirtualApicRegs::lint0_route()`
- `"[VLAPIC] LINT0 write: ..."` 日志

作用：

- 让上层能读当前 `LINT0` 路由语义
- 真写到 `LINT0` 时可以直接打点

### `tgoskits/virtualization/x86_vlapic/src/lib.rs`

- `pub use vlapic::LegacyPicLint0Route`
- `EmulatedLocalApic::lint0_route()`

### `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
### `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs`

- `lint0_route()`

作用：

- 让 `axvm` 可以查询当前 vCPU 的 `LINT0` 语义，而不需要额外引入一层新框架

### `tgoskits/virtualization/axdevice/src/device.rs`

- `x86_pic_read_irq_vector()`

作用：

- 给 `ExtINT` fallback 提供 PIC vector 读取入口

### `tgoskits/virtualization/axvm/src/vm.rs`

- `inject_due_x86_pic_lint0_irq()`
- `inject_due_x86_pit_irq0()` fallback 调整

作用：

- `GSI2` 无路时，再探一次 `LINT0`

## 已验证的命令和结果

### 单测 / 构建

跑过：

```bash
cargo test -p x86_vlapic channel0_is_live_after_reset -- --nocapture
cargo test -p x86_vlapic lint0_route_reports_fixed_vector -- --nocapture
cargo fmt --all
cargo check -p axvisor --target x86_64-unknown-none --features fs,vmx
cargo +nightly-2026-04-27 check -p axvm --target x86_64-unknown-linux-musl --features plat-dyn
```

结果：

- 新增回归测试通过
- `fmt` 通过
- `axvisor fs+vmx` 构建通过
- `axvm x86_64-unknown-linux-musl + plat-dyn` 构建通过

### Linux UEFI smoke

命令：

```bash
cd /home/ssdns/work/axvisor-uefi/tgoskits/os/axvisor
timeout 80 cargo xtask qemu \
  --config /home/ssdns/work/axvisor-uefi/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml \
  --qemu-config /home/ssdns/work/axvisor-uefi/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml \
  --vmconfigs /home/ssdns/work/axvisor-uefi/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-linux-smoke.toml \
  --rootfs /home/ssdns/work/axvisor-uefi/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img
```

当前高信号日志：

- `tmp/ovmf-linux-smoke-pit-reset.log`
- `tmp/ovmf-linux-smoke-pit-lint0.log`
- `tmp/ovmf-linux-smoke-lint0-write.log`

关键观察：

- `x86 PIT IRQ0 due ...`
- `x86 PIT IRQ0 due but neither guest GSI2 nor LAPIC LINT0 had an injectable route`
- 没有任何 `"[VLAPIC] LINT0 write"` 日志
- Linux 最终仍然 panic：`IO-APIC + timer doesn't work!`

## 切新上下文后先读什么

建议顺序：

1. `CLAUDE.md`
2. `axvisor-2026-notebooks/develop/linux-uefi-timer-apic/00-pit-reset-and-lint0-status.md`
3. `axvisor-2026-notebooks/develop/linux-uefi-timer-apic/01-apic-lvt0-write-gap.md`
4. `axvisor-2026-notebooks/subagent_watch/linux-lvt0-write-path-audit.md`
5. `axvisor-2026-notebooks/subagent_watch/vmx-apic-write-path-contract.md`
6. `axvisor-2026-notebooks/subagent_watch/virtual-wire-extint-contract-matrix.md`
7. 当前文件

## 下一个窄目标

只做这件事：

- 核死 **VMX APIC-access / virtual-APIC / APIC-access-page / EPT** 的精确合同

不要回头做这些：

- 再改 PIT
- 重新泛化成“中断框架不完整”
- 提前展开 modern virtio / virtio-pci
- 做 direction-2 本地替代框架

当前真正要回答的是：

> Linux 对 `0xfee00350` 的 xAPIC MMIO 写，为什么没有落到我们当前可观察的 `LINT0` 写路径？

## 如果需要外部廉价模型，大题目只问这三类

### Prompt A: Intel 合同核查

审计 Intel SDM 中 VMX local APIC virtualization 的精确合同，目标是 guest 对
`0xfee00350` (`APIC_LVT0`) 的 xAPIC MMIO 写。

要求：

- 只引用 Intel SDM 原文，不要引用博客或二手文章
- 明确区分这些控制位/机制：
  - `virtualize APIC accesses`
  - `use TPR shadow`
  - `virtual-interrupt delivery`
  - `APIC-access page`
  - `virtual-APIC page`
  - `APIC-register virtualization`
- 回答这几个问题：
  1. `LVT0` 写是否应该触发 `APIC_ACCESS` VM-exit？
  2. 是否可能触发 `APIC_WRITE` VM-exit？
  3. 是否可能在当前控制位组合下被硬件“静默处理”而不经过软件可见写路径？
  4. APIC-access page 的 EPT 映射和这个行为是什么关系？
- 输出必须带卷号、章节号、页码或精确标题
- 不做设计建议，只给合同事实和结论

### Prompt B: AxVisor 本地实现核查

只审计当前仓库里 AxVisor 的 VMX APIC 相关实现，不做新设计。

目标：

- 判断 guest 对 `0xfee00350` 的写，在当前实现里“理论上应该走哪条路径”
- 判断现有 VMCS / EPT / APIC page 设置有没有让这次写落到别处的风险

必须阅读：

- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
- `tgoskits/virtualization/x86_vlapic/src/lib.rs`
- `tgoskits/virtualization/x86_vlapic/src/vlapic.rs`
- `tgoskits/virtualization/axvm/src/vm.rs`

输出要求：

- 列出相关 VMCS 控制位、VMCS 字段、物理页、EPT 映射
- 说明每条路径对应的 exit 类型和代码落点
- 说明哪种组合会导致“guest 写了 LVT0，但当前日志里看不到 `LINT0 write`”
- 不要泛化成新的中断框架建议

### Prompt C: 成熟实现对照

对照成熟实现，只回答这个窄问题：

- 当 guest 以 xAPIC MMIO 方式写 `APIC_LVT0` 时，成熟实现如何观测或故意不观测这次写？

限制：

- 优先看 `references/qemu/`、`references/cloud-hypervisor/`
- 如需看 KVM，优先官方内核代码或文档
- 不接受博客总结

输出要求：

- 分清“软件显式处理”和“交给硬件/KVM 内核处理”
- 给出具体文件、函数、行号
- 最终只回答哪些路径会让 VMM 自己看到 `LINT0` 写，哪些路径不会
