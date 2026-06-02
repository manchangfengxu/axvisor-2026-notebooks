# AxVisor OVMF 设备模型重构计划（精简版）

详细版见 `ovmf-device-model-refactor-plan-detailed.md`。

## 目标

把 `vm.rs` 中内联的 4 个模拟设备（debugcon、fw_cfg、virtio-blk I/O、QEMU exit）搬到独立的 `x86_qemu_device` crate，通过 TOML 配置声明，统一走 `AxVmDevices` 注册表分发。同时完成 `dev` 分支遗留的 I/O bitmap 从设备注册表自动生成的 TODO。

## 核心链路

```text
TOML emu_devices 声明设备
  ↓
axvmconfig 解析 → AxVMCrateConfig
  ↓
axdevice init() 按 emu_type 匹配，从 x86_qemu_device 实例化
  ↓
add_port_dev() 注册到 AxVmDevices 端口设备表
  ↓
refresh_io_bitmap() 从设备表自动生成 I/O bitmap
  ↓
guest I/O → VM exit → run_vcpu() → get_devices().handle_port_read/write → 设备
```

未声明的设备不注册、不拦截，自然穿透给外层 QEMU。

## 设备清单

| 设备 | emu_type | 端口 | 说明 |
|------|----------|------|------|
| debugcon | 0x40 | 0x402 | OVMF 日志输出 |
| fw_cfg | 0x41 | 0x510-0x51C | 固件配置接口，需 guest memory 引用 |
| qemu-exit | 0x42 | 0x604 | 关机信号，用 per-VM AtomicBool |
| ovmf-virtio-blk | 0x43 | 0x6000-0x6080 | legacy I/O BAR，需 guest memory 引用做 DMA 翻译 |

ACPI PM（0x600-0x60F）保持穿透，不注册为模拟设备。原因：当前是 host passthrough shim，且端口 0x600-0x60F 与 QEMU exit 0x604 重叠。

## 新建 crate

```
components/x86_qemu_device/
  src/
    lib.rs
    debugcon.rs
    fw_cfg.rs
    virtio_blk_io.rs
    qemu_exit.rs
```

每个设备实现 `BasePortDeviceOps` trait，构造时从 TOML config 读取端口范围。

## 关键设计决策

**1. TOML 声明，不硬编码**

和 aarch64 的 GIC 设备同一套模式：
```toml
emu_devices = [
    ["debugcon",       0x402,  0x1,  0, 0x40, []],
    ["fw-cfg",         0x510,  0xC,  0, 0x41, []],
    ["qemu-exit",      0x604,  0x4,  0, 0x42, []],
    ["ovmf-virtio-blk",0x6000, 0x80, 0, 0x43, []],
]
```

emu_type 使用 0x40..0x43（`#[repr(u8)]`，最大 0xFF）。BIOS 配置不声明 fw_cfg 和 ovmf-virtio-blk。debugcon 和 qemu-exit 出现在所有 x86 QEMU 配置中。

**2. GuestMemoryBytes trait**

`axaddrspace` 新增 object-safe 子 trait：
```rust
pub trait GuestMemoryBytes: Send + Sync {
    fn read_buffer(&self, gpa: GuestPhysAddr, buf: &mut [u8]) -> AxResult<()>;
    fn write_buffer(&self, gpa: GuestPhysAddr, buf: &[u8]) -> AxResult<()>;
    fn translate_and_get_limit(&self, gpa: GuestPhysAddr) -> Option<(HostPhysAddr, usize)>;
}
```

`translate_and_get_limit` 是 virtio-blk nested DMA 所需。设备通过 `Arc<dyn GuestMemoryBytes>` 访问 guest memory，不依赖 axvm 内部类型。

**3. QemuExitDevice 用 per-VM AtomicBool**

```rust
struct QemuExitDevice { shutdown_requested: Arc<AtomicBool> }
```

AxVM 创建 flag，clone 给设备，run_vcpu() 循环末尾检查。只停目标 VM，不影响其他 VM。保持当前行为：检查 width == Word 和 magic 值。

**4. FwCfgPlatformInfo 避免依赖循环**

fw_cfg 构造需要 memory_regions 和 cpu_count。定义在 axvmconfig 中：
```rust
pub struct FwCfgPlatformInfo {
    pub memory_regions: Vec<VMMemoryRegion>,
    pub cpu_count: usize,
}
```

`AxVmDevices::new(config, guest_mem, shutdown_flag, fw_cfg_info)` — 不传 AxVMConfig（会导致 axdevice → axvm 循环依赖）。

**5. I/O bitmap 从设备表自动生成**

vCPU 在 AxVmDevices 之前创建，不能在创建时传入设备引用。方案：AxVmDevices 创建后调用 `refresh_io_bitmap(devices)` 刷新。

**6. 未注册端口**

port read 返回 0xFF，port write ignore + warn。不 panic。

**7. 本轮只支持 VMX，只处理 port I/O**

SVM 后续再做。MMIO 设备（vIOAPIC 等）的 EPT 排除逻辑在加入时再实现。

## Phase 顺序

| Phase | 内容 | 依赖 |
|-------|------|------|
| 1 | 新增 GuestMemoryBytes trait + BaseDeviceOps string I/O 方法 | 无 |
| 2 | VmGuestMemory wrapper + FwCfgPlatformInfo + AxVmDevices::new() 签名扩展 + 测试适配 | 1 |
| 3 | 创建 x86_qemu_device crate，搬出 4 个设备 | 2 |
| 4 | 增加 EmulatedDeviceType 变体（0x40..0x43），TOML 声明设备，init() 注册 | 3 |
| 5 | 删除 vm.rs inline handler，简化 run_vcpu() | 4 |
| 6 | refresh_io_bitmap() 自动联动 | 5 |
| 7 | 回归验证 | 6 |

关键路径：1 → 2 → 3 → 4 → 5 → 6 → 7

## 文件清单

| 文件 | 修改 |
|------|------|
| `components/axaddrspace/src/memory_accessor.rs` | 新增 GuestMemoryBytes trait |
| `components/axdevice_base/src/lib.rs` | BaseDeviceOps 增加 string I/O 默认方法 |
| `components/axvmconfig/src/lib.rs` | 新增 FwCfgPlatformInfo + EmulatedDeviceType 变体 0x40..0x43 |
| `components/axvm/src/vm.rs` | VmGuestMemory wrapper + 删 inline handler + 简化 run_vcpu() |
| `components/axdevice/src/device.rs` | 扩展 new() 签名 + init() 注册逻辑 |
| `components/axdevice/tests/test.rs` | 适配 new(config, None, None, None) |
| `components/x86_qemu_device/` | 新 crate，4 个设备实现 |
| `components/x86_vcpu/src/vmx/vcpu.rs` | 删 0x604 SystemDown + refresh_io_bitmap() |
| `os/axvisor/configs/vms/ovmf-x86_64-qemu-smp1.toml` | emu_devices 声明 |
| `Cargo.toml`（workspace 根） | members 增加 x86_qemu_device |
| `components/axdevice/Cargo.toml` | 增加 x86_qemu_device 依赖 |

## 验证

1. `cargo build --target x86_64-unknown-none`
2. `cargo fmt --check`
3. `cargo xtask clippy --package axdevice --package x86_qemu_device --package axvm`
4. `make ovmf-run` — OVMF 走到 BDS、加载 BOOTX64.EFI
5. `arceos-x86_64-qemu-smp1.toml` 回归正常
6. `vm.rs` 行数 < 900
