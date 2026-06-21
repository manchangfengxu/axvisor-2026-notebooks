# Linux x86_64 Early Timer Bring-Up: Source-Code Fact-Check

Kernel version: 7.1 (local tree at `/home/ssdns/work/axvisor-uefi/linux/`)

---

## 1. Source Inventory

| File | Key contents |
|------|-------------|
| `arch/x86/kernel/apic/io_apic.c` | `check_timer()`, `timer_irq_works()`, `find_isa_irq_pin/apic()`, `ioapic_i8259`, `setup_IO_APIC()`, `setup_IO_APIC_irqs()`, `mp_alloc_timer_irq()`, `replace_pin_at_irq_node()`, `unlock_ExtINT_logic()`, `lapic_register_intr()` |
| `arch/x86/kernel/apic/apic.c` | `apic_intr_mode_init()`, `__apic_intr_mode_select()`, `apic_bsp_setup()`, `init_bsp_APIC()`, `setup_boot_APIC_clock()`, `setup_local_APIC()` |
| `arch/x86/kernel/time.c` | `timer_interrupt()` (IRQ0 handler), `setup_default_timer_irq()`, `hpet_time_init()`, `x86_late_time_init()` |
| `arch/x86/kernel/i8253.c` | `pit_timer_init()`, `use_pit()` |
| `drivers/clocksource/i8253.c` | `clockevent_i8253_init()`, `i8253_clockevent` struct, PIT mode/counter programming |
| `drivers/irqchip/irq-i8259.c` | `enable_8259A_irq()`, `disable_8259A_irq()`, `make_8259A_irq()`, `mask_and_ack_8259A()` |
| `arch/x86/include/asm/irq_vectors.h` | `FIRST_EXTERNAL_VECTOR` (0x20), `ISA_IRQ_VECTOR()`, `NR_IRQS_LEGACY` (16) |
| `arch/x86/include/asm/apicdef.h` | `APIC_DM_EXTINT` (0x700), `APIC_DM_FIXED` (0x000), `APIC_DELIVERY_MODE_EXTINT` (7) |
| `arch/x86/include/asm/apic.h` | `enum apic_intr_mode_id` (APIC_PIC, APIC_VIRTUAL_WIRE, APIC_VIRTUAL_WIRE_NO_CONFIG, APIC_SYMMETRIC_IO, APIC_SYMMETRIC_IO_NO_ROUTING) |
| `arch/x86/include/asm/i8259.h` | PIC port constants: `PIC_MASTER_CMD` (0x20), `PIC_MASTER_IMR` (0x21), `PIC_SLAVE_CMD` (0xa0), `PIC_SLAVE_IMR` (0xa1), `PIC_CASCADE_IR` (2) |
| `include/linux/i8253.h` | `PIT_MODE` (0x43), `PIT_CH0` (0x40), `PIT_TICK_RATE` (1193182), `PIT_LATCH` |

---

## 2. Code-Path Walk

### 2.1 Boot-time call ordering

The timer probe happens inside `x86_late_time_init()` (called from `time_init()` -> late init path):

```
x86_late_time_init()                          [arch/x86/kernel/time.c:67]
  |
  +-- x86_init.irqs.intr_mode_select()        [arch/x86/kernel/time.c:73]
  |     -> apic_intr_mode_select()             [arch/x86/kernel/apic/apic.c:1308]
  |       -> __apic_intr_mode_select()          returns APIC_SYMMETRIC_IO for SMP
  |
  +-- x86_init.timers.timer_init()            [arch/x86/kernel/time.c:76]
  |     -> hpet_time_init()                    [arch/x86/kernel/time.c:57]
  |       -> pit_timer_init()                  [arch/x86/kernel/i8253.c:41]
  |         -> clockevent_i8253_init(true)     [drivers/clocksource/i8253.c:198]
  |         -> global_clock_event = &i8253_clockevent
  |       -> setup_default_timer_irq()         [arch/x86/kernel/time.c:43]
  |         -> request_irq(0, timer_interrupt, ...)
  |
  +-- x86_init.irqs.intr_mode_init()          [arch/x86/kernel/time.c:82]
        -> apic_intr_mode_init()               [arch/x86/kernel/apic/apic.c:1365]
          -> apic_bsp_setup()                  [arch/x86/kernel/apic/apic.c:2341]
            -> connect_bsp_APIC()
            -> setup_local_APIC()
            -> enable_IO_APIC()
            -> setup_IO_APIC()                 [arch/x86/kernel/apic/io_apic.c:2272]
              -> setup_IO_APIC_irqs()          programs all IOAPIC pins from MP/ACPI
              -> check_timer()                 <-- THE PROBE
            -> lapic_update_legacy_vectors()
```

