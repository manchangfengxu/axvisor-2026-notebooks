# PIT IRQ0 Poll-Window Audit

**Date**: 2026-06-19
**Log file**: `tmp/ovmf-linux-smoke-pic-imr.log`
**Goal**: Find why `x86 PIT IRQ0 due` logs appear abundantly during OVMF DXE but disappear during Linux `check_timer()`'s 8259/ExtINT window.

---

## 1. Log timeline summary

| Time (host) | Event | Line |
|---|---|---|
| 4.89s | PIT initialized (PitState::new, period ~54.9 us) | 307 |
| 5.22s | 1st "PIT IRQ0 due but no injectable route" (LINT0 virtual_page=0x0) | 335 |
| 5.52s | Route transitions to virtual_page=0x700, ExtInt | 566 |
| 5.99s | 16th (last rate-limited) "PIT IRQ0 due but no injectable route" | 1035 |
| 6.0s-19.3s | OVMF hands off to Linux; no AxVisor PIT logs (rate limit exhausted) | - |
| 19.40s | Linux PIC ICW1 init, irq_base=0x30 | 2675-2676 |
| 19.46s | Linux OCW1=0xfe (unmask IRQ0 for 8259A attempt) | 2729 |
| 19.49s | Linux OCW1=0xff (8259A attempt failed) | 2730 |
| 19.49s | Linux OCW1=0xfe (Virtual Wire attempt) | 2735 |
| 19.51s | Virtual Wire attempt failed; PIC re-init | 2740 |
| 19.51s | ExtINT attempt begins | 2738 |
| 19.64s | ExtINT attempt failed | 2744 |
| 19.65s | Kernel panic: IO-APIC + timer doesn't work | 2747 |

Key observation: **zero** "PIT IRQ0 due" messages appear after 5.999s. Zero `[VLAPIC] LINT0 write` messages appear anywhere. Only one `APIC_ACCESS` exit appears (line 2777, after the panic).

---

## 2. Answer to each question

### Q1: Is `consume_irq0_if_due()` only observed after "guest exit"?

**Yes, in the logging sense.** The log message is emitted only when `inject_due_x86_pit_irq0` runs, which happens on specific VM-exit paths. But the rate limiter (`X86_LEGACY_IRQ_LOG_COUNT`, vm.rs line 71) caps messages at 16; all 16 are consumed during OVMF DXE (5.2-6.0s). During the Linux window, `consume_irq0_if_due()` IS called on every eligible VM exit but no message appears.

**Polling paths** (vm.rs `run_vcpu` inner loop, lines 759-927; vcpus.rs `vcpu_run` outer loop, lines 546-556):
- `IoRead`, `IoWrite`, `IoStringRead`, `IoStringWrite` -> `poll_emulated_x86_irqs = true` -> `inject_pending_x86_legacy_irqs` -> `inject_due_x86_pit_irq0` (vm.rs lines 924-927)
- `MmioRead`, `MmioWrite` -> same path
- `PreemptionTimer` -> `inject_due_pit_irq0` (vcpus.rs line 553)
- `ExternalInterrupt` -> `inject_due_pit_irq0` (vcpus.rs line 542)
- `Halt` -> `inject_due_pit_irq0` (vcpus.rs line 566)

**Non-polling paths** (no PIT check):
- `Nothing` (returned by `handle_apic_access` for LAPIC MMIO, vmx/vcpu.rs line 1322) -> falls through with `poll_emulated_x86_irqs = false`
- `NestedPageFault` -> break, no poll
- `InterruptEnd` -> break, no poll

### Q2: If guest doesn't exit frequently enough, could AxVisor miss PIT ticks entirely?

**No, but the polling frequency is the limiting factor.** The VMX preemption timer fires every ~59 us (`VMX_PREEMPTION_TIMER_SET_VALUE=100000` at 1700 MHz TSC, vmx/vcpu.rs line 57). This is the only guaranteed poll source when the guest runs a tight loop without I/O. The PIT period is ~54.9 us. Since `consume_irq0_if_due` advances the deadline by whole periods (pit.rs lines 291-296), missed ticks are not lost -- they are collapsed into the next poll, which fires at the next future deadline.

During the Linux calibration loop (24-25 ms windows), the preemption timer fires ~400+ times. Each fire polls the PIT. The PIT deadline tracking is sound; no ticks are missed at the logical level.

