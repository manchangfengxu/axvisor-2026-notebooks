# Intel VMX APIC Virtualization Contract: Guest xAPIC Write to APIC_LVT0

## Scope

This document traces the exact processor behavior when a guest in xAPIC mode writes
to GPA 0xFEE00350 (APIC_LVT0 register, offset 0x350 within the 4-KByte APIC page).
All conclusions cite SDM chapter/section and page, or KVM source file and line number.

---

## 1. Control Bit Positions (Confirmed from SDM)

### Primary Processor-Based VM-Execution Controls (Table 25-6)

| Bit | Name | Description |
|-----|------|-------------|
| 19 | CR8-load exiting | VM exit on MOV to CR8 |
| 20 | CR8-store exiting | VM exit on MOV from CR8 |
| 21 | Use TPR shadow | Enables TPR virtualization and other APIC-virtualization features |
| 31 | Activate secondary controls | Enables secondary processor-based controls |

*SDM Vol. 3C, Section 25.5, Table 25-6 (confirmed by pdftotext extraction from the
combined SDM PDF, Vol. 3C pages 25-6 through 25-7).*

### Secondary Processor-Based VM-Execution Controls (Table 25-7)

| Bit | Name | Description |
|-----|------|-------------|
| 0 | Virtualize APIC accesses | Treats accesses to the APIC-access page specially |
| 4 | Virtualize x2APIC mode | Treats RDMSR/WRMSR to APIC MSRs specially |
| 8 | APIC-register virtualization | Virtualizes certain APIC register accesses |
| 9 | Virtual-interrupt delivery | Enables evaluation/delivery of virtual interrupts |

*SDM Vol. 3C, Section 25.6, Table 25-7 (confirmed by pdftotext extraction).*

### KVM Cross-Reference (for bit position sanity check)

| VMCS control | KVM define | VMX_FEATURE value |
|---|---|---|
| Use TPR shadow (pri bit 21) | `CPU_BASED_TPR_SHADOW` | `VMX_FEATURE_VIRTUAL_TPR = (1*32+21)` |
| Virtualize APIC accesses (sec bit 0) | `SECONDARY_EXEC_VIRTUALIZE_APIC_ACCESSES` | `VMX_FEATURE_VIRT_APIC_ACCESSES = (2*32+0)` |
| Virtualize x2APIC mode (sec bit 4) | `SECONDARY_EXEC_VIRTUALIZE_X2APIC_MODE` | `VMX_FEATURE_VIRTUAL_X2APIC = (2*32+4)` |
| APIC-register virt (sec bit 8) | `SECONDARY_EXEC_APIC_REGISTER_VIRT` | `VMX_FEATURE_APIC_REGISTER_VIRT = (2*32+8)` |
| Virtual-interrupt delivery (sec bit 9) | `SECONDARY_EXEC_VIRTUAL_INTR_DELIVERY` | `VMX_FEATURE_VIRT_INTR_DELIVERY = (2*32+9)` |

*`linux/arch/x86/include/asm/vmxfeatures.h` lines 50, 62, 66, 70-71.*

---

## 2. Dependency Diagram

```
                        Use TPR shadow  (pri bit 21)
                              |
              +---------------+----------------+----------------+
              |               |                |                |
              v               v                v                v
   APIC-register virt   Virtualize      Virtual-interrupt   (on x86_64:
   (sec bit 8)          x2APIC mode     delivery (sec bit 9)  independent)
                        (sec bit 4)
                              |
                              v
                    IPI virtualization
                    (tertiary bit 4)
```

### Confirmed Dependencies

**TPR shadow (bit 21) is a prerequisite for bits 8, 4, and 9:**

> SDM Vol. 3C, Section 25.6.8 (page 25-15):
> "Several processor-based VM-execution controls (see Section 25.6.2) control such
> accesses. These are 'use TPR shadow', 'virtualize APIC accesses', 'virtualize x2APIC
> mode', 'virtual-interrupt delivery', 'APIC-register virtualization', and 'IPI
> virtualization'."