### 2.2 `check_timer()` full walk

Source: `arch/x86/kernel/apic/io_apic.c:2049-2190`

**Phase 0 -- setup:**
```c
legacy_pic->mask(0);                          // mask IRQ0 in 8259 PIC
apic_write(APIC_LVT0, APIC_LVT_MASKED | APIC_DM_EXTINT);  // mask LINT0, ExtINT mode
legacy_pic->init(1);                          // re-init the 8259 PIC
```
LINT0 is masked to prevent stray ExtINT delivery during the probe. The PIC is re-initialized to clear stale state.

**Phase 1 -- lookup:**
```c
pin1  = find_isa_irq_pin(0, mp_INT);          // MP table: which IOAPIC pin has ISA IRQ 0 as mp_INT?
apic1 = find_isa_irq_apic(0, mp_INT);         // which IOAPIC controller?
pin2  = ioapic_i8259.pin;                     // IOAPIC pin wired as ExtINT (cascaded 8259)
apic2 = ioapic_i8259.apic;
```

`find_isa_irq_pin()` (line 635) scans `mp_irqs[]` for an entry where:
- the source bus is an ISA bus (`mp_bus_not_pci` bit set)
- the interrupt type is `mp_INT`
- the source bus IRQ matches 0

and returns the destination IOAPIC pin from that entry. The `ioapic_i8259` struct is populated during `enable_IO_APIC()` (line 1272) by scanning IOAPIC redirection table entries for one with `delivery_mode == APIC_DELIVERY_MODE_EXTINT`.

**Phase 2 -- IOAPIC direct (IRQ0 -> IOAPIC pin 2):**
```c
irq_domain_deactivate_irq(irq_data);
irq_domain_activate_irq(irq_data, false);     // program the IOAPIC redirection entry
if (timer_irq_works()) { goto out; }          // test
// if fail:
clear_IO_APIC_pin(apic1, pin1);
pr_err("..MP-BIOS bug: 8254 timer not connected to IO-APIC\n");
```

This tries IRQ0 directly routed through the IOAPIC pin found in the MP table (typically pin 2). If PIT interrupts arrive at the CPU through this path, `timer_irq_works()` succeeds and we are done.

**Phase 3 -- 8259A through IOAPIC (ExtINT cascade):**
```c
pr_info("...trying to set up timer (IRQ0) through the 8259A ...\n");
replace_pin_at_irq_node(data, node, apic1, pin1, apic2, pin2);
irq_domain_deactivate_irq(irq_data);
irq_domain_activate_irq(irq_data, false);     // reprogram IOAPIC to pin2 (8259 ExtINT)
legacy_pic->unmask(0);                        // unmask IRQ0 in 8259 PIC
if (timer_irq_works()) { goto out; }          // test
// if fail:
legacy_pic->mask(0);
clear_IO_APIC_pin(apic2, pin2);
```

This re-routes IRQ0 through the IOAPIC pin that carries the cascaded 8259A ExtINT signal. The chain is: PIT -> IRQ0 -> 8259 master -> IOAPIC ExtINT pin -> LAPIC LINT0 -> CPU. This works if the IOAPIC-to-LAPIC-ExtINT path is functional.

**Phase 4 -- Virtual Wire:**
```c
pr_info("...trying to set up timer as Virtual Wire IRQ...\n");
lapic_register_intr(0);                        // IRQ0 edge-triggered, lapic_chip
apic_write(APIC_LVT0, APIC_DM_FIXED | cfg->vector);  // LINT0 = Fixed delivery, vector 0x30
legacy_pic->unmask(0);
if (timer_irq_works()) { goto out; }
```

