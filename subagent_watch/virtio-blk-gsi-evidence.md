# virtio-blk INTx GSI 证据分析

## 分析目标

证明、否定或明确说明"无法证明"以下命题：

**当前 AxVisor 的 nested OVMF legacy virtio-blk INTx 应该接到哪个 GSI？**

---

## 1. 直接证据

**未看到直接证据。**

当前 AxVisor 本地代码里，没有任何"nested OVMF legacy virtio-blk → 某个 GSI"的直接证据。

具体说明：

- `tgoskits/virtualization/axdevice/src/virtio_blk.rs`：设备核心只关心 port I/O 和 queue，不涉及 GSI。`LEGACY_BLK_IO_BASE = 0x6000` 是 port 地址，不是 IRQ。
- `tgoskits/virtualization/axvm/src/vm.rs`：`handle_ovmf_virtio_blk_io_write` 行 1230 用的是 trace 日志 `INTx injection is not wired yet`，没有任何 GSI 值。
- `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`：`emu_devices = []`，无 IOAPIC 配置，无 IRQ 路由信息。
- `tgoskits/virtualization/axvm/src/runtime/x86_irq.rs`：硬编码 PIT `GSI = 0`、COM1 `GSI = 4`，与 virtio-blk 无关。

---

## 2. 间接证据

### 证据 A：mptable.rs 的 pci_intx_gsi 函数

**文件**：`tgoskits/os/axvisor/src/images/x86/mptable.rs`，行 144-155

```rust
const fn pci_intx_gsi(dev: u8, pin: u8) -> u8 {
    if dev == 3 && pin == 0 {
        18
    } else {
        16 + ((dev + pin) & 3)
    }
}
```

注释（行 145-149）：

> The current x86 smoke setup passes the outer q35 00:03.0 virtio-blk device through to the guest. QEMU routes that device's INTA# to host IOAPIC GSI 18, and Axvisor forwards host IOAPIC GSIs by number.

**能证明什么**：

- 这是 AxVisor **唯一的** virtio-blk GSI 直接声明。
- 明确指出 outer QEMU 的 00:03.0 virtio-blk 对应 **host IOAPIC GSI 18**。
- 这个值是 `18` 而不是标准 q35 公式算出的 `23`。

**关键限制**（三个）：

1. **mptable 只用于 Linux boot 路径**。`mod.rs` 行 497-504 中，`x86_mptable::build()` 只在 `build_x86_linux_images_from_filesystem` 中调用。`load_uefi_ovmf_images()`（行 331-371）**不加载 MP table**。

2. **这个 mptable 是给 AxArceOS guest 用的**（Linux 直接启动路径），不是给 nested OVMF 用的。

3. **这是 passthrough 模式下的映射**，即 host 物理 IOAPIC GSI 直通。切换到 emulated IOAPIC 后，GSI 由 AxVisor 自己定义，与 host 无关。

---

### 证据 B：QEMU q35 ICH9 PIRQ 路由公式

**文件**：`references/qemu/hw/isa/lpc_ich9.c`，行 124-128

```c
for (slot = 0; slot < PCI_SLOT_MAX; slot++) {
    for (intx = 0; intx < PCI_NUM_PINS; intx++) {
        lpc->irr[slot][intx] = (slot + intx) % 4 + 4;
    }
}
```

**文件**：`references/qemu/hw/isa/lpc_ich9.c`，行 239-242

```c
static int ich9_pirq_to_gsi(int pirq) {
    return pirq + ICH9_LPC_PIC_NUM_PINS;  // pirq + 16
}
```

**推导链**：

- slot 1 INTA：`(1+0)%4 + 4 = 5` → PIRQF → GSI 5+16 = **21**
- slot 3 INTA：`(3+0)%4 + 4 = 7` → PIRQH → GSI 7+16 = **23**

**与证据 A 的矛盾**：

mptable.rs 说 slot 3 INTA → GSI 18，但 q35 标准公式算出 slot 3 INTA → GSI 23。两个值不一致。可能的解释：
- mptable.rs 针对 AxVisor 实际 host IOAPIC 情况做了覆盖（注释暗示 QEMU 实际路由就是 18）。
- 或者 mptable.rs 的计算使用了特殊值而非标准 q35 PIRQH 公式。

**无法确认哪个是正确的**，因为没有 AxVisor 外层 QEMU 的 IOAPIC 路由表实际内容的日志证据。

---

### 证据 C：OVMF BdsPlatform.c SetPciIntLine 算法

**文件**：`edk2/OvmfPkg/Library/PlatformBootManagerLib/BdsPlatform.c`，行 35-40

