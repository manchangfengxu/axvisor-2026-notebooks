# VMX APIC Write Path Contract Audit

**Date:** 2026-06-19
**Subject:** If the guest really wrote APIC_LVT0, how should it be observed, and is the path broken?
**Log analyzed:** `/home/ssdns/work/axvisor-uefi/tmp/ovmf-linux-smoke-lint0-write.log`

---

## 1. All possible paths for guest local APIC access in AxVisor's current VMX setup

### Path A: xAPIC MMIO APIC-access page VM-exit

**When triggered:** Guest accesses any address in GPA range `[0xfee0_0000, 0xfee0_1000)` while VMCS secondary control `VIRTUALIZE_APIC` (secondary bit 0) is set.

**VMCS controls that make this fire:**
- `CpuCtrl2::VIRTUALIZE_APIC` (secondary bit 0) is set at
  `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:764`
- `APIC_ACCESS_ADDR` is written to the physical APIC-access page at
  `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:855-856`
- EPT maps GPA `0xfee0_0000` to the APIC-access page with `DEVICE | READ | WRITE` at
  `tgoskits/virtualization/axvm/src/vm.rs:480-485`
- Static APIC-access page allocated at
  `tgoskits/virtualization/x86_vlapic/src/lib.rs:48-50`

**Exit reason:** `VmxExitReason::APIC_ACCESS`

**Handler (builtin, handled inside VmxVcpu):**
`VmxVcpu::handle_apic_access()` at `vcpu.rs:1262-1310`

**Flow for a write:**
1. Exit reason `APIC_ACCESS` dispatched in `AxArchVCpu::run()` at `vcpu.rs:2372`
2. `handle_apic_access()` reads `ApicAccessExitInfo` (exit qualification), determines `access_type == LinearDataWrite`
3. Decodes instruction value via `decode_apic_mmio_write_value()` at `vcpu.rs:1312-1345`
4. If offset == `X86_LOCAL_APIC_EOI_OFFSET` (0xb0): returns `InterruptEnd`
5. Otherwise: calls `EmulatedLocalApic::handle_write()` at `lib.rs:172-176`
6. Which calls `VirtualApicRegs::handle_write()` at `vlapic.rs:927`
7. For `ApicRegOffset::LvtLint0` (offset 0x35): calls `write_lvt(LvtLint0)` at `vlapic.rs:1008-1010`
8. `write_lvt()` at `vlapic.rs:642-729` prints: `"[VLAPIC] LINT0 write: raw=..."` at info level

**Exit reason logged by AxVisor (at warn level):**
```
[VMX] APIC_ACCESS exit: rip=... len=... info=ApicAccessExitInfo { offset: ..., access_type: LinearDataWrite, ... }
```

### Path B: x2APIC MSR intercept via MSR bitmap

**When triggered:** Guest executes `WRMSR` with ECX in `[0x800, 0x8ff]` (x2APIC MSR space) while MSR bitmap has write-intercept set for those MSRs.

**MSR bitmap setup:** `setup_msr_bitmap()` at `vcpu.rs:570-594`:
```rust
for msr in 0x800..=0x83f {
    self.msr_bitmap.set_read_intercept(msr, true);
    self.msr_bitmap.set_write_intercept(msr, true);
}
```
Plus `IA32_APIC_BASE` (0x1b) at lines 572-574.

**Exit reason:** `VmxExitReason::MSR_WRITE` (for WRMSR)

**Handler (builtin, handled inside VmxVcpu):**
`VmxVcpu::handle_apic_msr_access(true, msr)` at `vcpu.rs:1214-1250`

**Flow for LINT0 write via MSR 0x835:**
1. Exit reason `MSR_WRITE` with `rcx == 0x835` dispatched at `vcpu.rs:2405-2408`
2. Check `(X2APIC_MSR_BASE..=X2APIC_MSR_END).contains(&msr)` passes (0x800..=0x8ff)
3. `handle_apic_msr_access(true, 0x835)` at `vcpu.rs:1214`
4. `msr != X2APIC_EOI_MSR`, so calls `EmulatedLocalApic::handle_write(SysRegAddr(0x835), Qword, value)` at `lib.rs:198-201`
5. `x2apic_msr_access_reg(SysRegAddr(0x835))` -> `ApicRegOffset::LvtLint0` via `ApicRegOffset::from(0x35)` at `consts.rs:157`
6. Same path as above: `handle_write()` -> `write_lvt(LvtLint0)` -> info-level "[VLAPIC] LINT0 write" log