> SDM Vol. 3C, Section 25.6.8 (page 25-15), TPR threshold paragraph:
> "The TPR threshold exists only on processors that support the 1-setting of the
> 'use TPR shadow' VM-execution control."

*Additionally, Section 30.4.3.1 (page 30-9) requires "use TPR shadow" = 1 as the
first condition for any APIC-access page write to be virtualized rather than causing
an APIC-access VM exit.*

**KVM enforces this dependency:**

> `linux/arch/x86/kvm/vmx/vmx.c` lines 2762-2766:
> ```c
> if (!(_cpu_based_exec_control & CPU_BASED_TPR_SHADOW))
>     _cpu_based_2nd_exec_control &= ~(
>         SECONDARY_EXEC_APIC_REGISTER_VIRT |
>         SECONDARY_EXEC_VIRTUALIZE_X2APIC_MODE |
>         SECONDARY_EXEC_VIRTUAL_INTR_DELIVERY);
> ```

**"Virtualize APIC accesses" (bit 0) is independent of "Use TPR shadow" on x86_64:**

> `linux/arch/x86/kvm/vmx/vmx.c` lines 2756-2760:
> ```c
> #ifndef CONFIG_X86_64
>     if (!(_cpu_based_2nd_exec_control &
>             SECONDARY_EXEC_VIRTUALIZE_APIC_ACCESSES))
>         _cpu_based_exec_control &= ~CPU_BASED_TPR_SHADOW;
> #endif
> ```
> On x86_64, TPR shadow can be set without "Virtualize APIC accesses".

**"IPI virtualization" (tertiary bit 4) depends on APICv being active:**

> `linux/arch/x86/kvm/vmx/vmx.c` lines 4647-4648:
> ```c
> if (!enable_ipiv || !kvm_vcpu_apicv_active(&vmx->vcpu))
>     exec_control &= ~TERTIARY_EXEC_IPI_VIRT;
> ```

### Summary of Dependencies

| Control bit | Requires |
|---|---|
| APIC-register virt (sec 8) | Use TPR shadow (pri 21) |
| Virtualize x2APIC mode (sec 4) | Use TPR shadow (pri 21) |
| Virtual-interrupt delivery (sec 9) | Use TPR shadow (pri 21) |
| IPI virt (tertiary 4) | APICv active (which implies sec 8 or sec 9) |
| Virtualize APIC accesses (sec 0) | Independent on x86_64 |

---

## 3. Behavior Matrix for Guest xAPIC MMIO Write to APIC_LVT0 (offset 0x350)

The APIC_LVT0 register lives at offset 0x350 within the 4-KByte APIC MMIO page
(physical 0xFEE00000, 4096 bytes).

### Step 1: Does the access reach the APIC-access page?

SDM Vol. 3C, Section 30.4.5 (page 30-12):

> "The 1-setting of the 'virtualize APIC accesses' VM-execution is guaranteed to
> apply only if translations to the APIC-access address use a 4-KByte page."

> "If EPT is in use, any guest-physical address that translates to an address on
> the APIC-access page should use a 4-KByte page. Any access to a linear address
> that translates to a guest-physical address that in turn translates to the
> APIC-access page using a larger page may operate as if the 'virtualize APIC
> accesses' VM-execution control were 0."

**Condition**: "Virtualize APIC accesses" = 1 AND the GPA 0xFEE00350 translates
(via EPT) to the APIC-access page AND the translation uses a 4-KByte page.

If this condition is NOT met, the write behaves as a normal memory store (no
APIC-related interception).

### Step 2: If the access reaches the APIC-access page -- does it cause an
APIC-access VM exit?

SDM Vol. 3C, Section 30.4.3.1 (page 30-9):

