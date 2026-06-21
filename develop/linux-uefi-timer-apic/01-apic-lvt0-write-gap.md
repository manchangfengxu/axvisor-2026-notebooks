# Linux UEFI timer/APIC 续篇：Linux 已写 LVT0，但 AxVisor 没观察到

## 这一步新增了什么结论

上一阶段已经确认：

- `PIT reset` 缺失已修；
- `PIT IRQ0` 已经会自然到期；
- 但 Linux 仍然 panic 在 `IO-APIC + timer doesn't work!`。

这一轮进一步把问题收敛到一个更尖锐的矛盾：

1. **Linux 源码和阶段日志都表明 guest 确实写了 `APIC_LVT0`。**
2. **AxVisor 的 VMX/APIC 观测日志里却完全看不到任何 `LINT0 write`。**

所以当前重点已经不是：

- PIT 是否工作；
- PIC ack 是否实现；
- ExtINT 语义是否理解错；

而是：

**guest 对 `0xfee00350` 的 xAPIC MMIO 写为什么没有按预期落到 AxVisor 当前的 vLAPIC 可观察路径里。**

## 本轮使用的三份定点调研

这一步的结论来自三份 `subagent_watch` 文档：

- `axvisor-2026-notebooks/subagent_watch/linux-lvt0-write-path-audit.md`
- `axvisor-2026-notebooks/subagent_watch/vmx-apic-write-path-contract.md`
- `axvisor-2026-notebooks/subagent_watch/virtual-wire-extint-contract-matrix.md`

它们不是最终结论本身，但足够作为当前阶段的交叉证据。

## 关键事实 1：Linux 确实写了 APIC_LVT0

`linux-lvt0-write-path-audit.md` 已经把 Linux 7.0/7.1 这条 timer fallback 路径核死了。

当前本地 Linux 源码树是：

- `/home/ssdns/work/axvisor-uefi/linux`

关键结论：

- `check_timer()` 会按顺序走：
  - 8259A through IO-APIC
  - Virtual Wire IRQ
  - ExtINT IRQ
- 在这几个阶段里，Linux 会对 `APIC_LVT0` 做 6 次写入。

按那份审计里的整理，这 6 次写分别是：

1. `0x10000`：masked
2. `0x00700`：ExtINT, unmasked
3. `0x10700`：ExtINT, masked
4. `0x00030`：Fixed, vector=0x30, unmasked
5. `0x10030`：Fixed, masked
6. `0x00700`：ExtINT, unmasked

而且这些写都不是 x2APIC MSR，而是：

- **xAPIC MMIO 写到 `0xfee00350`**

这条结论的意义很大：

- 现在不能再说 “可能 guest 根本没写 LVT0”。

## 关键事实 2：AxVisor 当前日志里完全看不到这些写

本轮最新的 smoke 日志是：

- `tmp/ovmf-linux-smoke-lint0-write.log`

我已经把 `x86_vlapic::vlapic::write_lvt(LvtLint0)` 路径加了明确日志：

- `"[VLAPIC] LINT0 write: raw=... route=..."`

但实际结果是：

- 这条日志 **一次都没有出现**。

同时，`vmx-apic-write-path-contract.md` 也指出：

- 整个日志里没有任何 `APIC_ACCESS` 的 `LinearDataWrite` 退出对应到 `LVT0`
- 只有 panic 之后出现过一次 `APIC_ACCESS` 读退出，而且 offset 不是 `0x350`

这说明：

- AxVisor 当前并没有从 VMX APIC 访问路径里观察到 Linux 的那 6 次 `LVT0` 写。

## 关键事实 3：ExtINT 语义本身不是当前第一阻塞点

`virtual-wire-extint-contract-matrix.md` 对照了 QEMU / Cloud Hypervisor / AxVisor。

它给出的高价值结论是：

- **ExtINT 模式不能只注入 vector。**
- 正确的 ExtINT 最小语义包括：
  - vector 来自 PIC
  - PIC ack（ISR/IRR 状态变化）
  - spurious 处理

而 AxVisor 当前这部分并不是完全空白：

- `PIC read_irq_vector()` 已经能做 ack
- `ExtINT` fallback 也已经会从 PIC 读 vector

所以当前最先需要搞定的不是 “重新发明 ExtINT 语义”，而是：

- **为什么 Linux 已经写了 `LVT0`，但 AxVisor 这边完全没看到。**

## 当前最合理的工作假设

