# VMX APIC Page Audit: Why Zero LINT0 Write Logs

**Date:** 2026-06-19
**Question:** Linux wrote APIC_LVT0 6 times, but we see zero "[VLAPIC] LINT0 write" logs. Where in our current code is this most likely broken?

---

## 1. All VMCS Control Bits Set by AxVisor

### Primary Processor-Based Controls (`vcpu.rs:742-758`)

| Bit | Status | Code location |
|-----|--------|--------------|
| `USE_IO_BITMAPS` | Always set | `vcpu.rs:747` |
| `HLT_EXITING` | Always set | `vcpu.rs:748` |
| `USE_MSR_BITMAPS` | Always set | `vcpu.rs:749` |
| `USE_TPR_SHADOW` | Always set | `vcpu.rs:750` |
| `SECONDARY_CONTROLS` | Always set | `vcpu.rs:751` |
| `CR3_LOAD_EXITING` | Always cleared | `vcpu.rs:753` |
| `CR3_STORE_EXITING` | Always cleared | `vcpu.rs:754` |
| `CR8_LOAD_EXITING` | Always cleared | `vcpu.rs:755` |
| `CR8_STORE_EXITING` | Always cleared | `vcpu.rs:756` |

### Secondary Processor-Based Controls (`vcpu.rs:761-796`)

| Bit | Status | Code location | Notes |
|-----|--------|--------------|-------|
| `ENABLE_EPT` | Always set | `vcpu.rs:762` | |
| `UNRESTRICTED_GUEST` | Always set | `vcpu.rs:762` | |
| `VIRTUALIZE_APIC` | **Conditional** | `vcpu.rs:764,768` | Set if `secondary_control_bits_allowed()` returns true. No CPUID feature check. |
| `VIRTUALIZE_APIC_REGISTER` | **Conditional** | `vcpu.rs:765,768` | Set if `secondary_control_bits_allowed()` returns true. No CPUID feature check. |
| `VIRTUAL_INTERRUPT_DELIVERY` | **Conditional** | `vcpu.rs:766,768` | Set if `secondary_control_bits_allowed()` returns true. No CPUID feature check. |
| `ENABLE_RDTSCP` | Conditional | `vcpu.rs:772-777` | Requires CPUID feature check. |
| `ENABLE_INVPCID` | Conditional | `vcpu.rs:778-783` | Requires CPUID feature check. |
| `ENABLE_XSAVES_XRSTORS` | Conditional | `vcpu.rs:784-789` | Requires CPUID feature check. |

**Key observation:** `VIRTUALIZE_APIC`, `VIRTUALIZE_APIC_REGISTER`, and `VIRTUAL_INTERRUPT_DELIVERY` do NOT have explicit CPUID feature checks. They are set if the VMX capability MSR allows them. On modern Intel CPUs supporting VMX, these bits are "allowed-1" and will be set.

### VMCS 64-bit Control Fields (`vcpu.rs:854-860`)

| Field | Value | Code location |
|-------|-------|--------------|
| `VIRT_APIC_ADDR` | HPA of dynamically allocated virtual-APIC page | `vcpu.rs:854` |
| `APIC_ACCESS_ADDR` | HPA of static `VIRTUAL_APIC_ACCESS_PAGE` | `vcpu.rs:855-856` |
| `EOI_EXIT0` | `u64::MAX` (all vectors) | `vcpu.rs:857` |
| `EOI_EXIT1` | `u64::MAX` (all vectors) | `vcpu.rs:858` |
| `EOI_EXIT2` | `u64::MAX` (all vectors) | `vcpu.rs:859` |
| `EOI_EXIT3` | `u64::MAX` (all vectors) | `vcpu.rs:860` |

### MSR Bitmap Configuration (`vcpu.rs:570-594`)

| MSR Range | Intercepted | Code location |
|-----------|------------|--------------|
| `IA32_APIC_BASE` (0x1B) | Read + Write | `vcpu.rs:572-574` |
| `IA32_UMWAIT_CONTROL` (0xE1) | Read + Write | `vcpu.rs:579-583` |
| `IA32_MTRR_DEF_TYPE` (0x2FF) | Read + Write | `vcpu.rs:584-586` |
| x2APIC MSRs (0x800-0x83F) | Read + Write | `vcpu.rs:589-593` |

