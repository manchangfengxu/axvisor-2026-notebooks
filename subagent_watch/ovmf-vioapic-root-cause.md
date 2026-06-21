# OVMF vIOAPIC 未访问根因分析

## 结论

- **OVMF virtio-blk 驱动是纯轮询的，不使用中断完成 I/O。** `VirtioFlush()` 设置 `VRING_AVAIL_F_NO_INTERRUPT` 并 busy-wait `Used.Idx`；没有任何中断处理程序。因此 "guest 完全不碰 IOAPIC" 是预期行为，不是故障。
- **OVMF 本身从不编程 IOAPIC redirection table。** OVMF 只编程 LPC bridge 的 PIRQ routing 寄存器和 PCI 设备的 InterruptLine 字段；IOAPIC redirection table 的编程是操作系统（Linux）的职责，依赖 ACPI MADT + _PRT 表。
- **nested OVMF 看到的是 outer QEMU 的 PCI 设备。** AxVisor 的 I/O bitmap 不拦截 0xCF8/0xCFC，PCI config space MMIO passthrough 到 host。因此 OVMF 枚举的是 outer QEMU 的 q35 PCI 总线，不是 AxVisor 自己的 PCI 模型。
- **AxVisor 的 emulated IOAPIC 架构上正确，但 OVMF 启动期间根本不需要它。** 0xFEC00000 未被 stage-2 页表映射（正确），MMIO 访问会触发 vmexit 路由到 EmulatedIoApic（正确），但 OVMF 永远不会发出这个 MMIO。
- **当前不应继续 virtio-blk INTx 接线。** 接线的前提（guest 编程 emulated IOAPIC）不成立。如果要推进，需要先决定谁负责给 nested guest 提供 IRQ 路由信息（MP table 或 ACPI tables），这是一个平台设计决策，不是小尾巴。
- **OVMF 的 PciAcpiInitialization 只写了 LPC bridge PIRQ routing 寄存器（0x60-0x6B）和 PCI InterruptLine，不碰 IOAPIC。** 这个函数是 OVMF 对 IRQ 路由的全部贡献，IOAPIC 编程留给 OS。

---

## 事实

- 事实：OVMF virtio-blk 驱动文件头注释明确声明 "we stick to synchronous requests"。
  - 证据：`edk2/OvmfPkg/VirtioBlkDxe/VirtioBlk.c:10-11`
  - 解释：驱动是同步设计，不使用异步中断完成 I/O。

- 事实：OVMF 的 `VirtioPrepare()` 设置 `VRING_AVAIL_F_NO_INTERRUPT`，告诉 host 不要发中断。
  - 证据：`edk2/OvmfPkg/Library/VirtioLib/VirtioLib.c:176`
  - 解释：即使 host 想发中断，guest 也声明了不需要。这是标准 UEFI DXE 模式——boot services 时间段没有 virtio 中断基础设施。

- 事实：`VirtioFlush()` 在写入 `SetQueueNotify` 后，busy-wait 轮询 `*Ring->Used.Idx != NextAvailIdx`，用 `gBS->Stall()` 做自适应退避（1→2→4→...→1024 µs）。
  - 证据：`edk2/OvmfPkg/Library/VirtioLib/VirtioLib.c:329-337`
  - 解释：没有中断处理程序，没有 eventfd，没有回调。纯粹是 CPU 自旋等待。

- 事实：AxVisor I/O bitmap 不拦截 0xCF8/0xCFC。
  - 证据：`tgoskits/virtualization/x86_vcpu/src/svm/vcpu.rs:385-406`（SVM）、`tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:481-510`（VMX）。`setup_io_bitmap` 只拦截 QEMU exit port、PIT、COM1、debugcon、fw_cfg、virtio-blk 端口。
  - 解释：guest PCI config space 访问直接穿透到 host，OVMF 枚举的是 outer QEMU 的 PCI 设备。

- 事实：当前 OVMF VM 配置中 IOAPIC 是 emulated device，不在 passthrough_devices 中。
  - 证据：`tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml:62`（`emu_devices` 含 `X86 IO APIC`）；行 69 的 `passthrough_devices` 只有 PCI Low MMIO、PCI MMIO、HPET，没有 IO APIC。
  - 解释：0xFEC00000 未被 stage-2 页表线性映射，guest 访问会触发 NPT/EPT violation 并路由到 EmulatedIoApic。架构正确。

