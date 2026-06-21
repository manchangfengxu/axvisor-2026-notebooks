# Linux `check_timer()` / PIT / PIC / LVT0 Timing Contract Audit

Conclusions first, then code evidence.

---

## Conclusions

### Q1: Does Linux depend on PIT producing a NEW tick after `legacy_pic->unmask(0)`?

**Yes.** `timer_irq_works()` needs jiffies to advance by MORE than 4 during its ~4ms observation window. A single latched IRQ from a PIT tick that arrived while PIC was masked only increments jiffies by 1, which is insufficient. The remaining ~4 ticks must come from NEW PIT ticks that fire after unmask.

### Q2: If IRQ0 arrived at PIC while masked, then unmask happens -- should `timer_irq_works()` still succeed?

**Yes, provided PIT continues to tick.** The 8259A IRR latches the edge regardless of IMR mask. When unmasked, the pending IRR bit propagates to INT immediately, giving the first jiffy. Then subsequent PIT ticks (new edges) deliver the remaining jiffies needed. The 8259A IRR is edge-latched: multiple PIT ticks while masked only store ONE pending bit, so at most 1 "free" jiffy comes from the latch. The rest must be new.

### Q3: Does `check_timer()` reprogram PIT channel 0?

**No.** `check_timer()` contains zero writes to PIT ports (0x40--0x43). It only touches PIC ports (0x20, 0x21, 0xa0, 0xa1), IO-APIC REDTBL registers, and local APIC LVT0. PIT is assumed pre-configured and already ticking from firmware POST and the Linux `pit_timer_init()` call that runs earlier in boot.

---

## Timing Contract: `timer_irq_works()`

**File**: `linux/arch/x86/kernel/apic/io_apic.c`, lines 1519--1544

```
t1 = jiffies;
local_irq_enable();           // IRQs open
delay_with_tsc();             // loop up to 4 jiffies
local_irq_disable();
return time_after(jiffies, t1 + 4);   // need jiffies > t1+4, i.e. >= t1+5
```

- **Window duration**: Up to 4 jiffies. At HZ=1000, that is ~4ms. The TSC-based upper bound (`40_000_000_000 / HZ` = 40M TSC cycles) exits first only below ~10 MHz effective TSC, so the jiffies limit is the binding constraint on any normal (virtual or physical) CPU.
- **Minimum PIT ticks needed**: 5. After the delay loop exits (when jiffies first exceeds `t1+4`), the return check passes. With PIT rate matched to HZ (both 1000 on typical config), 5 ticks take just over 4ms.
- **No grace for lost ticks**: The comment says "at least one tick may be lost due to delays," but the math still requires `jiffies > t1+4`, which tolerates at most 1 lost tick out of the 5 needed (because the loop runs until jiffies passes `t1+4`, allowing some margin).

---

## Phase-by-Phase Walkthrough of `check_timer()`

**File**: `linux/arch/x86/kernel/apic/io_apic.c`, lines 2049--2190

### Phase 0: Setup (lines 2061--2078)

| Action | Port / Register | Effect |
|--------|----------------|--------|
| `local_irq_disable()` | -- | CPU cannot take interrupts |
| `legacy_pic->mask(0)` | PIC_MASTER_IMR (0x21) | Sets bit 0 in IMR; IRQ0 masked |
| `apic_write(APIC_LVT0, APIC_LVT_MASKED \| APIC_DM_EXTINT)` | LVT0 (0xFEE00350) | Masks LVT0, sets ExtINT delivery |
| `legacy_pic->init(1)` | PIC_MASTER_CMD (0x20), PIC_MASTER_IMR (0x21), PIC_SLAVE_CMD (0xA0), PIC_SLAVE_IMR (0xA1) | Full 8259A init: ICW1=0x11, ICW2=0x30 (ISA_IRQ_VECTOR(0)), ICW3=0x04 (cascade on IR2), ICW4=AEOI mode; restores `cached_master_mask` (IRQ0 still masked) |

**After Phase 0**: PIC fully initialized. IRQ0 masked. LVT0 masked in ExtINT mode. PIT is ticking but IRQ0 cannot reach CPU.

### Phase 1: IO-APIC direct (lines 2105--2126)

| Action | Port / Register | Effect |
|--------|----------------|--------|
| `irq_domain_activate_irq(irq_data, false)` | IO-APIC REDTBL | Routes IRQ0 directly through IO-APIC pin |
| `timer_irq_works()` | -- | ~4ms observation window |

- **PIC IRQ0 unmasked?** NO.
- **APIC LVT0 written?** No change (still masked ExtINT).
- **Depends on**: PIT -> IO-APIC -> CPU path working for IRQ0.
- **If this fails**: Falls through to Phase 2.

### Phase 2: Cascaded 8259A via IO-APIC (lines 2132--2150)

