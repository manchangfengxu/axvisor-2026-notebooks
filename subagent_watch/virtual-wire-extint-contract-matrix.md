# Virtual Wire / ExtINT Semantic Contract Matrix

Comparison of QEMU, Cloud Hypervisor, and AxVisor for the
"8259A -> LAPIC LINT0 (Virtual Wire / ExtINT)" interrupt delivery path,
specifically during legacy timer (PIT IRQ0) fallback.

## QEMU Findings

### How 8259A pending IRQ becomes visible on LAPIC LINT0

There are two delivery paths, selected by the guest's LVT_LINT0 configuration:

**Path A: IOAPIC GSI path with ExtINT delivery mode**

1. `gsi_handler()` in `x86-common.c:457-498` routes ISA IRQ lines to both
   PIC and IOAPIC simultaneously (lines 482-491).
2. `ioapic_set_irq()` in `ioapic.c:160-197` sets the IOAPIC IRR for the
   GSI pin.
3. `ioapic_service()` in `ioapic.c:97-149` calls `ioapic_entry_parse()`.
4. `ioapic_entry_parse()` in `ioapic.c:68-95` checks delivery_mode: when
   `delivery_mode == IOAPIC_DM_EXTINT` (line 83), it calls
   `pic_read_irq(isa_pic)` (line 84) to obtain the vector FROM the PIC,
   not from the IOAPIC redirection entry's vector field.
5. The resulting MSI message (with the PIC-provided vector) is sent to the
   LAPIC via `address_space_stl_le()` (line 144) which delivers into the
   LAPIC IRR as a Fixed delivery.

**Path B: Direct LAPIC LINT0 with ExtINT delivery mode**

1. `apic_deliver_pic_intr()` in `apic.c:185-203` is called when the LAPIC
   is configured to accept PIC interrupts.
2. On `level == 1`, it calls `apic_local_deliver(s, APIC_LVT_LINT0)`
   (line 188).
3. `apic_local_deliver()` in `apic.c:153-183` reads `s->lvt[APIC_LVT_LINT0]`
   and switches on `(lvt >> 8) & 7`:
   - `APIC_DM_EXTINT` (line 172): calls `cpu_interrupt(CPU(s->cpu),
     CPU_INTERRUPT_HARD)` (line 173). This does NOT inject a vector into
     the LAPIC IRR. Instead, it signals the CPU to poll for external
     interrupts.
   - `APIC_DM_FIXED` (line 176): calls `apic_set_irq(s, lvt & 0xff,
     trigger_mode)` (line 181). The vector comes from the LVT entry's
     low 8 bits.
4. The CPU's interrupt handler calls `apic_get_interrupt()` in
   `apic.c:706-737`. When `apic_check_pic(s)` (line 723) detects a PIC
   interrupt, it returns `-1`, deferring to the CPU to call `pic_read_irq()`
   directly.

### Who provides vector in ExtINT mode

- IOAPIC path: `pic_read_irq(isa_pic)` called inside
  `ioapic_entry_parse()` at `ioapic.c:84`.
- LAPIC LINT0 path: the CPU interrupt handler calls `pic_read_irq()`
  after `cpu_interrupt(CPU_INTERRUPT_HARD)` signals external interrupt.

### Who provides vector in Fixed Virtual Wire mode

The LVT_LINT0 entry's low 8 bits (the vector field) are used directly.
`apic_local_deliver()` at `apic.c:181`: `apic_set_irq(s, lvt & 0xff,
trigger_mode)`.

### When PIC ack happens

`pic_read_irq()` at `i8259.c:173-211` calls `pic_intack()` (line 183-184
for slave, line 195 for master). `pic_intack()` at `i8259.c:157-171` sets
the ISR bit (`s->isr |= (1 << irq)`) and clears the IRR bit
(`s->irr &= ~(1 << irq)`). This is the complete PIC acknowledge cycle.

In the IOAPIC ExtINT path, PIC ack happens inside `ioapic_entry_parse()`
before the interrupt reaches the LAPIC.

In the LAPIC LINT0 ExtINT path, PIC ack happens when the CPU reads the
vector from the PIC (deferred from `apic_get_interrupt` returning -1).

