## 本阶段行为准则
- 每推进一个有意义的小阶段，就更新本文档。
- 每条记录必须说明：看到什么日志、判断出现什么问题、根据`docs/OVMF-Boot-Overview.md`判断处于哪个阶段(如果进入新的阶段需要写)、做了什么改动（描述简洁，改动位置定位具体）。
- 不为了“继续推进”而盲目扩展范围；如果下一步需要大改，就明确记录边界。

## 基线
- 已确认可用的 DEBUG OVMF 产物已经在 `edk2/Build/OvmfX64/DEBUG_GCC/FV/` 下生成，包含 `OVMF_CODE.fd`、`OVMF_VARS.fd` 和 `OVMF.fd`。
- 这套 OVMF 已经被 AxVisor smoke test 实际加载并运行到早期固件阶段，不再停留在镜像缺失或构建失败问题上。
- 当前现场仍是同一个早期异常路径：`guest_rip=0x82997a`，`IDT-vectoring` 显示 `#BR bound range exceeded`，随后触发 `#GP general protection`，错误码指向 `GDT selector 0x10`。
- 目前只保留小步推进边界：AxVisor 和 OVMF 只做日志增强或单点适配；一旦需要大规模逻辑改动、复杂平台补齐或 `sudo` 权限，就停止并记录。
- 现在 guest rootfs 里的 `/guest/ovmf/OVMF_CODE.fd` 已经被新编译的 DEBUG 版本覆盖，`OVMF_VARS.fd` 也与当前 build 一致。
- 从现在开始，后续阶段记录都写到这个文件。

## Step 1：把新 DEBUG OVMF 放进 guest rootfs 并重新跑 smoke test

### 做了什么
- 用当前新编出来的 `edk2/Build/OvmfX64/DEBUG_GCC/FV/OVMF_CODE.fd` 覆盖了 guest rootfs 里的 `/guest/ovmf/OVMF_CODE.fd`。
- 核对了 rootfs 内的 `/guest/ovmf/OVMF_CODE.fd` 与本次 DEBUG build 的 SHA256 一致，`OVMF_VARS.fd` 也已和当前 build 一致。
- 重新运行了 AxVisor OVMF smoke test。

### 看到什么日志
- 先看到 AxVisor 正常加载：
  - `Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000`
  - `Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000`
  - `VMX guest entry setup: entry=0xfffffff0, reset_vector=true, cs=0xf000, cs_base=0xffff0000, rip=0xfff0`
- 这次 guest 侧不再停留在之前那个 VMX exit 视角的 `0x82997a` 现场，而是直接打印出 OVMF 自己的异常摘要：
  - `!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!`
  - `RIP - 000000000084F404`
  - `CS - 0000000000000038`
  - `CR0 - 0000000080000023`
  - `CR3 - 0000000000800000`
  - `CR4 - 0000000000002668`
  - `Find image based on IP(0x84F404) ... PlatformPei.dll (ImageBase=0000000000849340, EntryPoint=00000000008523EE)`

### 判断出现什么问题
- 当前问题已经从“reset vector / 早期 VMX 异常投递”推进到了 `docs/OVMF-Boot-Overview.md` 里的 **PEI** 阶段，具体落在 `OvmfPkg/PlatformPei/PlatformPei.dll`。
- 这说明新 DEBUG OVMF 已经真正被 guest 使用，当前卡点不再是镜像未更新。
- 现在的 failure site 是 `InternalQemuFwCfgDmaBytes()`，对应源码 `OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c:245-246`。
- 这是一个更靠后的、也更具体的阶段问题：OVMF 在 PEI 里走到了 QEMU fw_cfg DMA 相关路径，然后触发了 `#BR`。

### 根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段
- 已经进入 **PEI**。
- 更具体地说，当前是在 PEI 里处理 `PlatformPei` / `QemuFwCfgPei` 相关逻辑，而不是 SEC 或 reset-vector 阶段。

### 做了什么改动
- 只做了最小的 runtime 适配：把 guest rootfs 里的 OVMF_CODE 切换成当前 DEBUG build。
- 没有改 OVMF 复杂逻辑，也没有引入大规模平台能力。

### 下一步最小动作是什么
- 先沿着 `PlatformPei.dll` 里的 `InternalQemuFwCfgDmaBytes()` 继续加最小日志或做更细的调用点定位。
- 目前不扩展到 ACPI / PCI / virtio 大改；如果后续发现需要大规模平台补齐，再停下来记录边界。