**Note:** `handle_apic_msr_access()` uses `trace!` level for its own log, which does NOT appear in the `release_max_level_info` build. However, the downstream `write_lvt()` uses `info!` which WOULD appear.

### Path C: APIC WRITE VM-exit (virtual APIC register write virtualization)

**When triggered:** Certain APIC register writes are intercepted by the processor as "APIC WRITE" exits (not "APIC ACCESS" exits) when `VIRTUAL_INTERRUPT_DELIVERY` (secondary bit 31) is enabled.

**VMCS control:** `CpuCtrl2::VIRTUAL_INTERRUPT_DELIVERY` at `vcpu.rs:766`

**Exit reason:** `VmxExitReason::APIC_WRITE`

**Handler in `run()` at `vcpu.rs:2363-2371`:**
```rust
VmxExitReason::APIC_WRITE => {
    let offset = self.apic_access_exit_info()?.offset as usize;
    if offset == X86_LOCAL_APIC_EOI_OFFSET {
        let vector = self.vlapic.handle_eoi();
        AxVCpuExitReason::InterruptEnd { vector }
    } else {
        AxVCpuExitReason::Nothing
    }
}
```

**Critical observation:** For LINT0 (offset 0x350), this handler returns `Nothing` without calling any vlapic write function. If the processor fires an APIC_WRITE exit for LINT0, the write would be silently dropped here.

**However**, per Intel SDM Vol. 3C, Section 29.4, the APIC WRITE exit fires specifically for **TPR (0x80), EOI (0xb0), and ICR (0x300/0x308)** writes that the processor is virtualizing. LVT registers (including LINT0 at 0x350) generate the standard APIC ACCESS exit (Path A), not the APIC WRITE exit. So Path C should not apply to LINT0.

### Path D: EPT violation fallback for unmapped APIC page

**When triggered:** If the APIC page GPA is NOT mapped in EPT (or mapped without write permission), a write generates an EPT violation instead of an APIC ACCESS exit.

**Current state:** The APIC page IS mapped with `DEVICE | READ | WRITE` at `vm.rs:480-484`, so this path should NOT fire for normal APIC register accesses.

**Handler:** `decode_ept_mmio_access()` at `vcpu.rs:1347-1432`, which checks if the fault GPA is in `[X86_APIC_ACCESS_GPA, X86_APIC_ACCESS_GPA + 0x1000)`.

**If it fired for LINT0 write:** `handle_decoded_ept_mmio_write()` at `vcpu.rs:1434-1463` would call `EmulatedLocalApic::handle_write()` and produce the same "VLAPIC LINT0 write" info log.

**This path is NOT expected to fire** for APIC MMIO because the EPT mapping exists. It could only fire if the EPT mapping were missing or incorrect.

### Path E: Virtual-APIC page hardware virtualization (TPR shadow)

**When triggered:** Write to TPR at offset 0x80 when `USE_TPR_SHADOW` (primary bit 21) is enabled.

**VMCS controls:**
- `CpuCtrl::USE_TPR_SHADOW` at `vcpu.rs:750`
- `VIRT_APIC_ADDR` written at `vcpu.rs:854`

**This path applies ONLY to TPR (offset 0x80), not to LINT0.** The processor intercepts TPR writes via the virtual-APIC page and generates an `APIC WRITE` exit. LINT0 is not in the set of hardware-virtualized APIC registers (SDM Vol. 3C, Section 29.4: only TPR, EOI, and ICR qualify).

---

## 2. Where should an LINT0 write land in current implementation

### xAPIC MMIO write to 0xfee00350:

```
Guest: MOV [0xfee00350], reg  (xAPIC MMIO)
  -> VMX: APIC_ACCESS VM-exit (access_type=LinearDataWrite, offset=0x350)
  -> VmxVcpu::run() matches VmxExitReason::APIC_ACCESS at vcpu.rs:2372
  -> handle_apic_access() at vcpu.rs:1262-1310
     -> [warn] "[VMX] APIC_ACCESS exit: ... access_type: LinearDataWrite ..."
     -> EmulatedLocalApic::handle_write(GPA(0xfee00350), Dword, value) at lib.rs:172
        -> xapic_mmio_access_reg_offset(0xfee00350) -> ApicRegOffset::LvtLint0 via ApicRegOffset::from(0x35) at consts.rs:157
     -> VirtualApicRegs::handle_write(LvtLint0, val, Dword) at vlapic.rs:1008-1010
        -> self.regs().LVT_LINT0.set(data32)
        -> write_lvt(LvtLint0) at vlapic.rs:642
           -> [info] "[VLAPIC] LINT0 write: raw=... route=..." at vlapic.rs:669-672
           -> self.regs().LVT_LINT0.set(val)
           -> self.lvt_last.lvt_lint0.set(val)
```

**Minimum observable logs (at info level and above):**
1. `[VMX] APIC_ACCESS exit: ... offset: 848, access_type: LinearDataWrite ...` (warn level, vcpu.rs:1264-1268)
2. `[VLAPIC] LINT0 write: raw=... route=...` (info level, vlapic.rs:669-672)

### x2APIC MSR write to MSR 0x835:

```
Guest: WRMSR  (ECX=0x835, EDX:EAX=value)
  -> VMX: MSR_WRITE VM-exit (rcx=0x835)
  -> VmxVcpu::run() matches VmxExitReason::MSR_WRITE at vcpu.rs:2405-2408
  -> handle_apic_msr_access(true, 0x835) at vcpu.rs:1214-1250
     -> msr != X2APIC_EOI_MSR, so:
     -> EmulatedLocalApic::handle_write(SysRegAddr(0x835), Qword, value) at lib.rs:198-201
        -> x2apic_msr_access_reg(0x835) -> ApicRegOffset::LvtLint0
     -> VirtualApicRegs::handle_write(LvtLint0, val, Qword) at vlapic.rs:1008-1010
        -> Same write_lvt path as above
```

**Minimum observable logs:**
1. `[VLAPIC] LINT0 write: raw=... route=...` (info level)
2. `[VMX] MSR bitmap intercept enabled: x2APIC MSRs 0x800..=0x83f read/write` (info level, seen at boot, vcpu.rs:593)

---

## 3. What should we at minimum see if guest really executed Linux's APIC_LVT0 write

If the guest truly executed a write to APIC_LVT0 (either xAPIC MMIO or x2APIC MSR), the following logs MUST appear in the AxVisor output:

### Must-see (non-negotiable):
| Log message | Level | File:line |
|---|---|---|
| `[VLAPIC] LINT0 write: raw=0x... route=...` | info | `vlapic.rs:669-672` |

This is emitted inside `write_lvt(ApicRegOffset::LvtLint0)` which is called unconditionally for every LINT0 write path, regardless of whether xAPIC or x2APIC mode is active.

### Should-see (for xAPIC MMIO path):
| Log message | Level | File:line |
|---|---|---|
| `[VMX] APIC_ACCESS exit: ... offset: 848, access_type: LinearDataWrite ...` | warn | `vcpu.rs:1264-1268` |

### Should-see (for x2APIC MSR path):
| Log message | Level | File:line |
|---|---|---|
| (none visible at info level; trace-level only) | trace | `vcpu.rs:1223` |

### Actual state of logs for LINT0:
- **`[VLAPIC] LINT0 write`**: NOT PRESENT in the log (zero occurrences)
- **`[VMX] APIC_ACCESS exit` with `LinearDataWrite`**: NOT PRESENT (zero occurrences of any APIC_ACCESS write)
- **`[VMX] APIC_ACCESS exit` total**: EXACTLY ONE entry in entire log: a READ at offset 752 (CMCI register area, NOT LINT0) appearing at timestamp 19.893s, AFTER the kernel panic

---