This programs LAPIC LINT0 as Fixed delivery with the ISA vector (0x30). The PIT fires IRQ0 into the 8259, the 8259 asserts INT# to the LAPIC LINT0 (which is physically wired as the 8259 cascade), and the LAPIC delivers vector 0x30 directly to the CPU.

**Phase 5 -- ExtINT:**
```c
pr_info("...trying to set up timer as ExtINT IRQ...\n");
legacy_pic->init(0);                           // re-init PIC
legacy_pic->make_irq(0);                       // set IRQ0 handler to handle_level_irq via i8259A_chip
apic_write(APIC_LVT0, APIC_DM_EXTINT);        // LINT0 = ExtINT mode
legacy_pic->unmask(0);
unlock_ExtINT_logic();                         // send INTA cycles to 8259 to clear glue logic
if (timer_irq_works()) { goto out; }
```

This programs LINT0 as ExtINT. In ExtINT mode, the LAPIC passes through the full 8259 INTA cycle: the CPU fetches the vector from the 8259 master via INTA# bus cycles. The vector is determined by the 8259 itself (ICW2), not by the LAPIC LVT0 vector field.

**Phase 6 -- panic:**
```c
panic("IO-APIC + timer doesn't work!  Boot with apic=debug and send a "
      "report.  Then try booting with the 'noapic' option.\n");
```

### 2.3 `timer_irq_works()` test

Source: `arch/x86/kernel/apic/io_apic.c:1519-1544`

```c
static int __init timer_irq_works(void)
{
    unsigned long t1 = jiffies;
    if (no_timer_check) return 1;
    local_irq_enable();
    if (boot_cpu_has(X86_FEATURE_TSC))
        delay_with_tsc();
    else
        delay_without_tsc();
    local_irq_disable();
    return time_after(jiffies, t1 + 4);   // jiffies advanced by > 4?
}
```

The function enables interrupts, busy-waits for roughly 4 PIT ticks (using TSC-based or calibrated delay), then checks whether `jiffies` advanced by more than 4. The `timer_interrupt` handler (registered via `request_irq(0, ...)`) increments `jiffies` through `global_clock_event->event_handler()`. If no interrupt arrives, jiffies stays frozen and the test fails.

The delay loop (`delay_with_tsc` at line 1486) waits progressively longer: 10M/HZ, 20M/HZ, ... up to ~4094M/HZ cycles total, roughly 10-40 ms depending on CPU frequency and HZ setting.

---

## 3. Answers to Q1-Q8

### Q1. For early timer bring-up, which delivery path does Linux expect IRQ0 to arrive on CPU?

**source-direct**

Linux tries four paths in order, each progressively more fallback:

| Path | Delivery chain | IOAPIC pin | LINT0 mode |
|------|---------------|------------|------------|
| **IOAPIC direct** | PIT -> IRQ0 wire -> IOAPIC pin (pin1 from MP table, typically pin 2) -> LAPIC -> CPU vector 0x30 | pin 2 (mp_INT entry for IRQ0) | masked initially |
| **8259A through IOAPIC** | PIT -> IRQ0 -> 8259 master INT# -> IOAPIC ExtINT pin (pin2 = ioapic_i8259.pin, typically pin 0) -> LAPIC -> CPU | pin 0 (ExtINT cascade) | masked initially |
| **Virtual Wire** | PIT -> IRQ0 -> 8259 master INT# -> LAPIC LINT0 (Fixed delivery, vector 0x30) -> CPU | not used | Fixed, vector 0x30 |
| **ExtINT** | PIT -> IRQ0 -> 8259 master INT# -> LAPIC LINT0 (ExtINT mode, vector from 8259 ICW2) -> CPU | not used | APIC_DM_EXTINT |

In all cases, the PIT (8254) is the interrupt source. The question is whether the delivery goes through the IOAPIC or through the 8259->LAPIC path.

### Q2. What is the source of `pin1=2`? Why does IRQ0 correspond to IOAPIC pin/GSI 2?

**source-direct**

`pin1` is obtained by `find_isa_irq_pin(0, mp_INT)` (line 2080). This scans the MP table interrupt entries (`mp_irqs[]`) looking for an entry where:
- the source bus is ISA (`mp_bus_not_pci` bit set)
- the interrupt type is `mp_INT` (standard interrupt, not ExtINT/SMI/NMI)
- the source bus IRQ is 0

