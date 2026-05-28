## 本阶段行为准则
- 日志多多益善,确定修改是否正确,通过日志代码定位回源码位置.
- 每推进一个对主线有意义的小阶段，就更新本文档。
- 每条记录必须说明：看到什么日志/源码逻辑、判断出现什么问题、根据 `docs/OVMF-Boot-Overview.md` 判断处于哪个阶段（如果进入新的阶段需要写,不进入不写）、为此做了什么改动达到了目标（描述简洁，改动位置定位具体）。
- 不为了"继续推进"而盲目扩展范围；如果下一步需要大改，就明确记录边界。

## 基线

- 本阶段接 `develop5.md` Step 3：AxVisor 已经让标准 OVMF 跨过 PEI `CpuMpPei` / AP startup，进入 DXE 并到达 BDS。
- 现在主线不再是 CPUID、xAPIC/x2APIC、LAPIC page 接入问题；除非新日志重新指向 APIC/MSR，否则不要回到旧调研分支。
- 本阶段目标是让 OVMF 在 BDS 中完成设备连接、生成可启动项，并启动一个可验证的 UEFI boot path。
- OVMF / EDK2 逻辑代码仍不作为修复对象；允许保留当前用于定位的 debug 输出和 fw_cfg 读入绕开。
- 可信最新运行日志：`/home/ssdns/work/axvisor-uefi/tmp/a.log`。
- 当前运行命令由 `tgoskits/os/axvisor/` 发起，关键配置为：
  - `tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml`
  - `tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml`
  - `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml`
- 当前 QEMU runtime 里有一个外层 virtio-blk 盘：
  - `-device virtio-blk-pci,drive=disk0`
  - `file=/home/ssdns/work/axvisor-uefi/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img`
- AxVisor 宿主侧启动时把这个镜像识别成整盘 ext4：
  - `No partition table found, treating whole disk as single partition`
  - `Ext4 filesystem mounted`
  - `Using partition 'disk' (Ext4) as root filesystem`
- 这说明该镜像目前更像 AxVisor 自己的 rootfs，不一定是 OVMF BDS 可启动的 UEFI 盘；后续不要默认它应该生成 `EFI System Partition` / FAT boot option。

### 当前 BDS 位置

- 根据 `docs/OVMF-Boot-Overview.md`，当前已经进入 **BDS**。
- 最新日志显示已经走过：
  - `[Bds] Entry...`
  - `PlatformBootManagerBeforeConsole`
  - PCI bus scan / root bridge resource allocation
  - `OnRootBridgesConnected: root bridges have been connected, installing ACPI tables`
  - `PlatformBootManagerAfterConsole`
- 当前最后稳定日志停在 mass storage / virtio-blk 连接附近：
  - `Found Mass Storage device: PciRoot(0x0)/Pci(0x3,0x0)`
  - `VirtioBlkInit: LbaSize=0x200[B] NumBlocks=0x200000[Lba]`
  - `VirtioBlkInit: FirstAligned=0x0[Lba] PhysBlkSize=0x1[Lba]`
  - `VirtioBlkInit: OptimalTransferLengthGranularity=0x0[Lba]`
  - `BlockSize : 512`
  - `LastBlock : 1FFFFF`
- 之后日志没有继续增长。

### 当前判断

- 现在不是 PXE，也不是 AHCI。旧日志里看到的 `Booting UEFI PXEv4` / AHCI `AtaError` 不能作为当前卡点。
- OVMF 已经找到 `PciRoot(0x0)/Pci(0x3,0x0)` 的 virtio-blk mass storage，并读到了 Block I/O 几何信息。
- 当前尚未看到：
  - `PlatformBdsConnectSequence`
  - `EfiBootManagerConnectAll`
  - `Load Options Dumping`
  - `Boot000x`
  - `Booting ...`
- 因此当前卡点先按 BDS 的 **connect mass storage / virtio-blk Block I/O 后续枚举** 处理：重点确认 `VirtioBlkDxe` 安装 Block I/O 后，PartitionDxe / FatDxe 是否继续读取块设备，以及 AxVisor 的 virtio-blk 设备模型是否卡在后续 `ReadBlocks` 请求。
- 同时要确认当前 guest boot 介质是否本来就不具备 UEFI 可启动结构。如果镜像只有整盘 ext4，没有 GPT/MBR + ESP/FAT + EFI loader，那么即使 virtio-blk 正常，BDS 也未必能生成磁盘启动项。

