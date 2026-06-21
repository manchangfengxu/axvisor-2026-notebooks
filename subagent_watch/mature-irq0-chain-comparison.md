# Mature IRQ0/PIT/PIC/IOAPIC/LAPIC Chain Comparison

## 1. Source Inventory

### QEMU (local checkout)
| File | Role |
|------|------|
| `hw/intc/ioapic.c` | IOAPIC: IRQ0->GSI2 mapping, ExtINT->pic_read_irq vector lookup, level/edge dispatch |
| `hw/intc/i8259.c` | PIC: read_irq (cascade logic), intack, set_irq_line, EOI, poll mode |
| `hw/intc/i8259_common.c` | PIC: pic_reset_common, i8259_init_chip (master/slave init, irq_base=0/0x08) |
| `hw/timer/i8254.c` | PIT: channel 0 timer callback, qemu_set_irq(s->irq, level) to ISA bus IRQ 0 |
| `hw/timer/i8254.h` | PIT init: `i8254_pit_init()` connects GPIO out to `isa_bus_get_irq(bus, isa_irq)` |
| `hw/i386/pc.c` | PC machine: pc_i8259_create, pc_msos_init (PIT init with pit_isa_irq=0) |
| `hw/i386/x86-common.c` | gsi_handler: GSI 0-15 -> PIC + IOAPIC; ioapic_init_gsi: wire IOAPIC pins to GSI state |
| `hw/i386/kvm/ioapic.c` | KVM IRQ routing: GSI 0->IOAPIC pin 2, GSI 0-7->PIC_MASTER, GSI 8-15->PIC_SLAVE |
| `hw/intc/apic.c` | LAPIC: apic_local_deliver(LINT0, EXTINT) -> CPU_INTERRUPT_HARD; apic_deliver_pic_intr |
| `include/hw/i386/pc.h` | ISA_NUM_IRQS=16, IOAPIC_NUM_PINS=24 |
| `include/hw/i386/x86.h` | GSIState: i8259_irq[16] + ioapic_irq[24] |

### Cloud Hypervisor (local checkout)
| File | Role |
|------|------|
| `devices/src/ioapic.rs` | Userspace IOAPIC: MSI-based routing, delivery_mode enum includes External=0b111 |
| `devices/src/interrupt_controller.rs` | InterruptController trait definition |
| `devices/src/legacy/i8042.rs` | i8042 stub: port B returns 0x20 to avoid pit_calibrate_tsc hang |
| `devices/src/legacy/serial.rs` | Serial: LegacyIrqGroupConfig based interrupt |
| `arch/src/x86_64/interrupts.rs` | LINT0=ExtINT, LINT1=NMI setup (Virtual Wire mode) |
| `arch/src/x86_64/mptable.rs` | MP table: reports IOAPIC, ISA bus, IRQ 0-15 -> IOAPIC pin 0-15 (1:1) |
| `vmm/src/interrupt.rs` | LegacyUserspaceInterruptManager wraps IOAPIC for legacy IRQs; MsiInterruptManager for MSI |
| `vmm/src/device_manager.rs` | IOAPIC creation and wiring; legacy interrupt manager setup |

### AxVisor (current tree)
| File | Role |
|------|------|
| `tgoskits/virtualization/x86_vlapic/src/vioapic.rs` | vIOAPIC: ExtINT path calls read_extint_vector callback; GSI0->GSI2 in axvm glue |
| `tgoskits/virtualization/x86_vlapic/src/pic.rs` | PIC: full 8259 master/slave with cascade, intack, ISR/IRR/IMR, read_irq_vector |
| `tgoskits/virtualization/x86_vlapic/src/pit.rs` | PIT: channel 0 periodic timer with consume_irq0_if_due |
| `tgoskits/virtualization/axdevice/src/device.rs` | AxVmDevices: wires PIC->IOAPIC ExtINT, PIT->PIC IRQ0->IOAPIC GSI2 |
| `tgoskits/virtualization/axvm/src/runtime/x86_irq.rs` | inject_due_pit_irq0: PIT->PIC assert->IOAPIC assert_gsi(2)->vcpu inject |

---

## 2. QEMU Findings

### 2.1 IRQ0 -> GSI2 Mapping