- 事实：EmulatedIoApic 的 redirection table 初始化为全 masked。
  - 证据：`tgoskits/virtualization/x86_vlapic/src/vioapic.rs:38`（`[REDIRECTION_ENTRY_MASKED; 24]`）
  - 解释：即使 AxVisor 尝试通过 `assert_gsi()` 注入中断，如果 guest 没有 unmask 对应的 redirection entry，`assert_gsi()` 返回 `None`，中断被静默丢弃。

- 事实：OVMF 的 `PciAcpiInitialization()` 只编程 LPC bridge 的 PIRQ routing 寄存器和 PCI 设备的 InterruptLine，不碰 IOAPIC。
  - 证据：`edk2/OvmfPkg/Library/PlatformBootManagerLib/BdsPlatform.c:1357-1364`（Q35 LPC bridge PIRQ E-H routing 寄存器 0x60-0x6B）、行 1390（`VisitAllPciInstances(SetPciIntLine)`）
  - 解释：OVMF 假设 IOAPIC 由硬件/固件预初始化或由 OS 编程。它自己不写 IOAPIC redirection table。

- 事实：ACPI MADT（描述 IOAPIC 地址和 GSI base）由 QEMU 生成，通过 fw_cfg 传递给 OVMF，不是 OVMF 自己生成的。
  - 证据：`edk2/OvmfPkg/AcpiPlatformDxe/EntryPoint.c:44`（`InstallQemuFwCfgTables()` 从 fw_cfg 读取 QEMU 预生成的 ACPI 表）
  - 解释：nested OVMF 场景下，AxVisor 的 fw_cfg 不提供 ACPI 表。nested guest 看不到 MADT。

- 事实：OVMF 在 Q35 平台上使用 ECAM（MMCONFIG at 0xE0000000）做 PCI config 访问，不使用 0xCF8/0xCFC。
  - 证据：`edk2/OvmfPkg/Library/DxePciLibI440FxQ35/PciLib.c:92-96`（Q35 上 `PciRead8`/`PciWrite8` dispatch 到 `PciExpressRead8`/`PciExpressWrite8`，使用 ECAM MMIO 窗口）
  - 解释：但 0xE0000000 在 `passthrough_devices` 的 PCI MMIO 范围内（`0xe000_0000` 起 256MB），所以 ECAM 访问也穿透到 host。两种 config 访问方法殊途同归，都看到 outer QEMU 设备。

- 事实：两次 smoke 运行日志中，emulated IOAPIC 初始化成功，但没有任何 vIOAPIC MMIO 访问日志。
  - 证据：`/tmp/ovmf-ioapic-smoke.log`、`/tmp/ovmf-ioapic-smoke2.log`；`vioapic.rs` 中 IOREGSEL/IOWIN 的 `info!()` 日志（行 207-218、259-261、281-284、288-290）无触发
  - 解释：OVMF 在整个启动过程中从未访问 0xFEC00000。

- 事实：OVMF 成功发现 outer QEMU PCI 设备，包括 virtio-blk。
  - 证据：日志中 `PciBus: Discovered PCI @ [00|01|00] [VID = 0x1AF4, DID = 0x1001]`
  - 解释：设备发现正常，I/O 完成正常（通过轮询），不需要中断。

---

## 对 6 个问题的逐项回答

### Q1: nested OVMF 是通过什么路径发现 `PciRoot(0x0)/Pci(0x1,0x0)` 的？

**通过 outer QEMU 的 PCI config space，经由 ECAM MMIO passthrough。**

证据链：

1. AxVisor 的 I/O bitmap（SVM `svm/vcpu.rs:385-406`、VMX `vmx/vcpu.rs:481-510`）不拦截 0xCF8/0xCFC。
2. AxVisor 的 stage-2 页表把 `0xE000_0000` 起 256MB 的 PCI MMIO 空间线性映射到 host（`passthrough_devices` 中的 `["PCI MMIO", 0xe000_0000, 0xe000_0000, 0x1000_0000, 0x1]`）。
3. OVMF 在 Q35 上使用 ECAM（`DxePciLibI440FxQ35/PciLib.c:92-96`），通过 MMIO 访问 PCI config space。
4. ECAM MMIO 穿透到 host 的 MCFG 窗口，OVMF 枚举的是 outer QEMU 的 q35 PCI 总线。
5. Outer QEMU 的 `virtio-blk-pci` 配置在 `qemu-x86_64.toml:12`（`virtio-blk-pci,drive=disk0`），自动分配到 slot 1。