| Action | Port / Register | Effect |
|--------|----------------|--------|
| `clear_IO_APIC_pin(apic1, pin1)` | IO-APIC REDTBL | Removes Phase 1 direct route |
| `replace_pin_at_irq_node(...)` | IO-APIC REDTBL | Routes IRQ0 to 8259A cascade pin (typically IO-APIC pin for IRQ2/ExtINT) |
| `irq_domain_activate_irq(irq_data, false)` | IO-APIC REDTBL | Activates new route |
| **`legacy_pic->unmask(0)`** (line 2140) | **PIC_MASTER_IMR (0x21)** | **Clears bit 0 in IMR; IRQ0 unmasked in PIC** |
| `timer_irq_works()` | -- | ~4ms observation window |

- **PIC IRQ0 unmasked?** YES (line 2140).
- **APIC LVT0 written?** No change (still masked ExtINT from Phase 0).
- **Depends on**: PIT -> PIC (IRQ0 unmasked) -> IO-APIC cascade (ExtINT) -> CPU.
- **If this fails**: `legacy_pic->mask(0)` re-masks IRQ0, falls to Phase 3.

### Phase 3: Virtual Wire Fixed mode (lines 2153--2165)

| Action | Port / Register | Effect |
|--------|----------------|--------|
| `lapic_register_intr(0)` | IRQ descriptor | Sets IRQ0 to edge-triggered, `lapic_chip`, `handle_edge_irq` |
| **`apic_write(APIC_LVT0, APIC_DM_FIXED \| cfg->vector)`** (line 2156) | **LVT0 (0xFEE00350)** | **Fixed delivery mode, specific vector, UNMASKED** |
| **`legacy_pic->unmask(0)`** (line 2157) | **PIC_MASTER_IMR (0x21)** | **Clears bit 0 in IMR; IRQ0 unmasked in PIC** |
| `timer_irq_works()` | -- | ~4ms observation window |

- **PIC IRQ0 unmasked?** YES (line 2157).
- **APIC LVT0 value**: `APIC_DM_FIXED | cfg->vector` (e.g., `0x00000030` if vector=0x30). No mask bit set, so LVT0 is unmasked.
- **Depends on**: PIT -> PIC (IRQ0 unmasked) -> CPU INT -> local APIC LVT0 Fixed -> ISR.
- **If this fails**: `legacy_pic->mask(0)` re-masks IRQ0; `apic_write(APIC_LVT0, APIC_LVT_MASKED | APIC_DM_FIXED | cfg->vector)` masks LVT0; falls to Phase 4.

### Phase 4: ExtINT mode (lines 2167--2187)

| Action | Port / Register | Effect |
|--------|----------------|--------|
| `legacy_pic->init(0)` (line 2169) | PIC ports 0x20, 0x21, 0xA0, 0xA1 | Full 8259A re-init in NON-AEOI (normal EOI) mode |
| `legacy_pic->make_irq(0)` (line 2170) | IRQ descriptor | Sets IRQ0 to level-triggered, `i8259A_chip`, `handle_level_irq`, `lapic_assign_legacy_vector` |
| **`apic_write(APIC_LVT0, APIC_DM_EXTINT)`** (line 2171) | **LVT0 (0xFEE00350)** | **ExtINT delivery mode, UNMASKED** |
| **`legacy_pic->unmask(0)`** (line 2172) | **PIC_MASTER_IMR (0x21)** | **Clears bit 0 in IMR; IRQ0 unmasked in PIC** |
| `unlock_ExtINT_logic()` (line 2174) | IO-APIC REDTBL (for IRQ8/RTC pin), RTC ports | Temporarily routes IRQ8 via IO-APIC ExtINT, sends INTA cycles to 8259A to clear glue logic, restores original IO-APIC entry |
| `timer_irq_works()` | -- | ~4ms observation window |

- **PIC IRQ0 unmasked?** YES (line 2172).
- **APIC LVT0 value**: `APIC_DM_EXTINT` (0x00000700). No mask bit, so LVT0 is unmasked.
- **Depends on**: PIT -> PIC (IRQ0 unmasked) -> local APIC ExtINT (transparent pass-through of PIC INT) -> CPU.
- **If this fails**: PANIC at line 2186.

---

## Ports Written per Phase

| Phase | PIC ports | PIT ports (0x40-0x43) | IO-APIC REDTBL | Local APIC LVT0 (0xFEE00350) |
|-------|-----------|----------------------|----------------|-------------------------------|
| 0 (setup) | 0x20 (CMD), 0x21 (IMR), 0xA0, 0xA1 | NONE | -- | Masked \| ExtINT |
| 1 (IO-APIC direct) | NONE | NONE | IRQ0 pin routing | No change |
| 2 (cascade 8259A) | 0x21 (unmask IRQ0) | NONE | Cascade pin routing | No change |
| 3 (Fixed LVT0) | 0x21 (unmask IRQ0) | NONE | No change | FIXED \| vector (unmasked) |
| 4 (ExtINT) | 0x20, 0x21 (re-init), 0xA0, 0xA1, 0x21 (unmask IRQ0) | NONE | IRQ8 ExtINT (temp) | ExtINT (unmasked) |

**PIT ports are never written by check_timer().**

---

## Key 8259A PIC Mechanism

**File**: `linux/arch/x86/kernel/i8259.c`