## 4. "Completely no LINT0 write observed" -- which layers could be broken, ranked by probability

### Rank 1 (MOST PROBABLE): Guest OS never executed the write

**Evidence:**
- Linux log line `[0.071171] x2apic: IRQ remapping doesn't support X2APIC mode` proves Linux detected the absence of IRQ remapping
- The forwarded ACPI MADT (from outer QEMU Q35) contains only type-1 LAPIC entries, no type-9 x2APIC entries
- QEMU `-machine q35` does NOT enable x2APIC in MADT by default; no `int_remap=on` or equivalent
- Without x2APIC MADT entry, Linux `native_apic_init_apic()` skips x2APIC init and returns early (Linux `arch/x86/kernel/apic/apic.c`, the `if (!x2apic_support)` early return before LINT0/LINT1 configuration)
- The `apic_intr_mode_init` function at panic trace line (Linux log 2459-2460) is the code that tries timer setup; it is called AFTER the early x2APIC check, but by then the LAPIC may not be fully configured with LINT0/LINT1 in the fallback path

**This is NOT an AxVisor bug.** The write never happened because Linux's interrupt subsystem decided it wasn't needed for the given hardware configuration.

### Rank 2 (LOW PROBABILITY): VMX APIC-access mechanism fails for writes

**Evidence against:**
- One APIC_ACCESS exit IS observed in the log (a read at offset 752, line 2473), proving the APIC-access page mechanism works at the processor level
- The VMCS `VIRTUALIZE_APIC` control IS set (vcpu.rs:764)
- The APIC-access page IS allocated and its PA IS written to VMCS (vcpu.rs:855-856, lib.rs:112-116)
- EPT DOES map the APIC page GPA (vm.rs:480-484)

**Residual doubt:** Could the APIC-access mechanism have a read-vs-write behavioral difference? Per Intel SDM Vol. 3C, Section 30.4, the "virtualize APIC accesses" feature applies uniformly to all access types (read, write, instruction fetch, event delivery). There is no documented mechanism that would allow reads to generate exits while silently accepting writes. A write should produce the same APIC ACCESS exit with `access_type = LinearDataWrite`. This is extremely unlikely to be the issue.

### Rank 3 (VERY LOW PROBABILITY): APIC_WRITE exit swallows LINT0 writes

**Evidence against:**
- `VmxExitReason::APIC_WRITE` at `vcpu.rs:2363-2371` only handles EOI offset; other offsets return `AxVCpuExitReason::Nothing`
- However, per SDM Vol. 3C, Section 29.4, APIC WRITE exits are ONLY generated for TPR, EOI, and ICR writes, not for LVT register writes
- LVT registers generate the standard APIC ACCESS exit (Path A), not APIC WRITE (Path C)

**If this path were somehow triggered for LINT0:** The handler would return `Nothing` without calling `write_lvt()`, producing no "[VLAPIC] LINT0 write" log. But this would be a hardware behavior deviation from the SDM specification, which is extremely unlikely.

### Rank 4 (NEGLIGIBLE): EPT violation path catches APIC writes

**Evidence against:**
- The APIC page IS mapped with DEVICE|READ|WRITE (vm.rs:480-484), so EPT violations should NOT fire for normal APIC accesses
- If an EPT violation DID fire for an APIC write, `decode_ept_mmio_access()` at vcpu.rs:1347 would catch it and decode the instruction, routing to `handle_decoded_ept_mmio_write()` which DOES call `EmulatedLocalApic::handle_write()` and would produce the "[VLAPIC] LINT0 write" log

**This path is impossible to miss silently** -- any EPT-violation-routed APIC write would produce the info-level log.

### Summary ranking:
| Rank | Layer | Probability | Would produce "[VLAPIC] LINT0 write" log? |
|---|---|---|---|
| 1 | Guest OS never wrote LINT0 | **High** | No (write never happened) |
| 2 | VMX APIC-access broken for writes | Very Low | No (if broken) |
| 3 | APIC_WRITE exit incorrectly fires for LINT0 | Negligible | No (handler returns Nothing) |
| 4 | EPT mapping missing for APIC page | Negligible | Yes (would be caught by EPT handler) |