**结论：nested OVMF 看到的 PCI 设备完全是 outer QEMU 的，不是 AxVisor 自己定义的。** AxVisor 没有自己的 PCI config space emulation。

### Q2: OVMF virtio-blk 访问是依赖中断还是轮询？

**纯轮询。OVMF virtio-blk 驱动不使用中断，也不期望中断。**

证据链：

1. 驱动文件头：`VirtioBlk.c:10-11`："we stick to synchronous requests and EFI_BLOCK_IO_PROTOCOL for now"
2. `VirtioPrepare()`（`VirtioLib.c:176`）：`*Ring->Avail.Flags = VRING_AVAIL_F_NO_INTERRUPT`——明确告诉 host 不要发中断
3. `VirtioFlush()`（`VirtioLib.c:329-337`）：写 `SetQueueNotify` 后，busy-wait `*Ring->Used.Idx != NextAvailIdx`，用 `gBS->Stall()` 自旋
4. 没有注册任何中断处理程序（UEFI DXE 环境下没有 virtio 中断基础设施）

**因此"OVMF 完全不碰 IOAPIC"是预期行为。** OVMF 的 virtio-blk boot 路径从头到尾都是 CPU 轮询，不需要 IOAPIC 参与。AxVisor 侧出现的 `irq=true` 只表示 `LegacyNotifyResult.should_raise_irq` 为真（设备核心认为"至少发布了一个 used buffer"），不表示 guest 需要中断。

### Q3: OVMF 在什么条件下才会访问 IOAPIC？

**OVMF 本身不编程 IOAPIC redirection table。IOAPIC 编程是操作系统的职责。**

OVMF 对 IRQ 路由的全部贡献只有两步（`BdsPlatform.c:1332-1396`，在 `PlatformBootManagerAfterConsole` 中调用）：

1. **写 LPC bridge PIRQ routing 寄存器**（Q35：`BdsPlatform.c:1357-1364`，写 0:1f.0 的 config space offsets 0x60-0x6B，值来自 `PciHostIrqs[] = {0x0a, 0x0a, 0x0b, 0x0b}`）
2. **写 PCI 设备的 InterruptLine 字段**（`BdsPlatform.c:1390`，`VisitAllPciInstances(SetPciIntLine)` 遍历所有 PCI 设备）

这些操作只写 PCI config space，不写 IOAPIC MMIO。

**IOAPIC 由以下机制提供给 OS：**
- ACPI MADT 表：描述 IOAPIC 地址（0xFEC00000）、GSI base（0）、entries 数量（24）。由 QEMU 生成，通过 fw_cfg 传递给 OVMF（`AcpiPlatformDxe/EntryPoint.c:44`）
- ACPI _PRT 方法：描述 PCI slot/function 到 GSI 的映射。由 QEMU 的 ACPI DSDT 生成
- OS 内核读取 MADT + _PRT，编程 IOAPIC redirection table

**当前 nested OVMF 的情况：**
- AxVisor 不生成 ACPI 表给 nested guest
- AxVisor 的 fw_cfg 不提供 MADT
- 没有 _PRT 信息
- 没有 MP table（`load_uefi_ovmf_images()` 不加载，`mod.rs:331-371`）

**因此 nested OVMF 没有任何 IRQ 路由信息来源。** 即使它想编程 IOAPIC，也不知道该用哪个 GSI。

### Q4: 为什么日志里有 IOAPIC 初始化、irq=true、PCI 设备发现，但没有 vIOAPIC MMIO 访问？

**证据最闭合的解释：**

1. **`x86 IO APIC initialized`**：这是 AxVisor host 侧初始化 emulated IOAPIC 设备（`device.rs` init 路径，`vioapic.rs` `EmulatedIoApic::new_default()`），注册 GPA 0xFEC00000 的 MMIO handler。这是 host 侧行为，与 guest 无关。

2. **`[OVMF-VIRTIO-BLK-IO] ... irq=true`**：这是 `axvm/src/vm.rs:1230` 的 adapter 日志。`LegacyNotifyResult.should_raise_irq` 为真表示设备核心发布了 used buffer 并设置了 ISR。但 AxVisor 只打了日志，没有实际注入中断（行 1231：`INTx injection is not wired yet`）。`irq=true` 是设备侧信号，不表示 guest 期望中断。

