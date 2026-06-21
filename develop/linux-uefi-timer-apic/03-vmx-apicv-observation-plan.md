# Linux UEFI timer/APIC 续查：APICv 合同已对上，但还差 virtual-APIC page 实证

## 本轮先做的事

按当前 handoff 指定顺序重新读了：

1. `CLAUDE.md`
2. `02-context-handoff.md`
3. `00-pit-reset-and-lint0-status.md`
4. `01-apic-lvt0-write-gap.md`
5. `subagent_watch/linux-lvt0-write-path-audit.md`
6. `subagent_watch/vmx-apic-write-path-contract.md`
7. `subagent_watch/virtual-wire-extint-contract-matrix.md`

然后又补读了两个新 agent 结果：

- `subagent_watch/intel-vmx-apic-access-contract.md`
- `subagent_watch/axvisor-vmx-apic-page-audit.md`

再把这些结论和当前树里的实现重新对了一遍：

- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`
- `tgoskits/virtualization/x86_vlapic/src/lib.rs`
- `tgoskits/virtualization/x86_vlapic/src/vlapic.rs`
- `tgoskits/virtualization/axvm/src/vm.rs`

## 这轮确认下来的事实

### 1. “Linux 写了 LVT0，但没有 APIC_ACCESS write 日志”这件事，和 Intel 合同是对得上的

当前 VMX 路径明确打开了这些位：

- `USE_TPR_SHADOW`
- `VIRTUALIZE_APIC`
- `VIRTUALIZE_APIC_REGISTER`
- `VIRTUAL_INTERRUPT_DELIVERY`

对应代码：

- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:750`
- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:764-766`

Intel SDM 30.4.3.1 的关键点已经被新 agent 核死：

- guest 对 xAPIC MMIO `0xfee00350` (`APIC_LVT0`) 的 32-bit aligned 写，
  在
  - `use TPR shadow = 1`
  - `virtualize APIC accesses = 1`
  - `APIC-register virtualization = 1`
  时，
  **可以被 CPU 直接虚拟化到 virtual-APIC page，而不产生 `APIC_ACCESS` VM-exit。**

所以：

- 没有 `APIC_ACCESS` write 日志
- 没有 `"[VLAPIC] LINT0 write"` 日志

这两件事本身，已经不能再拿来证明 “Linux 没写 LVT0”。

### 2. AxVisor 当前 VMCS wiring 和 EPT wiring 基本符合这份合同

当前树里：

- `VIRT_APIC_ADDR` 指向每 vCPU 的 virtual-APIC page
- `APIC_ACCESS_ADDR` 指向独立的 APIC-access page
- EPT 把 guest GPA `0xfee0_0000` 映到 APIC-access page 的 HPA

对应代码：

- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:854-856`
- `tgoskits/virtualization/axvm/src/vm.rs:480-485`
- `tgoskits/virtualization/x86_vlapic/src/lib.rs:44-46`
- `tgoskits/virtualization/x86_vlapic/src/vlapic.rs:108-147`

这意味着：

- “APIC-access 机制压根没接上” 不是当前第一怀疑对象。
- 新日志里 panic 后那一次 `APIC_ACCESS` read，也说明这条硬件机制至少不是完全死的。

### 3. 这里有一个必须单独写下来的纠偏点

目前最容易滑进去的说法是：

> `APIC-register virtualization` 把 LVT0 写吃掉了，AxVisor 的 `lvt_last` shadow 没更新，所以 PIT handler 看到的还是 reset 值。

这句话只说对了一半。

因为当前 `PIT IRQ0 -> LINT0 fallback` 读路由时，走的不是 `lvt_last`：

- `axvm::inject_due_x86_pic_lint0_irq()`
- `vcpu.get_arch_vcpu().lint0_route()`
- `VmxVcpu::lint0_route()`
- `EmulatedLocalApic::lint0_route()`
- `VirtualApicRegs::lint0_route()`

而 `VirtualApicRegs::lint0_route()` 直接读的是：

- `self.regs().LVT_LINT0.get()`