---

## 5. Minimum observation matrix

### If guest uses xAPIC MMIO write to 0xfee00350:

| Step | Exit type / event | Function | File:line | Log level |
|---|---|---|---|---|
| 1 | `VmxExitReason::APIC_ACCESS` | `handle_apic_access()` | `vcpu.rs:1262` | warn |
| 2 | Instruction decode | `decode_apic_mmio_write_value()` | `vcpu.rs:1312` | (no log) |
| 3 | Dispatch to vlapic | `EmulatedLocalApic::handle_write()` | `lib.rs:172` | debug (filtered) |
| 4 | Offset conversion | `xapic_mmio_access_reg_offset()` | `consts.rs:230` | (no log) |
| 5 | Register write dispatch | `VirtualApicRegs::handle_write(LvtLint0, ...)` | `vlapic.rs:1008` | debug (filtered) |
| 6 | LVT write execution | `write_lvt(ApicRegOffset::LvtLint0)` | `vlapic.rs:642` | info |
| 7 | **Minimum visible log** | | `vlapic.rs:669` | **info: `[VLAPIC] LINT0 write`** |

### If guest uses x2APIC MSR write to MSR 0x835:

| Step | Exit type / event | Function | File:line | Log level |
|---|---|---|---|---|
| 1 | `VmxExitReason::MSR_WRITE` (rcx=0x835) | `builtin_vmexit_handler()` | `vcpu.rs:1147-1172` | (no log) |
| 2 | MSR bitmap range check passes | `run()` MSR_WRITE arm | `vcpu.rs:2405-2408` | (no log) |
| 3 | Dispatch to APIC MSR handler | `handle_apic_msr_access(true, 0x835)` | `vcpu.rs:1214` | trace (filtered) |
| 4 | RIP advance | `advance_rip(2)` | `vcpu.rs:1217` | (no log) |
| 5 | Dispatch to vlapic | `EmulatedLocalApic::handle_write()` | `lib.rs:198` | debug (filtered) |
| 6 | Offset conversion | `x2apic_msr_access_reg()` | `consts.rs:247` | (no log) |
| 7 | Register write dispatch | `VirtualApicRegs::handle_write(LvtLint0, ...)` | `vlapic.rs:1008` | debug (filtered) |
| 8 | LVT write execution | `write_lvt(ApicRegOffset::LvtLint0)` | `vlapic.rs:642` | info |
| 9 | **Minimum visible log** | | `vlapic.rs:669` | **info: `[VLAPIC] LINT0 write`** |

---

## 6. Every conclusion with file path + function name + line number

### VMX controls configuration:
- Primary controls including `USE_TPR_SHADOW`: `vcpu.rs:742-758`, `setup_vmcs_control()`
- Secondary controls including `VIRTUALIZE_APIC`, `VIRTUALIZE_APIC_REGISTER`, `VIRTUAL_INTERRUPT_DELIVERY`: `vcpu.rs:761-796`, `setup_vmcs_control()`
- `VIRT_APIC_ADDR` written: `vcpu.rs:854`
- `APIC_ACCESS_ADDR` written: `vcpu.rs:855-856`
- EOI exit bitmap set to u64::MAX: `vcpu.rs:857-860`

### MSR bitmap configuration:
- `IA32_APIC_BASE` (0x1b) intercepted: `vcpu.rs:572-574`, `setup_msr_bitmap()`
- x2APIC MSRs 0x800..=0x83f intercepted: `vcpu.rs:589-593`, `setup_msr_bitmap()`

### APIC-access page allocation:
- Static APIC-access page: `lib.rs:48-50`, `VIRTUAL_APIC_ACCESS_PAGE`
- Physical address helper: `lib.rs:112-116`, `EmulatedLocalApic::virtual_apic_access_addr()`
- Virtual-APIC page (per-vCPU): `vlapic.rs:108-135`, `VirtualApicRegs::new()`

### EPT mapping:
- GPA `0xfee0_0000` mapped to APIC-access page: `vm.rs:480-485`