**File**: `hw/intc/ioapic.c`, function `ioapic_set_irq()` (line 160-172)

```c
static void ioapic_set_irq(void *opaque, int vector, int level)
{
    /* ISA IRQs map to GSI 1-1 except for IRQ0 which maps
     * to GSI 2.  GSI maps to ioapic 1-1. */
    if (vector == 0) {
        vector = 2;
    }
    ...
}
```

This is the canonical IRQ0->GSI2 mapping. It is hardcoded in the IOAPIC's `ioapic_set_irq` callback, not in the IOAPIC's MMIO register read/write path.

**Also confirmed in KVM routing**: `hw/i386/kvm/ioapic.c`, function `kvm_pc_setup_irq_routing()` (line 22-47):
```c
for (i = 0; i < KVM_IOAPIC_NUM_PINS; ++i) {
    if (i == 0) {
        kvm_irqchip_add_irq_route(s, i, KVM_IRQCHIP_IOAPIC, 2);
    } else if (i != 2) {
        kvm_irqchip_add_irq_route(s, i, KVM_IRQCHIP_IOAPIC, i);
    }
}
```
KVM's in-kernel routing table maps GSI 0 to IOAPIC pin 2, confirming the same mapping.

### 2.2 IOAPIC ExtINT Delivery Mode -> Vector from PIC

**File**: `hw/intc/ioapic.c`, function `ioapic_entry_parse()` (line 68-95)

```c
if (info->delivery_mode == IOAPIC_DM_EXTINT) {
    info->vector = pic_read_irq(isa_pic);
} else {
    info->vector = entry & IOAPIC_VECTOR_MASK;
}
```

When the guest programs IOAPIC redirection entry for GSI 2 with delivery mode = ExtINT (0b111), the IOAPIC does NOT use the vector field from the entry. Instead, it calls `pic_read_irq(isa_pic)` to obtain the vector from the 8259 PIC at the moment the interrupt is dispatched.

This means:
- The vector is dynamic, determined by the PIC's current state
- The IOAPIC polls the PIC each time it services the GSI 2 interrupt
- The PIC must have a pending interrupt (IRQ 0 in IRR, unmasked) for the vector to be valid

### 2.3 PIC Master/Slave Cascade, read_irq, intack Semantics

**File**: `hw/intc/i8259.c`, function `pic_read_irq()` (line 173-211)

```c
int pic_read_irq(PICCommonState *s)
{
    int irq, intno;
    irq = pic_get_irq(s);
    if (irq >= 0) {
        int irq2;
        if (irq == 2) {
            irq2 = pic_get_irq(slave_pic);
            if (irq2 >= 0) {
                pic_intack(slave_pic, irq2);
            } else {
                irq2 = 7;  // spurious IRQ on slave
            }
            intno = slave_pic->irq_base + irq2;
            pic_intack(s, irq);
            irq = irq2 + 8;
        } else {
            intno = s->irq_base + irq;
            pic_intack(s, irq);
        }
    } else {
        irq = 7;
        intno = s->irq_base + irq;  // spurious on master
    }
    return intno;
}
```

Key semantics:
- `pic_get_irq()`: returns highest-priority pending IRQ (IRR & ~IMR, checked against ISR for priority)
- Cascade on IRQ 2: if master has IRQ 2 pending, read from slave_pic
- `pic_intack()`: sets ISR bit (unless auto_eoi), clears IRR for edge-triggered
- Return value is the interrupt vector number (`irq_base + irq`), NOT the IRQ number
- Default irq_base: master=0x00, slave=0x08 (set during PIC ICW2 init)

### 2.4 PIC read_irq Vector for Timer

For PIT (IRQ 0) specifically:
- Master PIC has IRQ 0 in IRR (set when PIT asserts IRQ 0)
- `pic_read_irq()` finds IRQ 0 on master, returns `master.irq_base + 0 = 0x00`
- Wait, irq_base is typically 0x00 after ICW2, so the vector would be 0x00? No, the irq_base is set by ICW2 to 0x20 (32) for the master PIC.