### `set_control` Logic (`vmcs.rs:654-693`)

The function takes the host MSR's allowed-0 and allowed-1 bits to determine the final VMCS value:
- `fixed1 = allowed0` (bits that MUST be 1)
- `flexible = !allowed0 & allowed1` (bits that CAN be 0 or 1)
- `default = flexible & old_value` (flexible bits preserve host MSR initial value)
- `final = fixed1 | default | set`

For `VIRTUALIZE_APIC` (secondary bit 0): if the host MSR's old_value has bit 0 set (which it does on modern Intel CPUs), the bit is preserved in the `default` mask. Additionally, `VIRTUALIZE_APIC` is in the `set` mask. So the final value includes bit 0.

**Net effect:** On any Intel CPU that supports VMX with APIC virtualization, VIRTUALIZE_APIC WILL be enabled.

---

## 2. APIC-Access Address vs Virtual-APIC Address

These are **two different pages**:

### APIC-Access Page (hardware use)
- **Location:** `x86_vlapic/src/lib.rs:48-50` — static `VIRTUAL_APIC_ACCESS_PAGE`
- **Allocation:** `#[repr(align(4096))]` static, zero-initialized at compile time
- **Physical address:** `EmulatedLocalApic::virtual_apic_access_addr()` at `lib.rs:112-116`
  - Calls `host::virt_to_phys()` on the static array's pointer
- **VMCS field:** `APIC_ACCESS_ADDR` at `vmcs.rs:135` (encoding 0x2014)
- **Written at:** `vcpu.rs:855-856`
- **Purpose:** The CPU compares this HPA against the GPA-after-EPT-translation. If they match and VIRTUALIZE_APIC is enabled, the CPU generates an APIC_ACCESS exit.

### Virtual-APIC Page (vLAPIC state)
- **Location:** `x86_vlapic/src/vlapic.rs:108-135` — per-vCPU `PhysFrame::alloc_zero()`
- **Physical address:** `VirtualApicRegs::virtual_apic_page_addr()` at `vlapic.rs:145-147`
  - Returns `self.apic_page.start_paddr()`
- **VMCS field:** `VIRT_APIC_ADDR` at `vmcs.rs:133` (encoding 0x2012)
- **Written at:** `vcpu.rs:854`
- **Purpose:** The CPU reads/writes this page for hardware-virtualized APIC registers (TPR shadow, EOI, ICR). The vLAPIC also writes register state here.

**Critical distinction:** The APIC-access page is what the CPU uses to decide whether to generate an APIC_ACCESS exit. The virtual-APIC page is where the CPU stores hardware-virtualized APIC register state. They must be separate physical pages.

---

## 3. EPT Mapping for 0xFEE00000 Region

### APIC-access page mapping (`vm.rs:480-485`)

```rust
#[cfg(all(target_arch = "x86_64", feature = "vmx"))]
inner_mut.address_space.map_linear(
    GuestPhysAddr::from(X86_APIC_ACCESS_GPA),   // 0xFEE00000
    x86_apic_access_page_addr(),                   // HPA of VIRTUAL_APIC_ACCESS_PAGE
    ax_memory_addr::PAGE_SIZE_4K,                  // 4096 bytes
    MappingFlags::DEVICE | MappingFlags::READ | MappingFlags::WRITE,
)?;
```

This creates a 4KB EPT mapping: GPA 0xFEE00000 -> HPA of the static APIC-access page.

### Passthrough device mapping (`vm.rs:467-477`)

Pass-through device regions are mapped BEFORE the APIC-access page. They use identity mapping:
```rust
inner_mut.address_space.map_linear(
    GuestPhysAddr::from(*gpa),   // e.g., 0xFEC00000 for IOAPIC
    HostPhysAddr::from(*gpa),     // identity: HPA = GPA
    *len,
    MappingFlags::DEVICE | MappingFlags::READ | MappingFlags::WRITE | MappingFlags::USER,
)?;
```

### IOAPIC vs Local APIC MMIO

- IOAPIC (GPA 0xFEC00000): Identity-mapped as pass-through device. Different GPA, no conflict with APIC-access page.
- Local APIC MMIO (GPA 0xFEE00000): Mapped to APIC-access page HPA (NOT identity). This is correct because the host has no real APIC hardware at 0xFEE00000.

