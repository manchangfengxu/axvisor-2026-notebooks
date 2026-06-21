# VMX Runtime Poll Points for Software-Injected Legacy IRQ

Scope: Which VM-exit mechanisms reliably deliver periodic PIT/serial/legacy IRQ to
a guest that may be in a tight busy-wait before its own timer subsystem is running.

---

## 1. Source Inventory

### 1.1 Intel SDM (local PDF)

- **File**: `axvisor-2026-notebooks/docs/x86_64/intel/intel-sdm-combined-volumes.pdf`
- **VMX preemption timer**: Vol. 3C, Chapter 24.7 (Pin-Based VM-Execution Controls),
  specifically the "Activate VMX-preemption timer" bit. VM-exit reason 52 in
  Table 24-1 when the timer reaches zero. Counts down by 1 each time bit X in
  the TSC changes; X is determined by `IA32_VMX_MISC[4:0]`.
- **External interrupt exiting**: Vol. 3C, Chapter 24.7, pin-based control bit
  "External-interrupt exiting" (bit 0). VM-exit reason 1.
- **HLT exiting**: Vol. 3C, Chapter 24.7, primary processor-based control bit
  "HLT exiting" (bit 7). VM-exit reason 12.
- **PAUSE-loop exiting**: Vol. 3C, Chapter 25.1.2, secondary processor-based
  control "PAUSE-loop exiting" (bit 15). VM-exit reason 40. For spin-wait
  loops of N+1 iterations where the PAUSE count exceeds threshold.
- **MWAIT exiting**: Vol. 3C, Chapter 24.7, primary processor-based control bit
  "Monitor trapping/MWAIT exiting" (bit 10). VM-exit reason 36.
- **Interrupt-window exiting**: Vol. 3C, Chapter 24.7, primary processor-based
  control bit "Interrupt-window exiting" (bit 2). VM-exit reason 7.
- **Posted interrupts**: Vol. 3C, Chapter 29.1. "Posted-interrupt notification"
  pin-based control bit (bit 1). Enables delivery of virtual interrupts without
  a VM exit via a posted-interrupt descriptor in memory. Requires APICv-capable
  processor.
- **APICv / Virtual interrupt delivery**: Vol. 3C, Chapter 29.4, secondary
  processor-based control "Virtual interrupt delivery" (bit 31). Enables direct
  injection of virtual interrupts into the guest via the APIC page without exit.

### 1.2 AxVisor codebase