Actually, looking at the PIC init more carefully:
- ICW2 for master: the high 5 bits of ICW2 become irq_base. Linux typically sets ICW2=0x20, so irq_base=0x20.
- For IRQ 0: vector = 0x20 + 0 = 0x20 (but Linux later re-maps this via IOAPIC to 0x30 or similar)
- Actually, the PIC irq_base is set by the OS during PIC initialization (ICW2 command). The default in QEMU after reset is 0, but Linux sets it to 0x20 (master) and 0x28 (slave).

### 2.5 PIT Connection to ISA Bus IRQ 0

**File**: `hw/timer/i8254.h`, function `i8254_pit_init()` (line 48-63)

```c
static inline ISADevice *i8254_pit_init(ISABus *bus, int base, int isa_irq,
                                        qemu_irq alt_irq)
{
    ...
    qdev_connect_gpio_out(dev, 0,
                          isa_irq >= 0 ? isa_bus_get_irq(bus, isa_irq)
                                       : alt_irq);
    return d;
}
```

Called from `hw/i386/pc.c` with `pit_isa_irq = 0` (when HPET is disabled).

### 2.6 GSI Handler: Dual PIC + IOAPIC Forwarding

**File**: `hw/i386/x86-common.c`, function `gsi_handler()` (line 457-494)

```c
void gsi_handler(void *opaque, int n, int level)
{
    GSIState *s = opaque;
    switch (n) {
    case 0 ... ISA_NUM_IRQS - 1:   // 0..15
        if (s->i8259_irq[n]) {
            qemu_set_irq(s->i8259_irq[n], level);  // -> PIC
        }
        /* fall through */
    case ISA_NUM_IRQS ... IOAPIC_NUM_PINS - 1:  // 16..23
        if (!bypass_ioapic) {
            qemu_set_irq(s->ioapic_irq[n], level);  // -> IOAPIC
        }
        break;
    }
}
```

ISA IRQs (GSI 0-15) are forwarded to BOTH the PIC and the IOAPIC. This is the standard dual-8259 + IOAPIC configuration. The PIC sees the raw ISA IRQ number; the IOAPIC sees the GSI number.

### 2.7 LAPIC LINT0 / Virtual Wire Mode

**File**: `hw/intc/apic.c`, function `apic_local_deliver()` (line 153-183)

```c
case APIC_DM_EXTINT:
    cpu_interrupt(CPU(s->cpu), CPU_INTERRUPT_HARD);
    break;
```