### EPT conflict check

The APIC-access page mapping (4KB) is applied AFTER all pass-through mappings. If a pass-through device included the 0xFEE00000 range and was mapped with a huge page (2MB or 1GB), the subsequent 4KB mapping would need to split it. The `map_linear` backend uses `allow_huge = true` (`axaddrspace/.../linear.rs:45`), so the 4KB mapping might be subsumed by a pre-existing huge page entry.

**Potential conflict:** If the VM config defines a pass-through device that includes GPA 0xFEE00000 in its range, the identity-mapped pass-through entry could conflict with the APIC-access page mapping. The 4KB APIC-access mapping is applied last and should win, but page table splitting semantics must be verified.

---

## 4. APIC-Access Page Allocation and Physical Address

### Allocation

The APIC-access page is a **static** allocation in `x86_vlapic/src/lib.rs`:
```rust
#[repr(align(4096))]
struct APICAccessPage([u8; PAGE_SIZE_4K]);

static VIRTUAL_APIC_ACCESS_PAGE: APICAccessPage = APICAccessPage([0; PAGE_SIZE_4K]);
```

- Compile-time zero-initialized, page-aligned
- Never freed (static lifetime)
- Physical address obtained at runtime via `host::virt_to_phys()`

### Physical address resolution

```rust
pub fn virtual_apic_access_addr() -> HostPhysAddr {
    host::virt_to_phys(HostVirtAddr::from_usize(
        VIRTUAL_APIC_ACCESS_PAGE.0.as_ptr() as usize,
    ))
}
```

This converts the static array's host virtual address to a host physical address. The result is written to `VMCS APIC_ACCESS_ADDR` (vmcs.rs:135, encoding 0x2014).

### Virtual-APIC page (per-vCPU)

```rust
let apic_frame = PhysFrame::alloc_zero().expect("allocate virtual-APIC page failed");
```

- Dynamically allocated at vCPU creation time
- Stored in `VirtualApicRegs::apic_page`
- Physical address: `self.apic_page.start_paddr()`
- Written to `VMCS VIRT_APIC_ADDR` (vmcs.rs:133, encoding 0x2012)

---

## 5. xAPIC Write Path Trace: Exit -> Handler -> Dispatch -> write_lvt

### Path A: xAPIC MMIO (APIC_ACCESS exit)

```
Guest: MOV [0xFEE00350], reg   (xAPIC MMIO write to LVT_LINT0)
  |
  v
CPU: GPA 0xFEE00000 -> EPT -> HPA of APIC-access page
     HPA matches APIC_ACCESS_ADDR?
     YES -> VIRTUALIZE_APIC enabled? YES
     -> Generate APIC_ACCESS VM exit (exit reason 44)
     -> Exit qualification: offset=0x350, access_type=LinearDataWrite
  |
  v
builtin_vmexit_handler() [vcpu.rs:1147-1172]
  -> APIC_ACCESS not matched -> returns None
  |
  v
AxArchVCpuImpl::run() [vcpu.rs:2372]
  -> VmxExitReason::APIC_ACCESS => self.handle_apic_access(&exit_info)?
  |
  v
handle_apic_access() [vcpu.rs:1262-1310]
  -> apic_access_exit_info() [vmcs.rs:838-847]
     -> Reads EXIT_QUALIFICATION bits [11:0] = 0x350 (page offset)
     -> Access type = LinearDataWrite
  -> [warn] "[VMX] APIC_ACCESS exit: rip=... offset: 848, access_type: LinearDataWrite ..."
  -> Write path:
     -> decode_apic_mmio_write_value() [vcpu.rs:1312-1345]
        -> Decodes MOV instruction, extracts register value
     -> reg = 0x350 (page offset from exit qualification)
     -> addr = GuestPhysAddr::from(0xFEE00000 + 0x350) = 0xFEE00350
     -> EmulatedLocalApic::handle_write(addr, Dword, value) [lib.rs:172-176]
        -> xapic_mmio_access_reg_offset(0xFEE00350)
           -> (0xFEE00350 & 0xFFF) >> 4 = 0x350 >> 4 = 0x35
           -> ApicRegOffset::from(0x35) = LvtLint0 [consts.rs:157]
        -> VirtualApicRegs::handle_write(LvtLint0, val, Dword) [vlapic.rs:927]
           -> self.regs().LVT_LINT0.set(data32) [vlapic.rs:1008]
           -> write_lvt(ApicRegOffset::LvtLint0) [vlapic.rs:1009]
  |
  v
write_lvt(LvtLint0) [vlapic.rs:662-694]
  -> mask = APIC_LVT_M | APIC_LVT_DS | APIC_LVT_VECTOR
         | LVT_LINT0::TriggerMode, RemoteIRR, Polarity, DeliveryMode masks
  -> val &= mask
  -> [info] "[VLAPIC] LINT0 write: raw=... route=..."   <--- THE MISSING LOG
  -> self.regs().LVT_LINT0.set(val)
  -> self.lvt_last.lvt_lint0.set(val)
```

