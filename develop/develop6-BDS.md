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