- `mask_8259A_irq(0)` (line 60): Sets bit 0 in `cached_irq_mask`, writes `cached_master_mask` to port 0x21. IRQ0 masked.
- `unmask_8259A_irq(0)` (line 79): Clears bit 0 in `cached_irq_mask`, writes `cached_master_mask` to port 0x21. IRQ0 unmasked.
- `init_8259A(auto_eoi)` (line 349): Masks all (0xFF to port 0x21), runs ICW1-ICW4 sequence, restores `cached_master_mask` at end (line 395).
- `i8259A_irq_real()` (line 130): Reads ISR register via OCW3 (0x0B to CMD port), checks if specific IRQ is in-service.

The 8259A IRR latches edge-triggered interrupts regardless of IMR mask. When IMR bit is cleared (unmask), any pending IRR bit immediately asserts INT to the CPU (subject to priority). Multiple PIT edges while masked only store ONE pending bit in IRR (edge-triggered: repeated edges on an already-set IRR bit are idempotent).

---

## PIC OCW1 IMR Values in check_timer()

At each unmask point, the value written to port 0x21:

| Point | `cached_master_mask` value | Binary (bit 0 = IRQ0) | Effect |
|-------|---------------------------|------------------------|--------|
| After Phase 0 mask(0) | `0xFF` (all masked) | `11111111` | IRQ0 masked |
| After Phase 2 unmask(0) | `0xFE` (IRQ0 clear) | `11111110` | IRQ0 unmasked, all others masked |
| After Phase 2 mask(0) | `0xFF` | `11111111` | IRQ0 masked again |
| After Phase 3 unmask(0) | `0xFE` | `11111110` | IRQ0 unmasked |
| After Phase 3 mask(0) | `0xFF` | `11111111` | IRQ0 masked again |
| After Phase 4 unmask(0) | `0xFE` | `11111110` | IRQ0 unmasked |

The AxVisor log `[VPIC] master OCW1 imr=0xfe` corresponds to one of the unmask(0) calls. The `[VPIC] master ICW2 irq_base=0x30` corresponds to an `init_8259A` call (ICW2 = `ISA_IRQ_VECTOR(0)` = `((0x20+16) & ~15) + 0` = `0x30`).

---

## Implications for AxVisor

1. **check_timer() never reprograms PIT.** It trusts PIT is already producing periodic IRQ0 ticks. If AxVisor's virtual PIT is not running or not producing IRQ0 edges, all four phases fail.

2. **Phase 2/3/4 each unmask IRQ0 in the PIC, then wait ~4ms for 5 jiffies.** The PIT must deliver at least 5 IRQ0 interrupts through the PIC to the CPU during that window. If AxVisor's PIT-to-PIC-to-CPU interrupt delivery path has any gap (PIC port intercept not forwarding IMR changes to actual interrupt delivery, PIT not firing, or CPU not receiving the vector), `timer_irq_works()` returns false.

3. **The `legacy_pic->unmask(0)` call writes `cached_master_mask` (0xFE) to port 0x21.** AxVisor must intercept this write AND update its internal PIC state so that the next PIT IRQ0 edge is delivered to the vCPU. If the write is intercepted but the PIC's internal "IRQ0 unmasked" state is not tracked, the PIC will continue to block PIT interrupts.

4. **`init_8259A(1)` at Phase 0 restores `cached_master_mask` at the end.** After `mask(0)` sets bit 0, the init sequence ends by writing that masked value back. AxVisor must process all four ICW bytes AND the final OCW1 write. If AxVisor only logs ICW1/ICW2 but doesn't track the final IMR restore, the PIC's mask state will be wrong.

5. **`timer_irq_works()` requires 5 PIT ticks in ~4ms.** At the standard PIT rate of ~1000 Hz (PIT_LATCH = `(1193182 + 500) / 1000` = 1193 for HZ=1000), this is exactly 5 ticks in 5ms, which exceeds the 4ms jiffies limit by 1ms -- just barely enough. If the virtual PIT runs slower than the host PIT (e.g., stalled during VM exit processing), the window may not be wide enough.

---

## Boot Order Context

**File**: `linux/arch/x86/kernel/time.c`, `linux/arch/x86/kernel/apic/apic.c`

```
start_kernel()
  -> init_IRQ()             [line 1079]
  -> time_init()            [line 1088]
     -> late_time_init = x86_late_time_init  [line 1138-1139]
        1. intr_mode_select()
        2. timer_init() = hpet_time_init()
           -> pit_timer_init()  -- registers PIT clockevent, sets global_clock_event
           -> setup_default_timer_irq()  -- request_irq(0, timer_interrupt, ...)
        3. intr_mode_init() = apic_intr_mode_init()
           -> apic_bsp_setup()
              -> setup_IO_APIC()
                 -> check_timer()    <-- THIS IS THE FUNCTION UNDER AUDIT
```

PIT clockevent registration and IRQ0 handler registration happen BEFORE `check_timer()`. The PIT is already ticking. `check_timer()` only tests whether the interrupt delivery path works, not whether PIT is configured.
