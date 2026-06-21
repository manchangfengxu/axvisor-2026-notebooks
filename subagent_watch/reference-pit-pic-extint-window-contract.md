# Reference PIT / 8259 / LINT0 ExtINT Runtime Contract

Source audited: QEMU (hw/timer/i8254.c, hw/timer/i8254_common.c, hw/intc/i8259.c, hw/intc/i8259_common.c, hw/intc/apic.c, hw/intc/apic_common.c, hw/intc/ioapic.c, hw/i386/x86-cpu.c, hw/i386/x86-common.c, hw/i386/pc.c), Cloud Hypervisor (arch/src/x86_64/interrupts.rs).

---

## 1. PIT channel 0: continuous periodic generation

### Reset state (i8254_common.c:pit_reset_common, line 149)

```
mode = 3 (square wave), count = 0x10000, gate = 1
```

On reset, channel 0 is armed: `pit_get_next_transition_time()` returns a finite time, `timer_mod()` is called (`i8254.c:pit_reset`, line 290-299). The PIT starts generating IRQ0 at ~18.2 Hz immediately after reset.

### Mode 3 behavior (i8254_common.c:pit_get_next_transition_time, line 76)

Mode 3 always returns a next transition time. The only case returning -1 (stop) is modes 0/1/4/5 after the count expires (`d >= s->count`). Mode 3 is self-rearming: `pit_irq_timer_update()` calls `pit_get_next_transition_time()` which always returns the next edge, and `timer_mod()` re-arms the QEMU virtual timer.

**Result: PIT channel 0 in mode 3 is continuous until guest reprograms it. No tick is ever "consumed" or "dropped" by the device model.** Each tick calls `qemu_set_irq(s->irq, irq_level)` which toggles the PIC IRQ0 input.

### Mode change (i8254.c:pit_ioport_write, line 160-173)

Guest mode writes go through `pit_ioport_write()`. Mode is written to `s->mode` (line 170). Count loads call `pit_load_count()` which re-arms the timer. **A mode change alone does not stop the timer**; only gate=0 or irq_disabled (HPET legacy) stops it.

### Gate control (i8254.c:pit_set_channel_gate, line 79)

Gate=0 on modes 2/3 disables counting but does not reset count_load_time. Gate=1 restarts. The default gate=1 means counting is always enabled.

---

## 2. 8259 PIC: masked IRQ0 latching and unmask delivery

### Edge detection and IRR latching (i8259.c:pic_set_irq, line 118-153)

```c
/* edge triggered */
if (level) {
    if ((s->last_irr & mask) == 0) {
        s->irr |= mask;
    }
    s->last_irr |= mask;
} else {
    s->last_irr &= ~mask;
}
```

IRQ0 arrives as edge: PIT calls `qemu_set_irq(pit->irq, level)` which calls `pic_set_irq()`. The `irr` bit for IRQ0 is SET regardless of IMR state. The only requirement is that `last_irr` transitions from 0 to 1 (edge detection).

**While masked (IMR bit 0 set): IRQ0 is latched in IRR. It is NOT dropped.**

### Unmask delivery (i8259.c:pic_ioport_write, line 294-299)

```c
case 0:  /* init_state == 0, normal mode, addr == 1 (OCW1) */
    s->imr = val;
    pic_update_irq(s);
    break;
```

When Linux writes `OCW1` with `imr=0xfe` (unmasking only IRQ0):
1. `s->imr = 0xfe`
2. `pic_update_irq(s)` is called immediately
3. `pic_get_irq()` computes `mask = s->irr & ~s->imr` -- if IRQ0 is in IRR, mask bit 0 is set
4. If IRQ0 has higher priority than current ISR, `qemu_irq_raise(s->int_out[0])` fires

**On unmask, any IRQ0 already latched in IRR is delivered within the same pic_update_irq() call.**

### Priority check (i8259.c:pic_get_irq, line 75-101)

`pic_get_irq()` returns IRQ0 if `irr & ~imr` has bit 0 set AND priority is higher than current ISR. After `pic_reset_common()`, `isr = 0`, so any pending IRQ0 will always win priority.

---

## 3. PIC output to LINT0 ExtINT / virtual wire path

### PIC to CPU interrupt wiring (x86-cpu.c:pic_irq_request, line 41-62)

```c
if (cpu_is_apic_enabled(cpu->apic_state) && !kvm_irqchip_in_kernel()) {
    CPU_FOREACH(cs) {
        if (apic_accept_pic_intr(cpu->apic_state)) {
            apic_deliver_pic_intr(cpu->apic_state, level);
        }
    }
} else {
    if (level) cpu_interrupt(cs, CPU_INTERRUPT_HARD);
}
```

When PIC raises `int_out[0]` (IRQ0 asserted), `pic_irq_request()` fires. If APIC is enabled in QEMU emulation mode, it goes through `apic_deliver_pic_intr()`.

### apic_deliver_pic_intr (apic.c:185-203)

```c
void apic_deliver_pic_intr(APICCommonState *s, int level) {
    if (level) {
        apic_local_deliver(s, APIC_LVT_LINT0);
    } else { /* level==0 path: clears level-triggered IRR or updates */ }
}
```