> "A write access to the APIC-access page causes an APIC-access VM exit if any of
> the following are true:
> - The 'use TPR shadow' VM-execution control is 0.
> - The access is more than 32 bits in size.
> - The access is part of an operation for which the processor has already
>   virtualized a write (with a different page offset or a different size) to
>   the APIC-access page.
> - The access is not entirely contained within the low 4 bytes of a naturally
>   aligned 16-byte region. That is, bits 3:2 of the access's address are 0,
>   and the same is true of the address of the highest byte accessed."

### Step 3: If no APIC-access exit -- is the write virtualized?

SDM Vol. 3C, Section 30.4.3.1 (page 30-9):

> "If none of the above are true, whether a write access is virtualized depends
> on the settings of the 'APIC-register virtualization', 'virtual-interrupt
> delivery', and 'IPI virtualization' VM-execution controls:
> - A write access is virtualized if its page offset is 080H (task priority)
>   regardless of the settings of the 'APIC-register virtualization' and
>   'virtual-interrupt delivery' VM-execution controls.
> - If the 'virtual-interrupt delivery' VM-execution control is 1, a write
>   access is virtualized if its page offset is 0B0H (end of interrupt) or
>   300H (interrupt command -- low).
> - If the 'IPI virtualization' VM-execution control is 1, a write access
>   is virtualized if its page offset is 300H.
> - If the 'APIC-register virtualization' VM-execution control is 1, a write
>   access is virtualized if it is entirely within one of the following ranges
>   of offsets:
>   - 020H-023H (local APIC ID);
>   - 080H-083H (task priority);
>   - 0B0H-0B3H (end of interrupt);
>   - 0D0H-0D3H (logical destination);
>   - 0E0H-0E3H (destination format);
>   - 0F0H-0F3H (spurious-interrupt vector);
>   - 280H-283H (error status);
>   - 300H-303H or 310H-313H (interrupt command);
> - 320H-323H, 330H-333H, 340H-343H, 350H-353H, 360H-363H, or 370H-373H
>   (LVT entries);
> - 380H-383H (initial count); or
> - 3E0H-3E3H (divide configuration).
> In all other cases, the access causes an APIC-access VM exit."

**Critical finding**: Offset 0x350 (APIC_LVT0) IS listed in the
"APIC-register virtualization" virtualizable range (350H-353H, LVT entries).

Wait -- re-reading more carefully:

SDM Vol. 3C, Section 30.4.3.1 (page 30-9), the indentation shows the LVT
entries are listed under the "APIC-register virtualization" condition:

> "If the 'APIC-register virtualization' VM-execution control is 1, a write
> access is virtualized if it is entirely within one of the following ranges:
> - ...
> - 320H-323H, 330H-333H, 340H-343H, 350H-353H, 360H-363H, or 370H-373H
>   (LVT entries);
> - ..."

This means: **If "APIC-register virtualization" = 1, a write to offset 350H IS
virtualized (no VM exit)**, because 350H falls within the 350H-353H range.

**CORRECTION**: Offset 350H IS in the virtualizable list when
"APIC-register virtualization" = 1.

### Complete Behavior Matrix

| TPR shadow | APIC reg virt | Virt int deliv | IPI virt | Offset 350H outcome |
|---|---|---|---|---|
| 0 | X | X | X | **APIC-access VM exit** (reason 44). Write not virtualized. |
| 1 | 0 | X | X | **APIC-access VM exit** (reason 44). Offset 350H not virtualizable. |
| 1 | 1 | X | X | **No VM exit**. Write is virtualized. Data goes to virtual-APIC page. |

**"X" means the value of the control is irrelevant to the outcome.**

### Why these outcomes

1. **TPR shadow = 0**: Section 30.4.3.1 says the very first condition for
   causing an APIC-access VM exit is "The 'use TPR shadow' VM-execution control
   is 0." This overrides all other considerations.