```c
CONST UINT8  PciHostIrqs[] = {
  0x0a, // LNKA, LNKE  (IRQ 10)
  0x0a, // LNKB, LNKF  (IRQ 10)
  0x0b, // LNKC, LNKG  (IRQ 11)
  0x0b  // LNKD, LNKH  (IRQ 11)
};
```

**文件**：`edk2/OvmfPkg/Library/PlatformBootManagerLib/BdsPlatform.c`，行 1197-1283（SetPciIntLine）

**算法**：
1. `Idx = InterruptPin - 1`（INTA → 0）
2. 遍历 device path，累加 `Idx += Device`（slot 1 → Idx = 1）
3. Q35 平台：若 `RootSlot <= 24`，不调整（PIRQ E-H 区间）
4. `Idx %= 4`
5. Q35 slot 1 INTA：`Idx = 1 % 4 = 1` → `IrqLine = PciHostIrqs[1] = 0x0a`（IRQ 10）

**能证明什么**：

OVMF 会把 slot 1 INTA 写入 PCI config space `InterruptLine = 0x0a`（IRQ 10）。然后通过 ACPI _PRT 找到对应的 PIRQ link 设备，再解出最终 GSI。

**关键限制**：这依赖 PCI 枚举和后续 IRQ 路由信息。当前 AxVisor 没有自己的 PCI config / ACPI IRQ 路由模型，但由于 `x86_vcpu` 没有拦 `0xcf8/0xcfc`，nested OVMF 仍然能直接枚举 outer QEMU 的 `virtio-blk-pci`。

---

### 证据 D：OVMF 不接收 AxVisor 的 MP table

**文件**：`tgoskits/os/axvisor/src/images/mod.rs`

行 331-371：`load_uefi_ovmf_images()` 只加载 OVMF_CODE 和 OVMF_VARS，**不加载 MP table**。

行 497-504：MP table 只在 `build_x86_linux_images_from_filesystem` 中加载（Linux boot 路径）。

行 723-730：UEFI boot 路径的 `load_vm_images_from_filesystem` 只调用 `load_uefi_ovmf_images` 和 `load_virtio_blk_disk_from_filesystem`。

**能证明什么**：

nested OVMF guest 完全没有来自 AxVisor 的 MP table。OVMF 是 UEFI firmware，本身不搜索 EBDA 区域的 MP table（那是 BIOS/SMBIOS/SeaBIOS 的做法）。

---

### 证据 E：nested OVMF 没有 AxVisor 自己提供的 IRQ 路由信息来源

综合以上证据：

| IRQ 路由信息来源 | 是否提供给 nested OVMF |
|---|---|
| AxVisor MP table | 否（`load_uefi_ovmf_images` 不加载） |
| AxVisor ACPI tables（MADT/_PRT/DSDT） | 否（AxVisor 不生成 ACPI 表给 nested guest） |
| outer QEMU PCI config space InterruptPin/Line | 是（`0xcf8/0xcfc` 未被 AxVisor 拦截，OVMF 可直接枚举 outer QEMU `virtio-blk-pci`） |
| fw_cfg IRQ 路由 | 未看到证据（fw_cfg 当前提供 e820 和 CPU 拓扑，未看到 IRQ 路由相关 item） |

**结论**：nested OVMF 能看见 outer QEMU 的 PCI 设备，但 AxVisor 没有把这条 PCI 发现链延续成自己的 IRQ 路由模型。guest 看得到设备，不等于 guest 会编程 emulated IOAPIC。

---

### 证据 F：当前 passthrough 模式下的中断路径

**文件**：`tgoskits/virtualization/axvm/src/runtime/x86_irq.rs`

行 27-35：`forward_passthrough_irq_from_vmexit` 将 host IOAPIC 中断转发给 guest vCPU。

**文件**：`tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml`

行 58：`interrupt_mode = "passthrough"`

行 69：`["IO APIC", 0xfec0_0000, 0xfec0_0000, 0x1000, 0x1]`

**能证明什么**：

当前 nested OVMF 看到的是 host 物理 IOAPIC（passthrough），不是 AxVisor 的 emulated IOAPIC。guest 的 IOAPIC MMIO 访问直接穿透到 host，不触发 vmexit。

host 物理 IOAPIC 的 redirection table 由 host Linux 内核管理。nested OVMF 不能重新编程 host IOAPIC 的 redirection table（它的 MMIO 写穿透到 host，但 host 内核可能不接受 guest 对 IOAPIC 的写入）。