## Step 2：确认 `#BR` 已进入 PEI 的 `QemuFwCfgPei` 路径

### 做了什么
- 运行 smoke test 时保留了新 DEBUG OVMF。
- 继续观察 OVMF 自己打印的异常摘要和 image lookup 结果。

### 看到什么日志
- OVMF 直接打印出：
  - `!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!`
  - `RIP - 000000000084F404`
  - `Find image based on IP(0x84F404) ... PlatformPei.dll`
- 结合反汇编，`0x84F404` 落在 `OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` 的 `InternalQemuFwCfgDmaBytes()`，靠近：
  - `AccessHigh = ...`
  - `AccessLow = ...`
  - `IoWrite32 (FW_CFG_IO_DMA_ADDRESS, ...)`

### 判断出现什么问题
- 当前异常点不是 reset-vector 相关，而是 PEI 阶段的 fw_cfg DMA 路径。
- 这说明 OVMF 已经从 SEC 进入 PEI，并在处理 QEMU fw_cfg 时触发 `#BR`。
- 下一步最小可观测点不是大改平台，而是给 `QemuFwCfgPei` 加一条更细的日志，打印：
  - 当前 `FwCfgItem`
  - 当前 `DataSize`
  - `QEMU_FW_CFG_WORK_AREA` 里的 ongoing item / offset
  - DMA 访问是 read 还是 write

### 根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段
- 已经进入 **PEI**。
- 更具体地说，卡在 PEI 里的 `QemuFwCfgPei` / fw_cfg DMA 路径。

### 做了什么改动
- 目前还没有改 OVMF 逻辑，只是把阶段记录推进到 PEI 内部的 `QemuFwCfgPei`。

### 下一步最小动作是什么
- 给 `OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` 加最小日志，优先打印 `InternalQemuFwCfgDmaBytes()` 的输入参数和 work area 状态。
- 继续保持“只做日志增强或单点适配”的边界，不展开平台大改。

## Step 4：重新跑 smoke test，确认 fw_cfg 日志位置和新的故障点

### 做了什么
- 重新运行了 AxVisor smoke test，使用的是已经更新进 guest rootfs 的新 DEBUG OVMF。
- 继续观察 `PlatformPei.dll` 里的 `InternalQemuFwCfgDmaBytes()` 相关路径。

### 看到什么日志
- AxVisor 仍然正常加载：
  - `Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000`
  - `Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000`
  - `VMX guest entry setup: entry=0xfffffff0, reset_vector=true, cs=0xf000, cs_base=0xffff0000, rip=0xfff0`
- guest 侧这次仍然报告 `#BR`，但故障点向前/向后移动到了：
  - `RIP - 000000000084F5EB`
  - `Find image based on IP(0x84F5EB) ... PlatformPei.dll`
- 反汇编对照后，`0x84F5EB` 仍在 `OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c` 的 `InternalQemuFwCfgDmaBytes()`，更接近：
  - `Status = SwapBytes32 (Access.Control)`
  - `ASSERT ((Status & FW_CFG_DMA_CTL_ERROR) == 0)`
  - `while (Status != 0)`

### 判断出现什么问题
- 这次比上一步更明确：`#BR` 已经不是在 `PlatformPei` 泛泛的位置，而是落在 `InternalQemuFwCfgDmaBytes()` 的 DMA 完成轮询附近。
- 我加的那条 `AXOVMF` 日志还没有在当前 console 上看到，说明下一步应该优先确认日志级别/输出路径，而不是继续改逻辑。
- 当前更像是 fw_cfg DMA 访问本身触发了异常，或轮询状态字段在 guest 当前状态下出现了异常访问。

### 根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段
- 仍然是 **PEI**。
- 但现在已经具体到 PEI 里的 `QemuFwCfgPei` DMA 等待路径。

### 做了什么改动
- 新增的 OVMF 改动仍然只有一条最小 DEBUG 日志。
- AxVisor 侧没有引入额外复杂适配。

### 下一步最小动作是什么
- 把这条日志换到更容易在当前控制台看到的级别，或者换一个更靠近调用点的日志位置。
- 先只做日志可见性调整，不碰 fw_cfg DMA 行为本身。