### Path B: x2APIC MSR (MSR_WRITE exit)

```
Guest: WRMSR  (ECX=0x835, EDX:EAX=value)   (x2APIC MSR write to LVT_LINT0)
  |
  v
CPU: MSR bitmap check -> MSR 0x835 is intercepted -> MSR_WRITE VM exit (exit reason 32)
  |
  v
builtin_vmexit_handler() [vcpu.rs:1147-1172]
  -> MSR_WRITE with rcx=0x835, not APIC_BASE -> returns None
  |
  v
AxArchVCpuImpl::run() [vcpu.rs:2405-2408]
  -> VmxExitReason::MSR_WRITE with msr=0x835 in x2APIC range
  -> self.handle_apic_msr_access(true, 0x835)?
  |
  v
handle_apic_msr_access(true, 0x835) [vcpu.rs:1214-1250]
  -> advance_rip(2) [vcpu.rs:1217]
  -> msr = 0x835, reg = 0x835
  -> msr != X2APIC_EOI_MSR (0x80B)
  -> EmulatedLocalApic::handle_write(SysRegAddr(0x835), Qword, value) [lib.rs:198-201]
     -> x2apic_msr_access_reg(SysRegAddr(0x835))
        -> addr.addr() - X2APIC_MSE_REG_BASE = 0x835 - 0x800 = 0x35
        -> ApicRegOffset::from(0x35) = LvtLint0 [consts.rs:157]
     -> VirtualApicRegs::handle_write(LvtLint0, val, Qword) [vlapic.rs:927]
        -> Same path as above through write_lvt()
```

### Path C: APIC_WRITE exit (should NOT apply to LINT0)

```
VmxExitReason::APIC_WRITE [vcpu.rs:2363-2371]
  -> offset = apic_access_exit_info()?.offset as usize
  -> if offset == X86_LOCAL_APIC_EOI_OFFSET (0xB0): handle EOI
  -> else: returns AxVCpuExitReason::Nothing   <--- SILENT DROP
```

Per Intel SDM Vol. 3C Section 29.4, APIC_WRITE exits are generated ONLY for TPR (0x80), EOI (0xB0), and ICR (0x300/0x308) writes when VIRTUAL_INTERRUPT_DELIVERY is enabled. LVT registers (including LINT0 at 0x350) should generate APIC_ACCESS (exit 44), not APIC_WRITE (exit 56). **If LINT0 writes somehow reached this path, they would be silently dropped.**

---

## 6. Potential Failure Points Ranked by Probability

### Rank 1 (HIGH): Guest OS never executed the LVT0 writes

**Evidence from existing analysis (`vmx-apic-write-path-contract.md`):**
- Linux log shows `x2apic: IRQ remapping doesn't support X2APIC mode`
- The forwarded ACPI MADT (from outer QEMU Q35) contains only type-1 LAPIC entries, no type-9 x2APIC entries
- Without x2APIC MADT entry, Linux `native_apic_init_apic()` may skip full APIC init
- In this configuration, `clear_local_APIC()` at `apic.c:1136` writes LVT0=0x10000 (masked), and `setup_local_APIC()` at `apic.c:1353` writes LVT0=0x0700 (ExtINT). These ARE unconditional writes that should occur.