也就是 **virtual-APIC page 上 offset `0x350` 的当前值**，
不是 `lvt_last.lvt_lint0`。

对应代码：

- `tgoskits/virtualization/axvm/src/vm.rs:313-345`
- `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs:245-247`
- `tgoskits/virtualization/x86_vlapic/src/lib.rs:148-150`
- `tgoskits/virtualization/x86_vlapic/src/vlapic.rs:195-197`

所以，当前仍然缺一条关键实证：

> CPU 直接虚拟化掉的那几次 LVT0 写，到底有没有真实落到当前 `VIRT_APIC_ADDR` 指向的 virtual-APIC page 上？

在这个问题被直接观测之前，不能只靠 `lvt_last` 过时，就解释
`x86 PIT IRQ0 due but neither guest GSI2 nor LAPIC LINT0 had an injectable route`。

## 现在最该回答的精确问题

不是：

- 要不要再改 PIT
- 要不要先重做 ExtINT 语义
- 要不要直接回退 APICv 整套控制位

而是：

> 在当前 VMCS 组合下，Linux 的 LVT0 写是否已经被 CPU 写进了 virtual-APIC page；
> 如果写进去了，AxVisor 为什么在 `inject_due_x86_pic_lint0_irq()` 那个时刻仍然读不到可用 route？

## 下一步的窄计划

### A. 先补“只观测不改语义”的诊断

优先补三类低侵入日志：

1. VMCS 最终控制位和两个 APIC page 地址
   - 记录 secondary controls 最终值
   - 记录 `VIRT_APIC_ADDR`
   - 记录 `APIC_ACCESS_ADDR`

2. APIC mode 切换证据
   - `IA32_APIC_BASE` read/write 升到 `info!`
   - 若 guest 真走了 x2APIC，补 `MSR 0x835` 等 LVT 相关写日志

3. virtual-APIC page 上 LVT0 的真实值
   - 在安全的 VM-exit 观察点上直接读 `regs().LVT_LINT0`
   - 同时把 `lvt_last.lvt_lint0` 也一起打出来
   - 目标不是做新框架，只是确认：
     - virtual page 是否变化
     - shadow 是否没跟上
     - “无 route” 发生时两者分别是什么

### B. 只有在观测完成后，才决定修复路径

目前不要直接选“禁 APIC-register virtualization”或“直接改注入逻辑”。

先看观测结果再分叉：

1. **如果 virtual-APIC page 上的 `LVT0` 确实变成过 `0x30` / `0x700`**
   - 说明 Linux 写确实到了 `VIRT_APIC_ADDR`
   - 接下来就要查：
     - route 窗口是否太短
     - 我们的观测时点是否晚了
     - 是否还存在 guest 自己又很快 mask 回去
     - 哪些依赖副作用仍然绑在 `write_lvt()` / `lvt_last`

2. **如果 virtual-APIC page 上的 `LVT0` 一直停在 reset/masked**
   - 才能继续往“guest 实际没走这条写”或“当前 VMCS / EPT / APIC page wiring 仍有缝”收紧

### C. 额外需要盯的一点

`EmulatedLocalApic::handle_xapic_write_offset()` 现在是死代码：

- `tgoskits/virtualization/x86_vlapic/src/lib.rs:94-104`

它看起来正是为“硬件已完成 APIC write，再补一次软件副作用”预留的窄入口，
但当前 `VmxExitReason::APIC_WRITE` 分支并没有调用它。

这件事要记住，但现在先不要据此直接改行为，因为：

- Intel 合同对 `LVT0` 的更强结论是：在当前控制位组合下，它很可能连 `APIC_WRITE` exit 都不会有，而是 **完全无 exit**。

所以它更像下一步可能的修补点，不是当前已经证实的主因。

## 本轮结论

当前最稳的结论是：

1. `APIC-register virtualization` 足以解释“为什么没有 LINT0 write / APIC_ACCESS write 日志”。
2. 但它**还不足以单独解释** “为什么 PIT fallback 始终读不到 LINT0 route”，因为当前 route 查询读的是 virtual-APIC page，不是 `lvt_last`。
3. 所以下一轮不该直接拍板修复方案，而应该先把 **virtual-APIC page 的 `LVT0@0x350` 真实变化** 观测出来。