### Q3: Are there code paths that could change channel 0 to non-periodic, push the deadline far out, or make subsequent ticks invisible?

**No code path changes the PIT mode after boot.** The PIT channel 0 is:
- Initialized to mode 3 (SquareWaveGenerator) with divisor 0x10000 at VM creation (pit.rs `PitState::new`, lines 250-261)
- `PitChannel::new()` defaults to `SquareWaveGenerator` (pit.rs line 97)
- `program_reload(0, now_ns)` sets `period_ns = Some(54925492 ns)` and `next_deadline_ns = now_ns + period_ns`
- `is_periodic_irq()` returns true for `RateGenerator | SquareWaveGenerator` (pit.rs line 62)

During OVMF DXE, the PIT channel 0 is not reprogrammed (no "PIT channel0 command" or "PIT channel0 armed" logs appear for channel 0 after initialization). The Linux kernel does not write to PIT ports 0x40-0x43 during early boot (its APIC/HPET timer init happens much later). So channel 0 remains in mode 3 with its reset period throughout the entire window.

The deadline tracking in `consume_irq0_if_due` (pit.rs lines 281-303):
```rust
if channel.mode.is_periodic_irq() {
    let elapsed = now_ns.saturating_sub(channel.next_deadline_ns);
    let missed_periods = elapsed / period_ns;
    channel.next_deadline_ns = channel.next_deadline_ns
        .saturating_add((missed_periods + 1).saturating_mul(period_ns));
}
```
This advances the deadline past `now_ns` by an integer number of periods. Subsequent calls will not return true until `now_ns >= new_deadline_ns`. This is correct for a periodic timer and does not create "invisible" ticks.

### Q4: What is the polling frequency?

| Source | Period | Where configured |
|---|---|---|
| VMX preemption timer | ~58.8 us (100000 / 1700 MHz) | vmx/vcpu.rs line 57, vmcs setup line 725 |
| PIT channel 0 | ~54.9 us (0x10000 / 1193182 Hz) | pit.rs `PitState::new`, line 254 |

Polling happens on every eligible VM exit. In the Linux calibration loop (no I/O), only the preemption timer provides exits: ~17,000 exits/second. In the OVMF DXE phase (heavy I/O), polling is much more frequent (every I/O exit).

---

## 3. Root cause: PIC ISR stuck after first LINT0-ExtINT intack

The PIC-interrupt path in `inject_due_x86_pit_irq0` (vm.rs lines 273-321) has a structural issue that blocks sustained IRQ0 delivery.

### Flow trace (first tick, Linux check_timer 8259A attempt)

```
1. consume_irq0_if_due() -> true                    [pit.rs:281]
2. PIC assert_irq(0, true)  -> IRR bit 0 = 1        [vm.rs:284, pic.rs:134]
3. PIC assert_irq(0, false) -> last_irr bit 0 = 0   [vm.rs:285, pic.rs:152]
4. x86_ioapic_assert_gsi(GSI2) -> None               [vm.rs:289, vioapic.rs:57]
   (GSI2 is masked; read_extint_vector closure NOT called; PIC IRR untouched)
5. inject_due_x86_pic_lint0_irq                      [vm.rs:290]
   a. lint0_route() -> Some(ExtInt)                  [vm.rs:326, vlapic.rs:214]
   b. x86_pic_read_irq_vector() -> Some(0x30)        [vm.rs:329, pic.rs:309]
      ** INTACK SIDE EFFECT: IRR bit 0 cleared, ISR bit 0 SET **
      (pic.rs:120-132 intack clears IRR, sets ISR)
   c. inject_interrupt_with_trigger(vector=0x30)      [vm.rs:345, vcpu.rs:2509]
      -> queued in pending_events VecDeque
6. VM entry -> guest receives interrupt (IF=1)
7. Guest ISR handler runs -> sends LAPIC EOI
   ** Guest does NOT send PIC EOI (OCW2) **
```

### Flow trace (second tick, same attempt)