and returns `mp_irqs[i].dstirq` -- the destination IOAPIC pin.

In QEMU's default SeaBIOS-generated MP table, the IOAPIC pin mapping is:
- Pin 0: IRQ 2 (cascade, ExtINT) -- the 8259A slave cascade
- Pin 1: IRQ 1 (keyboard controller)
- Pin 2: IRQ 0 (PIT timer)

This is a firmware (BIOS/SeaBIOS) choice, not a hardware requirement. QEMU's PIIX/ICH9 chipset model does not hard-wire the ISA-to-IOAPIC mapping; the MP table or ACPI MADT defines it. Different firmware could map IRQ0 to a different pin.

The GSI number equals `gsi_base + pin`, where `gsi_base` is 0 for the first IOAPIC (both in ACPI MADT and MP table with `gsi_top=0` at registration time). So IOAPIC pin 2 = GSI 2.

### Q3. When Linux tries "through the 8259A", does it require real programmable 8259 PIC semantics?

**source-direct**

Yes. The "through the 8259A" path (Phase 3, line 2132) requires:

1. **IOAPIC redirection entry** for `ioapic_i8259.pin` programmed with `delivery_mode = APIC_DELIVERY_MODE_EXTINT`. This was set up by `enable_IO_APIC()` (line 1341):
   ```c
   entry.delivery_mode = APIC_DELIVERY_MODE_EXTINT;
   ioapic_write_entry(ioapic_i8259.apic, ioapic_i8259.pin, entry);
   ```

2. **8259A PIC** must be functional: `legacy_pic->unmask(0)` (line 2140) writes to `PIC_MASTER_IMR` (port 0x21) to clear the IRQ0 mask bit. The 8259 must then:
   - Accept the IRQ0 assertion from the PIT
   - Present the interrupt to the LAPIC via the ExtINT signal path
   - Respond to INTA# cycles from the LAPIC/CPU with the correct vector

3. **LAPIC LINT0** must be in ExtINT mode (set at Phase 0, line 2077):
   ```c
   apic_write(APIC_LVT0, APIC_LVT_MASKED | APIC_DM_EXTINT);
   ```
   Note: LINT0 is actually masked at this point in Phase 3. The delivery goes through the IOAPIC's ExtINT entry, which is unmasked by the `irq_domain_activate_irq` call.

4. The 8259 must mask/unmask correctly via I/O port writes (PIC_MASTER_IMR = 0x21, PIC_SLAVE_IMR = 0xa1) and acknowledge via specific EOI commands to ports 0x20/0xa0.

So the requirement is: PIT fires IRQ0 -> 8259 master accepts it -> IOAPIC ExtINT pin delivers to LAPIC -> LAPIC delivers to CPU. This requires both the 8259 PIC and the IOAPIC ExtINT mode to work.

### Q4. When Linux falls back to Virtual Wire / ExtINT, does it require LAPIC LINT0 to have ExtINT/virtual-wire semantics?

**source-direct**

**Virtual Wire (Phase 4, line 2153):**
```c
apic_write(APIC_LVT0, APIC_DM_FIXED | cfg->vector);  // vector = 0x30
```
LINT0 is programmed as `APIC_DM_FIXED` delivery mode. This does NOT require ExtINT semantics on LINT0. It requires that:
- LINT0 is wired to receive the 8259 master INT# signal
- The LAPIC treats LINT0 as a standard fixed-delivery interrupt input
- The LAPIC uses the vector from LVT0 (0x30) when delivering

This is a "virtual wire" in the sense that the 8259 physical INT# line is connected to LINT0, but the LAPIC handles the vector, not the 8259's INTA cycle.

**ExtINT (Phase 5, line 2167):**
```c
apic_write(APIC_LVT0, APIC_DM_EXTINT);
```
LINT0 is programmed as `APIC_DM_EXTINT`. This DOES require full ExtINT semantics:
- When LINT0 receives an interrupt, the LAPIC must generate INTA# bus cycles
- The 8259 responds with the vector (typically 0x08 for IRQ0)
- The LAPIC passes this vector through to the CPU
- `unlock_ExtINT_logic()` (line 2174) sends dummy INTA cycles to clear potential glue logic that holds the interrupt line