2. **TPR shadow = 1, APIC-register virt = 0**: After passing the "use TPR
   shadow" gate, Section 30.4.3.1 says the processor checks whether offset 350H
   is virtualizable. Without "APIC-register virtualization" = 1, offset 350H
   is not in any of the virtualizable offset lists (080H for TPR, 0B0H/300H for
   VID, 300H for IPI virt, and the APIC-register virt list). So the processor
   falls through to "In all other cases, the access causes an APIC-access VM exit."

3. **TPR shadow = 1, APIC-register virt = 1**: Section 30.4.3.1 explicitly
   lists 350H-353H (LVT entries) as virtualizable when "APIC-register
   virtualization" = 1. The write is virtualized -- no VM exit occurs.

---

## 4. Cases Where Write Succeeds Without APIC_ACCESS Exit

A write to offset 350H succeeds without generating an APIC-access exit if and
only if:

1. **"Virtualize APIC accesses" (sec bit 0) = 1**
   - SDM Vol. 3C, Section 30.4.3.1: the virtualization path is only entered if
     the access reaches the APIC-access page, which requires this bit.

2. **"Use TPR shadow" (pri bit 21) = 1**
   - SDM Vol. 3C, Section 30.4.3.1, first condition for APIC-access VM exit.

3. **Access size <= 32 bits and alignment within a 16-byte boundary**
   - SDM Vol. 3C, Section 30.4.3.1, conditions 2 and 4.

4. **"APIC-register virtualization" (sec bit 8) = 1**
   - SDM Vol. 3C, Section 30.4.3.1: offset 350H is only virtualizable under
     this control. Without it, offset 350H falls through to "In all other cases,
     the access causes an APIC-access VM exit."

**Minimum required set**: "Virtualize APIC accesses" + "Use TPR shadow" +
"APIC-register virtualization" all equal to 1, with a standard 32-bit aligned
write.

**No VM exit at all** means the processor writes the value directly to offset
350H on the **virtual-APIC page** (not the APIC-access page).

SDM Vol. 3C, Section 30.4.3 (page 30-9):
> "The processor virtualizes a write access to the APIC-access page by writing
> data to the corresponding page offset on the virtual-APIC page."

---

## 5. State Location Analysis

### Virtual-APIC Page (VMCS field: Virtual-APIC address)

When a write to APIC_LVT0 is virtualized (no VM exit), the new value lives at
offset 0x350 on the **virtual-APIC page**. This is a 4-KByte region whose
physical address is specified in the VMCS "Virtual-APIC address" field.

SDM Vol. 3C, Section 25.6.8 (page 25-15):
> "Virtual-APIC address (64 bits). This field contains the physical address of
> the 4-KByte virtual-APIC page. The processor uses the virtual-APIC page to
> virtualize certain accesses to APIC registers and to manage virtual interrupts;
> see Chapter 30."

The VMM can read offset 0x350 on the virtual-APIC page to observe the LVT0
state. The hardware updates it transparently.

### APIC-access Page (VMCS field: APIC-access address)

The APIC-access page is a **trap page** -- it exists solely to trigger
VM exits. The processor does NOT store APIC register state there. When a write
is virtualized, the processor writes to the virtual-APIC page, not the
APIC-access page.

SDM Vol. 3C, Section 25.6.8 (page 25-15):
> "APIC-access address (64 bits). This field contains the physical address of
> the 4-KByte APIC-access page. If the 'virtualize APIC accesses' VM-execution
> control is 1, access to this page may cause VM exits or be virtualized by the
> processor. See Section 30.4."

### Processor Shadow (TPR shadow)

The "TPR shadow" mechanism is specifically for the TPR register (offset 080H /
CR8). The processor maintains an internal shadow of VTPR bits 7:4. Other LVT
registers like LVT0 have no processor shadow -- they are fully stored in the
virtual-APIC page.

SDM Vol. 3C, Section 25.6.8 (page 25-15), TPR threshold:
> "Bits 3:0 of this field determine the threshold below which bits 7:4 of VTPR
> (see Section 30.1.1) cannot fall."