```
1. consume_irq0_if_due() -> true
2. PIC assert_irq(0, true)  -> IRR bit 0 = 1
3. PIC assert_irq(0, false)
4. x86_ioapic_assert_gsi(GSI2) -> None
5. inject_due_x86_pic_lint0_irq
   a. lint0_route() -> Some(ExtInt)
   b. x86_pic_read_irq_vector():
      get_irq(): mask = IRR & !IMR = 0x01 & 0x01 = 1, priority = 0
      in_service = ISR = 0x01 -> current_priority = 0
      priority (0) < current_priority (0) -> FALSE -> returns None
      ** NO INTACK. IRR stays at 1, ISR stays at 1 **
   c. returns false
6. "no injectable route" (rate-limited, not logged)
```

**After the first tick, PIC ISR bit 0 is permanently set** because:
- `intack()` (pic.rs line 120-132) sets ISR unconditionally
- The guest only sends LAPIC EOI (via `ack_APIC_irq()`), never PIC EOI (`ack_8259A_irq()`) for ExtINT-delivered interrupts
- `get_irq()` (pic.rs line 101-118) rejects same-priority re-delivery: `priority(0) < current_priority(0)` is false
- This blocks ALL subsequent IRQ0 delivery through the PIC

### Why this differs from real hardware

In real hardware, the PIC INTA cycle is only issued by the CPU when it actually delivers the interrupt. If the CPU doesn't deliver (IF=0, or interrupt already in-service), no INTA happens, and the PIC ISR is not set.

In AxVisor, `x86_pic_read_irq_vector()` performs the INTA as a host-side side effect during the injection path (vm.rs line 329 -> pic.rs line 309-337), **before** the interrupt is delivered to the guest. The PIC ISR is set even if:
- The guest has IF=0 and the interrupt remains pending
- The guest never sends a PIC EOI

### Impact on check_timer()

Linux's `check_timer()` (arch/x86/kernel/apic/apic.c) tests three timer delivery modes in sequence:
1. **8259A**: unmasks IRQ0, waits ~24ms, checks timer -> fails (PIC ISR stuck after 1st tick)
2. **Virtual Wire**: unmasks IRQ0, waits ~25ms -> fails (same PIC ISR issue)
3. **ExtINT**: re-inits PIC, sets LINT0 to ExtINT, waits ~128ms -> fails (same PIC ISR issue)

All three fail because the PIC ISR blocks sustained IRQ0 delivery through the LINT0-ExtINT path. The timer IRQ is delivered exactly once per attempt, which is insufficient for calibration.

### Why the PIC ISR is not cleared between attempts

Between the 8259A and Virtual Wire attempts:
- Linux masks IRQ0 (OCW1=0xff) but does NOT re-init the PIC
- PIC ISR bit 0 remains set
- Next attempt: same stuck ISR

Between the Virtual Wire and ExtINT attempts:
- Linux DOES re-init the PIC (ICW1 at line 2740)
- `reset_common()` (pic.rs line 69-85) clears ISR
- But the ExtINT attempt then hits the same first-tick-ISR-stuck issue

---

## 4. Additional finding: APIC_ACCESS exits bypass PIT polling

`handle_apic_access` (vmx/vcpu.rs line 1301) returns `AxVCpuExitReason::Nothing` for LAPIC MMIO reads/writes (line 1322). In the vm.rs inner loop, `Nothing` falls through without setting `poll_emulated_x86_irqs` (line 759: initialized to false, only set for I/O and MMIO exits). This means LAPIC MMIO accesses (including LINT0 configuration writes by Linux) do not trigger PIT polling.

This is not the primary cause of the check_timer failure (the preemption timer provides sufficient polling), but it means that a LAPIC MMIO write immediately followed by a PIC read could have a small window where PIT ticks are not polled.

---

## 5. Most likely cause ranking

| Rank | Cause | File:Line | Severity |
|---|---|---|---|
| **1** | PIC intack in LINT0-ExtINT path sets ISR bit as host-side side effect; guest never sends PIC EOI; same-priority IRQ0 blocked after 1st tick | vm.rs:329 (calls x86_pic_read_irq_vector), pic.rs:120-132 (intack), pic.rs:101-118 (get_irq priority check) | **Primary** |
| **2** | PIT IRQ0 log rate limit (X86_LEGACY_IRQ_LOG_COUNT < 16) masks all activity during Linux window, making the PIC ISR issue invisible in logs | vm.rs:71, vm.rs:293 | Diagnostic blocker |
| **3** | `handle_apic_access` returns Nothing, bypassing PIT poll in vm.rs inner loop | vmx/vcpu.rs:1322, vm.rs:759 | Minor (preemption timer compensates) |