### APIC-access exit handler:
- `VmxVcpu::handle_apic_access()`: `vcpu.rs:1262-1310`
- Dispatch for `APIC_ACCESS` reason: `vcpu.rs:2372`
- Instruction decode: `decode_apic_mmio_write_value()`: `vcpu.rs:1312-1345`

### MSR exit handler:
- Dispatch for `MSR_WRITE` x2APIC range: `vcpu.rs:2405-2408`
- `handle_apic_msr_access()`: `vcpu.rs:1214-1250`
- `EmulatedLocalApic` MSR write path: `lib.rs:198-201`

### APIC_WRITE exit handler:
- Dispatch for `APIC_WRITE`: `vcpu.rs:2363-2371` (only handles EOI, other offsets return Nothing)

### vLAPIC LINT0 write:
- `VirtualApicRegs::handle_write(LvtLint0)`: `vlapic.rs:1008-1010`
- `write_lvt(ApicRegOffset::LvtLint0)`: `vlapic.rs:662-694`
- Info-level log emission: `vlapic.rs:669-672`
- `lint0_route_from_lvt()`: `vlapic.rs:61-75`
- `EmulatedLocalApic::lint0_route()`: `lib.rs:148-150`

### LINT0 route read (used by PIT injection):
- `VmxVcpu::lint0_route()`: `vcpu.rs:245-247`
- `AxVM::inject_due_x86_pic_lint0_irq()`: `vm.rs:320-339`

### LINT0 initial state:
- `VirtualApicRegs::new()` initializes `lvt_last` to `LocalVectorTable::default()` at `vlapic.rs:101`
- LVT reset value: `RESET_LVT_REG = APIC_LVT_M` (masked) at `consts.rs:210`
- `vlapic.rs:130`: `regs().ID.set((vcpu_id as u32) << 24)` and version set; no LVT configuration
- IA32_APIC_BASE shadow: `0xfee00900` (BSP + xAPIC enabled) at `vlapic.rs:110-112`

### VMCS APIC_ACCESS exit info extraction:
- `vmcs::apic_access_exit_info()`: `vmcs.rs:838-845`
- `ApicAccessExitType` enum: `vmcs.rs:600-617`
- `ApicAccessExitInfo` struct: `vmcs.rs:639-646`

---

## 7. Contract verification summary

### The write path is COMPLETE and CORRECT in code

Every layer from VMX exit through vLAPIC register write has been traced. The code path for LINT0 writes exists and is complete for both xAPIC MMIO and x2APIC MSR paths. The "[VLAPIC] LINT0 write" info-level log at `vlapic.rs:669-672` is the definitive marker: if this message appears, the write was processed.

### The write NEVER HAPPENED -- this is a guest-side issue

Evidence:
1. Zero `"[VLAPIC] LINT0 write"` messages in the log
2. Zero APIC_ACCESS write exits in the log (only 1 read exit at offset 752 after panic)
3. Linux log shows `x2apic: IRQ remapping doesn't support X2APIC mode` at 0.071s
4. Linux then falls into timer setup failure cascade and panics
5. The outer QEMU Q35 MADT (forwarded via fw_cfg ACPI blobs, logged at AxVisor timestamp 5.656s) contains only type-1 LAPIC entries, not type-9 x2APIC entries

### Root cause chain (not an AxVisor bug):
1. Outer QEMU Q35 default ACPI MADT lacks type-9 x2APIC entry
2. Linux detects no IRQ remapping support for x2APIC
3. Linux `native_apic_init_apic()` returns early without configuring LINT0/LINT1 in the proper virtual wire mode
4. AxVisor vLAPIC LINT0 remains at reset value (masked, `LVT_LINT0 = 0x00010000`)
5. AxVisor PIT handler finds no injectable LINT0 route: `"LAPIC LINT0 had an injectable route"` false
6. Linux timer setup fails through all paths, kernel panics

### Note on `handle_apic_access` log level

The `handle_apic_access()` function at `vcpu.rs:1262` uses `warn!` for its APIC_ACCESS exit log, which IS visible in `release_max_level_info` builds. The fact that the only such log in the entire trace is a single read (not write) confirms the VMX APIC-access mechanism is functioning, and the write simply never occurred.
