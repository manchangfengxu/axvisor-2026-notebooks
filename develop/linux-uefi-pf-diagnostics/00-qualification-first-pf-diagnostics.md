# Linux UEFI #PF 诊断：先看 qualification，再看 CR2

## 这一步做了什么

这轮不再继续把 Linux 早期失败笼统归因到 “virtio-blk 还不够完整”。

当前链路已经能稳定走到：

- OVMF 打开 `\EFI\BOOT\BOOTX64.EFI`
- EFI payload 打开 `\vmlinuz.efi`
- EFI stub 打开 `initramfs.cpio`
- 然后 guest Linux 在更早期的内存映射阶段反复触发 `#PF`

所以这一步只做两类基础设施整理：

1. 给 `x86_vcpu::vmx::vcpu` 补一套更可信的 `#PF` 诊断。
2. 给 `x86_64 + UEFI + OVMF + image_location="fs"` 这条加载路径补清楚契约日志，避免继续误读 `kernel_path` / `ramdisk_path`。

## 代码变更

### 1. `tgoskits/virtualization/x86_vcpu/src/vmx/vcpu.rs`

- 把 page-walk helper 从中间态修平，恢复可编译状态：
  - `fmt::Result` 和普通 `Result<T, E>` 分离。
  - 显式引入 `alloc::format`。
- 保留并完善了 guest page walk / EPT walk 的调试辅助：
  - `walk_4level_page_tables()`
  - `walk_guest_page_table_4level_debug()`
  - `walk_ept_4level_debug()`
  - `translate_guest_phys_via_ept()`
  - `read_guest_phys_u64()`
- `#PF` 诊断不再默认把 `guest_cr2` 当成唯一 fault VA。
  - 新增 `select_page_fault_addr()`。
  - 选择顺序改成：
    1. `EXIT_QUALIFICATION`
    2. `guest_cr2`
    3. `GUEST_LINEAR_ADDR`
- 在 VMX `#PF` 日志里明确打印：
  - `fault_va`
  - `guest_cr2`
  - `qualification`
  - `gla`
  - guest page walk 结果
  - Linux direct-map 候选 GPA
  - 对应 EPT walk 结果

### 2. `tgoskits/os/axvisor/src/images/mod.rs`

- 给 `x86_64 UEFI+OVMF fs` 路径增加契约日志：
  - 这条路径只预加载 `OVMF_CODE` / `OVMF_VARS` / 可选 `disk_path`
  - `kernel_path` 在这里是配置元数据，不会被预装进 guest RAM
  - EFI payload 由 OVMF 从 nested disk 文件系统自行加载
  - 配了 `disk_path` 时会明确打印实际暴露给 OVMF 的磁盘路径

## 验证

### 构建与单测

- `cargo test -p x86_vcpu --lib`
- `cargo check -p axvisor --target x86_64-unknown-none --features fs,vmx`

结果：

- `x86_vcpu` 68 个单测通过。
- `axvisor fs+vmx` 构建通过。

### Linux UEFI smoke

命令：

```bash
cd "$WORKSPACE/tgoskits/os/axvisor"
timeout 60 cargo xtask qemu \
  --config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64.toml" \
  --qemu-config "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/qemu-x86_64-runtime.toml" \
  --vmconfigs "$WORKSPACE/tgoskits/os/axvisor/tmp/configs/ovmf-x86_64-linux-smoke.toml" \
  --rootfs "$WORKSPACE/tgoskits/tmp/axbuild/rootfs/rootfs-x86_64-alpine.img"
```

本轮重定向日志：

- `tmp/ovmf-linux-smoke-pfdiag.log`

## 关键观测

### 1. UEFI+OVMF 契约日志已经进了真实 smoke

日志里能看到：

- `x86_64 UEFI+OVMF fs path preloads OVMF_CODE/OVMF_VARS and disk_path; EFI payload loading is left to OVMF`
- `x86_64 UEFI+OVMF fs path keeps kernel_path=/guest/uefi/empty-kernel-placeholder.bin as config metadata; it is not preloaded into guest RAM on this path`
- `x86_64 UEFI+OVMF fs path will expose disk_path=/guest/disks/linux-uefi-smoke.img as the nested virtio-blk disk for OVMF`

这说明当前加载路径的行为已经被日志直接说清楚了。

### 2. OVMF -> Linux EFI stub 链路仍然成立

日志里能看到：

- `FSOpen: Open '\EFI\BOOT\BOOTX64.EFI' Success`
- `FSOpen: Open '\vmlinuz.efi' Success`
- `FSOpen: Open 'initramfs.cpio' Success`

所以当前阻塞点已经不是 “disk 没挂上” 或 “EFI stub 没读到 kernel/initrd”。

### 3. `guest_cr2` 在这个 VMX #PF 退出路径上是 stale 的

旧日志一直把 `fault_va` 打成 `0x0`，这会直接误导 page walk。

本轮修正后，日志明确显示：

- `fault_va=0xffff88800e276000`
- `guest_cr2=0x0`
- `qualification=0xffff88800e276000`
- `using exit qualification as the fault VA because guest CR2 is stale on this VM-exit path`

也就是说，对这类 VMX `#PF` 退出，`EXIT_QUALIFICATION` 比 `guest_cr2` 更可靠。

### 4. 现在能把问题切到 guest 页表，而不是 EPT

同一组日志里能看到：

- guest page walk:
  - `fault_va=0xffff88800e276000`
  - `PML4 entry_addr=0x3d60888`
  - `entry=0x0`
- Linux direct-map 候选：
  - `candidate_gpa=0xe276000`
- EPT walk:
  - `candidate_gpa=0xe276000 -> host_phys=0xee76000`
  - `page_size=0x200000`

这组证据说明：

1. `0xe276000` 这块 guest physical memory 本身是被 EPT 正常映射出来的。
2. Linux 访问 `ffff88800e276000` 时，guest 自己的页表没有把这段 direct-map VA 建起来。
3. 所以当前失败点更像是 **guest 页表构造 / early memory map / early memblock** 这一层，而不是 virtio-blk I/O 面或 EPT 缺页。

## 当前结论

- 当前 `virtio-blk` 仍然是 legacy-only，但它已经足够把 OVMF 和 Linux EFI payload 拉起来。
- 这条链路的主阻塞点已经进一步缩小到：
  - Linux 早期页表 / direct-map 建立不完整
  - 或更上层的 EFI memory map / early memory discoverability 问题
- 新的 `#PF` 诊断已经把 “fault VA 选错” 这个干扰项拿掉了。

## 下一步建议

下一轮不要回头把问题重新解释成 “先补 modern virtio 再说”。

更值得做的是：

1. 结合 `RIP=0xffffffff834d2400` 去定位这段 Linux 代码在做什么。
2. 继续核对：
   - Linux 看到的 EFI memory map
   - early memblock / initrd / reserved range
   - direct-map 建表前后 `0xe276000` 是否被排除
3. 如果还需要更细的证据，再在 `#PF` 日志里补：
   - 当前 fault VA 对应的 direct-map 区间判定
   - 更完整的 guest page-table chain dump
   - 触发点前后的 guest memory map / e820 / EFI memory descriptor 关键摘要