This calls `apic_local_deliver(APIC_LVT_LINT0)`.

### apic_local_deliver for ExtINT (apic.c:154-183)

```c
if (lvt & APIC_LVT_MASKED) return;       // <-- gate: must be unmasked

switch ((lvt >> 8) & 7) {
    case APIC_DM_EXTINT:                   // delivery mode 7
        cpu_interrupt(CPU(s->cpu), CPU_INTERRUPT_HARD);
        break;
}
```

**Two prerequisites for PIC interrupts to reach the CPU via LINT0:**
1. `lvt[APIC_LVT_LINT0]` delivery mode must be `APIC_DM_EXTINT` (bits [10:8] = 7)
2. `lvt[APIC_LVT_LINT0]` mask bit (bit 16) must be 0 (unmasked)

### CPU vector retrieval (x86-cpu.c:cpu_get_pic_interrupt, line 69-87)

```c
intno = apic_get_interrupt(cpu->apic_state);
if (intno >= 0) return intno;
if (!apic_accept_pic_intr(cpu->apic_state)) return -1;
// fall through:
intno = pic_read_irq(isa_pic);   // reads vector from PIC
return intno;
```

`apic_get_interrupt()` calls `apic_check_pic()` (line 723) which checks `pic_get_output(isa_pic)` -- if the PIC has a pending IRQ, it delivers via LINT0 ExtINT, and returns -1 to let the CPU read the vector from the PIC directly via `pic_read_irq()`.

### apic_accept_pic_intr (apic.c:739-753)

```c
lvt0 = s->lvt[APIC_LVT_LINT0];
if ((s->apicbase & ENABLE) == 0 || (lvt0 & MASKED) == 0)
    return isa_pic != NULL;    // accepts if PIC exists
return 0;                      // masked LINT0 -> no PIC intr
```

**If LINT0 is masked, `apic_accept_pic_intr()` returns 0, and the PIC path is completely blocked.**

---

## 4. IOAPIC ExtINT path (IOAPIC pin 2 = GSI2 = PIT IRQ0)

### IOAPIC IRQ0 to GSI2 mapping (ioapic.c:ioapic_set_irq, line 160-172)

```c
if (vector == 0) vector = 2;  // ISA IRQ0 -> GSI 2
```

### IOAPIC ExtINT vector resolution (ioapic.c:ioapic_entry_parse, line 83-87)

```c
if (info->delivery_mode == IOAPIC_DM_EXTINT) {
    info->vector = pic_read_irq(isa_pic);   // reads PIC vector
}
```

**When IOAPIC GSI2 pin is configured as ExtINT, the IOAPIC reads the vector from the PIC at delivery time.**

---

## 5. APIC reset default vs what Linux expects

### APIC reset (apic_common.c:196-197)

```c
for (i = 0; i < APIC_LVT_NB; i++) {
    s->lvt[i] = APIC_LVT_MASKED;   // all LVT masked, delivery mode = 0 (Fixed)
}
```

**After APIC reset, LINT0 is masked with delivery mode Fixed (0).** This blocks PIC interrupts entirely.

### What Cloud Hypervisor does (interrupts.rs:set_lint, line 26-38)

```rust
pub fn set_lint(vcpu: &dyn hypervisor::Vcpu) -> Result<()> {
    let mut klapic = vcpu.get_lapic()?;
    let lvt_lint0 = klapic.get_klapic_reg(APIC_LVT0);
    klapic.set_klapic_reg(APIC_LVT0, set_apic_delivery_mode(lvt_lint0, APIC_MODE_EXTINT));
    let lvt_lint1 = klapic.get_klapic_reg(APIC_LVT1);
    klapic.set_klapic_reg(APIC_LVT1, set_apic_delivery_mode(lvt_lint1, APIC_MODE_NMI));
    vcpu.set_lapic(&klapic)
}
```

Cloud Hypervisor explicitly sets LINT0 to ExtINT and LINT1 to NMI before each vCPU starts. QEMU does this through firmware/OVMF or through Linux itself writing the LVT registers.

### What Linux does during check_timer()

Linux `check_timer()` in arch/x86/kernel/apic/apic.c:
1. Masks all 8259 IRQs via OCW1
2. Sets APIC LINT0 to ExtINT delivery mode + level trigger
3. Unmasks LINT0 (clears mask bit)
4. Programs PIT in mode 0 or mode 2 to fire a single/periodic tick
5. Unmasks PIT IRQ0 via 8259 OCW1
6. Reads APIC IRR to check if the vector appeared

**Linux writes `APIC_LVT0` MMIO to `0xfee00350` to set LINT0.** This is a standard MMIO write to the local APIC register space.

---

## 6. Q1-Q4 Answers

### Q1: If PIT started ticking during early firmware, should Linux check_timer() naturally see new ticks?

**Yes.** PIT mode 3 is continuous. Every ~54.9 us (1/18.2 Hz) the PIT toggles IRQ0. If the 8259 is configured (ICW1-ICW4 written) and IRQ0 is unmasked, each tick latches in IRR and is delivered. Even if Linux masks IRQ0 during probe setup, the ticks continue arriving at the PIT and latching in IRR. When Linux unmasks IRQ0, the most recent latched tick is delivered immediately.

