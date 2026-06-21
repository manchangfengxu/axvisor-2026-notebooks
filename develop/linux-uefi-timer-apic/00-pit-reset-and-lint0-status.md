# Linux UEFI timer/APIC 现状：PIT reset 已修，当前卡在 guest APIC 路由建立

## 这一步做了什么

这轮不是在原地打转，已经把问题往前推了两层：

1. 先确认 Linux 的早期 `IO-APIC + timer` panic 不是因为“我们完全收不到 VM-exit”。
2. 再确认不是“IRQ0 注入代码没写”，而是 **PIT channel 0 在 reset 后根本没有运行**。
3. 修掉 PIT reset 之后，Linux 的失败点继续前移，当前收敛到：
   - `PIT IRQ0` 已经到期；
   - 但 guest 直到 panic 都没有建立可用的 timer 注入路由；
   - 具体表现是 `vIOAPIC GSI2` 一直 masked，同时 `LAPIC LINT0` 也没有进入可注入状态。

换句话说，当前不再是“没有 timer tick”，而是“guest timer route 没立起来”。

## 关键证据链

### 1. 先前确认：VMX preemption timer 在工作

在 `tmp/ovmf-linux-smoke-io-poll-diag.log` 和 `tmp/ovmf-linux-smoke-pit-arm.log` 里都能稳定看到：

- `VM[1] VCpu[0] left guest on VMX preemption timer`

这说明 guest 运行期间我们是会周期性回到 VMM 的，问题不在“永远回不来所以没机会补 IRQ”。

### 2. 修复前：guest 从未显式编程 PIT channel 0

先前加过 PIT 诊断：

- `x86 PIT channel0 command`
- `x86 PIT channel0 armed`

在 `tmp/ovmf-linux-smoke-pit-arm.log` 里，这两类日志完全没有出现。

而对照 `references/qemu/hw/timer/i8254_common.c` 的 `pit_reset_common()` 可以看到，QEMU 在 reset 时就让 channel 0 进入运行态：

- `mode = 3`
- `gate = 1`
- `count = 0x10000`
- `next_transition_time = pit_get_next_transition_time(...)`

我们的 `tgoskits/virtualization/x86_vlapic/src/pit.rs` 原先没有这层 reset 语义，channel 0 只有在 guest 后续写端口时才会开始跑。

### 3. 修复后：PIT IRQ0 已经开始到期

修完 PIT reset 之后，新的 smoke 日志 `tmp/ovmf-linux-smoke-pit-reset.log` 明确出现了：

- `x86 PIT IRQ0 due but guest GSI2 still has no injectable route`
- `vIOAPIC GSI2 is still masked while the guest waits for IRQ0`

这说明：

1. `PIT channel 0` 已经在 reset 后自然开始工作。
2. `consume_irq0_if_due()` 已经真的触发。
3. 当前剩下的问题不在 PIT 本身，而在 guest timer route。

### 4. 当前进一步确认：不仅 GSI2 没路，LINT0 也没路

我在 AxVM 侧又补了一层非常窄的 fallback：

- 如果 `IOAPIC GSI2` 没有可注入路由，就检查当前 `LAPIC LINT0` 是否被 guest 配成：
  - `Fixed`
  - 或 `ExtINT`

同时在 `x86_vlapic` 里补了一个纯逻辑解码：

- `masked` -> `None`
- `Fixed + vector` -> `Some(Fixed { vector })`
- `ExtINT` -> `Some(ExtInt)`

然后重新跑 smoke，新的日志 `tmp/ovmf-linux-smoke-pit-lint0.log` 表明：

- `x86 PIT IRQ0 due but neither guest GSI2 nor LAPIC LINT0 had an injectable route`

对应的 guest 内部日志仍然停在：

- `...trying to set up timer (IRQ0) through the 8259A ...`
- `...trying to set up timer as Virtual Wire IRQ...`
- `...trying to set up timer as ExtINT IRQ...`
- `Kernel panic - not syncing: IO-APIC + timer doesn't work!`

所以这一步把问题继续缩小了：

- 现在不是 `PIT`；
- 不是 “没轮询到”；
- 也不是 “只差一条 GSI2 注入 glue”；
- 更可能是 **guest APIC 写路径 / LAPIC LINT0 建立过程没有真正落到我们当前可观察的状态上**。

## 代码变更

### 1. `tgoskits/virtualization/x86_vlapic/src/pit.rs`

- `PitState::new(now_ns)`
- `EmulatedPit::new()`
- `channel0_is_live_after_reset()`

作用：

