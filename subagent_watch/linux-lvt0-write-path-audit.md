# Linux x86 APIC_LVT0 Write Path Audit: IO-APIC Timer Fallback

## 1. Source and Log Version Note

- **Source tree under audit**: `/home/ssdns/work/axvisor-uefi/linux/` = Linux **7.1.0** (`Makefile`: VERSION=7, PATCHLEVEL=1)
- **Log file**: `/home/ssdns/work/axvisor-uefi/tmp/ovmf-linux-smoke-lint0-write.log` = Linux **7.0.0** (`Not tainted 7.0.0-gdd6c438c3e64-dirty #3`)
- **Where they match**: The `check_timer()` function body, APIC constants, and the entire timer fallback chain are unchanged between 7.0.0 and 7.1.0. The `io_apic.c` lines cited below are from the 7.1.0 source, and the same logic executed in the 7.0.0 log kernel.

---

## 2. Exact Function Chain That Produces the "Trying to set up timer" Messages

```
start_kernel()
  -> x86_late_time_init()
    -> apic_intr_mode_init()                           [apic.c:1365]
      -> apic_bsp_setup()                              [apic.c:2341]
        -> setup_local_APIC()                          [apic.c:1509]
        -> (returns)
    -> setup_IO_APIC()                                 [io_apic.c:2272]
      -> setup_IO_APIC_irqs()                          [io_apic.c:2289]
      -> init_IO_APIC_traps()                          [io_apic.c:2290]
      -> check_timer()                                 [io_apic.c:2292, called when nr_legacy_irqs() != 0]
```

Inside `check_timer()` (io_apic.c:2049-2190), the three fallback phases execute in order:

| Phase | Log message | Source line |
|-------|------------|-------------|
| 8259A via IO-APIC cascade pin | `"...trying to set up timer (IRQ0) through the 8259A ..."` | io_apic.c:2132 |
| Virtual Wire via LVT0 Fixed | `"...trying to set up timer as Virtual Wire IRQ..."` | io_apic.c:2153 |
| ExtINT via LVT0 ExtINT | `"...trying to set up timer as ExtINT IRQ..."` | io_apic.c:2167 |
| Final panic | `"IO-APIC + timer doesn't work!"` | io_apic.c:2186 |

---

## 3. All APIC_LVT0 Writes in check_timer()

Every LVT0 write in `check_timer()` goes through `apic_write(APIC_LVT0, val)` (declared io_apic.c:2049-2190).

### Phase 0 -- Pre-check initial mask (before any timer test)

**io_apic.c:2077**
```c
apic_write(APIC_LVT0, APIC_LVT_MASKED | APIC_DM_EXTINT);
```

| Field | Constant | Hex value | Binary bits [16:8] |
|-------|----------|-----------|---------------------|
| Mask (bit 16) | `APIC_LVT_MASKED` = `(1 << 16)` | 0x10000 | bit 16 = 1 |
| Delivery Mode (bits 10:8) | `APIC_DM_EXTINT` = `0x700` | 0x700 | bits 10:8 = `111` |
| Vector (bits 7:0) | (implicit 0) | 0x000 | bits 7:0 = `00000000` |
| **Value written** | | **0x10700** | Masked + ExtINT + vector 0 |

Purpose: Disable the LINT0 virtual wire while the 8259A is being initialized for the 8259A test phase.

### Phase 2 -- Virtual Wire (Fixed mode), try 1: UNMASKED

**io_apic.c:2156**
```c
lapic_register_intr(0);  /* clears IRQ_LEVEL, sets edge handler (io_apic.c:1930-1934) */
apic_write(APIC_LVT0, APIC_DM_FIXED | cfg->vector);  /* Fixed mode */
```

| Field | Constant / Source | Hex value | Binary bits [16:8] |
|-------|------------------|-----------|---------------------|
| Mask (bit 16) | (not set) | 0x0 | bit 16 = 0 (UNMASKED) |
| Delivery Mode (bits 10:8) | `APIC_DM_FIXED` = `0x000` | 0x000 | bits 10:8 = `000` |
| Vector (bits 7:0) | `cfg->vector` from `irqd_cfg(irq_data)` | 0x30 (from log: `..TIMER: vector=0x30`) | bits 7:0 = `00110000` |
| **Value written** | | **0x00030** | Unmasked + Fixed + vector 0x30 |

`cfg->vector` is assigned at io_apic.c:2053 (`struct irq_cfg *cfg = irqd_cfg(irq_data)`) and is 0x30 per the log (`..TIMER: vector=0x30`).

### Phase 2 -- Virtual Wire (Fixed mode), fallback on failure: MASKED