The only way ticks stop is:
- Guest reprograms PIT to a one-shot mode (0, 1, 4, 5) and the count expires
- Gate is set to 0
- `irq_disabled` is set (HPET legacy override)
- PIT mode is reprogrammed and count is reloaded

None of these happen during early firmware-to-Linux handoff for the standard OVMF path.

### Q2: Is IRQ0 continuously periodic until guest changes PIT mode, or can it be stopped/consumed?

**Continuously periodic.** Mode 3 generates a square wave; `pit_get_next_transition_time()` always returns a future time. `pit_irq_timer_update()` always re-arms the timer. No tick is consumed. The timer runs until the guest writes a new mode/count to channel 0, or gate goes low.

### Q3: When IRQ0 arrives while PIC is masked, what happens on unmask in QEMU 8259?

**Immediate delivery.** IRQ0 is latched in IRR on the rising edge (while masked or not). When OCW1 writes `imr = 0xfe`, `pic_update_irq()` is called in the same function. `pic_get_irq()` sees `irr & ~imr` with bit 0 set, priority check passes (ISR is typically empty during Linux probe), and `qemu_irq_raise(s->int_out[0])` fires in the same call. The CPU sees the interrupt on the next `cpu_get_pic_interrupt()` check.

### Q4: AxVisor current symptoms vs reference contract difference table

| Contract item | QEMU/CH reference | AxVisor current status |
|---|---|---|
| PIT mode 3 generates continuous IRQ0 | PIT timer never stops in mode 3; re-arms on every transition | `x86 PIT IRQ0 due` appears during OVMF DXE, then no more PIT IRQ0 logs during Linux check_timer window |
| 8259 IRR latches masked IRQ0 | `pic_set_irq()` sets `irr \|= mask` regardless of IMR; `pic_update_irq()` called on every OCW1 write | PIC ICW/OCW1 writes appear in logs; `master OCW1 imr=0xfe` logged; unclear whether IRR actually contains IRQ0 at unmask time |
| 8259 delivers immediately on unmask | `pic_update_irq()` after IMR write -> `pic_get_irq()` sees unmasked IRR -> raises int_out | No evidence of PIC int_out raise reaching LINT0 path |
| LINT0 ExtINT accepts PIC delivery | LINT0 delivery mode must be 7 (ExtINT), mask bit must be 0 | Linux writes `APIC_LVT0` to `0xfee00350` but AxVisor sees no LINT0 delivery event |
| LINT0 unmask check | `apic_accept_pic_intr()` returns true only when LINT0 mask=0 | `GSI2 masked` log suggests IOAPIC/GSI2 pin 2 is masked, separate from LINT0; LINT0 mask state unknown |
| IOAPIC GSI2 = PIT IRQ0 | `ioapic_set_irq(vector=0)` maps to GSI 2; ExtINT reads PIC vector | AxVisor logs `GSI2` but no delivery; if IOAPIC pin 2 is configured as ExtINT, it should call through to PIC |
| Cloud Hypervisor sets LINT0 ExtINT before vCPU run | `set_lint()` explicitly writes `APIC_MODE_EXTINT` to LINT0 | Not confirmed whether AxVisor pre-configures LINT0 to ExtINT or relies on guest |

---

## 7. Key source locations for follow-up

| File | Function | Line | What to check |
|---|---|---|---|
| hw/timer/i8254_common.c | pit_reset_common | 149 | Default mode=3, count=0x10000 |
| hw/timer/i8254.c | pit_irq_timer_update | 260 | Timer always re-arms in mode 3 |
| hw/timer/i8254_common.c | pit_get_next_transition_time | 76 | Only returns -1 for modes 0/1/4/5 after expiry |
| hw/intc/i8259.c | pic_set_irq | 118 | IRR set regardless of IMR |
| hw/intc/i8259.c | pic_ioport_write | 298 | OCW1: `s->imr = val; pic_update_irq(s)` |
| hw/intc/i8259.c | pic_update_irq | 104 | Calls `pic_get_irq()`, raises/lowers int_out |
| hw/intc/apic.c | apic_local_deliver | 154 | Checks MASKED, then delivery mode EXTINT |
| hw/intc/apic.c | apic_deliver_pic_intr | 185 | PIC output -> apic_local_deliver(LINT0) |
| hw/intc/apic.c | apic_accept_pic_intr | 739 | Returns 0 if LINT0 masked |
| hw/intc/apic_common.c | apic_reset_common | 190 | LVT all masked on reset |
| hw/intc/ioapic.c | ioapic_entry_parse | 83 | ExtINT: vector from pic_read_irq() |
| hw/intc/ioapic.c | ioapic_set_irq | 170 | IRQ0 maps to GSI 2 |
| hw/i386/x86-cpu.c | cpu_get_pic_interrupt | 69 | PIC vector read for ExtINT |
| arch/src/x86_64/interrupts.rs | set_lint | 26 | CH pre-configures LINT0=ExtINT |