**这意味着**：即使 AxVisor 知道正确的 GSI 并尝试注入，当前架构下 OVMF 无法自行配置 IOAPIC 来接收 virtio-blk 中断。

---

## 3. 判断

**当前只能证明 OVMF 连 emulated IOAPIC 都没有访问，候选 GSI 仍不成立。**

具体缺口：

1. **旧的 GSI 18 证据只来自 `mptable.rs`**，且那是 Linux passthrough 场景硬编码，不是 nested OVMF emulated IOAPIC 的路由证据。
2. **q35 标准公式和 `mptable.rs` 仍然矛盾**，但这次真实运行不仅没有 IOREDTBL 写入，连 `IOREGSEL/IOWIN` 日志都没有，说明问题已经不在“挑哪个候选 GSI”，而在“OVMF 根本没访问 emulated IOAPIC”。
3. **guest 访问到了 PCI 设备**：日志里出现 `PciRoot(0x0)/Pci(0x1,0x0)`，说明外层 QEMU 的 `virtio-blk-pci` 确实被看见了。
4. **I/O bitmap 没有拦 PCI config 端口**，所以 nested OVMF 仍然在直接枚举外层 QEMU 的 PCI 设备，而不是 AxVisor 自己的 PCI/IRQ 模型。

---

## 4. 实现建议

**应先补证据，不要接线。**

具体原因：

1. **本次运行没有任何 emulated IOAPIC MMIO 访问**：`EmulatedIoApic::handle_read()` / `handle_write()` 扩大日志后，仍然没有出现 `vIOAPIC IOREGSEL write ...`、`vIOAPIC IOWIN read ...`、`vIOAPIC IOWIN write ...`，更没有 `vIOAPIC redirection entry ...`。
2. **OVMF 还是在走 PCI 枚举**：`PciRoot(0x0)/Pci(0x1,0x0)` 证明它至少看见了一个 PCI 设备，但没有把这个发现转成 IOAPIC 编程。
3. **因此现在不该接 INTx**：当前不是缺一个 GSI 常量，而是缺 guest 到 emulated IOAPIC 的整条访问链。
4. **如果后续继续做**：应先确认 OVMF 当前采用的 APIC/IRQ 平台信息来源，再决定要不要补最小 IRQ 路由尾巴。

---

## 附录：证据文件索引

| 文件 | 关键行 | 证据类型 |
|---|---|---|
| `tgoskits/os/axvisor/src/images/x86/mptable.rs` | 144-155 | virtio-blk GSI 18 声明（Linux passthrough 场景） |
| `references/qemu/hw/isa/lpc_ich9.c` | 124-128, 239-242 | q35 PIRQ 路由公式和 PIRQ→GSI 映射 |
| `edk2/OvmfPkg/Library/PlatformBootManagerLib/BdsPlatform.c` | 35-40, 1197-1283 | OVMF PCI IRQ 分配算法 |
| `tgoskits/os/axvisor/src/images/mod.rs` | 331-371, 497-504 | UEFI 路径不加载 MP table |
| `tgoskits/virtualization/x86_vlapic/src/vioapic.rs` | 38 | IOAPIC redirection entry 默认 masked |
| `tgoskits/os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` | 58, 62, 69 | interrupt_mode=passthrough, emu_devices=[], IOAPIC passthrough |
| `tgoskits/virtualization/axvm/src/vm.rs` | 1230-1233 | INTx 注入未接线的 trace 日志 |
| `tgoskits/virtualization/axvm/src/runtime/x86_irq.rs` | 27-35, 37-61 | passthrough IRQ 转发和 PIT/COM1 硬编码 GSI |
| `tgoskits/os/axvisor/configs/qemu/qemu-x86_64.toml` | 6, 12 | QEMU machine=q35, virtio-blk-pci |
| `tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs` | `SvmVcpu::setup_io_bitmap()` | 未拦 `0xcf8/0xcfc`，PCI config 访问仍然 passthrough |
| `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs` | `VmxVcpu::setup_io_bitmap()` | 同上 |
| `/tmp/ovmf-ioapic-smoke.log` | 1992-2145 | `PciRoot(0x0)/Pci(0x1,0x0)` + `OVMF-VIRTIO-BLK-IO irq=true`，但无 IOREDTBL 写入 |
| `/tmp/ovmf-ioapic-smoke2.log` | 1893-2150 | 扩大到 `IOREGSEL/IOWIN` 后仍无任何 emulated IOAPIC 访问日志 |

---

## 本次文档修改记录

新增文件：

- `axvisor-2026-notebooks/subagent_watch/virtio-blk-gsi-evidence.md`