3. **`PciRoot(0x0)/Pci(0x1,0x0)` 发现**：OVMF 通过 ECAM MMIO（0xE0000000 起，passthrough 到 host）枚举 outer QEMU PCI 总线，发现 `1AF4:1001`（virtio-blk）。这条路径完全不涉及 AxVisor 的 IOAPIC。

4. **没有 vIOAPIC MMIO 访问**：因为 OVMF 的 virtio-blk 驱动是纯轮询（`VirtioFlush` busy-wait），不需要中断。OVMF 不编程 IOAPIC redirection table（那是 OS 的职责）。因此 guest 永远不会发出对 0xFEC00000 的 MMIO 访问。

**四条日志条目完全自洽，没有矛盾。** IOAPIC 初始化是 host 侧，irq=true 是设备侧信号，PCI 发现是 config space passthrough，没有 vIOAPIC 访问是因为 guest 不需要中断。

### Q5: 现在应该继续 virtio-blk INTx 接线吗？

**结论 A：现在不该接，因为 guest 根本没访问 vIOAPIC，而且当前启动路径不需要中断。**

证据链：

1. OVMF virtio-blk 驱动是纯轮询（`VirtioLib.c:329-337` busy-wait），设置 `VRING_AVAIL_F_NO_INTERRUPT`（`VirtioLib.c:176`）。即使 AxVisor 注入中断，guest 也会忽略。
2. OVMF 不编程 IOAPIC（`BdsPlatform.c:1332-1396` 只写 PIRQ routing 和 InterruptLine）。
3. Emulated IOAPIC 的 redirection table 全 masked（`vioapic.rs:38`），guest 从未 unmask 任何 entry。
4. Nested OVMF 没有 IRQ 路由信息来源（无 MP table、无 ACPI MADT、无 _PRT）。
5. Guest 看到的是 outer QEMU 的 PCI 设备（ECAM passthrough），不是 AxVisor 自己的 PCI 模型。

**接 INTx 需要先解决的前置问题：**
- 决定谁给 nested guest 提供 IRQ 路由信息（AxVisor 的 fw_cfg 或 MP table 或 ACPI 表生成）
- 确认 OVMF 在有 IRQ 路由信息的情况下是否会编程 IOAPIC（OVMF 本身不编程，但如果有 ACPI 表，OVMF 的 ACPI platform driver 可能会安装它们，而 OS loader 或 UEFI driver 可能会使用）
- 如果目标是让 OVMF 在 DXE 阶段就用中断做 I/O，需要修改 OVMF 驱动（超出范围）

**当前的 virtio-blk 轮询路径已经能完成 OVMF boot。** INTx 只有在后续阶段（比如 ArceOS UEFI payload 需要中断驱动 I/O，或 OVMF 后期需要中断服务）才会有价值。

### Q6: 如果下一步继续调查，最值得加日志/继续读的 3 个点是什么？

**1. 确认 OVMF ACPI platform driver 是否会在 nested 环境下安装 ACPI 表**

- 文件：`edk2/OvmfPkg/AcpiPlatformDxe/EntryPoint.c`，函数 `InstallQemuFwCfgTables()`
- 为什么：这个函数从 fw_cfg 读取 QEMU 预生成的 ACPI 表。如果 AxVisor 的 fw_cfg 不提供 ACPI 表，OVMF 就没有 MADT，也就没有 IOAPIC 描述。确认这一点可以闭合"为什么 OVMF 不知道 IOAPIC 存在"的证据链。
- 具体动作：在 AxVisor fw_cfg 的 file directory 日志中，记录 OVMF 请求了哪些 fw_cfg 文件名，特别关注是否请求了 `etc/acpi/tables` 或类似的 ACPI 相关 item。

**2. 确认 OVMF 的 `SetPciIntLine` 对 nested 环境的实际执行结果**