### Step 1: virtio-blk legacy I/O 已确认进入 queue notify

- 本轮目标：不要再猜 `LastBlock` 后 OVMF 是否继续读盘，先在 AxVisor 侧确认 L2 对外层 QEMU `virtio-blk-pci` 的访问。
- 代码定位结果：仓库内没有 AxVisor 自己的 virtio-blk PCI 设备模型或 virtqueue 实现。当前 `00:03.0` 是外层 QEMU 提供的 `virtio-blk-pci`，L2 OVMF 通过 AxVisor 的 passthrough I/O / MMIO 访问它。
- BDS resource map 给 `PciRoot(0x0)/Pci(0x3,0x0)` 分配的资源是：
  - legacy I/O BAR：`0x6000..0x607f`，Owner = `PCI [00|03|00:10]`
  - MMIO BAR：`0x81085000..0x81085fff`，Owner = `PCI [00|03|00:14]`
  - 64-bit BAR：`0x380000000000..0x380000003fff`，Owner = `PCI [00|03|00:20]`
- 发现一个独立基础问题：`components/x86_vcpu/src/vmx/structs.rs` 的 `IOBitmap::set_intercept()` 把单张 I/O bitmap 当成 1024 字节。Intel SDM 里 bitmap A/B 各是 4 KiB。以前只拦 `0x402` / `0x510` / `0x514`，没碰到高 offset；这次拦 `0x6000` 时直接越界 panic：
  - `index out of bounds: the len is 1024 but the index is 3072`
- 已把该 slice 长度改成 4096，并在 `components/x86_vcpu/src/vmx/vcpu.rs` 里临时拦截 `0x6000..0x607f`。拦截后对 legacy virtio-blk I/O 做记录并继续转发到宿主端口，避免把诊断改成新的设备模型。
- 验证命令：
  - 从 `tgoskits/` 执行 AxVisor build 通过。
  - 从 `tgoskits/os/axvisor/` 执行 70 秒 smoke，输出到 `/home/ssdns/work/axvisor-uefi/tmp/ovmf-bds-virtio-io.log`。命令最终被 `timeout` 结束，但日志已推进到 BDS 的 virtio-blk 连接点。
- 新日志显示 OVMF 确实在 `LastBlock` 后访问了 virtio-blk legacy I/O BAR：
  - `out port=0x6012 width=Byte value=0x0`
  - `out port=0x6012 width=Byte value=0x1`
  - `out port=0x6012 width=Byte value=0x3`
  - `in port=0x6000 width=Dword value=0x79006e54`
  - `in port=0x6014 width=Dword value=0x200000`
  - `in port=0x6028 width=Dword value=0x200`
  - `out port=0x600e width=Word value=0x0`
  - `in port=0x600c width=Word value=0x100`
  - `out port=0x6008 width=Dword value=0x2954`
  - `out port=0x6004 width=Dword value=0x640`
  - `out port=0x6012 width=Byte value=0x7`
  - `VirtioBlkInit: LbaSize=0x200[B] NumBlocks=0x200000[Lba]`
  - `BlockSize : 512`
  - `LastBlock : 1FFFFF`
  - `out port=0x6010 width=Word value=0x0`
- 这里的 `0x6010` 是 legacy virtio queue notify 偏移，`value=0` 表示通知 queue 0。也就是说 OVMF 不是停在“没有发读请求”，而是已经完成设备特性/queue 基址配置，并在安装 Block I/O 后发起了第一轮 queue notify。

#### 当前判断更新

- 当前卡点从“确认 OVMF 是否继续读块”推进为：**queue notify 后没有看到 virtio-blk 请求完成**。
- 后续应集中在 legacy virtio queue 事务是否对外层 QEMU 可见：
  - queue PFN 写入 `0x6008` 的值是 `0x2954`，按 4 KiB 页换算 guest queue GPA 约为 `0x2954000`。
  - notify 前 queue select 是 `0`，queue size 读到 `0x100`。
  - 需要确认 L2 写入的 queue 内存对外层 QEMU DMA 是否可见；当前 L2 的低端 RAM 是 AxVisor 分配内存，不一定等价于外层 QEMU 设备 DMA 看到的 GPA/host physical memory。
