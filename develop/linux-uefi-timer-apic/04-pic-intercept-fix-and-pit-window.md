# Linux UEFI timer/APIC 续查：VMX PIC 端口拦截已修通，当前阻塞前移到 PIT tick 与 8259 unmask 窗口错位

## 这轮先确认的根因

前一轮已经确认：

- `APIC_LVT0` 的 xAPIC MMIO 写被 APICv 直接写进了 virtual-APIC page
- `LINT0=0x700` 的 `ExtINT` decode 也已经修通

所以原来的“Linux 写了 LVT0，但 AxVisor 没观察到写”已经不是主阻塞。

继续往下看后，新的可疑点落到了 **PIC 端口 I/O 拦截**：

- `x86_vlapic::pic` 里已经有完整的 8259 设备
- `AxVmDevices` 也已经把 `0x20/0x21/0xa0/0xa1/0x4d0/0x4d1` 注册成了 port device
- 但 `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs` 里的 `VmxVcpu` 实际从 `IOBitmap::passthrough_all()` 起步
- `setup_io_bitmap()` 只显式拦了 PIT / COM1 / debugcon / fw_cfg / virtio-blk，没有把 PIC 端口加入 intercept 集

也就是说：

> 在 VMX 路径里，Linux 对 8259 PIC 的 `ICW`/`OCW1` 写原来根本不会形成 `IO_INSTRUCTION` VM-exit，自然也到不了 `x86_vlapic::pic`。

## 已做代码修改

### 1. 修正 VMX 默认 I/O 拦截集

文件：

- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`

改动：

- 抽出 `configure_default_io_intercepts(io_bitmap, config)`
- 在这份默认拦截集中补上：
  - `0x20/0x21`
  - `0xa0/0xa1`
  - `0x4d0/0x4d1`
- 顺手把 `setup_io_bitmap()` 里原来误导性的注释改掉：
  - 实际不是 “默认 intercept_all”
  - 而是从 `passthrough_all()` 出发，再选择性打开需要的 VM-exit

### 2. 为这条合同补最小单测

文件：

- `tgoskits/virtualization/x86_vcpu/src/vmx/structs.rs`
- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
- `tgoskits/virtualization/x86_vcpu/src/test_utils.rs`

改动：

- 给 `IOBitmap` 增加 `#[cfg(test)] is_intercepted(port)` 观察接口
- 新增单测：
  - `setup_io_bitmap_intercepts_legacy_pic_ports`
- 为了让 `x86_vcpu` 单测能链接通过，补了最小 `x86_vlapic::host::X86VlapicHostIf` mock
  - 注意：这是测试支撑，不是运行时语义改动

## 这轮验证结果

### 1. 单测先红后绿

先加测试后运行：

```bash
cargo test -p x86_vcpu setup_io_bitmap_intercepts_legacy_pic_ports -- --nocapture
```

先看到失败：

- `legacy PIC port 0x20 must be intercepted`

补完 VMX PIC intercept 后再跑，同一测试通过。

### 2. 目标编译通过

```bash
cargo check -p axvisor --target x86_64-unknown-none --features fs,vmx
```

通过。

### 3. Linux smoke 现场前进

新日志：

- `tmp/ovmf-linux-smoke-pic-ports.log`
- `tmp/ovmf-linux-smoke-pic-imr.log`

关键新证据：

1. PIC `ICW` 写现在已经能到 AxVisor：

- `[VPIC] master ICW1 init ...`
- `[VPIC] master ICW2 irq_base=0x30`
- `[VPIC] slave ICW1 init ...`
- `[VPIC] slave ICW2 irq_base=0x38`

这说明：

> “PIC 端口写没被截到” 这个阻塞已经被解决。

2. Linux 确实还会在 timer fallback 里短暂 unmask IRQ0：

- `[VPIC] master OCW1 imr=0xfe`

随后又会重新 mask：

- `[VPIC] master OCW1 imr=0xff`

这和 Linux `check_timer()` 里 `legacy_pic->unmask(0)` / `mask(0)` 的节奏是对得上的。

## 当前最新阻塞

虽然 PIC 端口 intercept 已修通，但 panic 还在。

现在最关键的新事实是：