### Summary Table

| Control state | Where LVT0 value lives | Who can read it |
|---|---|---|
| TPR shadow = 1, APIC reg virt = 1 | virtual-APIC page, offset 0x350 | Processor hardware (for interrupt delivery) + VMM (via VMCS pointer) |
| TPR shadow = 1, APIC reg virt = 0 | Causes APIC-access VM exit | VMM reads it from the VMCS exit qualification / emulates |
| TPR shadow = 0 | Causes APIC-access VM exit | VMM must emulate the full instruction |

---

## 6. Required / Not-Required / Mutually-Exclusive Conditions

### Required Conditions (for virtualized write, no VM exit)

| Condition | Control | Bit | SDM reference |
|---|---|---|---|
| Virtualize APIC accesses = 1 | Secondary, bit 0 | sec 0 | Section 30.4.3.1, page 30-9 |
| Use TPR shadow = 1 | Primary, bit 21 | pri 21 | Section 30.4.3.1, page 30-9 (first exit condition) |
| APIC-register virtualization = 1 | Secondary, bit 8 | sec 8 | Section 30.4.3.1, page 30-9 (LVT range virtualizable only when this = 1) |
| Access size <= 32 bits | -- | -- | Section 30.4.3.1, page 30-9 (second exit condition) |
| Alignment within 16-byte boundary | -- | -- | Section 30.4.3.1, page 30-9 (fourth exit condition) |
| Translation uses 4-KByte page | -- | -- | Section 30.4.5, page 30-12 |

### Not-Required Conditions

| Condition | Why not required for offset 350H |
|---|---|
| Virtual-interrupt delivery (sec bit 9) | Not needed for LVT0 offset virtualization. VID only virtualizes offsets 0B0H and 300H. |
| Virtualize x2APIC mode (sec bit 4) | Irrelevant for xAPIC MMIO writes. This controls RDMSR/WRMSR behavior. |
| IPI virtualization (tertiary bit 4) | Irrelevant for LVT0. Only virtualizes offset 300H (ICR low). |

### Mutually-Exclusive Conditions

There are no mutually-exclusive control bit combinations among the six controls
listed. Any subset of them can be set simultaneously without architectural
conflict. However:

- If "Use TPR shadow" = 0, the other APICv bits (8, 9, 4) are architecturally
  meaningless for MMIO writes because all APIC-access page writes cause
  APIC-access exits unconditionally (Section 30.4.3.1, page 30-9).
- The KVM source confirms that bits 8, 9, 4 are forcibly cleared when bit 21 is
  clear (`linux/arch/x86/kvm/vmx/vmx.c` lines 2762-2766).

### The Two VM Exit Types and Their Difference

| Exit type | Exit reason | When it fires for offset 350H | VMM observation |
|---|---|---|---|
| APIC-access VM exit | Reason 44 | TPR shadow = 0, OR TPR shadow = 1 but APIC reg virt = 0 | Exit qualification contains: access type + offset. VMM must emulate the write from the instruction stream. |
| APIC-write VM exit | Reason 45 | TPR shadow = 1 AND APIC reg virt = 1, but offset not virtualizable | -- (offset 350H IS virtualizable when APIC reg virt = 1, so this exit type does NOT fire for LVT0) |

**For offset 350H specifically**: APIC-write VM exit (reason 45) does NOT occur
when "APIC-register virtualization" = 1, because offset 350H-353H is in the
virtualizable range. The write is fully virtualized.

SDM Vol. 3C, Section 30.4.3.2 (page 30-10):
> "APIC-write emulation takes priority over system-management interrupts (SMIs),
> INIT signals, and lower priority events."

SDM Vol. 3C, Section 30.4.3.3 (page 30-11):
> "APIC-write VM exits are invoked by APIC-write emulation, and APIC-write
> emulation occurs after an operation that performs a write access to the
> APIC-access page. Because of this, every APIC-write VM exit is trap-like: it
> occurs after completion of the operation containing the write access that
> caused the VM exit."