**However**, there is a subtlety: `setup_local_APIC()` is called from `apic_bsp_setup()` which is called from `apic_intr_mode_init()`. If the APIC driver initialization path takes an early return before reaching `setup_local_APIC()`, the LVT0 writes would not happen. The `check_timer()` function in `io_apic.c` (called during `setup_IO_APIC()`) also writes LVT0 four times, but these are only reached if `nr_legacy_irqs() != 0`.

**Verdict:** Partially likely. The early `clear_local_APIC()` and `setup_local_APIC()` writes should happen unconditionally. If they don't, it's a guest OS configuration issue, not an AxVisor bug. But we cannot fully rule out an AxVisor-side interception problem until we add diagnostics (see minimum observation matrix below).

### Rank 2 (MEDIUM): APIC_ACCESS_ADDR in VMCS does not match EPT-resolved HPA

**The mechanism:**
When the guest accesses GPA 0xFEE00000, the CPU:
1. Translates GPA via EPT to HPA
2. Compares HPA against APIC_ACCESS_ADDR
3. If match AND VIRTUALIZE_APIC enabled -> APIC_ACCESS exit

If the HPA from step 1 does NOT match APIC_ACCESS_ADDR from step 2, no exit is generated. The access proceeds as normal memory access (the guest reads/writes the APIC-access page contents directly).

**Why this could happen:**
- `host::virt_to_phys()` might return a different value in the EPT mapping context vs the VMCS write context (e.g., if the host uses different address space layouts at different points)
- The static `VIRTUAL_APIC_ACCESS_PAGE` might be mapped differently by the host's memory allocator

**How to verify:** Read both values at VM init time and compare:
```rust
info!("[VMX] APIC_ACCESS_ADDR VMCS={:#x} EPT_GPA={:#x} mapped_HPA={:#x}",
    EmulatedLocalApic::virtual_apic_access_addr(),
    X86_APIC_ACCESS_GPA,
    <HPA from EPT mapping>);
```

### Rank 3 (LOW): VIRTUALIZE_APIC is actually NOT enabled

**Why it seems unlikely:**
- `set_control()` includes VIRTUALIZE_APIC in the `set` mask (vcpu.rs:764)
- On modern Intel CPUs, this is an allowed-1 bit
- The one observed APIC_ACCESS exit (offset 0x2F0, DCR_TIMER read) proves the mechanism is active

**Why it could still be true for writes:**
- There is NO documented mechanism in the Intel SDM that would allow reads to generate APIC_ACCESS exits while silently accepting writes. The APIC-access page mechanism applies uniformly to all access types.
- The single APIC_ACCESS exit for a read confirms the mechanism works for reads. If it works for reads, it MUST work for writes.

**Verdict:** Extremely unlikely.

### Rank 4 (LOW): APIC_WRITE exit (exit 56) fires instead of APIC_ACCESS (exit 44) for LINT0

**The code at vcpu.rs:2363-2371:**
```rust
VmxExitReason::APIC_WRITE => {
    let offset = self.apic_access_exit_info()?.offset as usize;
    if offset == X86_LOCAL_APIC_EOI_OFFSET {
        let vector = self.vlapic.handle_eoi();
        AxVCpuExitReason::InterruptEnd { vector }
    } else {
        AxVCpuExitReason::Nothing  // SILENT DROP
    }
}
```

**Per Intel SDM Vol. 3C Section 29.4:** APIC_WRITE exits fire ONLY for TPR, EOI, and ICR writes. LVT register writes generate standard APIC_ACCESS exits. This path should NOT fire for LINT0.

**If it DID fire:** The handler returns `Nothing` without calling `write_lvt()`, producing no log. The write would be silently lost.

**Verdict:** Contradicts SDM specification. Hardware behavior deviation is extremely unlikely.

### Rank 5 (NEGLIGIBLE): EPT mapping conflict at 0xFEE00000

If a pass-through device's mapping includes GPA 0xFEE00000 with a huge page (2MB or 1GB), the subsequent 4KB APIC-access mapping might not take effect. The guest would access the identity-mapped pass-through region instead of the APIC-access page.

**Impact:** The CPU would resolve the access via EPT to HPA = 0xFEE00000 (identity), which does NOT match APIC_ACCESS_ADDR. No APIC_ACCESS exit. The guest reads/writes garbage.