1. 所有 `x86 PIT IRQ0 due ...` 日志都出现在更早阶段，而且和 OVMF DXE debugcon 日志交错
2. 当 Linux 真正进入 8259 / ExtINT fallback，开始做：
   - `ICW1/ICW2`
   - `OCW1 imr=0xfe`
   这些动作时，
   **我们已经看不到新的 `x86 PIT IRQ0 due`**
3. 也就是说，Linux 打开 IRQ0 观察窗口时：
   - `LINT0` 路由是 `ExtINT`
   - PIC 写路径也已经通了
   - 但 AxVisor 没有再给它一个可消费的 PIT tick

这轮后，主问题已经变成：

> PIT channel 0 在 Linux `check_timer()` 的 8259/ExtINT 探测窗口里，为什么没有再提供一个新的 IRQ0 tick？

注意这不是回到旧的 “PIT reset 不工作”：

- reset 语义仍然是修好的
- 这是一个**更晚阶段**的 PIT/轮询/重装/时序问题

## 当前最窄的下一步

不要回退 PIC intercept 修复，也不要再回到 APICv 根因讨论。

下一步只该继续回答下面两个精确问题：

1. `inject_due_x86_pit_irq0()` 在 Linux `check_timer()` 关键窗口里，是否还在被调用？
2. 如果被调用了，`x86_pit_consume_irq0_if_due(now_ns)` 为什么此时返回 false？
   - PIT 被 guest 改模式了？
   - deadline 被推远了？
   - channel0 被重新装载成了非周期 / 非可注入状态？
   - 还是 AxVM 的 poll 点没有覆盖到 Linux 的 `timer_irq_works()` 观察窗口？

下一轮调试应优先补：

- `PIT channel0 observation`
  - `mode`
  - `reload/divisor`
  - `period_ns`
  - `next_deadline_ns`
  - `now_ns`
- 观察点放在：
  - `inject_due_x86_pit_irq0()` 的 `not due` 分支
  - Linux 已经把 `LINT0` 切到 `ExtINT` 的窗口附近

## 6.19 会话末尾：`intack()` ISR stuck 根因

会话末尾已精确缩小到 PIC `intack()` 行为，以下是原始调用链分析（来自 6.19 codex 会话用户笔记）：

### 调用链

```
inject_due_x86_pit_irq0()  [vm.rs:290]
  → x86_pic_read_irq_vector()  [vm.rs:329]
    → intack()  [pic.rs:120-132]  ← ISR bit 0 SET, IRR bit 0 CLEARED
  → inject_interrupt_with_trigger(vector=0x30)
```

第一次 tick 正常：`intack()` 把 ISR bit 0 set，IRR bit 0 clear。

### 第二次 tick 时

```
get_irq()  [pic.rs:101-118]
  → priority(0) < current_priority(0) → FALSE → returns None
  → "no injectable route"（但被 log rate limit 吃掉了，日志看不到）
```

ISR bit 0 已经 set，`get_irq()` 里 priority 检查把后续所有 IRQ0 都拒绝了。

### 与真实硬件的根本区别

- **真实硬件**：CPU 只在实际 deliver 中断时才发 INTA cycle；如果 `IF=0` 或中断已在 service，不会 INTA，ISR 不会被 set
- **AxVisor 当前**：`intack()` 作为注入路径的 host-side 副作用**无条件执行**，ISR 被过早设置，且 guest 永远不会发 PIC EOI 来清除它

### 为什么日志看不到

`X86_LEGACY_IRQ_LOG_COUNT = 16`（vm.rs:71），16 条配额在 OVMF DXE 阶段就用完了。Linux `check_timer` 窗口里 PIT tick 实际仍在到达，但消息被 rate limit 吞掉，加上 ISR stuck 导致全部被拒绝。

### 为什么 `timer_irq_works()` 失败

`timer_irq_works()` 需要 5 个 jiffies（~4ms），但每个 phase 只能成功投递 1 次 IRQ0，之后 ISR stuck，远不够。

### 修复方向

`intack()` 不应在注入路径中无条件执行——应该延迟到 guest 实际 deliver 中断之后（比如 guest EOI 路径），或者用 AEOI 模式（Linux Phase 0 的 `init_8259A(1)` 设了 AEOI，但 Phase 4 的 `init_8259A(0)` 用的是 normal EOI）。

**注意**：这个方向还未在代码中落地，是 6.19 codex 会话的最终结论。