---

## Cloud Hypervisor Findings

### Does it implement the legacy PIC -> LAPIC LINT0 fallback?

**No.**

Cloud Hypervisor configures LAPIC LINT0 as ExtINT and LINT1 as NMI in
`interrupts.rs:26-38` (`set_lint()`):

```rust
klapic.set_klapic_reg(APIC_LVT0, set_apic_delivery_mode(lvt_lint0, APIC_MODE_EXTINT));
klapic.set_klapic_reg(APIC_LVT1, set_apic_delivery_mode(lvt_lint1, APIC_MODE_NMI));
```

However, Cloud Hypervisor relies entirely on KVM's kernel irqchip to handle
the actual PIC -> LAPIC LINT0 routing. The hypervisor sets up GSI routing
via KVM_IRQCHIP_PIC_MASTER, KVM_IRQCHIP_PIC_SLAVE, and KVM_IRQCHIP_IOAPIC
through KVM's kernel infrastructure. The Cloud Hypervisor IOAPIC
(`ioapic.rs`) translates IOAPIC redirection entries into MSI messages and
pushes them to KVM via `irqfd`. It validates `DeliveryMode::External`
(ioapic.rs:371) and embeds it in the MSI data, but KVM's kernel resolves
the actual PIC -> LAPIC path.

There is no userspace PIC emulation, no `pic_read_irq()` equivalent, and
no explicit LINT0 delivery path in the Cloud Hypervisor source. The
`LegacyUserspaceInterruptGroup` (`interrupt.rs:312-349`) is a thin wrapper
that calls `ioapic.service_irq()` but does not implement PIC ack or
LINT0 delivery.

---

## Comparison Matrix

| Semantic Item | QEMU | Cloud Hypervisor | AxVisor Current |
|---|---|---|---|
| **LINT0 mode detection** | Present. `apic_local_deliver()` checks `(lvt >> 8) & 7` for each LVT entry (`apic.c:163`). | Present. `set_lint()` writes `APIC_MODE_EXTINT` to LVT0 (`interrupts.rs:32`). | Present. `lint0_route_from_lvt()` checks delivery mode field of LVT_LINT0 (`vlapic.rs:61-75`). |
| **ExtINT vector from PIC** | Present. `pic_read_irq()` called in `ioapic_entry_parse()` (`ioapic.c:84`) and by CPU interrupt handler. | Not implemented in userspace. Delegated to KVM kernel. | Present. `read_irq_vector()` called in `inject_due_x86_pic_lint0_irq()` (`vm.rs:324`). |
| **Fixed Virtual Wire vector from LVT** | Present. `apic_local_deliver` Fixed case uses `lvt & 0xff` (`apic.c:181`). | N/A (no userspace delivery path). | Present. `LegacyPicLint0Route::Fixed { vector }` branch in `inject_due_x86_pic_lint0_irq()` (`vm.rs:329`). |
| **PIC ack (INTA)** | Present. `pic_intack()` sets ISR, clears IRR (`i8259.c:157-171`). Called by `pic_read_irq()`. | Not implemented in userspace. | Present. `intack()` called inside `read_irq_vector()` (`pic.rs:282-295`). Sets PIC ISR, clears PIC IRR. |
| **CPU_EXTERNAL_INTERRUPT signal** | Present. `cpu_interrupt(CPU(s->cpu), CPU_INTERRUPT_HARD)` for ExtINT (`apic.c:173`). | N/A (KVM handles). | Not needed. AxVisor directly injects the vector via `inject_interrupt_with_trigger()` (`vm.rs:340-343`), bypassing the need for an external-interrupt poll cycle. |
| **Vector source is PIC, not LVT/IOAPIC field** | Present. ExtINT path always calls `pic_read_irq()` for the vector (`ioapic.c:84`). | N/A. | Present. ExtINT branch reads vector from PIC (`vm.rs:330`), Fixed branch reads from LVT (`vm.rs:329`). |
| **IOAPIC ExtINT handling** | Present. `ioapic_entry_parse()` special-cases `IOAPIC_DM_EXTINT` (`ioapic.c:83-84`). | Validates `DeliveryMode::External` (`ioapic.rs:371`), embeds in MSI data. | Present. `interrupt_for_entry()` calls `read_extint_vector` callback (`vioapic.rs:64-72`). |
| **EOI handling** | Present. `apic_eoi()` clears ISR, broadcasts to IOAPIC for level-triggered (`apic.c:494-506`). | Present. `end_of_interrupt()` clears Remote IRR (`ioapic.rs:404-412`). | Present. `process_eoi()` clears ISR in virtual APIC page, returns vector for IOAPIC broadcast (`vlapic.rs:235-272`). `end_of_interrupt()` clears Remote IRR in vIOAPIC (`vioapic.rs:167-185`). |
| **Level-triggered support for PIC path** | Present. `apic_deliver_pic_intr` with level=0 clears IRR bit (`apic.c:193-198`). | N/A. | Partial. `inject_due_x86_pic_lint0_irq` always injects `EdgeTriggered` (`vm.rs:342`). Correct for PIT IRQ0 (edge-triggered), but would be wrong for level-triggered PIC IRQs. |
| **Spurious vector on no pending PIC IRQ** | Present. `pic_read_irq()` returns `irq_base + 7` when no IRQ pending (`i8259.c:196-199`). | N/A. | Present. `read_irq_vector()` returns `None` when no IRQ pending (`pic.rs:283`); the caller (`inject_due_x86_pic_lint0_irq`) returns `Ok(false)` and does not inject (`vm.rs:325`). |