- 文件：`edk2/OvmfPkg/Library/PlatformBootManagerLib/BdsPlatform.c`，函数 `SetPciIntLine()`
- 为什么：这个函数遍历所有 PCI 设备并写 InterruptLine。如果 nested OVMF 枚举到了 outer QEMU 的 virtio-blk-pci（slot 1），它会计算并写 InterruptLine 值。这个值虽然不直接影响 IOAPIC，但能揭示 OVMF 认为这个设备应该用哪个 IRQ line（10 或 11），以及 OVMF 是否认为自己在 q35 上。
- 具体动作：在 AxVisor 的 PCI config space passthrough 日志中，记录 OVMF 对 virtio-blk 设备（00:01.0）的 config space 0x3C（InterruptLine）和 0x3D（InterruptPin）的读写。

**3. 确认 emulated IOAPIC 是否被 guest 完全忽略（而不是访问了但被 stage-2 映射意外拦截）**

- 文件：`tgoskits/virtualization/x86_vlapic/src/vioapic.rs`，`EmulatedIoApic::handle_read()` / `handle_write()`
- 为什么：需要排除一种可能性——guest 的 IOAPIC 访问被 stage-2 页表的其他映射意外覆盖（比如 PCI Low MMIO 的 0x80000000-0xE0000000 范围是否可能包含 0xFEC00000？答案是不包含，但需要确认 passthrough_devices 合并逻辑没有 bug）。
- 具体动作：确认 `vm.rs` 中 `init()` 的 passthrough region 合并逻辑（行 300-357）没有把 0xFEC00000 包含在任何映射区域内。这可以通过在 stage-2 页表构建后打印最终映射范围来验证。

---

## 未闭合点

- **AxVisor 的 fw_cfg 是否提供 ACPI 表给 nested OVMF？** 当前只知道 fw_cfg 提供 e820 和 CPU 拓扑，没有看到 ACPI tables 相关的 fw_cfg item。如果 AxVisor 不提供，nested OVMF 连 IOAPIC 的存在都不知道（没有 MADT）。需要读 fw_cfg file directory 的完整列表或加日志确认。

- **OVMF 在 Q35 上使用 ECAM（0xE0000000），但 AxVisor 的 passthrough 映射是否正确覆盖了完整的 ECAM 窗口？** `PCI MMIO` passthrough 从 0xE000_0000 起 0x1000_0000（256MB），这应该覆盖 bus 0-255 的完整 ECAM 空间。但如果 OVMF 编程了 MCH PCIEXBAR（`Platform.c:278-316`）指向不同的地址，可能不匹配。需要确认 AxVisor 的 stage-2 映射是否与 OVMF 编程的 PCIEXBAR 一致。

- **`mptable.rs` 中 GSI 18 的来源仍然不清楚。** 注释说 "QEMU routes that device's INTA# to host IOAPIC GSI 18"，但标准 q35 公式算出 slot 3 INTA → GSI 23。这个矛盾没有闭合。如果未来需要 passthrough 中断，需要确认 outer QEMU 的实际 IOAPIC routing。但在当前纯轮询路径下，这个矛盾不影响功能。

---

## 下一步最小调查动作

1. **查 AxVisor fw_cfg file directory 完整列表**：在 `axdevice/src/fw_cfg.rs` 的 file directory 构建代码中加日志，打印所有注册的 fw_cfg 文件名（特别是 `etc/acpi/tables`、`etc/table-loader`、`etc/acpi/rsdp` 等 ACPI 相关 item）。这决定了 nested OVMF 能否看到 ACPI 表。文件：`tgoskits/virtualization/axdevice/src/fw_cfg.rs`，搜索 `FwCfgFileDirectory` 构建逻辑。

2. **查 OVMF 的 ECAM 基址是否与 AxVisor passthrough 匹配**：确认 OVMF 编程的 MCH PCIEXBAR 寄存值（`edk2/OvmfPkg/Library/PlatformInitLib/Platform.c:278-316`）与 AxVisor 的 `PCI MMIO` passthrough 起始地址（0xE0000000）一致。如果不一致，OVMF 的 ECAM 访问会穿透到错误的 host 地址。

3. **确认 stage-2 映射的完整性**：在 `axvm/src/vm.rs` 的 `init()` 函数中，passthrough region 合并后（行 300-357），打印最终的映射范围列表。验证 0xFEC00000 确实不在映射内（应该被 EmulatedIoApic MMIO 拦截），且 0xE0000000-0xF0000000 确实在映射内（ECAM passthrough）。

---

## 文档位置

`axvisor-2026-notebooks/subagent_watch/ovmf-vioapic-root-cause.md`