## 本轮代码修改清单

### x86_vlapic/vlapic.rs
- 新增 `Lint0Observation` 结构体和 `lint0_observation_from_values()` 纯函数
- `VirtualApicRegs::lint0_observation()` 改用 `lint0_observation_from_values()`
- `lint0_route()` 改用 `observation.virtual_page_route`
- 新增 `LVT_DELIVERY_MODE_FIXED = 0b000 << 8` / `LVT_DELIVERY_MODE_EXTINT = 0b111 << 8`
- `lint0_route_from_lvt()` 和 `write_lvt()` 改用上述 raw 常量（修复 tock-registers `.mask()` 误用）
- 新增单测：`lint0_observation_surfaces_virtual_page_shadow_split`

### x86_vlapic/lib.rs
- Re-export `Lint0Observation`
- `EmulatedLocalApic::lint0_observation()` 转发到 vlapic

### x86_vlapic/pic.rs
- 新增 `[VPIC]` 诊断日志（rate-limited）：
  - ICW1 init / ICW2 irq_base（`PIC_INIT_LOG_COUNT < 16`）
  - OCW1 imr 写（`PIC_MASK_LOG_COUNT < 32`）
  - `read_irq_vector()` 的 master/slave irq、base、irr、imr（`PIC_VECTOR_LOG_COUNT < 32`）

### x86_vcpu/vmx/vcpu.rs
- `lint0_observation()` 转发到 vlapic
- 新增 VMX exec/APICv 控制位 info 日志（`setup_vmcs` 末尾）
- `VIRT_APIC_ADDR` / `APIC_ACCESS_ADDR` 地址 info 日志
- `IA32_APIC_BASE` read/write 提升到 info
- x2APIC LVT0 write info 日志
- 抽出 `configure_default_io_intercepts()` helper
- 在默认 I/O 拦截集里补上 PIC 端口 `0x20/0x21/0xa0/0xa1/0x4d0/0x4d1`
- 修正误导性注释（实际是 `passthrough_all()`，不是 "默认 intercept_all"）
- 新增单测 `setup_io_bitmap_intercepts_legacy_pic_ports`（用 `IOBitmap` 纯 helper，绕过 `VmxVcpu::new()` 段错）

### x86_vcpu/vmx/structs.rs
- `IOBitmap` 新增 `#[cfg(test)] is_intercepted(port)` 观察接口

### x86_vcpu/test_utils.rs
- 新增 `MockVlapicHal`（实现 `X86VlapicHostIf`），与 `MockMmHal` 分开以避免 `ax_crate_interface` 宏符号冲突

### x86_vcpu/svm/vcpu.rs
- `lint0_observation()` 转发到 vlapic（SVM 对称实现）

### axvm/vm.rs
- `inject_due_x86_pit_irq0()` 在无路由时打印 `virtual_page` vs `shadow` LINT0 状态

## 6.20：参考实现审计 + 诊断日志

### 参考实现审计结论

对比 QEMU (`hw/intc/i8259.c`, `hw/intc/apic.c`) 和 KVM (`arch/x86/kvm/i8259.c`, `arch/x86/kvm/lapic.c`)：

1. **PIC intack 时机**：intack 发生在 host 侧读取 PIC vector 时（INTA cycle 模拟），ISR 在 non-AEOI 模式下被 set。这和 AxVisor 当前行为一致。

2. **ExtINT 路径 EOI**：guest 收到 ExtINT 中断后，发 LAPIC EOI（MMIO 写 offset 0xB0）清 LAPIC ISR，同时发 PIC EOI（OCW2 写 port 0x20）清 PIC ISR。两者独立，互不级联。

3. **AEOI 模式**：intack 时 ISR set 后立即 auto-clear（KVM）或根本不 set（QEMU）。后续不需要显式 EOI。

4. **AxVisor PIC OCW2 处理**：`pic.rs::command_write()` 已经正确处理 OCW2 command 1/3/5/7（清 ISR）。Guest 写 port 0x20 的 I/O exit 通过 `vm.rs` 的 `IoWrite` 分支正确到达 PIC 的 `handle_port_write()`。

### 之前 "ISR stuck" 结论的修正