When LAPIC LINT0 is configured as ExtINT delivery mode (Virtual Wire), the LAPIC acts as a passthrough: it signals the CPU to take an external interrupt (CPU_INTERRUPT_HARD). The CPU then reads the interrupt vector from the PIC via INTA cycle (or in QEMU's case, via `pic_read_irq()`).

Function `apic_deliver_pic_intr()` (line 185-203):
```c
void apic_deliver_pic_intr(APICCommonState *s, int level)
{
    if (level) {
        apic_local_deliver(s, APIC_LVT_LINT0);
    } else {
        // handle deassert: clear IRR if level-triggered, or update
    }
}
```

This function is the bridge between the IOAPIC's ExtINT delivery and the LAPIC's LINT0. When the IOAPIC services an ExtINT entry, it calls this function on the target LAPIC, which then triggers the LAPIC's LINT0 delivery mode.

---

## 3. Cloud Hypervisor Findings

### 3.1 No 8259 PIC Implementation

Cloud Hypervisor does NOT implement an 8259 PIC. There is no file corresponding to PIC emulation in the entire codebase. The `devices/src/legacy/` directory contains:
- `cmos.rs`, `debug_port.rs`, `fw_cfg.rs`, `fwdebug.rs`, `gpio_pl061.rs`, `i8042.rs`, `mod.rs`, `rtc_pl031.rs`, `serial.rs`, `uart_pl011.rs`

No PIC, no PIT.

### 3.2 No PIT Implementation

No PIT/8254 timer device is implemented. The i8042 PS/2 controller (`devices/src/legacy/i8042.rs`) has a critical workaround:

```rust
// Like kvmtool, we return bit 5 set in I8042_PORT_B_REG to
// avoid hang in pit_calibrate_tsc() in Linux kernel.
data[0] = 0x20;
```

This returns bit 5 (PIT Channel 2 gate) as always-set, which tells Linux that PIT Channel 2 is functional. Without this, Linux's `pit_calibrate_tsc()` would hang waiting for PIT Channel 2 to tick during TSC calibration.

### 3.3 Cloud Hypervisor Platform Prerequisites

Cloud Hypervisor relies on:

1. **KVM in-kernel IOAPIC/PIC**: KVM's kernel module provides a built-in IOAPIC and PIC pair. The default IRQ routing in KVM (via `kvm_setup_default_irq_routing()` in Linux kernel) maps:
   - GSI 0 -> IOAPIC pin 2 (same as QEMU)
   - GSI 1-15 -> IOAPIC pin 1-15
   - GSI 0-7 -> PIC_MASTER
   - GSI 8-15 -> PIC_SLAVE

2. **Linux kernel's own PIC->LAPIC LINT0 setup**: Linux boot code configures LAPIC LINT0 to ExtINT mode (Virtual Wire) and LINT1 to NMI mode. Cloud Hypervisor confirms this via `set_lint()` in `arch/src/x86_64/interrupts.rs`.

3. **Userspace IOAPIC for MSI routing only**: Cloud Hypervisor's `Ioapic` device is an MSI-based interrupt source. It translates IOAPIC redirection table entries into MSI messages and uses KVM irqfd for direct injection. This is NOT used for early ISA IRQ handling.

4. **MP table for Linux boot**: The MP table (`arch/src/x86_64/mptable.rs`) reports the interrupt topology, but the actual IRQ routing is handled by KVM's kernel-level irq routing table.

### 3.4 How Cloud Hypervisor Handles Early Linux Timer

The chain is:
1. KVM provides in-kernel PIC + IOAPIC
2. KVM's default irq routing maps GSI 0 -> IOAPIC pin 2
3. Linux boot code sets up LAPIC LINT0 = ExtINT (Virtual Wire mode)
4. PIT fires (handled by KVM in-kernel or host-side PIT)
5. KVM routes IRQ 0 through IOAPIC pin 2 with ExtINT delivery mode
6. In-kernel IOAPIC reads the PIC vector and delivers to LAPIC
7. LAPIC LINT0 in ExtINT mode passes the interrupt to the CPU

Cloud Hypervisor's userspace IOAPIC only becomes relevant after Linux has switched to IOAPIC-based MSI routing for devices like virtio-net, NVMe, etc.

---

## 4. Gap Matrix

### Row: PIT (8254) Timer

| Component | QEMU | Cloud Hypervisor | AxVisor |
|-----------|------|-----------------|---------|
| PIT device | `hw/timer/i8254.c` - full 8254 with all 6 modes, read-back, gate control | **Missing** - relies on host KVM/KVMTOOL for PIT | `x86_vlapic/src/pit.rs` - `EmulatedPit`: channels 0 and 2, mode command, count latch, periodic/one-shot, speaker port 0x61 |
| Channel 0 periodic tick | `pit_irq_timer_update()` fires `qemu_set_irq(s->irq, irq_level)` to ISA bus IRQ 0 | Handled by KVM kernel PIT | `consume_irq0_if_due()` checks deadline, returns bool; caller asserts PIC IRQ0 |
| Port 0x40-0x43 | Full I/O port emulation | **Not emulated** (KVM handles it) | `EmulatedPit` registered at ports 0x40..=0x43 + 0x61 |
| Port 0x61 (speaker) | Connected to `pit_irq_control()` to enable/disable channel 0 | i8042 returns 0x20 to fake bit 5 | `speaker_control` field stored but only read as 0; no timer gating via port 0x61 |

### Row: PIC (8259) Master/Slave

| Component | QEMU | Cloud Hypervisor | AxVisor |
|-----------|------|-----------------|---------|
| PIC device | `hw/intc/i8259.c` + `i8259_common.c` - full 8259 pair with master/slave cascade | **Missing** - relies on KVM kernel PIC | `x86_vlapic/src/pic.rs` - `EmulatedPic8259` with `PicState` (master+slave) |
| Cascade on IRQ 2 | `pic_read_irq()`: if master IRQ 2 pending, read from `slave_pic`, return slave vector | N/A (no PIC) | `read_irq_vector()`: if master IRQ 2, `slave.get_irq()` -> `slave.intack()` -> return slave vector |
| `read_irq` / `intack` | `pic_read_irq()` returns intack'd vector; `pic_intack()` sets ISR, clears IRR | N/A | `read_irq_vector()` calls `get_irq()` -> `intack()` -> returns `irq_base + irq` |
| ISR/IRR/IMR | Full IRR/IMR/ISR with priority, special mask, fully nested mode | N/A | `PicChipState` has all fields: irr, imr, isr, priority_add, irq_base, special_mask, auto_eoi, etc. |
| ICW1-ICW4 init | `pic_ioport_write()` handles all ICW/OCW commands | N/A | `command_write()` handles ICW1 (reset), ICW2 (irq_base), ICW3 (cascade), ICW4 (auto_eoi, special_fully_nested) |
| EOI | OCW2 non-specific EOI, specific EOI, rotate-on-EOI | N/A | `command_write()` handles OCW2 commands 0-7 (EOI variants) |
| Poll mode | OCW3 poll command, returns highest priority pending with bit 7 set | N/A | `read()` with `poll` flag returns `get_irq().map_or(0, |irq| { intack(irq); irq | 0x80 })` |
| Port 0x20/0x21 | Master command/data | N/A | `EmulatedPicMasterPort` at 0x20/0x21 |
| Port 0xa0/0xa1 | Slave command/data | N/A | `EmulatedPicSlavePort` at 0xa0/0xa1 |
| Port 0x4d0/0x4d1 | ELCR (edge/level control) | N/A | `EmulatedPicElcrPort` |

### Row: IOAPIC

| Component | QEMU | Cloud Hypervisor | AxVisor |
|-----------|------|-----------------|---------|
| IOAPIC device | `hw/intc/ioapic.c` - standard 82093AA, 24 pins, `ioapic_set_irq` with IRQ0->GSI2 remap | `devices/src/ioapic.rs` - MSI-based delivery, no GSI remapping, no PIC interaction | `x86_vlapic/src/vioapic.rs` - `EmulatedIoApic` with ExtINT callback |
| IRQ0 -> GSI2 | **Explicit** in `ioapic_set_irq()`: `if (vector == 0) vector = 2;` | Not applicable (no IRQ0/PIC path) | Handled in `axvm/src/runtime/x86_irq.rs`: caller asserts GSI 2 directly |
| ExtINT delivery mode | `ioapic_entry_parse()`: calls `pic_read_irq(isa_pic)` to get vector from PIC | `DeliveryMode::External = 0b111` defined but `update_entry()` passes vector through MSI data directly; no PIC callback | `interrupt_for_entry()`: `REDIRECTION_ENTRY_DELIVERY_MODE_EXTINT` calls `read_extint_vector()` callback which calls PIC `read_irq_vector()` |
| EOI handling | `ioapic_eoi_broadcast()`: clears Remote IRR for level-triggered entries | `end_of_interrupt()`: clears Remote IRR for matching vector | `end_of_interrupt()`: clears Remote IRR, re-injects pending level IRQ |
| Redirection table | 24 entries, full register read/write | 24 entries, MSI message generation | 24 entries, MMIO register interface |
| Level-triggered coalesce | Remote IRR bit checked; new delivery skipped if set | Remote IRR bit managed | Remote IRR bit checked in `interrupt_for_entry()` |
| GSI -> IOAPIC pin mapping | GSI 0 -> pin 2 (via `ioapic_set_irq` remap) | No remapping; GSI i -> IOAPIC entry i | No remapping; caller is responsible for GSI 2 |

### Row: LAPIC LINT0 / Virtual Wire

| Component | QEMU | Cloud Hypervisor | AxVisor |
|-----------|------|-----------------|---------|
| LINT0 ExtINT setup | `apic.c::apic_local_deliver()`: EXTINT -> `CPU_INTERRUPT_HARD` | `arch/src/x86_64/interrupts.rs::set_lint()`: LINT0 = APIC_MODE_EXTINT | **Not yet implemented** - vLAPIC exists but LINT0 ExtINT path not verified |
| LINT1 NMI setup | `apic_local_deliver()`: NMI -> `CPU_INTERRUPT_SMI` | LINT1 = APIC_MODE_NMI | Not explicitly verified |
| `apic_deliver_pic_intr()` | Bridges IOAPIC ExtINT -> LAPIC LINT0 | Not needed (KVM handles it) | N/A - AxVisor injects directly via `vcpu.inject_interrupt_with_trigger()` |
| LAPIC register access | Full LAPIC MMIO/MSR emulation | KVM LAPIC (kernel) | `EmulatedLapic` in `x86_vlapic/src/vlapic.rs` (exists but LINT0 path not exercised) |

### Row: Interrupt Injection Path (PIT -> CPU)

| Step | QEMU | Cloud Hypervisor | AxVisor |
|------|------|-----------------|---------|
| 1. PIT fires | `pit_irq_timer()` -> `qemu_set_irq(s->irq, level)` to ISA bus IRQ 0 | KVM kernel PIT -> KVM irq routing -> GSI 0 | `inject_due_pit_irq0()` polls `x86_pit_consume_irq0_if_due()` |
| 2. GSI routing | `gsi_handler()`: GSI 0 -> PIC master IRQ 0 + IOAPIC pin 2 | KVM kernel: GSI 0 -> IOAPIC pin 2 | PIC assert_irq(0, true/false), then ioapic_assert_gsi(2) |
| 3. PIC sees IRQ 0 | Master PIC: IRR bit 0 set, `pic_update_irq()` raises INT output to CPU | N/A (KVM kernel PIC) | PIC: `assert_irq(0, true)` sets IRR bit 0 |
| 4. IOAPIC services GSI 2 | `ioapic_service()` -> `ioapic_entry_parse()` -> if ExtINT: `pic_read_irq(isa_pic)` -> return PIC vector | N/A (KVM kernel IOAPIC) | `interrupt_for_entry(2, read_extint_vector)` -> ExtINT -> `pic.read_irq_vector()` -> return vector |
| 5. LAPIC delivery | `apic_deliver_pic_intr()` -> `apic_local_deliver(LINT0)` -> ExtINT -> `CPU_INTERRUPT_HARD` | KVM kernel LAPIC handles it | `vcpu.inject_interrupt_with_trigger(vector, trigger_mode)` |
| 6. CPU takes interrupt | CPU exits to interrupt handler, reads vector | KVM injects into guest | VMX/SVM injection |

---

## 5. Concrete Implementation Implications for AxVisor

### 5.1 What AxVisor Already Has (Verified)

AxVisor's current implementation covers all major protocol semantics:

1. **PIC**: Full 8259 emulation with cascade, ISR/IRR/IMR, intack, poll mode, ICW/OCW, ELCR, auto_EOI. The `read_irq_vector()` function correctly handles cascade on IRQ 2 and returns the interrupt vector.

2. **PIT**: Channel 0 periodic timer with `consume_irq0_if_due()` deadline checking. Mode command, count latch, and basic I/O port emulation.

3. **IOAPIC**: ExtINT delivery mode with PIC callback. The `read_extint_vector` closure correctly calls PIC `read_irq_vector()`. Level-triggered support with Remote IRR bit.

4. **IRQ0 -> GSI2**: Handled correctly in `x86_irq.rs` where `PIT_TIMER_GSI = 2` is used for IOAPIC assertion.

5. **Full injection chain**: `inject_due_pit_irq0()` correctly sequences PIT -> PIC -> IOAPIC -> vCPU injection.

### 5.2 What AxVisor Has That Cloud Hypervisor Does Not

- **Userspace PIC emulation**: Cloud Hypervisor has no PIC at all, relying on KVM's kernel PIC. AxVisor's PIC is necessary because AxVisor IS the hypervisor and handles interrupts in userspace.
- **Userspace PIT emulation**: Same reasoning. Cloud Hypervisor relies on KVM kernel PIT.
- **PIC -> IOAPIC ExtINT bridging**: AxVisor's `x86_ioapic_assert_gsi()` with PIC callback is the correct way to handle ExtINT when the IOAPIC is emulated in userspace. Cloud Hypervisor never needs this because KVM handles it.

### 5.3 Gaps and Missing Pieces

1. **LAPIC LINT0 ExtINT path**: Cloud Hypervisor explicitly configures LINT0 = ExtINT via `set_lint()`. AxVisor's vLAPIC exists but the LINT0 ExtINT delivery path is not verified as working. The current injection path bypasses LAPIC LINT0 entirely by calling `vcpu.inject_interrupt_with_trigger()` directly. This is acceptable for the current bring-up but would need verification if a guest OS (like Linux) relies on LINT0 for PIC interrupts.

2. **PIC IRQ vector remapping**: QEMU's PIC irq_base is set by the OS during ICW2 init. AxVisor's PIC correctly tracks `irq_base` per chip. However, the current `inject_due_pit_irq0()` path does not go through LAPIC delivery, so the PIC vector (typically 0x20 for IRQ 0) is what gets injected. If the guest expects the LAPIC LINT0 ExtINT path to work, the vector would be the PIC's answer, which is correct.

3. **IOAPIC ExtINT vector caching**: QEMU's `ioapic_entry_parse()` calls `pic_read_irq()` every time the IOAPIC services a GSI 2 interrupt. AxVisor's `interrupt_for_entry()` does the same via the `read_extint_vector` callback. This is correct - the vector must be read fresh each time because the PIC state changes.

4. **KVM IRQ routing table**: Cloud Hypervisor relies on KVM's default IRQ routing table which maps GSI 0 -> IOAPIC pin 2. AxVisor does not use KVM's irq routing (it handles everything in userspace), so the GSI 0 -> pin 2 mapping is handled manually in the `inject_due_pit_irq0()` function. This is correct.

5. **port 0x61 PIT gate**: QEMU's PIT uses port 0x61 bit 0 to gate Channel 0. Cloud Hypervisor's i8042 stub returns 0x20 (bit 5) to fake Channel 2 gate. AxVisor's PIT stores `speaker_control` but does not use it to gate Channel 0. This is acceptable for current OVMF boot but may matter for Linux.

### 5.4 Key Design Differences

| Aspect | QEMU | Cloud Hypervisor | AxVisor |
|--------|------|-----------------|---------|
| PIC/PIT/IOAPIC ownership | All emulated in userspace (QEMU) or KVM kernel (KVM) | KVM kernel handles all; userspace IOAPIC for MSI only | All emulated in userspace (x86_vlapic crate) |
| IRQ routing | gsi_handler dual-forwards ISA IRQs to PIC + IOAPIC | KVM kernel routing table | Manual routing in x86_irq.rs |
| ExtINT delivery | IOAPIC reads PIC vector, LAPIC LINT0 bridges to CPU | KVM handles internally | IOAPIC reads PIC vector, direct vCPU injection |
| GSI 0 -> pin 2 | Hardcoded in ioapic_set_irq() | KVM default routing | Hardcoded as PIT_TIMER_GSI = 2 |

### 5.5 Summary of Minimum Semantics for Timer Bring-Up

For the PIT -> PIC -> IOAPIC -> LAPIC -> CPU chain to work:

1. **PIT**: Channel 0 periodic mode (mode 3) fires periodic ticks. Must support mode command (port 0x43), count latch, and read-back. Port 0x61 gating is optional for basic operation.

2. **PIC**: Master/slave pair must handle:
   - ICW1 (init command): reset, set init_state
   - ICW2 (irq_base): sets vector base (typically 0x20/0x28)
   - ICW3 (cascade): master knows slave is on IRQ 2
   - ICW4 (auto_eoi, special_fully_nested): optional for basic operation
   - `set_irq_line()`: edge/level triggered IRQ assertion
   - `get_irq()`: priority-based pending interrupt selection
   - `intack()`: acknowledge, set ISR, clear IRR
   - `read_irq_vector()`: cascade-aware vector read (the key function for ExtINT)
   - EOI (OCW2): clear ISR bit

3. **IOAPIC**: Must handle:
   - Redirection table programming (MMIO write to IOREDTBL)
   - ExtINT delivery mode: call PIC to get vector
   - Edge/level trigger support with Remote IRR bit
   - EOI: clear Remote IRR for level-triggered entries

4. **LAPIC LINT0**: (Optional for current AxVisor bring-up)
   - If used: LINT0 configured as ExtINT mode
   - Bridges IOAPIC ExtINT to CPU external interrupt
   - Current AxVisor bypasses this with direct injection

5. **GSI routing**: IRQ 0 -> IOAPIC pin 2 is mandatory for PC-compatible x86.