**io_apic.c:2164**
```c
apic_write(APIC_LVT0, APIC_LVT_MASKED | APIC_DM_FIXED | cfg->vector);
```

| Field | Value | Binary |
|-------|-------|--------|
| Mask (bit 16) | set | bit 16 = 1 |
| Delivery Mode | Fixed | bits 10:8 = `000` |
| Vector | `cfg->vector` (0x30) | bits 7:0 = `00110000` |
| **Value written** | **0x10030** | Masked + Fixed + vector 0x30 |

Purpose: Re-mask LINT0 before attempting ExtINT path.

### Phase 3 -- ExtINT mode

**io_apic.c:2171**
```c
legacy_pic->init(0);   /* re-initialize 8259A PIC (io_apic.c:2169) */
legacy_pic->make_irq(0);  /* connect IRQ0 to cascade (io_apic.c:2170) */
apic_write(APIC_LVT0, APIC_DM_EXTINT);  /* ExtINT mode */
```

| Field | Constant / Value | Hex value | Binary bits [16:8] |
|-------|-----------------|-----------|---------------------|
| Mask (bit 16) | (not set) | 0x0 | bit 16 = 0 (UNMASKED) |
| Delivery Mode (bits 10:8) | `APIC_DM_EXTINT` = `0x700` | 0x700 | bits 10:8 = `111` |
| Vector (bits 7:0) | (implicit 0, vector supplied by 8259A via INTA) | 0x000 | bits 7:0 = `00000000` |
| **Value written** | | **0x00700** | Unmasked + ExtINT + vector 0 |

After this write, `legacy_pic->unmask(0)` (io_apic.c:2172) unmasks IRQ0 on the 8259A, then `unlock_ExtINT_logic()` (io_apic.c:2174) performs INTA cycles on the IO-APIC cascade pin.

---

## 4. Other APIC_LVT0 Writes Before check_timer()

Two writes occur before `check_timer()` is entered, both in `setup_local_APIC()` (called from `apic_bsp_setup()`):

### clear_local_APIC() during BSP init

**apic.c:1136** (inside `clear_local_APIC()`, called before `setup_local_APIC`)
```c
apic_write(APIC_LVT0, APIC_LVT_MASKED);
```
| Value | 0x10000 | Masked, all other bits cleared |

### setup_local_APIC() virtual wire setup

**apic.c:1353** (inside `setup_local_APIC()`)
```c
apic_write(APIC_LVT0, APIC_DM_EXTINT);
```
| Value | 0x0700 | Unmasked + ExtINT + vector 0 |

This is the same configuration that Phase 3 of `check_timer()` will attempt. The log confirms `APIC: Switch to symmetric I/O mode setup` (from `apic_intr_mode_init()` at apic.c:1380) ran, and `setup_local_APIC()` executed before `check_timer()`.

---

## 5. APIC_LVT0 Writes in io_apic.c Outside check_timer()

For completeness, two IRQ chip helpers in io_apic.c also write LVT0:

- **`mask_lapic_irq()`** (io_apic.c:1904): reads LVT0, sets APIC_LVT_MASKED, writes back. Called when IRQ0 IRQ chip is masked.
- **`unmask_lapic_irq()`** (io_apic.c:1911): reads LVT0, clears APIC_LVT_MASKED, writes back. Called when IRQ0 IRQ chip is unmasked.

These are invoked indirectly by `check_timer()` through `legacy_pic->unmask(0)` and `legacy_pic->mask(0)` in some paths, but do not change the delivery mode or vector.

---

## 6. xAPIC MMIO vs x2APIC MSR Semantics

### How apic_write() is dispatched

`apic_write()` is declared as an `__always_inline` wrapper around `static_call(apic_call_write)` (arch/x86/include/asm/apic.h:403-405). The `static_call` target is resolved at boot time from the `struct apic` driver's `.write` function pointer.

### Default x86_64 path (flat APIC mode)

The `.write = native_apic_mem_write` assignment is in `arch/x86/kernel/apic/apic_flat_64.c:58`.

```c
static inline void native_apic_mem_write(u32 reg, u32 v)
{
    volatile u32 *addr = (volatile u32 *)(APIC_BASE + reg);
    /* X86_BUG_11AP alternative uses xchgl instead of movl */
}
```

**Result**: For APIC_LVT0 (offset `0x350`), the MMIO address is `0xfee00000 + 0x350 = 0xfee00350`. The write is a 32-bit store to this physical memory address, i.e. **xAPIC MMIO semantics**.

### x2APIC path (MSR)

The `.write = native_apic_msr_write` assignment is in `arch/x86/kernel/apic/x2apic_phys.c:153`:

```c
static inline void native_apic_msr_write(u32 reg, u32 v)
{
    wrmsrq(APIC_BASE_MSR + (reg >> 4), v);
}
```

For APIC_LVT0 (offset `0x350`): MSR address = `0x800 + (0x350 >> 4) = 0x800 + 0x35 = 0x835`. The write uses `WRMSR` to MSR `0x835`.

### Which path the log kernel uses

The log shows `APIC: Switch to symmetric I/O mode setup` (not "x2apic mode"). The `x2apic_phys` driver is only selected when x2APIC is enabled. Since no x2APIC setup message appears, the kernel is using the flat APIC driver with **xAPIC MMIO semantics**. All APIC_LVT0 writes in this boot are MMIO stores to `0xfee00350`.

### Hardware interception in AxVisor

With VMX APIC_ACCESS VM-execution controls configured, a guest MMIO write to `0xfee00350` causes a VM exit with reason `APIC_ACCESS` (vmx/vcpu.rs:1262, `handle_apic_access()`). The exit info `offset` field contains `0x350`. The handler at vmx/vcpu.rs:1284 decodes the instruction and calls `VLapic::handle_write()` (vlapic.rs:927), which routes to `write_lvt(ApicRegOffset::LvtLint0)` (vlapic.rs:1008-1010), storing the value in the vAPIC page's LVT_LINT0 register.

---

## 7. What the AxVisor vLAPIC Does with the Received Values

The `write_lvt()` function (vlapic.rs:642-693) re-reads the value from the vAPIC page, applies a mask preserving writable bits, then writes the masked result back:

```rust
// vlapic.rs:662-693 (LvtLint0 branch)
mask = APIC_LVT_M | APIC_LVT_DS | APIC_LVT_VECTOR  // 0x10000 | 0x1000 | 0xFF = 0x110FF
       | LVT_LINT0::TriggerMode::SET.mask()          // bit 15
       | LVT_LINT0::RemoteIRR::SET.mask()            // bit 14
       | LVT_LINT0::InterruptInputPinPolarity::SET.mask()  // bit 13
       | LVT_LINT0::DeliveryMode::SET.mask();        // bits 10:8 = 0x700
// mask = 0x1F7FF (bits 16,15,14,13,10,9,8,7,6,5,4,3,2,1,0)
val &= mask;
info!("[VLAPIC] LINT0 write: raw={val:#x} route={:?}", lint0_route_from_lvt(val));
self.regs().LVT_LINT0.set(val);
self.lvt_last.lvt_lint0.set(val);
```

The `lint0_route_from_lvt()` function (vlapic.rs:61-74) is called for the log and returns:

| Input value | Mask bit set? | DeliveryMode | Result |
|------------|---------------|--------------|--------|
| 0x10700 (Phase 0) | YES (bit 16=1) | -- | `None` (masked) |
| 0x00030 (Phase 2 try 1) | NO | Fixed (0x000) | `Some(Fixed { vector: 0x30 })` |
| 0x10030 (Phase 2 fallback) | YES | Fixed | `None` (masked) |
| 0x00700 (Phase 3) | NO | ExtINT (0x700) | `Some(ExtInt)` |

The PIT IRQ handler (axvm/src/vm.rs:288-298) calls `lint0_route()` and can inject interrupts via the Fixed or ExtINT route if `LVT_LINT0` is properly configured.

---

## 8. What Linux Requires for Success on This Path

The timer fallback succeeds (via `timer_irq_works()` returning true, io_apic.c:1519-1543) if, within a brief polling window (~4+ jiffies / ~25ms at HZ=250), the CPU receives at least one IRQ0 tick from the configured interrupt source. The required register writes for each successful path are:

### Fixed mode (Phase 2) success requires:
1. `apic_write(APIC_LVT0, 0x00030)` -- LVT0 unmasked, Fixed delivery, vector 0x30
2. `legacy_pic->unmask(0)` -- unmask IRQ0 on 8259A
3. PIT fires IRQ0 -> 8259A asserts IRQ0 -> IO-APIC pin 2 delivers vector 0x30 to CPU (since the 8259A cascade is connected to IO-APIC pin 2 per the log: `apic2=-1 pin2=-1` with fallback to `pin2=pin1=2`)

**Wait -- correction on Phase 2**: Phase 2 does NOT use IO-APIC routing. It sets up LVT0 as Fixed + unmasked, and expects the PIC to deliver interrupts. Since `apic2=-1 pin2=-1`, the 8259A is not connected to IO-APIC. Phase 2 expects PIT -> 8259A -> LINT0 pin directly. But in a VM without a real 8259A LINT0 wire, this path cannot work unless the vLAPIC ExtINT/fixed route is functional.