## Step 5：确认 OVMF DEBUG 输出没有接到 AxVisor 当前可见路径

### 做了什么
- 检查了 OVMF 的 `PlatformDebugLibIoPort` 实现和 OVMF 默认调试端口配置。
- 继续检查 AxVisor 的 x86 VM-exit / I/O 处理路径，确认 debug 输出能否从 guest 侧被宿主看到。

### 看到什么日志
- OVMF 默认 `DEBUG()` 输出会走 `PcdDebugIoPort`，而 `OvmfPkg/OvmfPkgX64.dsc` 默认配置的是 `0x402`。
- AxVisor 当前只在 `x86_vcpu/src/vmx/vcpu.rs` 里拦截了 `QEMU_EXIT_PORT = 0x604`，没有把 `0x402` 当成 debugcon 处理。
- AxVisor 的主 vCPU 退出循环里也没有专门的 `IoRead` / `IoWrite` 分支去打印 `0x402` 的字节内容。

### 判断出现什么问题
- 这解释了为什么我在 OVMF 里加的 `DEBUG_INFO` 没有直接出现在当前控制台：guest 的 debugcon 端口没有被 AxVisor 暴露成宿主侧日志。
- 当前不是 OVMF 逻辑错误，而是 **日志路径缺失**。
- 这属于一个很小、边界清晰的 AxVisor 适配点，不是大平台重构。

### 根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段
- 仍然是 **PEI**。
- 目前阶段问题还是 PEI 中的 `QemuFwCfgPei`，只是需要先把 OVMF debugcon 输出接出来。

### 做了什么改动
- 目前还没有改代码，只确认了修改位置。
- 候选的最小改动点是：
  - `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`：把 `0x402` 加进 I/O bitmap 拦截范围。
  - `tgoskits/os/axvisor/src/vmm/vcpus.rs`：在 `AxVCpuExitReason::IoWrite` 里专门打印 `0x402` 的单字节写出。

### 下一步最小动作是什么
- 先做这两个小改动，把 OVMF `DEBUG()` 输出接到宿主控制台。
- 不碰 fw_cfg DMA 行为本身，也不扩展到 ACPI / PCI / virtio。

## Step 6：接上 OVMF debugcon 端口后推进到 VMX 字符串 I/O

### 做了什么
- 在 AxVisor 设备层补了最小 `0x402` debugcon 处理。
- 重新运行 AxVisor OVMF smoke test。

### 看到什么日志
- 之前的 `emu_device read failed: device not found for port Port(0x402)` panic 消失。
- AxVisor 仍然正常加载 OVMF：
  - `Loading OVMF_CODE image from /guest/ovmf/OVMF_CODE.fd into GPA 0xffc84000`
  - `Loading OVMF_VARS image from /guest/ovmf/OVMF_VARS.fd into GPA 0xffc00000`
  - `VMX guest entry setup: entry=0xfffffff0, reset_vector=true, cs=0xf000, cs_base=0xffff0000, rip=0xfff0`
- 新的停点变成 VMX I/O exit：
  - `VMX unsupported IO-Exit: VmxIoExitInfo { access_size: 0x1, is_in: false, is_string: true, is_repeat: true, port: 0x402 }`
  - `guest_rip: 0xfffd654f`
  - `rcx: 0x2f`
  - `rdx: 0x402`
  - `rsi: 0x81eba0`

### 判断出现什么问题
- `0x402` 的 debugcon probe 已经不再导致设备层 panic，说明 AxVisor 设备层已能识别该端口。
- 新问题是 OVMF `DebugLib.c` 里的 `IoWriteFifo8()` 会生成 `rep outsb`，而 `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs` 当前把 `is_string || is_repeat` 的 I/O exit 直接判为 unsupported。
- 这不是 fw_cfg DMA 行为本身的新失败，而是 debugcon 输出链路继续向前暴露出的最小 I/O 指令支持缺口。

### 根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段
- 仍然是 **PEI** 前后早期 OVMF 执行阶段。
- 当前不是新的 OVMF 阶段，而是为了观察 PEI 内部日志，需要先处理 debugcon 输出路径的字符串 I/O。