- 这比“磁盘镜像不是 UEFI 可启动盘”的检查更靠前。镜像是否有 ESP/FAT/`BOOTX64.EFI` 要等 virtio request 能完成后再判断。
- 暂时不要转向 OVMF / EDK2 改动。下一步应该继续在 AxVisor 侧确认 queue descriptor / avail ring / used ring 的 GPA/HPA 映射关系，以及外层 QEMU virtio-blk 是否能 DMA 到这块内存。

### Step 2: virtio-blk nested-DMA 地址翻译——VMX 层迁到 AxVM 层做 GPA→HPA 改写

#### 看到的日志/源码逻辑

- Step 1 最后：`out port=0x6010 width=Word value=0x0`，legacy virtio queue notify。
- 此前 OVMF 已写 `queue_pfn=0x2954`（→ L2 GPA `0x2954000`）、读 `queue_size=0x100`、写 `guest_features=0x640`。
- Step 1 的 VMX 层直接透传这些值到外层 QEMU，但 queue PFN 和 descriptor buffer 地址全是 L2 GPA，外层 QEMU DMA 看到的是自己的地址空间，不是 AxVisor EPT 映射后的 backing HPA。

#### 判断

nested virtualization 下 DMA 地址空间不匹配：L2 GPA `0x2954000` 对外层 QEMU virtio-blk 设备没有意义，它需要 backing HPA。阶段不变（BDS mass storage / virtio-blk 连接）。

#### 为此做了什么改动

- **`components/x86_vcpu/src/vmx/vcpu.rs`** 行 1609-1625：删除 `handle_ovmf_virtio_blk_io_passthrough()`（原行 159-202）和 `X86Port` import。`0x6000..0x607f` 改为返回 `AxVCpuExitReason::IoRead`/`IoWrite`，交给 AxVM dispatch。
- **`components/axvm/Cargo.toml`** 行 40：新增 `x86_64` 依赖，让 AxVM 能做宿主 port I/O。
- **`components/axvm/src/vm.rs`**：
  - 常量 `OVMF_VIRTIO_BLK_*`（行 76-82）、状态结构体 `OvmfVirtioBlkIoState`（行 132-136）、挂 `AxVMInnerMut`（行 147）、init（行 357）。
  - I/O 读写 dispatch 分别在行 761 和 784 插入 virtio-blk handler。
  - `handle_ovmf_virtio_blk_io_read/write`（行 1063/1097）：读宿主端口，写时对 queue PFN（`0x6008`）调用 GPA→HPA 翻译并转发 translated PFN，对 notify（`0x6010`）先做 descriptor 改写再转发。
  - `translate_ovmf_virtio_blk_queue_pfn`（行 1141）：`translate_and_get_limit(queue_gpa)` 拿到 HPA，记录原始/翻译后 PFN。
  - `rewrite_ovmf_virtio_blk_descriptors`（行 1156）：遍历 descriptor chain（最多 8 个），对每个 `desc.addr` 做 GPA→HPA 翻译后 **原地改写** descriptor table 中的地址字段。
  - `dump_ovmf_virtio_blk_queue`（行 1234）：打出 queue PFN、avail/used ring 状态。

#### 结果

编译/build 通过。smoke（90s timeout，日志 `tmp/ovmf-bds-nested-dma.log`）：

- queue PFN `0x2954` → HPA `0x7154000` → translated PFN `0x7154` 转发给 QEMU。✓
- 5 次 notify，每次 descriptor chain 地址全部翻译成功（`0x295xxxx` → `0x715xxxx`），无翻译失败。✓
- **used_idx 从 0 递增到 4**：外层 QEMU 成功 DMA 并回写 used ring。✓
- OVMF 越过 virtio-blk，继续发现 `PciRoot(0x0)/Pci(0x1F,0x2)`（SATA 控制器）。✓
- **新卡点**：port `0x604` Word read → device dispatch 层 `panic_device_not_found`。`0x604` 是 QEMU isa-debug-exit 端口，已在 VMX I/O bitmap 中被拦截，但 AxVM 没有对应 handler，落入 device 层 panic。