- **File**: `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
  - Pin-based controls set at line 729-731: enables `EXTERNAL_INTERRUPT_EXITING`
    and `VMX_PREEMPTION_TIMER` (along with `NMI_EXITING`).
  - Preemption timer value: `VMX_PREEMPTION_TIMER_SET_VALUE = 100_000` (line 221).
  - `handle_vmx_preemption_timer()` (line 1578): simply reloads the timer value
    to 100,000 and returns. No device polling or injection triggered.
  - `VmxExitReason::EXTERNAL_INTERRUPT` (line 2340): returns
    `AxVCpuExitReason::ExternalInterrupt` to the VM loop.
  - `VmxExitReason::PREEMPTION_TIMER` (line 2347): calls `handle_vmx_preemption_timer()`
    then returns `AxVCpuExitReason::PreemptionTimer`.
  - `VmxExitReason::HLT` (line 2351): advances RIP by instruction length,
    returns `AxVCpuExitReason::PreemptionTimer` (note: maps to PreemptionTimer,
    not Halt -- likely a code path simplification).
  - Interrupt-window exiting: `set_interrupt_window()` at line 505, used by
    `inject_pending_events()` when interrupts are blocked (line 1128).
  - `inject_pending_events()` (line 1111): checks `allow_interrupt()`, injects
    via `vmcs::inject_event()` if possible, otherwise enables interrupt-window
    exiting.
  - Posted interrupt support: `POSTED_INTERRUPT_NOTIFICATION_VECTOR` (VMCS field
    0x282E) and `POSTED_INTERRUPT_DESC_ADDR` (VMCS field 0x2016) are defined in
    `vmcs.rs` but not configured in the VCPU setup path. `VIRTUAL_INTERRUPT_DELIVERY`
    secondary control is conditionally enabled (line 761) if hardware supports it.
- **File**: `tgoskits/virtualization/x86_vcpu/src/vmx/definitions.rs`
  - Full VMX exit reason enum (line 91-230). Key entries: EXTERNAL_INTERRUPT=1,
    INTERRUPT_WINDOW=7, HLT=12, MWAIT_INSTRUCTION=36, PAUSE_INSTRUCTION=40,
    PREEMPTION_TIMER=52.

### 1.3 QEMU (local reference tree)

- **File**: `references/qemu/hw/timer/i8254.c`
  - PIT frequency: `PIT_FREQ = 1193182` Hz (from `include/hw/timer/i8254.h:33`).
  - Channel 0 timer is connected to IRQ line via `qdev_init_gpio_out()` (line 349).
  - `pit_irq_timer()` (line 283): QEMU timer callback, fires on
    `QEMU_CLOCK_VIRTUAL` (guest CPU time). Calls `pit_irq_timer_update()` which
    calls `qemu_set_irq(s->irq, irq_level)` to signal the PIC/IOAPIC.
  - `pit_irq_timer_update()` (line 260): computes next transition time from PIT
    count and schedules `timer_mod()` for the next edge.
  - Timer initialization: `timer_new_ns(QEMU_CLOCK_VIRTUAL, pit_irq_timer, s)`
    (line 348).
- **File**: `references/qemu/accel/kvm/kvm-all.c`
  - `kvm_set_irq()` (line 2137): issues `KVM_IRQ_LINE` ioctl to the kernel.
    This is the path QEMU uses when irqchip is not in-kernel.
  - `kvm_irqchip_in_kernel()` is checked throughout to decide between in-kernel
    and userspace interrupt delivery.
  - `KVM_EXIT_IRQ_WINDOW_OPEN` (line 3540): QEMU returns `EXCP_INTERRUPT`,
    which causes the main loop to re-inject pending interrupts.
  - Main vCPU run loop (line 3453): calls `kvm_arch_pre_run()` before each
    `KVM_RUN` ioctl.
- **File**: `references/qemu/target/i386/kvm/kvm.c`
  - `kvm_arch_pre_run()` (line 5901): If userspace PIC and a HARD interrupt is
    pending with IF=1, injects via `KVM_INTERRUPT` ioctl. If HARD interrupt is
    pending but guest not ready, sets `run->request_interrupt_window = 1` (line
    5979). This causes KVM to exit with `KVM_EXIT_IRQ_WINDOW_OPEN` when the
    interrupt window opens.
  - `kvm_handle_halt()` (line 6105): On `KVM_EXIT_HLT`, checks if there is a
    pending HARD interrupt with IF=1 or NMI. If so, returns 0 (guest un-halts).
    Otherwise sets `cs->halted = 1` and returns `EXCP_HLT`.
  - When `kvm_irqchip_in_kernel()` is true (line 6077), `kvm_handle_halt()`
    returns 0 immediately -- the kernel manages PIT->PIC->APIC->vCPU delivery
    entirely in-kernel. No exit needed.

### 1.4 Cloud-hypervisor (local reference tree)

- **File**: `references/cloud-hypervisor/devices/src/legacy/i8042.rs`
  - Line 40: `data[0] = 0x20` -- returns bit 5 set in port B register to
    **avoid hang in `pit_calibrate_tsc()` in Linux kernel**. This confirms
    cloud-hypervisor deliberately avoids PIT emulation and instead relies on
    TSC-based timing for Linux guests. Cloud-hypervisor does NOT emulate the
    8254 PIT timer.
- Cloud-hypervisor uses `devices/src/legacy/serial.rs` for serial, and
  `devices/src/ioapic.rs` + `devices/src/interrupt_controller.rs` for interrupt
  routing. No PIT device model found.

### 1.5 KVM headers

- **File**: `references/qemu/linux-headers/linux/kvm.h`
  - `KVM_EXIT_HLT = 5`, `KVM_EXIT_IRQ_WINDOW_OPEN = 7`
  - `KVM_IRQ_LINE = _IOW(KVMIO, 0x61, struct kvm_irq_level)` (line 1270)
  - `KVM_CAP_IRQCHIP = 0` (line 725)

---

## 2. Answers to Questions

### Q1. VMX preemption timer

**What it is**: A countdown timer in VMCS that decrements by 1 each time bit X
of the TSC changes. The value of X is set per-processor via `IA32_VMX_MISC[4:0]`.
When it reaches zero, a VM exit with reason 52 (`PREEMPTION_TIMER`) fires.

**Typical use in real implementations**:
- **KVM**: Does NOT use the VMX preemption timer for regular IRQ injection. KVM
  relies on its in-kernel irqchip (PIT/PIC/IOAPIC/LAPIC chain in kernel space)
  for PIT interrupt delivery. The preemption timer is used for nested virtualization
  (L2 guest time management) and for the `KVM_CAP_X86燚` TSC offset mechanism.
  (UNCONFIRMED for general KVM -- this is based on public KVM source patterns;
  the exact use may vary by kernel version.)
- **Xen**: Uses the VMX preemption timer for single-stepping and event injection
  timing in some configurations.
- **AxVisor**: Currently uses it as a periodic poll point. Set to 100,000, reloaded
  on each exit (line 1578-1583 of `vcpu.rs`).

**Reliability for early PIT injection**:
- **Guarantees triggering during busy-wait**: YES -- the timer counts in VMX-non-root
  mode regardless of what the guest executes. It fires even during tight loops
  with no HLT/PAUSE/MWAIT.
- **Reliable before timer bring-up**: CONDITIONALLY. The TSC may not be reliable
  before `calibrate_delay()` or TSC-frequency calibration in Linux. If the TSC
  is invariant (CPUID.80000007H:EDX[8]), the preemption timer rate is predictable.
  If the TSC is not invariant (older CPUs), the countdown rate varies with CPU
  frequency scaling, making timing unreliable during early boot.
- **Key limitation**: The preemption timer exit is a VM exit to the VMM. The VMM
  must then process the exit, check for pending PIT IRQ, and inject it. This adds
  latency. For a 100 Hz PIT (10 ms period) the timer value must be chosen to
  expire within that window. With AxVisor's value of 100,000 and unknown TSC rate
  at boot, the actual wall-clock period is unpredictable.
- **SDM reference**: Vol. 3C, Chapter 24.7 (pin-based control), Chapter 24.8
  (VM-exit information for reasons).

### Q2. Host external interrupt exits

**What it is**: When pin-based control bit "External-interrupt exiting" (bit 0)
is set, any host external interrupt received while the guest is running causes a
VM exit with reason 1 (`EXTERNAL_INTERRUPT`). The VMM then handles the interrupt
and can inject a virtual interrupt into the guest before re-entering.

**Commonly used as poll point?**: YES, this is the primary mechanism mature VMMs
use to ensure timely interrupt delivery to the guest. The pattern:

1. VMM enables `EXTERNAL_INTERRUPT_EXITING` in pin-based controls.
2. VMM enables `INTERRUPT_WINDOW_EXITING` in primary processor-based controls when
   it has a pending virtual interrupt but the guest cannot accept it (IF=0 or
   interrupt blocked by STI shadow / MOV-SS).
3. On each VM exit (whether from external interrupt, I/O, HLT, or any other
   reason), the VMM calls `inject_pending_events()` which checks if the guest can
   accept the interrupt and either injects directly via VMCS interrupt field or
   enables interrupt-window exiting.

**Evidence from QEMU**:
- `kvm_arch_pre_run()` (kvm.c:5901): checks `run->ready_for_interrupt_injection`
  and `cpu_test_interrupt(cpu, CPU_INTERRUPT_HARD) && (env->eflags & IF_MASK)`.
  If guest is ready, injects via `KVM_INTERRUPT`. If not ready but interrupt is
  pending, sets `run->request_interrupt_window = 1` (line 5979).
- `KVM_EXIT_IRQ_WINDOW_OPEN` handler (kvm-all.c:3540): returns `EXCP_INTERRUPT`,
  causing the main loop to inject the pending interrupt.

**Guarantees triggering during busy-wait**: NO -- external interrupt exits only
fire when the host actually receives a hardware interrupt. If no host IRQ arrives
(e.g., the PIT device is emulated entirely in the VMM, not via a host timer),
this exit never fires. However, if the VMM uses a host-side timer to emulate
PIT (as QEMU does with `timer_new_ns(QEMU_CLOCK_VIRTUAL, ...)`), the host
timer interrupt will trigger this exit path.

**Reliable before timer bring-up**: YES, if the VMM has already configured the
emulated PIT timer to fire host-side interrupts. The host IRQ delivery is
independent of guest state.

### Q3. Linux `setup_IO_APIC` busy-wait

During Linux early boot, `setup_IO_APIC()` (in `arch/x86/kernel/apic/io_apic.c`)
programs the I/O APIC RTEs and then runs a calibration busy-wait loop. This loop:

- Does NOT execute `HLT` -- it is a pure spin-wait reading the PIT counter.
- May or may not have `RFLAGS.IF = 1` -- early setup code typically runs with
  interrupts disabled or partially enabled.
- Does NOT execute `PAUSE` in most kernel versions (some kernels use `rep; nop`
  in the delay loop, but `setup_IO_APIC` calibration may use a tight counter loop).

**Most likely exit type during this stage**:
1. **HLT exits**: UNLIKELY -- the loop does not execute HLT.
2. **PAUSE exits**: UNLIKELY -- unless the kernel's delay loop contains `PAUSE`.
   Modern Linux `cpu_relax()` contains `rep; nop` (PAUSE), but the PIT
   calibration loop in `setup_IO_APIC` may not use `cpu_relax()`.
   (UNCONFIRMED for the specific kernel version in use.)
3. **VMX preemption timer**: GUARANTEED to fire eventually, but timing is
   non-deterministic (depends on TSC rate at boot).
4. **I/O exits**: GUARANTEED if the PIT is port-intercepted -- the calibration
   loop reads port 0x42 (PIT channel 2 count) via I/O. Each read causes an
   I/O exit. This is the most reliable and frequent exit during PIT calibration.
5. **External interrupt exits**: Only if a host interrupt arrives during the
   busy-wait window.

**Conclusion**: For Linux `setup_IO_APIC` specifically, the most likely stably
occurring exit is the **I/O exit** from PIT counter reads (port 0x40-0x43). The
VMX preemption timer is a secondary guarantee. HLT and PAUSE exits are unlikely.

### Q4. Most robust trigger point combination for legacy timer before normal clock

Before the guest has established its own timer (TSC calibrated, local APIC
timer configured, HPET/PIT recognized), the VMM needs to deliver IRQ0 (PIT)
and potentially IRQ4 (serial) periodically. The most robust combination:

1. **VMX preemption timer as the primary periodic poll point** -- guarantees
   the VMM gets control at a roughly known interval regardless of guest behavior.
   Set the timer value so it expires within one PIT period (e.g., for 100 Hz PIT,
   set it to expire within ~10 ms of guest CPU time). This is the ONLY mechanism
   that works during a pure busy-wait with interrupts disabled.

2. **HLT exits as a secondary (low-frequency) poll point** -- when the guest
   eventually executes HLT (e.g., idle loop), the VMM gets a clean exit and can
   inject all pending interrupts. Enable HLT exiting in primary processor-based
   controls.

3. **Interrupt-window exiting as the injection guarantee** -- whenever the VMM
   has a pending virtual interrupt but the guest cannot accept it (IF=0 or
   blocking), enable interrupt-window exiting. This ensures injection happens
   as soon as the guest re-enables interrupts.

4. **Host external interrupt exits for timing accuracy** -- if the VMM implements
   PIT as a host-side timer (like QEMU), the host timer IRQ will cause an
   external interrupt exit, giving the VMM a natural trigger to inject the
   virtual PIT IRQ. This is the most accurate mechanism when available.

5. **Posted interrupts (APICv)**: If the processor supports it and the VMM
   enables virtual interrupt delivery (secondary processor-based control bit 31),
   pending virtual interrupts can be delivered directly to the guest via the
   posted-interrupt mechanism without a VM exit. This is the lowest-latency path
   but requires the VMM to have the interrupt ready before the guest enters
   non-root mode.

**Recommended combination for a type-1 hypervisor** (no host OS IRQ infrastructure):
- Primary: VMX preemption timer (guaranteed exit, works during busy-wait)
- Secondary: HLT exiting (catches idle states)
- Always-on: interrupt-window exiting (ensures injection when guest is ready)

### Q5. Recommended priority ordering with justification

| Priority | Poll Point | Reliable before timer bring-up | Guarantees exit during busy-wait | Notes |
|----------|-----------|-------------------------------|--------------------------------|-------|
| **1** | VMX preemption timer | YES (if TSC is invariant) | YES | Only mechanism that fires during pure tight loops. Must be reloaded on each exit. Rate depends on TSC; use conservative value. |
| **2** | HLT exits | YES | NO (only on HLT) | Catches idle states reliably. Useless during busy-wait but essential for energy-efficient idle. |
| **3** | Interrupt-window exiting | YES | NO (only when window opens) | Not a poll point per se, but guarantees injection delivery once the VMM has a pending interrupt. Must always be enabled when a virtual IRQ is pending and guest can't accept it. |
| **4** | Host external interrupt exits | YES | Depends on host timer | Best accuracy when VMM runs PIT as host-side timer. Type-2 VMMs (KVM) use this naturally. Type-1 VMMs may not have host IRQ infrastructure. |
| **5** | PAUSE-loop exiting | NO (depends on guest code) | NO (only on PAUSE) | Only fires if guest executes PAUSE in a loop with count exceeding threshold. Not useful during `setup_IO_APIC` busy-wait. Useful for Linux `cpu_relax()`-based spin-waits. |
| **6** | MWAIT exiting | NO (depends on guest code) | NO (only on MWAIT) | Only fires if guest executes MWAIT. Linux uses MWAIT in deep idle, not during early boot. |
| **7** | Posted interrupts / APICv | YES (when available) | N/A (delivery without exit) | Not a poll point. Lowest-latency interrupt delivery. Requires APICv hardware + VMM support. Complementary to all poll points above. |

---

## 3. Evidence Table

| Claim | Source | Evidence |
|-------|--------|----------|
| VMX preemption timer counts down by 1 per TSC bit-X change | Intel SDM Vol. 3C, Ch. 24.7; AxVisor `vcpu.rs:1578-1583` | Comment in `handle_vmx_preemption_timer()` quotes SDM directly. Value `VMX_PREEMPTION_TIMER_SET_VALUE = 100_000`. |
| External interrupt exiting is enabled in AxVisor | AxVisor `vcpu.rs:729-731` | `PinCtrl::EXTERNAL_INTERRUPT_EXITING` in pin-based controls. |
| QEMU PIT fires via QEMU_CLOCK_VIRTUAL host timer | QEMU `i8254.c:348` | `timer_new_ns(QEMU_CLOCK_VIRTUAL, pit_irq_timer, s)` |
| QEMU PIT connects to IRQ via `qemu_set_irq()` | QEMU `i8254.c:270` | `qemu_set_irq(s->irq, irq_level)` |
| KVM uses interrupt-window for userspace PIC delivery | QEMU `kvm.c:5978-5979` | `run->request_interrupt_window = 1` when HARD interrupt pending but guest not ready. |
| KVM in-kernel irqchip handles PIT without exits | QEMU `kvm.c:6077` | `if (kvm_irqchip_in_kernel()) { return 0; }` in `kvm_arch_post_run()` -- no userspace injection needed. |
| Cloud-hypervisor avoids PIT, uses TSC for timing | Cloud-hypervisor `i8042.rs:40-41` | `data[0] = 0x20` to "avoid hang in pit_calibrate_tsc() in Linux kernel." |
| PIT frequency is 1,193,182 Hz | QEMU `include/hw/timer/i8254.h:33` | `#define PIT_FREQ 1193182` |
| AxVisor interrupt-window mechanism exists | AxVisor `vcpu.rs:505-514, 1111-1136` | `set_interrupt_window()`, `inject_pending_events()`, `handle_interrupt_window()`. |
| PAUSE-loop exiting is defined but not used in AxVisor | AxVisor `definitions.rs:169` | `PAUSE_INSTRUCTION = 40` exit reason defined. No handler found in vcpu.rs exit dispatch. |
| Posted interrupt VMCS fields defined but not configured | AxVisor `vmcs.rs:105,137` | `POSTED_INTERRUPT_NOTIFICATION_VECTOR` and `POSTED_INTERRUPT_DESC_ADDR` fields exist. Not set in VCPU setup. |