当前最值得沿着往下挖的假设是：

> Linux 对 `0xfee00350` 的 xAPIC MMIO 写已经发生，但这几次写没有经过我们预期的
> `APIC_ACCESS exit -> EmulatedLocalApic::handle_write() -> write_lvt(LvtLint0)` 路径。

这并不自动等于“VMX APIC virtualization 整体坏了”。

更细一点，当前可能性分两类：

### 类 A：这类写在当前 VMX 控制位组合下本来就不一定产生我们期待的软件可见路径

这需要继续对照 Intel 手册确认：

- `virtualize APIC accesses`
- `virtual-APIC page`
- `APIC-access page`
- `APIC-register virtualization`
- `virtual-interrupt delivery`

之间的精确合同。

### 类 B：AxVisor 当前 APIC-access / EPT / VMCS 配置让这几次写落到了“别的地方”

例如：

- APIC-access page
- virtual-APIC page
- 普通 GPA 映射

三者之间的地址和语义关系可能没有按我们目前的预期协同工作。

## 这一轮还新增了什么代码

除了上一阶段的 `PIT reset` 修复，这一轮又补了这些窄改动：

### 1. `tgoskits/virtualization/x86_vlapic/src/vlapic.rs`

- `LegacyPicLint0Route`
- `lint0_route_from_lvt()`
- `VirtualApicRegs::lint0_route()`
- `LVT0` 写日志

作用：

- 既能让上层查询当前 `LINT0` 语义；
- 又能在 guest 真正写到 `LINT0` 时直接出日志。

### 2. `tgoskits/virtualization/x86_vlapic/src/lib.rs`

- `EmulatedLocalApic::lint0_route()`

作用：

- 把 `LINT0` 路由状态以很窄的方式暴露给 VMX/SVM 层。

### 3. `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
### 4. `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs`

- `lint0_route()`

作用：

- 让 `AxVM` 侧可以查询当前 vCPU 的 `LINT0` 状态。

### 5. `tgoskits/virtualization/axdevice/src/device.rs`

- `x86_pic_read_irq_vector()`

作用：

- 给 `ExtINT` fallback 提供 PIC vector 读取入口。

### 6. `tgoskits/virtualization/axvm/src/vm.rs`

- `inject_due_x86_pic_lint0_irq()`
- `inject_due_x86_pit_irq0()` fallback 调整

作用：

- 在 `GSI2` 没路时，再探一次 `LAPIC LINT0` 的 `Fixed / ExtINT` 路由。

结果：

- 当前日志已经能明确区分：
  - `GSI2` 没路
  - `LINT0` 也没路

## 当前可复现的现场

当前最新可复现日志：

- `tmp/ovmf-linux-smoke-pit-reset.log`
- `tmp/ovmf-linux-smoke-pit-lint0.log`
- `tmp/ovmf-linux-smoke-lint0-write.log`

它们共同表明：

1. `PIT IRQ0` 已到期；
2. `GSI2` 一直 masked；
3. `LINT0` 从未进入可注入态；
4. `LINT0 write` 日志完全不存在；
5. Linux 最终 panic 仍停在 `IO-APIC + timer doesn't work!`

## 切换上下文后该先读什么

如果从新窗口接手，不要从头看所有对话。

建议读这个顺序：

1. `CLAUDE.md`
2. `axvisor-2026-notebooks/develop/linux-uefi-timer-apic/00-pit-reset-and-lint0-status.md`
3. `axvisor-2026-notebooks/develop/linux-uefi-timer-apic/01-apic-lvt0-write-gap.md`
4. `axvisor-2026-notebooks/subagent_watch/linux-lvt0-write-path-audit.md`
5. `axvisor-2026-notebooks/subagent_watch/vmx-apic-write-path-contract.md`
6. `axvisor-2026-notebooks/subagent_watch/virtual-wire-extint-contract-matrix.md`

## 下一步建议

下一步优先级已经很明确：

1. **不要再回头改 PIT。**
2. **不要再先讲 ExtINT 大语义。**
3. 先核死两件事：
   - Intel VMX 对 xAPIC MMIO 写 `LVT0` 的精确硬件合同
   - AxVisor 当前 VMCS / APIC-access page / virtual-APIC page / EPT 映射是否满足这个合同

当前真正的问题已经从“timer 不动”推进到了：

**Linux 写了 `APIC_LVT0`，但 AxVisor 没观察到这次写。**