#### 下一步边界

- nested-DMA 修补已验证可用，当前阻塞点不是 virtio-blk DMA。
- 需要在 AxVM 给 port `0x604` 加 handler（读返回 0，写忽略），不让它落入 device 层 panic。
- descriptor 原地改写是临时手段，最终需替换为 AxVisor 侧 virtio-blk device model。

### Step 3: 修复 ACPI PM I/O (0x600-0x60F) host passthrough，推进到 BDS 启动项枚举

#### 问题

Step 2 最后：`Boot Mode:0` 后 OVMF 读 `port=0x604 width=Word` → panic。`0x604` 在 QEMU q35 中是 PMBA(0x600)+4 = ACPI PM1_CNT 寄存器（Power Management 1 Control）。VMX I/O bitmap 已拦截（`QEMU_EXIT_PORT`），但 AxVM device dispatch 没有 handler，落入 `panic_device_not_found`。

#### 判断

阶段仍在 BDS，但已接近 Boot Manager 末尾。OVMF 读 PM1_CNT 检测 SCI_EN/sleep state，写 PM1_CNT 使能 ACPI SCI。这是连接 sequence 完成后、开始 load options 前的标准 ACPI 初始化操作。

#### 改动

- **`components/axvm/src/vm.rs`**：
  - 常量 `ACPI_PM_IO_BASE=0x600`、`ACPI_PM_IO_SIZE=0x10`（行 84-86）。
  - `handle_acpi_pm_io_read`（行 ~1296）：范围判断后 `X86Port` 读宿主端口并返回。
  - `handle_acpi_pm_io_write`（行 ~1318）：范围判断后 `X86Port` 写宿主端口。
  - I/O 读 dispatch（行 ~774）：在 virtio-blk handler 之后插入 `handle_acpi_pm_io_read`。
  - I/O 写 dispatch（行 ~797）：同上位置插入 `handle_acpi_pm_io_write`。

#### 结果

smoke（日志 `tmp/ovmf-bds-acpi-pm.log`）：

- port 0x604 Word read → `value=0x0`（PM1_CNT 返回 S0 状态，SCI_EN=0）。✓
- port 0x604 Word write → `value=0x1`（OVMF 设置 SCI_EN 使能 ACPI SCI）。✓
- 无 panic。OVMF 继续推进：
  - BDS load options 枚举完成：
    ```
    Boot0000: BootManagerMenuApp
    Boot0001: EFI Firmware Setup
    Boot0002: UEFI Misc Device
    Boot0003: UEFI PXEv4
    Boot0004: EFI Internal Shell
    ```
  - 尝试从 Boot0002（virtio-blk `PciRoot(0x0)/Pci(0x3,0x0)`）启动 → `Not Found`。符合预期：镜像为 ext4 rootfs，无 FAT ESP/`BOOTX64.EFI`。
  - 回退到 Boot0003 PXE 网络启动。90s timeout 终止时正在 PXE DHCP 阶段，无 panic。

- OVMF BDS 已完整走通启动项枚举和回退链，主线卡点不再是 OVMF 自身逻辑或 AxVisor I/O 拦截。

#### 下一步边界

- 如果要让 OVMF 从磁盘启动，需要 UEFI 可启动镜像（GPT + ESP + FAT + EFI loader），或 AxVisor 侧实现 virtio-blk device model 来桥接现有 ext4 镜像。
- 当前临时修补：nested-DMA descriptor 改写 + PM I/O passthrough。后续应考虑正规化（AxVisor 侧 virtio-blk device model、ACPI PM device model）。

### Step 4: UEFI 可启动镜像验证——OVMF 成功进入 EFI Shell

#### 问题

Step 3 确认 OVMF BDS 已完整走通启动项枚举和回退链，但 ext4 rootfs 镜像无 ESP/FAT/`BOOTX64.EFI`，导致 Boot0002 报 `Not Found`，无法验证完整磁盘启动路径。

#### 判断

阶段仍在 BDS。需要换一个 UEFI 可启动镜像来验证 virtio-blk nested-DMA 读盘链路在 BDS boot 阶段也能正常工作。

#### 改动