---

## 4. Recommended Poll-Point Strategy for AxVisor

### 4.1 Current state

AxVisor already enables:
- `EXTERNAL_INTERRUPT_EXITING` (pin-based)
- `VMX_PREEMPTION_TIMER` (pin-based)
- `NMI_EXITING` (pin-based)

But does NOT enable:
- `HLT_EXITING` (primary processor-based control bit 7)
- `PAUSE_LOOP_EXITING` (secondary processor-based control bit 15)
- Posted interrupt notification (pin-based bit 1, fully)

The VMX preemption timer is set to 100,000 and simply reloaded on each exit
without any device polling or injection work.

### 4.2 Recommended changes (informational, not prescriptive)

**Priority 1: Enable HLT exiting.**
Add `CpuCtrl::HLT_EXITING` to the primary processor-based controls in
`vcpu.rs:743-748`. This catches guest HLT instructions (idle loop) and gives
the VMM a chance to inject pending interrupts. Currently, HLT exits are not
enabled, so the guest executing HLT in VMX non-root mode may not cause an exit
depending on other control settings. (Note: the current code at line 2351 handles
`VmxExitReason::HLT` but this reason may never fire if `HLT_EXITING` is not set.)

**Priority 2: Use preemption timer exit for periodic device polling.**
The current `handle_vmx_preemption_timer()` does no useful work beyond reloading
the timer. It should be extended to call device-model tick handlers (PIT counter
decrement, serial timeout check, etc.) before returning. The timer value should
be calibrated to the desired PIT tick rate. For a 100 Hz PIT, the preemption
timer should expire within 10 ms of guest CPU time. The exact value depends on
`IA32_VMX_MISC[4:0]` (the TSC bit-shift factor X):
- Timer counts down by 1 per 2^X TSC increments.
- At 1 GHz TSC, X=5 (typical modern Intel), rate = 1 GHz / 32 = 31.25 MHz.
  10 ms = 312,500 counts.