- 把 channel 0 的 reset 语义对齐到 PC 兼容行为：
  - mode 3
  - divisor 0x10000
  - reset 后立即开始计时
- 加了最小回归测试，确认 reset 后就已经装载并运行。

### 2. `tgoskits/virtualization/x86_vlapic/src/vlapic.rs`

- `LegacyPicLint0Route`
- `lint0_route_from_lvt()`
- `VirtualApicRegs::lint0_route()`
- `lint0_route_reports_fixed_vector()`
- `lint0_route_reports_extint_mode()`
- `lint0_route_ignores_masked_entries()`

作用：

- 纯逻辑解码 `LAPIC LINT0` 的 guest 可见路由状态。
- 这一步不建新框架，只给现有 x86 本地 APIC 路径补一个可检查的状态接口。

### 3. `tgoskits/virtualization/x86_vlapic/src/lib.rs`

- `pub use vlapic::LegacyPicLint0Route`
- `EmulatedLocalApic::lint0_route()`

作用：

- 把 `LINT0` 路由状态以很窄的方式暴露给上层。

### 4. `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
### 5. `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs`

- `lint0_route()`

作用：

- 让 `axvm` 在 x86 路径上可以查询当前 vCPU 的 `LAPIC LINT0` 状态，而不用下沉成新的中断框架。

### 6. `tgoskits/virtualization/axdevice/src/device.rs`

- `AxVmDevices::x86_pic_read_irq_vector()`

作用：

- 提供 PIC 当前待处理中断向量的窄接口，供 `ExtINT` 路径读取和确认。

### 7. `tgoskits/virtualization/axvm/src/vm.rs`

- `inject_due_x86_pic_lint0_irq()`
- `inject_due_x86_pit_irq0()` fallback 调整

作用：

- 在 `IOAPIC GSI2` 没路时，尝试走 `LAPIC LINT0` 的 `Fixed` / `ExtINT` fallback。
- 当前 smoke 结果证明：这条 fallback 代码已经在跑，但 guest 还没有把 `LINT0` 置到可用状态。

## 验证

### 单测 / 构建

跑过这些：

```bash
cargo test -p x86_vlapic channel0_is_live_after_reset -- --nocapture
cargo test -p x86_vlapic lint0_route_reports_fixed_vector -- --nocapture
cargo fmt --all
cargo check -p axvisor --target x86_64-unknown-none --features fs,vmx
cargo +nightly-2026-04-27 check -p axvm --target x86_64-unknown-linux-musl --features plat-dyn
```

结果：

- 两个新加的回归测试通过；
- `fmt` 通过；
- `axvisor fs+vmx` 构建通过；
- `axvm x86_64-unknown-linux-musl + plat-dyn` 构建通过。

### Linux UEFI smoke

命令：

```bash
cd "$WORKSPACE/tgoskits/os/axvisor"
timeout 80 cargo xtask qemu \
  --config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml" \
  --qemu-config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml" \
  --vmconfigs "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-linux-smoke.toml" \
  --rootfs "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"
```

日志：

- `tmp/ovmf-linux-smoke-pit-reset.log`
- `tmp/ovmf-linux-smoke-pit-lint0.log`

结果摘要：

- 修 PIT reset 后，日志已经出现 `x86 PIT IRQ0 due ...`
- 但 Linux 仍然 panic 在 `IO-APIC + timer doesn't work!`
- 最新状态显示：
  - `GSI2` 一直 masked
  - `LINT0` 也没有进入 `Fixed` / `ExtINT` 可注入态

## 当前结论

当前最准确的结论是：

1. **PIT reset 语义缺失** 这个问题已经解决。
2. 现在的主阻塞点已经前移到 **guest APIC timer route 建立**。
3. 更具体地说，问题很可能在下面两类之一：
   - guest 对 `LAPIC LVT0` 的写入没有真正落到我们当前观察的本地 APIC 状态里；
   - 或者 guest 根本没有成功走到应有的 `LINT0` 建立路径，原因在 APIC access / APIC virtualization / local APIC state 同步这条链上。

## 下一步

下一轮不要再碰 PIT，也不要回头泛化成 “中断框架还不完整”。

更直接的动作是：

1. 在 `LVT0` 写路径上补针对性日志，确认 guest 到底有没有真正写 `APIC_LVT0`。
2. 如果没有看到写入，就继续沿着：
   - APIC access exit
   - xAPIC / x2APIC 选择
   - virtual APIC page / APIC-access page
   这条线追 guest 本地 APIC 写路径。
3. 如果看到了写入，但状态读不出来，再检查本地 APIC shadow 与 guest 可见寄存器之间是否脱节。