---

## Explicit Answers

### Is "just inject a vector" sufficient?

**No, not in general.**

For the **ExtINT** delivery mode, simply injecting a vector is NOT sufficient.
The minimum semantic contract requires:

1. **PIC acknowledge at delivery time**: The PIC's ISR must be set and IRR
   must be cleared when the interrupt is delivered. Without this, the PIC
   retains the same IRQ as pending and could deliver it again on the next
   read. The PIC uses ISR state for priority arbitration and EOI.

2. **Vector must come from PIC**: The vector must be read from the PIC via
   `pic_read_irq()` / `intack()`, not taken from the LVT or IOAPIC
   entry's vector field. The PIC dynamically computes the vector from
   `irq_base + pending_irq_number`.

3. **Spurious handling**: If no IRQ is pending in the PIC when ExtINT is
   triggered, a spurious vector (master IRQ7 -> `irq_base + 7`) must be
   returned, and the PIC ISR must NOT be set for spurious interrupts.

For the **Fixed Virtual Wire** delivery mode, just injecting the vector IS
sufficient because the PIC is not involved -- the vector comes directly from
the LVT entry.

### If not, what minimum side effects are missing?

For a correct ExtINT implementation, these side effects are required:

| Side Effect | Why Required |
|---|---|
| PIC ISR set / IRR clear (ACK) | Prevents re-delivery of the same IRQ; enables correct priority arbitration |
| Vector sourced from PIC | The PIC computes the vector from `irq_base + irq_number`; hardcoding would bypass PIC state |
| Spurious vector when no IRQ pending | Hardware delivers spurious IRQ7 on master when no real IRQ is pending; ISR must NOT be set for spurious |

### AxVisor's current compliance

AxVisor's `inject_due_x86_pic_lint0_irq()` in `vm.rs:320-345` implements
all three side effects:

- PIC ack: `read_irq_vector()` calls `intack()` internally (`pic.rs:282-295`)
- Vector from PIC: ExtINT branch uses PIC-provided vector (`vm.rs:330`)
- Spurious handling: `read_irq_vector()` returns `None` when no IRQ pending,
  and the caller does not inject (`vm.rs:325`)

The one gap: `inject_due_x86_pic_lint0_irq` always uses
`EdgeTriggered` (`vm.rs:342`). For the PIT IRQ0 timer fallback (which is
edge-triggered), this is correct. For level-triggered PIC IRQs routed
through LINT0, this would need to respect the LINT0 trigger mode setting.

---

## All Conclusions: File Path + Function Name + Line Number

### QEMU