### 做了什么改动
- `tgoskits/components/axdevice/src/device.rs`：新增最小 `OvmfDebugConDevice`，覆盖 port `0x402`。
- `tgoskits/components/axdevice/src/device.rs`：`IoRead8(0x402)` 返回 Bochs debug port magic `0xE9`，让 OVMF `DebugIoPortQemu.c` probe 通过。
- `tgoskits/components/axdevice/src/device.rs`：`IoWrite8(0x402)` 按行聚合并通过 AxVisor `info!` 输出。
- 保留已有 `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs` 的 `0x402` I/O bitmap 拦截。

### 下一步最小动作是什么
- 不在 AxVisor 里大规模实现完整字符串 I/O。
- 最小可回滚方案是只改 OVMF debug 输出路径：把 `OvmfPkg/Library/PlatformDebugLibIoPort/DebugLib.c` 中 debugcon 输出从 `IoWriteFifo8()` 改成逐字节 `IoWrite8()`，避免触发 VMX 未支持的 `rep outsb`，继续暴露 `QemuFwCfgPei` 日志。

## Step 7：OVMF debugcon 日志可见后确认 `QemuFwCfg` DMA 首次读触发 `#BR`
- OVMF 通过向 0x402 做端口 I/O 输出日志，AxVisor 用 VMX 拦截这个端口，把字节读出来，再用 debug!/info! 打到宿主终端。
### 做了什么
- 按 Step 6 的最小方案修改 OVMF debug 输出路径，避免 `IoWriteFifo8()` 生成 `rep outsb`。
- 重新构建 DEBUG OVMF，并把新的 `OVMF_CODE.fd` 写回 guest rootfs 的 `/guest/ovmf/OVMF_CODE.fd`。
- 核对 rootfs 内 `OVMF_CODE.fd` 与新 build 的 SHA256 一致。
- 重新运行 AxVisor OVMF smoke test。

### 看到什么日志
- OVMF debugcon 日志已经能通过 AxVisor 设备层输出，例如：
  - `OVMF debugcon: SecCoreStartupWithStack(0xFFFCC000, 0x820000)`
  - `OVMF debugcon: Platform PEIM Loaded`
  - `OVMF debugcon: QemuFwCfgProbe: Supported 1, DMA 1`
- `PlatformPei.efi` 被加载并进入执行：
  - `Loading PEIM at 0x00000848D40 EntryPoint=0x00000851DF2 PlatformPei.efi`
- 新增的 OVMF 日志已经出现，说明 `QemuFwCfgPei.c` 的诊断点可见：
  - `AXOVMF InternalQemuFwCfgDmaBytes: size=0x4 buffer=81DF28 control=0x2 workarea=815931 item=0x0 offset=0x0 reading=0`
- 紧接着 OVMF 抛出异常：
  - `!!!! X64 Exception Type - 05(#BR - BOUND Range Exceeded) !!!!`
  - `RIP - 0000000000848FA8`
  - `RAX - 0000000000000002`
  - `RCX - 0000000002000000`
  - `RDX - 0000000000000518`
  - `RDI - 0000000000000004`
  - `R15 - 000000000081DF28`
  - `Find image based on IP(0x848FA8) ... PlatformPei.dll (ImageBase=0000000000848D40, EntryPoint=0000000000851DF2)`

### 判断出现什么问题
- debugcon 链路现在已经打通，之前的 `0x402` 设备缺失和 VMX 字符串 I/O 两个日志路径问题都已越过。
- 当前真实 failure 是 PEI 阶段 `PlatformPei` 中 `QemuFwCfg` DMA 读路径触发 `#BR`。
- 第一次可见的 DMA 请求参数是：
  - `Size = 0x4`
  - `Buffer = 0x81DF28`
  - `Control = 0x2`
  - `WorkArea = 0x815931`
  - `WorkArea->FwCfgItem = 0x0`
  - `WorkArea->Offset = 0x0`
  - `WorkArea->Reading = 0`
- 结合前置日志 `QemuFwCfgProbe: Supported 1, DMA 1` 和 `Select Item: 0x19`，当前问题更具体地落在 OVMF 认为 fw_cfg DMA 可用后，对 DMA control/address port 的第一次读请求处理。

### 根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段
- 已进入 **PEI**。
- 更具体地说，是 PEI 驱动 `PlatformPei.efi` 中的 QEMU fw_cfg 初始化/读取路径。
- 尚未进入 DXE；当前还不是 ACPI / PCI / virtio 大平台阶段。