### KVM Handler Confirmation

> `linux/arch/x86/kvm/vmx/vmx.c` lines 5831-5850, `handle_apic_access()`:
> The handler checks `exit_qualification` for access type and offset.
> For EOI writes, it short-circuits with `kvm_lapic_set_eoi()`.
> For other offsets (including LVT0), it calls `kvm_emulate_instruction()`.

> `linux/arch/x86/kvm/vmx/vmx.c` lines 5863-5878, `handle_apic_write()`:
> Comment: "APIC-write VM-Exit is trap-like, KVM doesn't need to advance RIP
> and hardware has done any necessary aliasing, offset adjustments, etc...
> for the access. I.e. the correct value has already been written to the
> vAPIC page for the correct 16-byte chunk."

### KVM Virtual-APIC Mode Selection

> `linux/arch/x86/kvm/vmx/vmx.c` lines 6872-6929, `vmx_set_virtual_apic_mode()`:
> - LAPIC_MODE_XAPIC: sets `SECONDARY_EXEC_VIRTUALIZE_APIC_ACCESSES`
> - LAPIC_MODE_X2APIC: sets `SECONDARY_EXEC_VIRTUALIZE_X2APIC_MODE`
> These are mutually exclusive in practice (xAPIC vs x2APIC).

---

## 7. Key Takeaway

For a guest xAPIC mode write to APIC_LVT0 (GPA 0xFEE00350):

1. **TPR shadow = 0**: Always causes APIC-access VM exit (reason 44). No
   virtualization possible.

2. **TPR shadow = 1, APIC reg virt = 0**: Still causes APIC-access VM exit
   (reason 44) because offset 350H is not in any virtualizable offset list
   without this control.

3. **TPR shadow = 1, APIC reg virt = 1**: Write is fully virtualized. Processor
   writes to the virtual-APIC page at offset 0x350. No VM exit. The VMM can
   observe the LVT0 value by reading the virtual-APIC page through the VMCS
   Virtual-APIC address pointer.

There is no intermediate state where the write "succeeds" (reaches the
virtual-APIC page) but the VMM cannot observe it. If APIC-register
virtualization = 1, both the processor and the VMM (via the virtual-APIC page
pointer) can see the new value.

---

## Citations Index

| Reference | Location |
|---|---|
| Table 25-6: Primary processor-based controls | SDM Vol. 3C, Section 25.5, Table 25-6 (bit 21 = Use TPR shadow) |
| Table 25-7: Secondary processor-based controls | SDM Vol. 3C, Section 25.6, Table 25-7 (bit 0, 4, 8, 9) |
| APIC virtualization controls | SDM Vol. 3C, Section 25.6.8 (page 25-15) |
| Write virtualization conditions | SDM Vol. 3C, Section 30.4.3.1 (page 30-9) |
| APIC-write emulation | SDM Vol. 3C, Section 30.4.3.2 (page 30-10) |
| APIC-write VM exits | SDM Vol. 3C, Section 30.4.3.3 (page 30-11) |
| Page size / TLB requirements | SDM Vol. 3C, Section 30.4.5 (page 30-12) |
| MSR-based APIC virtualization | SDM Vol. 3C, Section 30.5 (page 30-14) |
| KVM dependency logic | `linux/arch/x86/kvm/vmx/vmx.c` lines 2756-2766 |
| KVM APIC-access handler | `linux/arch/x86/kvm/vmx/vmx.c` lines 5831-5850 |
| KVM APIC-write handler | `linux/arch/x86/kvm/vmx/vmx.c` lines 5863-5878 |
| KVM virtual-APIC mode | `linux/arch/x86/kvm/vmx/vmx.c` lines 6872-6929 |
| VMX feature bit definitions | `linux/arch/x86/include/asm/vmxfeatures.h` lines 50, 62, 66, 70-71, 92 |
