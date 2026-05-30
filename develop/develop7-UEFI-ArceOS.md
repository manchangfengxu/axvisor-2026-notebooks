## 本阶段行为准则

- 日志多多益善，确定修改是否正确，通过日志代码定位回源码位置。
- 每推进一个对主线有意义的小阶段，就更新本文档。
- 每条记录必须说明：看到什么日志/源码逻辑、判断出现什么问题、为此做了什么改动达到了目标（描述简洁，改动位置定位具体）。
- 不为了"继续推进"而盲目扩展范围；如果下一步需要大改，就明确记录边界。
- 保留原有 multiboot / 32-bit entry 兼容性，但它不是本阶段主线。不要为了兼容旧入口，把 UEFI 入口和 multiboot 入口写成一团难以验证的共享启动代码。
- 优先做可验证的小闭环：先让 UEFI entry 进入 ArceOS runtime 并打印，再考虑 `ExitBootServices`、内存图转换、分页、驱动、SMP。

## 基线

- 本阶段接 `develop6-BDS.md`：OVMF 已能通过 virtio-blk 启动 EFI Shell 和 PE32+ UEFI 应用。
- 主线：让 ArceOS 最小 guest 经 OVMF/UEFI loader 成功启动，不是补齐完整 Linux/UEFI 生态。
- 入口策略：新增并行 UEFI boot profile（`axplat-x86-pc` 的 UEFI entry + PE32+ 产物），旧 multiboot 路径不动。两条路径进入 runtime 前各自归一化 boot context，进入后复用 `rust_main`。
- 第一目标：最小可验证链路 OVMF → FAT32 ESP / `BOOTX64.EFI` → ArceOS UEFI entry → 打印 → 构造 boot context → 调用 `rust_main`。之后再推进 `ExitBootServices`、memory map 转换、ArceOS 自有 console/allocator。
- 参考：`tmp/uefi-hello/` 已证明 `x86_64-unknown-uefi` PE32+ 链路可用；`references/cloud-hypervisor/docs/uefi.md` 可借鉴 firmware/boot 边界；`references/cloud-hypervisor/arch/src/x86_64/layout.rs` 可借鉴 x86 保留区域划分思路。

## Step 1: 方案定界 —— 新增 UEFI entry，不破坏 multiboot entry

### 看到的源码逻辑

- `axplat-x86-pc/src/multiboot.S` 目前是 32-bit multiboot 专用入口，负责从 protected mode 切到 long mode。
- UEFI x86_64 application 已经由 OVMF 以 64-bit long mode 调用，不需要重复 multiboot 的 32-bit 入口、GDT 切换和临时页表流程。
- `axhal/linker.lds.S` 当前 `ENTRY(_start)` 和 `.text.boot` 仍按 multiboot/bare-metal kernel 组织。
- `axplat-x86-pc/src/mem.rs` 当前只认识 multiboot memory map。

### 判断

- UEFI 入口和 multiboot 入口的前置 CPU 状态不同，应并行存在，不应强行共用 `multiboot.S`。
- multiboot 兼容应保留在旧 feature/profile 下；UEFI 主线应使用单独 entry 文件和单独链接/构建产物。
- 最小可行改法是新增 UEFI entry profile，而不是重写 `axplat-x86-pc` 的默认启动路径。

### 建议改动边界

- 新增 UEFI entry 文件，例如：
  - `axplat-x86-pc/src/uefi.rs`
  - 或新平台 crate / feature：`axplat-x86-pc` 的 `uefi` feature。
- 新增 UEFI boot context：
  - 保存 `ImageHandle`
  - 保存 `SystemTable`
  - 保存 UEFI memory map 转换后的 RAM/reserved ranges
- `mem.rs` 保持 multiboot 默认逻辑；UEFI path 下走另一套 `init_from_uefi_memory_map()`。
- 初期只支持 `smp1`、无 `irq`、无 `fs`、无 `net`、无 `display`。
- 不在第一步处理 Linux EFI stub 解压失败。

### 下一步

1. 明确 ArceOS 构建系统如何增加一个 `x86_64-unknown-uefi` 或 PE32+ 产物。
2. 先做一个 ArceOS 外壳级 UEFI entry：打印 banner 后停机，确认它能替换 `tmp/uefi-hello`。
3. 再把 UEFI entry 接到 `axruntime::rust_main(0, boot_context_ptr)`。
4. 最后处理 UEFI memory map 到 ArceOS `phys_ram_ranges()` 的转换。

## Step 2: ArceOS 外壳级 UEFI app 已作为 BOOTX64.EFI 跑通

### 看到的日志/源码逻辑

- 新增 workspace member：
  - `tgoskits/os/arceos/examples/uefi-helloworld/Cargo.toml`
  - `tgoskits/os/arceos/examples/uefi-helloworld/src/main.rs`
- `tgoskits/Cargo.toml` 增加：
  - workspace member `os/arceos/examples/uefi-helloworld`
  - workspace dependency `ax-uefi-helloworld`
- 新示例是 `#![no_std]` / `#![no_main]`，导出 `efi_main(ImageHandle, SystemTable)`，使用 UEFI `SimpleTextOutputProtocol` 打印，并调用 `BootServices.GetMemoryMap()` 输出 memory map 摘要。
- 当前没有改动 `axplat-x86-pc/src/multiboot.S`、`boot.rs`、`mem.rs`，multiboot / 32-bit entry 路径保持不动。

### 判断

- 这一步验证的是 **ArceOS 目录内的 UEFI PE32+ 外壳产物**，还不是完整 ArceOS runtime。
- 当前目标是确认：ArceOS 侧可以生成可被 OVMF BDS 加载执行的 EFI application，且不会破坏旧 multiboot 入口。
- 根据 `develop6-BDS.md` 的阶段划分，本次仍处于 BDS 启动项加载之后的 EFI application 执行阶段。

### 为此做了什么改动

- 新增 `ax-uefi-helloworld` 示例 crate：
  - 使用 `x86_64-unknown-uefi` target 构建。
  - 通过 UEFI console 打印：
    - `ArceOS UEFI helloworld`
    - `This keeps the legacy multiboot entry untouched.`
    - UEFI memory map entries / descriptor size / descriptor version / total memory / conventional memory。
  - 结尾停在 `hlt` 循环，便于 smoke run 保持屏幕状态。
- 用 `mcopy` 将生成的 EFI app 写入 FAT32 ESP：
  - 镜像：`tmp/uefi-boot-test.img`
  - ESP 偏移：`@@1048576`（分区从 sector 2048 开始）
  - 目标路径：`::/EFI/BOOT/BOOTX64.EFI`
- EDK2 Shell 备份来源明确保留为：
  - `tmp/BOOTX64-shell-edk2.efi`

### 下一步边界

- 下一步不是继续扩展 UEFI app 功能，而是把这个 UEFI entry 和 ArceOS runtime 接起来。
- 建议先传入一个最小 boot context 指针，调用 `axruntime::rust_main(0, boot_context_ptr)`，并让 `ax-hal` 在 UEFI profile 下从 boot context 读取 memory map。
- `ExitBootServices` 暂时不要作为下一步的第一动作；先确认 runtime 早期打印、percpu 初始化和最小 memory region 逻辑可走通。