之前的审计结论 "PIC ISR stuck after first LINT0-ExtINT intack" 可能不完全准确：
- AxVisor 的 OCW2 EOI 处理代码本身是正确的
- Guest 写 port 0x20 的 dispatch 路径也是正确的
- 真正需要验证的是：**Linux check_timer() 的 guest handler 是否真的发了 PIC EOI (OCW2 写 port 0x20)**

### 需要验证的三个问题

1. **check_timer() 走到了哪个 phase？** — 如果 IOAPIC GSI2 在 phase 1/2 就成功了，PIC ISR 根本不会被 set
2. **Phase 4 的 guest PIC EOI 是否到达了 AxVisor？** — 需要新的 EOI 日志
3. **PIT tick 在 Linux 窗口里是否持续？** — 需要更高 log limit

### 新增诊断代码

改动都是纯诊断，不改变任何行为：

#### axvm/vm.rs
- `X86_LEGACY_IRQ_LOG_LIMIT`: 16 → 64，让 Linux check_timer 窗口里的消息不被 rate limit 吞掉
- `inject_due_x86_pit_irq0()`: 无路由时额外打印 IOAPIC GSI2 vector 和 PIC master_isr
- `inject_due_x86_pic_lint0_irq()`: 打印 intack 前后的 PIC master ISR 值（`isr_before` / `isr_after`）

#### x86_vlapic/pic.rs
- 新增 `PIC_EOI_LOG_COUNT`（上限 32）
- `command_write()` 中 OCW2 command 1/3/5/7 打印 EOI 日志：`[VPIC] master/slave OCW2 EOI cmd=X irq=Y isr: before -> after`
- 新增 `master_isr()` 诊断方法

#### axdevice/device.rs
- 新增 `x86_pic_master_isr()` 方法

### 下一步

跑一次 Linux smoke，看新日志里：
1. IOAPIC GSI2 vector 是否有值（如果有，说明走的是 IOAPIC 路径不是 LINT0 路径）
2. `[VPIC] master OCW2 EOI` 是否出现（如果不出现，说明 guest handler 没发 PIC EOI）
3. `isr_before` / `isr_after` 的值变化（确认 ISR 是否真的 stuck）

## 6.20 诊断结果 + ExtINT 修复 + 新 blocker

### 诊断结果

跑了多轮 smoke，逐步去掉 rate limit，精确定位了阻塞链：

**LINT0 路径确认通了但 ISR stuck 确实存在：**

```
17.930s: read_irq_vector: master_irq=Some(0) master_imr=0xfe master_isr=0x0  ← 第1次注入，intack set ISR
17.985s: read_irq_vector: master_irq=Some(0) master_imr=0xfe master_isr=0x0  ← 第2次注入（guest handler 中间发了 EOI 清了 ISR）
18.040s: read_irq_vector: master_irq=None  master_imr=0xfe master_isr=0x1  ← ISR stuck，get_irq() 被 priority 拒绝
18.058s: OCW2 specific-EOI irq=0 isr: 0x1 -> 0x0  ← guest 发了 PIC EOI 但太晚了
18.xxx:  Kernel panic - not syncing: IO-APIC + timer doesn't work!
```

问题链条：
1. `intack()` 在注入路径里 set ISR bit 0
2. Guest handler 在下一次 tick 到来前发了 PIC EOI（第1→2次之间）
3. 但第2次 `intack()` 又 set ISR，第3次 tick 到来时 ISR stuck
4. Guest 的 EOI 来了但太晚，`timer_irq_works()` 已经 timeout

**与真实硬件的区别：** 真实硬件 ExtINT 路径中，APIC 透明处理 INTA cycle，PIC ISR 根本不会被 set。AxVisor 的 `intack()` 是 host-side 副作用，在注入前就无条件执行了。

### 修复：read_irq_vector_extint()

**核心思路：** ExtINT 路径读 PIC vector 时只清 IRR，不 set ISR。匹配真实硬件行为。

#### x86_vlapic/pic.rs
- 新增 `PicState::read_irq_vector_extint()`: 和 `read_irq_vector()` 逻辑一样，但用 `self.master.irr &= !(1 << irq)` 代替 `self.master.intack(irq)`。不清 ISR。
- 新增 `EmulatedPic8259::read_irq_vector_extint()`: 公开方法

#### axdevice/device.rs
- 新增 `AxVmDevices::x86_pic_read_irq_vector_extint()`