## 本轮后续新增实证

后面又补了一轮最小诊断和 smoke，现场已经比写这份计划时更前进了一步。

### 1. APICv wiring 已被运行时日志直接确认

新日志文件：

- `tmp/ovmf-linux-smoke-apicv-observe.log`
- `tmp/ovmf-linux-smoke-apicv-fixed.log`

关键日志：

- `[VMX] exec controls: use_tpr_shadow=true secondary_controls=true ept=true`
- `[VMX] APICv controls: virt_apic_access=true apic_reg_virt=true virt_intr_delivery=true`
- `[VMX] APIC pages: guest_gpa=0xfee00000 virt_apic_hpa=... apic_access_hpa=...`

这说明这轮追的不是“控制位可能没开”，而是控制位开着以后，guest APIC 状态到底怎样被 CPU 和 AxVisor 共同观察。

### 2. 已直接看到 virtual-APIC page 和 shadow 分叉

最关键的新日志不是 `APIC_ACCESS`，而是 `inject_due_x86_pit_irq0()` 里的 LINT0 观测值：

- 早期多次：
  - `virtual_page=0x0 route=None, shadow=0x10000 route=None`
- 后续变成：
  - `virtual_page=0x700 ... , shadow=0x10000 ...`

这个变化本身已经足够说明：

- Linux 对 `LVT0` 的写确实已经落到 **virtual-APIC page**
- AxVisor 的 `lvt_last` shadow 没跟上，仍停在 reset masked 值

所以这里不再是“guest 可能根本没写到 LVT0”。

### 3. 还顺手抓到了一个本地 decode bug

在把 raw `LVT0` 值打出来以后，又发现一个额外问题：

- raw `0x700` 理论上应该是 `ExtINT`
- 但我们当时打印出来的 route 还是 `None`

最后定位到：

- `tgoskits/virtualization/x86_vlapic/src/vlapic.rs`
- `lint0_route_from_lvt()`

这里把 `tock-registers` 的 delivery-mode `.mask()` 当成了 raw value 去比较，导致：

- `0x700` 没被正确识别成 `ExtINT`

这个 bug 已修，相关小单测已跑通：

- `cargo test -p x86_vlapic lint0_route_reports_extint_mode -- --nocapture`
- `cargo test -p x86_vlapic lint0_observation_surfaces_virtual_page_shadow_split -- --nocapture`

### 4. 修完 decode 以后，现场又前进一步

在 `tmp/ovmf-linux-smoke-apicv-fixed.log` 里，已经首次看到：

- `x86 PIT IRQ0 due and injected via LAPIC LINT0 extint route: guest vector 0x0, PIC vector 0x0`

这说明当前阻塞已经不再是：

- `LVT0` 没写到
- `ExtINT` route 没识别出来

而是下一层更窄的问题：

- **PIC 在这次 ExtINT 注入点返回了 vector `0x0`**
- AxVisor 随后在 `inject_interrupt_with_trigger(vector=0)` 上本地 panic

所以当前最窄主问题已经更新成：

> APICv 下的 `LVT0 ExtINT` 路由已经开始建立，但 PIC vector/ack 现场不对，当前第一次真正的 LINT0 fallback 注入拿到了 `0x0`。

## 当前下一步

继续只盯这件事：

- `x86_vlapic/src/pic.rs::read_irq_vector()`
- Linux timer fallback 里 `legacy_pic->init()/unmask()/make_irq()` 对应的最小 PIC 合同
- 为什么在当前这次 `ExtINT` 注入点，master/slave PIC 的 `irq_base + irq` 最终给出的是 `0x0`

不要回头做这些：

- 不要再回头改 PIT
- 不要回头怀疑 “LVT0 根本没写”
- 不要扩成 generic IRQ framework
- 不要展开 x2APIC / modern virtio / direction-2