### Q5. Which guest-side registers/ports/interrupt delivery semantics must be correct?

**inference-from-source**

#### PIT (8254)
| Component | Details |
|-----------|---------|
| Port 0x40 (PIT_CH0) | Channel 0 counter register (read/write) |
| Port 0x43 (PIT_MODE) | Mode/command register |
| IRQ line | Physical wire from PIT IRQ0 to 8259 master IRQ0 input |
| Mode 2 periodic | Linux programs: `outb(0x34, 0x43); outb(LATCH_lo, 0x40); outb(LATCH_hi, 0x40)` |
| Mode 3 square wave | Same as mode 2 for continuous periodic interrupts |
| Mode 0 one-shot | Used for calibration: `outb(0x30, 0x43)` |
| Interrupt frequency | PIT_TICK_RATE (1,193,182 Hz) / PIT_LATCH |
| Output signal | Must generate IRQ0 pulse to 8259 at programmed rate |

#### 8259A PIC
| Component | Details |
|-----------|---------|
| Port 0x20 (PIC_MASTER_CMD) | Master PIC command register (OCW1/OCW2/OCW3, ICW1-4) |
| Port 0x21 (PIC_MASTER_IMR) | Master PIC interrupt mask register |
| Port 0xa0 (PIC_SLAVE_CMD) | Slave PIC command register |
| Port 0xa1 (PIC_SLAVE_IMR) | Slave PIC interrupt mask register |
| ICW1-4 initialization | `legacy_pic->init(1)` / `legacy_pic->init(0)` during `check_timer()` |
| IRQ0 mask | Bit 0 of PIC_MASTER_IMR (port 0x21) |
| IRQ2 cascade | Bit 2 of PIC_MASTER_IMR (connects to slave) |
| EOI | Specific EOI: `outb(0x60+irq, PIC_MASTER_CMD)` |
| IRQ0 assertion | When unmasked, PIT IRQ0 must be able to reach CPU |
| ExtINT delivery | 8259 must respond to INTA# with vector from ICW2 |

#### IOAPIC
| Component | Details |
|-----------|---------|
| I/O address | 0xfec00000 (standard for QEMU i440FX) |
| Register select | 0x00 (ID), 0x01 (version), 0x10+2*pin (redirection low), 0x11+2*pin (redirection high) |
| Redirection entry fields | vector (8-bit), delivery mode (3-bit), destination mode, mask, trigger, polarity |
| `delivery_mode=0` (FIXED) | LAPIC delivers the vector to the destination CPU |
| `delivery_mode=7` (ExtINT) | LAPIC generates INTA# cycles, 8259 provides vector |
| Pin 0 (ExtINT) | Must be programmed if 8259 cascade path is used |
| Pin 2 (IRQ0 direct) | Must be programmed for direct IOAPIC delivery |
| Mask bit | Bit 16 of redirection entry |
| Edge vs level | ISA IRQs are typically edge-triggered; `check_timer()` un-masks accordingly |
| EOI | Write vector to APIC EOI register (version >= 0x20) or mask-toggle trick |

#### LAPIC (Local APIC)
| Component | Details |
|-----------|---------|
| APIC base | 0xfee00000 (standard) |
| LVT0 (APIC_LVT0, offset 0x350) | LINT0 vector table entry |
| LVT1 (APIC_LVT1, offset 0x360) | LINT1 vector table entry |
| SPIR (APIC_SPIV, offset 0xf0) | Spurious interrupt register, enables APIC |
| EOI (APIC_EOI, offset 0xb0) | End-of-interrupt register |
| APIC_DM_EXTINT (0x700) | ExtINT delivery mode on LINT0 |
| APIC_DM_FIXED (0x000) | Fixed delivery mode |
| APIC_LVT_MASKED (0x10000) | Mask bit for LVT entries |
| ISR/IRR/TMR registers | Read to determine interrupt state |
| EOI write | Must clear in-service bit for the delivered vector |

### Q6. What is Linux's criterion for "timer works"?

**source-direct**

The criterion is: **`jiffies` must advance by more than 4 ticks during the delay window**.