#### axvm/vm.rs
- `inject_due_x86_pic_lint0_irq()` 的 ExtInt 分支改为调用 `x86_pic_read_irq_vector_extint()` 而不是 `x86_pic_read_irq_vector()`
- Fixed 分支保持用 `x86_pic_read_irq_vector()`（Fixed 模式需要正常的 intack）

### 修复后 smoke 结果

ISR 不再 stuck，guest handler 正常运行：

```
17.622s: injected via LINT0 extint: vector=0x30 isr=0x0  ← 第1次注入成功
17.677s: injected via LINT0 extint: vector=0x30 isr=0x0  ← 第2次注入成功
17.689s: OCW2 specific-EOI irq=0 isr: 0x0 -> 0x0         ← guest EOI，ISR 已经是 0
17.xxx:  Kernel panic - not syncing: IO-APIC + timer doesn't work!
```

### 当前新 blocker：只注入了 2 个 tick

前2个 tick 成功注入，guest handler 跑了、EOI 发了，ISR 不再 stuck。但第3个 tick 没有到达 guest。`timer_irq_works()` 需要 5 个 jiffies，只拿到 2 个。

诊断日志显示：
- `inject_due_x86_pit_irq0()` 确实在被调用
- `consume_irq0_if_due()` 返回 true
- `inject_interrupt_with_trigger(vector=0x30)` 被调了
- 但 guest 没有收到后续中断

**read_irq_vector_extint 日志：** 只在 17.62s 和 17.68s 出现了2次 Some(0)。之后再也没有被调用。说明 `inject_due_x86_pit_irq0()` 在第2次注入后没有再走到 LINT0 路径。

**INJECT_LOG 每200 tick 采样：**
```
3.17s:  gsi2=None lint0=None      pic_isr=0x0  ← OVMF DXE 阶段，LINT0 还没配
14.11s: gsi2=None lint0=Some(ExtInt) pic_isr=0x0  ← LINT0 已配好 ExtInt
25.10s: gsi2=None lint0=None      pic_isr=0x0  ← panic 之后
```

### 当前最可能的原因

`inject_pending_events()` (`x86_vcpu/src/vmx/vcpu.rs:1148`) 走传统 VMCS injection：
- 如果 `allow_interrupt()` 返回 false（guest IF=0），走 interrupt-window exiting 路径
- 但 AxVisor 同时开了 APICv virtual-interrupt delivery（`VIRTUAL_INTERRUPT_DELIVERY`）
- 这两个机制可能冲突：CPU 看到 VMCS 注入字段的同时，virtual-APIC page 上也有状态
- 或者 interrupt-window exit 本身没有正确触发

### 下一步方向

1. 在 `inject_pending_events()` 加日志：记录 vector、`allow_interrupt()` 返回值、是否走 interrupt-window exiting
2. 检查 VMX preemption timer exit 是否在 guest handler 返回后正常触发
3. 考虑绕过 VMCS injection，直接用 APICv PIR（Posted Interrupt Request）注入

### 其他改动记录

#### x86_vlapic/pic.rs 诊断日志（可后续清理）
- `read_irq_vector()`: 无 rate limit 的 `[VPIC] read_irq_vector: master_irq=... master_irr=... master_imr=... master_isr=...`
- `read_irq_vector_extint()`: 无 rate limit 的 `[VPIC] read_irq_vector_extint: master_irq=... master_irr=... master_imr=... master_isr=...`
- `command_write()` OCW2: `[VPIC] master/slave OCW2 EOI cmd=X irq=Y isr: before -> after`（PIC_EOI_LOG_COUNT 上限 32）
- `data_write()` OCW1: 无 rate limit 的 `[VPIC] master/slave OCW1 imr=...`（去掉了 PIC_MASK_LOG_COUNT 限制）

#### axvm/vm.rs 诊断日志（可后续清理）
- `inject_due_x86_pit_irq0()`: PIT_TICK_LOG 每200 tick 打一次 `tick due at ...ns`
- `inject_due_x86_pit_irq0()`: INJECT_LOG 每200 tick 打一次 `gsi2=... lint0=... pic_isr=...`
- `inject_due_x86_pic_lint0_irq()` ExtInt: 无 rate limit 的 `injected via LINT0 extint: vector=... isr=...`