| Item | File | Function | Line |
|---|---|---|---|
| GSI routes to PIC + IOAPIC | `references/qemu/hw/i386/x86-common.c` | `gsi_handler()` | 457-498 |
| IOAPIC ExtINT reads PIC vector | `references/qemu/hw/intc/ioapic.c` | `ioapic_entry_parse()` | 83-84 |
| LAPIC LINT0 ExtINT -> CPU_INTERRUPT_HARD | `references/qemu/hw/intc/apic.c` | `apic_local_deliver()` | 172-173 |
| LAPIC LINT0 Fixed -> vector from LVT | `references/qemu/hw/intc/apic.c` | `apic_local_deliver()` | 176-181 |
| PIC ack (ISR set, IRR clear) | `references/qemu/hw/intc/i8259.c` | `pic_intack()` | 157-171 |
| PIC vector read + ack | `references/qemu/hw/intc/i8259.c` | `pic_read_irq()` | 173-211 |
| LAPIC checks PIC passthrough | `references/qemu/hw/intc/apic.c` | `apic_deliver_pic_intr()` | 185-203 |
| CPU polls PIC via apic_get_interrupt | `references/qemu/hw/intc/apic.c` | `apic_get_interrupt()` | 706-737 |
| Spurious IRQ vector | `references/qemu/hw/intc/i8259.c` | `pic_read_irq()` | 196-199 |

### Cloud Hypervisor

| Item | File | Function | Line |
|---|---|---|---|
| LINT0 set to ExtINT | `references/cloud-hypervisor/arch/src/x86_64/interrupts.rs` | `set_lint()` | 26-38 |
| IOAPIC validates External mode | `references/cloud-hypervisor/devices/src/ioapic.rs` | `update_entry()` | 363-371 |
| No userspace PIC or LINT0 delivery | (absent) | (absent) | -- |

### AxVisor

| Item | File | Function | Line |
|---|---|---|---|
| LINT0 route detection | `tgoskits/virtualization/x86_vlapic/src/vlapic.rs` | `lint0_route_from_lvt()` | 61-75 |
| LINT0 route query | `tgoskits/virtualization/x86_vlapic/src/vlapic.rs` | `VirtualApicRegs::lint0_route()` | 195-197 |
| Vcpu LINT0 route pass-through | `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs` | `VmxVcpu::lint0_route()` | 245-247 |
| PIC intack (ISR set, IRR clear) | `tgoskits/virtualization/x86_vlapic/src/pic.rs` | `PicChipState::intack()` | 116-128 |
| PIC read_irq_vector (ack + vector) | `tgoskits/virtualization/x86_vlapic/src/pic.rs` | `PicState::read_irq_vector()` | 282-295 |
| PIC public API | `tgoskits/virtualization/x86_vlapic/src/pic.rs` | `EmulatedPic8259::read_irq_vector()` | 343-344 |
| PIC assertion | `tgoskits/virtualization/x86_vlapic/src/pic.rs` | `EmulatedPic8259::assert_irq()` | 338-340 |
| Device layer PIC read | `tgoskits/virtualization/axdevice/src/device.rs` | `x86_pic_read_irq_vector()` | 573-574 |
| LINT0 IRQ injection logic | `tgoskits/virtualization/axvm/src/vm.rs` | `inject_due_x86_pic_lint0_irq()` | 320-345 |
| PIT timer -> PIC -> LINT0 flow | `tgoskits/virtualization/axvm/src/vm.rs` | `inject_due_x86_pit_irq0()` | 273-317 |
| vIOAPIC ExtINT handling | `tgoskits/virtualization/x86_vlapic/src/vioapic.rs` | `IoApicState::interrupt_for_entry()` | 48-102 |
| vIOAPIC EOI broadcast | `tgoskits/virtualization/x86_vlapic/src/vioapic.rs` | `EmulatedIoApic::end_of_interrupt()` | 167-185 |
| VMCS event injection | `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs` | `VmxVcpu::inject_pending_events()` | 1116-1137 |
| vLAPIC ISR/TMR recording | `tgoskits/virtualization/x86_vlapic/src/vlapic.rs` | `VirtualApicRegs::accept_interrupt()` | 430-443 |
| vLAPIC EOI processing | `tgoskits/virtualization/x86_vlapic/src/vlapic.rs` | `VirtualApicRegs::process_eoi()` | 235-272 |