```c
// io_apic.c:1543
return time_after(jiffies, t1 + 4);
```

This is a polling test. Before calling `timer_irq_works()`, the kernel has already:
1. Registered `timer_interrupt` as the IRQ0 handler via `request_irq(0, timer_interrupt, ...)` in `setup_default_timer_irq()` (`arch/x86/kernel/time.c:52`)
2. `timer_interrupt` calls `global_clock_event->event_handler(global_clock_event)` which is the PIT clockevent handler
3. The clockevent handler calls `tick_handle_periodic()` which increments `jiffies`

So the mechanism is:
- PIT fires IRQ0 at ~100 Hz (PIT_TICK_RATE / PIT_LATCH)
- IRQ0 is delivered to CPU (through whichever path is being tested)
- IDT entry for vector 0x30 invokes `timer_interrupt`
- `timer_interrupt` calls the PIT clockevent handler
- Handler increments `jiffies`
- After ~40+ ms delay, `timer_irq_works()` checks `jiffies > t1 + 4`

If no interrupt is delivered, `jiffies` stays frozen, and the test returns false.

The `no_timer_check` kernel parameter skips this test entirely (returns 1 immediately).

### Q7. For a pure software VMM without in-kernel irqchip, what minimum behaviors are needed?

**inference-from-source**

The minimum is the intersection of what `check_timer()` tests and what the subsequent boot needs:

1. **PIT (8254) periodic interrupts**: Port 0x40/0x43 must produce IRQ0 pulses at the programmed frequency. The PIT does not need to be cycle-accurate; it needs to fire interrupts at approximately the right rate.

2. **One IOAPIC redirection entry** that can deliver the PIT interrupt to the CPU:
   - The simplest: program one IOAPIC pin with `delivery_mode=FIXED`, the correct vector (0x30), destination CPU, and unmasked. When the PIT asserts IRQ0 to that pin, the IOAPIC delivers the interrupt.
   - The MP table or ACPI MADT must map IRQ0 to this pin.

3. **LAPIC interrupt delivery**: The LAPIC must receive the IOAPIC interrupt, deliver it to the CPU via the IDT, and handle the EOI.

4. **8259 PIC** (for fallback paths): If the direct IOAPIC path fails, the kernel tries 8259-through-IOAPIC, Virtual Wire, and ExtINT. For a VMM, supporting just the direct IOAPIC path is sufficient if the MP table/ACPI is correctly configured.

5. **IDT vector 0x30**: Must be set up by the kernel to point to `timer_interrupt`.

6. **EOI handling**: After delivering the interrupt, the guest must write the LAPIC EOI register (offset 0xb0 at APIC base 0xfee00000). Without EOI, the LAPIC will not accept further interrupts on the same or lower priority.

### Q8. Categorization

**Must have** (Linux panics without these):
- PIT generates periodic IRQ0 at correct frequency via I/O port 0x40/0x43
- At least one path delivers IRQ0 to the BSP CPU: either IOAPIC with FIXED delivery to vector 0x30, or LAPIC LINT0 with appropriate mode
- LAPIC is enabled (SPIR register has APIC enable bit set)
- IDT vector 0x30 is wired to `timer_interrupt`
- LAPIC EOI is handled (write to APIC EOI register after interrupt delivery)
- IOAPIC/redirection entry is unmasked for the PIT pin

**Likely need** (for clean boot, not just check_timer):
- 8259 PIC respond to IMR writes (ports 0x20/0x21/0xa0/0xa1) without hanging -- `check_timer()` calls `legacy_pic->mask(0)`, `legacy_pic->init(1)`, `legacy_pic->unmask(0)` even on the IOAPIC direct path
- IOAPIC read/write registers at 0xfec00000 -- `setup_IO_APIC_irqs()` programs all pins from MP table
- ACPI MADT or MP table correctly describes IOAPIC, ISA IRQ routing
- PIT channel 0 counter read (for calibration, not strictly for `check_timer` but for subsequent LAPIC timer calibration)