**How to verify:** Check whether the VM config includes any pass-through device covering 0xFEE00000-0xFEE00FFF.

### Summary Table

| Rank | Failure Point | Probability | Would produce "[VLAPIC] LINT0 write"? | Root cause |
|------|--------------|-------------|--------------------------------------|------------|
| 1 | Guest never executed the writes | **HIGH** | No | Guest OS / ACPI MADT config |
| 2 | APIC_ACCESS_ADDR mismatch with EPT | MEDIUM | No | Host address translation inconsistency |
| 3 | VIRTUALIZE_APIC not enabled | LOW | No | VMX capability MSR (ruled out by evidence) |
| 4 | APIC_WRITE exit fires for LINT0 | LOW | No | Hardware deviation from SDM |
| 5 | EPT mapping conflict | NEGLIGIBLE | No | VM config overlap |

---

## 7. Minimum Observation Matrix for Debugging

### Step 1: Verify VIRTUALIZE_APIC is enabled in VMCS

Add to `setup_vmcs_control()` after `vmcs::set_control()` for secondary controls:

```rust
let verify = VmcsControl32::SECONDARY_PROCBASED_EXEC_CONTROLS.read()?;
info!("[VMX] SECONDARY_PROCBASED after setup: {:#x}", verify);
info!("[VMX] VIRTUALIZE_APIC={} VIRT_APIC_REG={} VIRT_INTR_DEL={} USE_EPT={} UNRESTRICTED={}",
    verify & (1 << 0) != 0,  // VIRTUALIZE_APIC
    verify & (1 << 1) != 0,  // VIRTUALIZE_APIC_REGISTER
    verify & (1 << 31) != 0, // VIRTUAL_INTERRUPT_DELIVERY
    verify & (1 << 2) != 0,  // ENABLE_EPT
    verify & (1 << 7) != 0,  // UNRESTRICTED_GUEST
);
```

### Step 2: Verify APIC_ACCESS_ADDR matches EPT mapping

Add to `setup_vmcs_control()` after writing APIC_ACCESS_ADDR:

```rust
let apic_access_hpa = EmulatedLocalApic::virtual_apic_access_addr();
info!("[VMX] APIC_ACCESS_ADDR written: {:#x}", apic_access_hpa.as_usize());
info!("[VMX] VIRT_APIC_ADDR written: {:#x}", self.vlapic.virtual_apic_page_addr().as_usize());
info!("[VMX] X86_APIC_ACCESS_GPA: {:#x}", X86_APIC_ACCESS_GPA);
```

### Step 3: Add logging to EPT mapping creation

Add to `AxVM::init()` at `vm.rs` after the APIC-access page mapping:

```rust
info!("[VM] APIC-access page: GPA={:#x} -> HPA={:#x} (4KB, DEVICE|RW)",
    X86_APIC_ACCESS_GPA,
    x86_apic_access_page_addr().as_usize());
```

### Step 4: Add logging to all APIC write paths

Add `info!` level logs to catch every write path:

**In `builtin_vmexit_handler()` for IA32_APIC_BASE MSR** (already has `trace!`):
```rust
// vcpu.rs:1197 - handle_apic_base_msr_access
// Change trace! to info!:
info!("[VMX] IA32_APIC_BASE {}: value={:#x}", if write {"write"} else {"read"}, value);
```

**In `handle_apic_msr_access()`** (currently `trace!` at vcpu.rs:1223):
```rust
// Change trace! to info! for LVT-related MSRs:
if (0x835..=0x837).contains(&msr) || msr == 0x832 { // LVT0, LVT1, LVTTimer, LVTError
    info!("[VMX] x2APIC MSR {}: msr={:#x} value={:#x}",
        if write {"write"} else {"read"}, msr, value);
}
```

**In `handle_apic_access()`** (already `warn!` at vcpu.rs:1264):
The warn-level log is sufficient. But add a separate log for write-only:
```rust
if write {
    info!("[VMX] APIC_ACCESS write: offset={:#x} reg_offset={:#x}",
        reg, reg >> 4);
}
```

### Step 5: Add early APIC initialization detection

Add to the first VM-exit after boot to detect if the guest is in xAPIC or x2APIC mode:

In `inner_run()` or at first `run()`, check if there are any MSR_WRITE exits for IA32_APIC_BASE (0x1B):

```rust
// After each run, log the first IA32_APIC_BASE access
if self.regs().rcx as u32 == 0x1B {
    info!("[VMX] IA32_APIC_BASE access: value={:#x} (MSR {})",
        ((self.regs().rdx & 0xFFFFFFFF) << 32) | (self.regs().rax & 0xFFFFFFFF),
        if exit_reason == VmxExitReason::MSR_WRITE {"WRITE"} else {"READ"});
}
```

### Step 6: Verify the APIC-access page is not identity-mapped

If there's any doubt, read the EPT entry for GPA 0xFEE00000 and verify it points to the APIC-access page HPA, NOT to 0xFEE00000:

```rust
// In handle_apic_access or at VM init:
let ept_hpa = self.translate_guest_phys_via_ept(X86_APIC_ACCESS_GPA)?;
let apic_page_hpa = EmulatedLocalApic::virtual_apic_access_addr();
info!("[VMX] EPT resolves 0xFEE00000 -> HPA={:#x}, expected APIC_ACCESS_ADDR={:#x}, match={}",
    ept_hpa.as_usize(), apic_page_hpa.as_usize(),
    ept_hpa.as_usize() == apic_page_hpa.as_usize());
```

### Expected diagnostic output

If the APIC-access mechanism is working correctly, for each LVT0 write:

**xAPIC MMIO path:**
```
[VMX] APIC_ACCESS exit: rip=... offset: 848, access_type: LinearDataWrite ...
[VMX] APIC_ACCESS write: offset=0x350 reg_offset=0x35
[VLAPIC] LINT0 write: raw=0x... route=...
```

**x2APIC MSR path:**
```
[VMX] x2APIC MSR write: msr=0x835 value=0x...
[VLAPIC] LINT0 write: raw=0x... route=...
```

If NONE of these appear for any of the 6 writes, the writes were never executed by the guest.

---

## 8. Code Reference Quick Index

| Item | File | Line(s) |
|------|------|---------|
| VIRTUALIZE_APIC set | `x86_vcpu/src/vmx/vcpu.rs` | 764, 768 |
| APIC_ACCESS_ADDR written | `x86_vcpu/src/vmx/vcpu.rs` | 855-856 |
| VIRT_APIC_ADDR written | `x86_vcpu/src/vmx/vcpu.rs` | 854 |
| APIC-access page static | `x86_vlapic/src/lib.rs` | 48-50 |
| APIC-access HPA helper | `x86_vlapic/src/lib.rs` | 112-116 |
| Virtual-APIC page alloc | `x86_vlapic/src/vlapic.rs` | 108-135 |
| Virtual-APIC HPA helper | `x86_vlapic/src/vlapic.rs` | 145-147 |
| EPT mapping of APIC page | `axvm/src/vm.rs` | 480-485 |
| handle_apic_access | `x86_vcpu/src/vmx/vcpu.rs` | 1262-1310 |
| handle_apic_msr_access | `x86_vcpu/src/vmx/vcpu.rs` | 1214-1250 |
| APIC_WRITE handler | `x86_vcpu/src/vmx/vcpu.rs` | 2363-2371 |
| write_lvt(LvtLint0) | `x86_vlapic/src/vlapic.rs` | 662-694 |
| handle_write(LvtLint0) | `x86_vlapic/src/vlapic.rs` | 1008-1010 |
| "[VLAPIC] LINT0 write" log | `x86_vlapic/src/vlapic.rs` | 669-672 |
| MSR bitmap setup | `x86_vcpu/src/vmx/vcpu.rs` | 570-594 |
| APIC_ACCESS exit info | `x86_vcpu/src/vmx/vmcs.rs` | 838-847 |
| set_control logic | `x86_vcpu/src/vmx/vmcs.rs` | 654-693 |
| xAPIC MMIO offset calc | `x86_vlapic/src/consts.rs` | 230-232 |
| x2APIC MSR offset calc | `x86_vlapic/src/consts.rs` | 247-249 |
| ApicRegOffset::from(0x35) | `x86_vlapic/src/consts.rs` | 157 |