### ExtINT mode (Phase 3) success requires:
1. `legacy_pic->init(0)` -- re-init 8259A
2. `legacy_pic->make_irq(0)` -- connect IRQ0
3. `apic_write(APIC_LVT0, 0x00700)` -- LVT0 unmasked, ExtINT delivery, vector 0
4. `legacy_pic->unmask(0)` -- unmask IRQ0 on 8259A
5. PIT fires IRQ0 -> 8259A drives LINT0 pin -> CPU performs INTA -> 8259A supplies vector

In a VM, the vLAPIC must:
- Recognize ExtINT delivery mode on LVT0
- On PIT IRQ0, route the interrupt as ExtINT through the vLAPIC to the vCPU
- Respond to INTA by reading the PIC's interrupt vector

---

## 9. Complete List of APIC_LVT0 Register Writes Before Panic

In chronological execution order, the guest Linux kernel writes to APIC_LVT0 four times before the panic:

| # | Function | io_apic.c or apic.c line | Value written | Delivery mode | Mask | Vector |
|---|----------|-------------------------|---------------|---------------|------|--------|
| 1 | `clear_local_APIC()` | apic.c:1136 | 0x10000 | (none, masked) | YES | 0 |
| 2 | `setup_local_APIC()` | apic.c:1353 | 0x00700 | ExtINT | no | 0 |
| 3 | `check_timer()` Phase 0 | io_apic.c:2077 | 0x10700 | ExtINT | YES | 0 |
| 4 | `check_timer()` Phase 2 try 1 | io_apic.c:2156 | 0x00030 | Fixed | no | 0x30 |
| 5 | `check_timer()` Phase 2 fallback | io_apic.c:2164 | 0x10030 | Fixed | YES | 0x30 |
| 6 | `check_timer()` Phase 3 | io_apic.c:2171 | 0x00700 | ExtINT | no | 0 |

All six writes use xAPIC MMIO semantics (32-bit store to GPA `0xfee00350`).

---

## 10. Log Evidence vs Source: Key Observation

The AxVisor vLAPIC logs `"[VLAPIC] LINT0 write: raw=... route=..."` in `write_lvt()` (vlapic.rs:669-671) for every LVT0 write received. The log file contains **zero** instances of this message anywhere.

The log also shows `"[VLAPIC] init: vm=1 vcpu=0 IA32_APIC_BASE shadow=0xfee00900"` (log line 304), confirming the vLAPIC was created. The APIC_ACCESS exit handler (vmx/vcpu.rs:1264) does log `[VMX] APIC_ACCESS exit: ...` for every intercepted APIC MMIO access. One such entry appears in the log (log line 2473):

```
[VMX] APIC_ACCESS exit: rip=0xffffffff812713d6 len=6 info=ApicAccessExitInfo { offset: 752, access_type: LinearDataRead, non_event_delivery_asynchronous: false }
```

This is a read at offset 752 (0x2F0), occurring *after* the panic (timestamp 19.893126 from outer AxVisor, vs the panic at inner time 0.284222). No APIC_ACCESS exits for writes to offset 0x350 (LVT0) appear in the log at any point during the nested Linux boot.

---

## 11. Summary of Bit Layout Reference

From `arch/x86/include/asm/apicdef.h` (lines 106-110):

```
APIC_LVT0          = 0x350      (register offset on APIC MMIO page)
APIC_LVT_MASKED    = (1 << 16)  = 0x10000   (bit 16)
APIC_DM_FIXED      = 0x00000    (bits 10:8 = 000)
APIC_DM_EXTINT     = 0x00700    (bits 10:8 = 111)
APIC_VECTOR_MASK   = 0x000FF    (bits 7:0)
```

From `x86_vlapic/src/regs/lvt/lint0.rs` (the vLAPIC register definition):

```
LVT_LINT0::Mask                OFFSET(16) NUMBITS(1)   = bit 16
LVT_LINT0::TriggerMode         OFFSET(15) NUMBITS(1)   = bit 15
LVT_LINT0::RemoteIRR           OFFSET(14) NUMBITS(1)   = bit 14
LVT_LINT0::InterruptInputPinPolarity OFFSET(13) NUMBITS(1) = bit 13
LVT_LINT0::DeliveryStatus      OFFSET(12) NUMBITS(1)   = bit 12 (read-only)
LVT_LINT0::DeliveryMode        OFFSET(8)  NUMBITS(3)   = bits [10:8]
LVT_LINT0::Vector              OFFSET(0)  NUMBITS(8)   = bits [7:0]
```

The Linux APIC constants and the AxVisor vLAPIC bit layout are consistent for the fields that matter: Mask, DeliveryMode, and Vector.