**Currently not needed** (for check_timer only):
- LAPIC timer (set up later by `setup_boot_APIC_clock()`)
- HPET (PIT is the default; HPET replaces PIT only if enabled by firmware)
- I/O APIC version >= 0x20 EOI register (older version mask-toggle trick works)
- APIC timer broadcast mode
- MSI/MSI-X interrupts
- IRR/ISR/TMR register reads
- Multiple CPU APIC delivery
- X2APIC mode

---

## 4. Minimal Must-Have Behavior List (for `check_timer()` to pass)

1. **PIT channel 0 periodic mode**: Guest writes `0x34` to port 0x43, then LATCH value to port 0x40. PIT generates IRQ0 pulse every LATCH ticks of 1.193182 MHz clock.

2. **IOAPIC redirection entry** for the pin mapped to IRQ0 (typically pin 2 per QEMU MP table):
   - vector = 0x30
   - delivery_mode = 0 (FIXED)
   - destination = BSP CPU (physical or logical ID)
   - masked = false
   - trigger = edge (for ISA IRQ0)
   - polarity = high

3. **LAPIC** enabled with:
   - SPIR register: APIC enabled, spurious vector = 0xFF
   - capable of receiving IOAPIC interrupts and delivering them to the CPU
   - EOI register writable and functional

4. **IDT** entry for vector 0x30 pointing to `timer_interrupt` (this is kernel setup, but the virtual CPU must support IDT and deliver the correct handler when vector 0x30 fires).

5. **Interrupt delivery chain**:
   - PIT asserts IRQ0 -> IOAPIC pin 2 detects edge -> IOAPIC delivers Fixed interrupt vector 0x30 to BSP LAPIC -> LAPIC delivers to CPU -> CPU dispatches to IDT[0x30] -> `timer_interrupt` -> `jiffies` increments

6. **EOI**: After `timer_interrupt` returns, the CPU (or kernel) writes EOI to LAPIC (APIC base + 0xB0, value 0). Without this, LAPIC blocks further interrupts.

---

## 5. Open Uncertainties

**UNCONFIRMED** -- Whether `check_timer()` Phase 2 (IOAPIC direct) requires IOAPIC pin 0 to be programmed at all. The source shows `check_timer()` programs pin 1 via `irq_domain_activate_irq()` (line 2121), but pin 0 (ExtINT cascade) was already set up by `enable_IO_APIC()` (line 1341). For a VMM that only cares about passing `check_timer()`, only the IRQ0 pin (pin 2) needs to be active.

**UNCONFIRMED** -- Whether the 8259 PIC must be functional for `check_timer()` Phase 2 (IOAPIC direct) to succeed. The code calls `legacy_pic->mask(0)` and `legacy_pic->init(1)` at Phase 0 (lines 2066-2078), and LINT0 is masked with ExtINT. If Phase 2 succeeds, the PIC state should not matter. However, some VMMs report hangs if PIC I/O port writes are not handled at all during this phase. This likely depends on whether the `legacy_pic` structure's `init()` function does something blocking or just writes to ports.

**UNCONFIRMED** -- The exact PIT behavior Linux expects during the `delay_with_tsc()` window. The PIT must fire at least 5-6 interrupts (since the check requires jiffies > t1 + 4, and the first tick may be lost). At 100 Hz, that requires ~50-60 ms of correct PIT operation.

**UNCONFIRMED** -- Whether Linux checks PIT counter value or only relies on jiffies. The source shows `timer_irq_works()` only checks `jiffies`; it does not read the PIT counter port. The PIT counter read path (`clocksource_i8253`) is used later for LAPIC timer calibration, not during `check_timer()`.

**UNCONFIRMED** -- Whether `init_bsp_APIC()` (called earlier, line 1316-1359) already sets LINT0 to ExtINT mode before `check_timer()`. The source shows `init_bsp_APIC()` writes `APIC_DM_EXTINT` to LVT0 (line 1353) but only when `!smp_found_config` (line 1324). For SMP systems (which is the typical QEMU + OVMF case), `init_bsp_APIC()` returns early and LINT0 setup is left to `check_timer()` itself.

**UNCONFIRMED** -- The exact QEMU MP table pin routing. The value `pin1=2` is from QEMU's SeaBIOS. If a different firmware (e.g., OVMF) generates a different MP table, `pin1` could be different. The code handles this dynamically via `find_isa_irq_pin()`.