### 做了什么改动
- `tgoskits/components/axdevice/src/device.rs`：保留最小 `OvmfDebugConDevice`，让 `0x402` debugcon read 返回 `0xE9`，write 按行输出。
- `edk2/OvmfPkg/Library/PlatformDebugLibIoPort/DebugLib.c`：把两个 debugcon 输出点从 `IoWriteFifo8()` 改为逐字节 `IoWrite8()`，避免 AxVisor VMX 当前不支持的 `rep outsb`。
- `edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c`：保留最小 `AXOVMF InternalQemuFwCfgDmaBytes` 日志，打印 DMA 请求参数和 work area 状态。
- `tgoskits/components/x86_vcpu/src/vmx/vcpu.rs`：保留已有 `0x402` I/O bitmap 拦截。

### 下一步最小动作是什么
- 继续保持小步边界，不实现完整 fw_cfg 设备模型。
- 下一步应先确认 `RIP=0x848FA8` 在 `PlatformPei.dll` / `InternalQemuFwCfgDmaBytes()` 中对应的具体指令，并对照 `FW_CFG_IO_DMA_ADDRESS` 的 port I/O 访问。
- 如果确认 AxVisor 没有 fw_cfg DMA port 的最小处理，再判断是否能做单点适配；如果需要补完整 fw_cfg 设备模型，则停止并记录为超出当前边界。

## Step 8：更新启动手册，默认使用自编译 DEBUG OVMF

### 做了什么
- 更新 `axvisor-2026-notebooks/start.md`，把默认 OVMF 来源从系统 `/usr/share/OVMF` 改为自编译 DEBUG OVMF。
- 保留系统 OVMF 作为备用来源，但明确当前调试不推荐默认使用。

### 看到什么日志
- 本步骤没有重新运行 smoke test，只更新运行手册。
- 修改依据来自 Step 7 已确认的运行结果：
  - `OVMF debugcon: SecCoreStartupWithStack(...)`
  - `OVMF debugcon: Platform PEIM Loaded`
  - `OVMF debugcon: QemuFwCfgProbe: Supported 1, DMA 1`
  - `OVMF debugcon: AXOVMF InternalQemuFwCfgDmaBytes: size=0x4 ...`

### 判断出现什么问题
- 原 `start.md` 仍把 `/usr/share/OVMF/OVMF_CODE_4M.fd` / `OVMF_VARS_4M.fd` 写成主要准备方式。
- 当前调试链路已经依赖自编译 DEBUG OVMF，否则看不到项目内新增的 `AXOVMF` 诊断日志，也无法稳定复现当前 `PlatformPei` / `QemuFwCfgPei` 定位流程。
- 另外 rootfs 更新目前可以用 `debugfs -w` 完成，不需要默认要求 sudo mount。

### 根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段
- 文档更新本身不改变 OVMF 阶段。
- 手册中的当前运行状态已同步为：已经能进入 **PEI**，并在 `PlatformPei.efi` / `QemuFwCfg` DMA 路径处触发 `#BR`。

### 做了什么改动
- `axvisor-2026-notebooks/start.md`：开头目标链路从 `fault 日志` 更新为 `debugcon 日志`。
- `axvisor-2026-notebooks/start.md`：新增自编译 DEBUG OVMF 作为默认来源。
- `axvisor-2026-notebooks/start.md`：新增 EDK2 build 命令，使用当前树可用的 `-t GCC`。
- `axvisor-2026-notebooks/start.md`：新增 `debugfs -w` 写入 rootfs 和 SHA256 校验流程。
- `axvisor-2026-notebooks/start.md`：保留系统 OVMF 作为备用来源，并提示必须检查 OVMF_CODE 文件大小和 `ovmf_code_base`。
- `axvisor-2026-notebooks/start.md`：更新预期关键日志，加入 `OVMF debugcon` 和 `AXOVMF InternalQemuFwCfgDmaBytes`。
- `axvisor-2026-notebooks/start.md`：更新常见错误，加入 `0x402` 字符串 I/O 和当前 PEI `#BR` 失败点。

### 下一步最小动作是什么
- 继续回到 Step 7 的技术路线：定位 `RIP=0x848FA8` 对应具体指令，并判断是否只是 fw_cfg DMA port 的单点适配，还是已经需要完整 fw_cfg 设备模型。