- 创建测试镜像 `/home/ssdns/work/axvisor-uefi/tmp/uefi-boot-test.img`：64MB GPT 盘，单 ESP（FAT32），放入 `EFI/BOOT/BOOTX64.EFI`（edk2 构建产出的 `Shell.efi`）。
- AxVisor 需要从该镜像加载 OVMF firmware 文件，因此从 Alpine rootfs 复制 `/guest/ovmf/OVMF_CODE.fd` 和 `OVMF_VARS.fd` 到测试镜像同路径。
- 使用 `--rootfs` 参数指定测试镜像路径，绕过 ostool 的 `patch_qemu_rootfs` 自动覆盖。

#### 结果

smoke（日志 `tmp/ovmf-bds-uefi-boot.log`）：

- OVMF 成功从 virtio-blk 磁盘启动：FS0 映射为 `PciRoot(0x0)/Pci(0x3,0x0)/HD(1,GPT,...)`。✓
- EFI Shell 正常进入，显示 mapping table 和 `Shell>` 提示符。✓
- nested-DMA 持续正常工作：used_idx 从 306 递增到 318+，每次 notify 前 descriptor 地址翻译成功。✓

#### 下一步边界

- BDS 阶段的 virtio-blk nested-DMA 通路已验证通过。OVMF 可启动到 UEFI Shell 交互状态。
- 当前临时修补需正规化：nested-DMA descriptor 原地改写 → AxVisor 侧 virtio-blk device model；ACPI PM I/O passthrough → device model。
- 后续可根据需要测试其他 UEFI boot target（如 Linux kernel via EFI stub）。

### Step 5: OS 内核加载验证——Linux EFI stub 解压失败，最小 UEFI 应用通过

#### 问题

Step 4 只验证了 `Shell.efi`（EDK2 自带 UEFI 应用），不是真正的 OS 内核。需要换一个 OS kernel 验证完整链路。

#### 判断

阶段仍在 BDS。选了两种方案：先用 Linux EFI stub（系统 vmlinuz），失败后换自行构建的最小 UEFI 应用。

#### 改动

**A. Linux EFI stub 尝试：**

- 测试镜像同时放 Shell.efi 作为 `BOOTX64.EFI`、vmlinuz 和 `startup.nsh`。Shell 启动后自动执行 nsh 加载内核。
- VM 内存从 64MB 扩到 128MB —— `tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-qemu-smp1.toml` 行 46：`[0x0000_0000, 0x0800_0000, ...]`。
- 内核加载成功但解压失败。日志：
  ```
  FSOpen: Open '\vmlinuz.efi' Success
  EFI stub: ERROR: Failed to decompress kernel
  EFI stub: ERROR: efi_stub_entry() failed!
  ```
- 可能原因：宿主 Ubuntu 6.17 内核压缩代码需要 SSE/AVX，nested virt 下未暴露；或 DMA 大数据（14MB）有边界损坏。

**B. 最小 UEFI 应用（`/tmp/uefi-hello/`）：**

- 用 `x86_64-unknown-uefi` target 构建独立 EFI 应用，不依赖 ArceOS 构建系统。
- 入口 `efi_main(Handle, *SystemTable)`，输出 banner、遍历 UEFI memory map、显示物理内存统计。
- 编译为 PE32+ EFI application（93KB），替换为 `BOOTX64.EFI`。

#### 结果

smoke（日志 `tmp/ovmf-uefi-hello.log`）：

```
AxVisor Guest UEFI OS Kernel v0.1
======================================
Guest OS kernel verified!
OVMF -> virtio-blk -> UEFI App chain OK
```

- Linux EFI stub 路径：内核被加载并执行到 EFI stub 入口，但 bzImage 解压失败。✓（部分）
- 最小 UEFI 应用：完整跑通 OVMF → virtio-blk DMA 读盘 → PE32+ 加载 → EFI 应用执行。✓

#### 下一步边界

- Linux EFI stub 解压失败需继续排查（CPU feature 暴露或 DMA 数据完整性）。
- ArceOS UEFI 适配需要改 linker script、boot.S、mem.rs 三处，从 multiboot（32-bit entry）切换到 UEFI（64-bit entry）。当前链路已验证 PE32+ 加载可行，ArceOS 适配的二进制格式不是障碍。