- At boot before TSC calibration, use a conservative (smaller) value to avoid
  missing the window. AxVisor's current 100,000 may be reasonable for early boot
  but should be increased once TSC frequency is known.

**Priority 3: Keep interrupt-window exiting as always-on injection guarantee.**
AxVisor already does this correctly in `inject_pending_events()`. No change needed.

**Priority 4 (future): Enable posted interrupts / APICv when infrastructure
allows.** This requires managing the posted-interrupt descriptor and the
notification vector. Defer until the vAPIC infrastructure is more mature.

### 4.3 Key insight for AxVisor OVMF path

For the OVMF bring-up, the guest is UEFI firmware (not Linux), and the current
smoke path is polling-based (virtio-blk reads, fw_cfg reads). The PIT is less
critical because UEFI firmware typically does not rely on PIT IRQ0 during early
DXE dispatch. However, if PIT injection is needed for future nested Linux guests
or for UEFI watchdog/timer events:

1. The VMX preemption timer is the only guaranteed poll point during busy-wait.
2. HLT exiting catches the UEFI firmware idle state (if it ever HLTs).
3. The VMM must implement PIT counting logic triggered by preemption timer exits
   and inject IRQ0 via the VMCS interrupt field or posted-interrupt descriptor.

---

## 5. Terminology

- **Poll point**: A VM-exit mechanism that gives the VMM periodic or event-driven
  control over the guest for device emulation purposes.
- **Busy-wait**: Guest executing a tight instruction loop with no HLT/PAUSE/MWAIT,
  such as `setup_IO_APIC` PIT calibration or early UEFI polling loops.
- **Before timer bring-up**: The guest phase before its own timer subsystem (PIT
  recognition, local APIC timer, HPET) is operational and generating periodic
  interrupts. For Linux, this is before `setup_APIC_clocks()`. For UEFI, this is
  before the firmware's timer event infrastructure is initialized.
