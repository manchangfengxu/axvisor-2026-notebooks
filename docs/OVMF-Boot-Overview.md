# OVMF 启动流程概览
本页给出了 OVMF 启动流程的高层概览。

## SEC
- UefiCpuPkg/ResetVector/Vtf0
    - Ia16/ResetVectorVtf0.asm：从 resetVector 开始执行
    - Ia16/Real16ToFlat32.asm：模式切换：实模式 => 32 位 Flat 模式
    - Ia32/SearchForBfvBase.asm：定位启动固件卷（Boot Firmware Volume, BFV）
    - Ia32/SearchForSecEntry.asm：在 BFV 中定位 SEC Core
    - Ia32/Flat32ToFlat64.asm：如果 PEI 是 64 位，则执行模式切换：32 位 Flat 模式 => 64 位 Long 模式
- SEC Core：OvmfPkg/Sec
    - OvmfPkg/Sec/SecMain.c：查找 PEI Core

## PEI
PEI（Pre-EFI Initialization，EFI 前初始化）

- PEI Core：MdeModulePkg/Core/Pei
    - 此时会运行各种 PEI 驱动
- DXE IPL（DXE Core 的加载器）
    - 查找 DXE Core
    - 如果 PEI 是 32 位而 DXE 是 64 位，则执行模式切换：32 位 Flat 模式 => 64 位 Long 模式
    - 跳转到 DXE Core

## DXE
DXE（Driver Execution Environment，驱动执行环境）

- DXE Core：MdeModulePkg/Core/Dxe
    - 此时会运行各种 DXE/UEFI 驱动
    - 当所有 DXE 架构协议都可用后，开始运行 BDS

## BDS
BDS（Boot Device Selection，启动设备选择）

- BDS：IntelFrameworkModulePkg/Universal/BdsDxe + IntelFrameworkModulePkg/Library/GenericBdsLib
    - OvmfPkg/Library/PlatformBdsLib 以平台特定的方式引导 BDS 的行为
- BDS 会决定从哪里启动，并初始化相应设备以完成启动
    - 如果系统固件文件系统中包含 EFI Shell，BDS 也可能加载它

---

# OVMF Boot Overview
This page provides a high-level overview of the boot process for OVMF.

## SEC
- UefiCpuPkg/ResetVector/Vtf0
    - Ia16/ResetVectorVtf0.asm: Starts at resetVector
    - Ia16/Real16ToFlat32.asm: Mode Switch: Real => 32-bit Flat
    - Ia32/SearchForBfvBase.asm: Locates the Boot FirmwareVolume (BFV)
    - Ia32/SearchForSecEntry.asm: Locates the SEC Core with BFV
    - Ia32/Flat32ToFlat64.asm: If 64-bit PEI, then Mode Switch: 32-bit Flat to 64-bit Long
- SEC Core: OvmfPkg/Sec
    - OvmfPkg/Sec/SecMain.c: Finds PEI Core

## PEI
Pre-EFI Initialization

- PEI Core: MdeModulePkg/Core/Pei
    - Various PEI drivers now run
- DXE IPL (Loader for the DXE Core)
    - Finds DXE Core
    - If PEI is 32-bit and DXE is 64-bit, then Mode Switch: 32-bit Flat to 64-bit Long
    - Jumps to DXE Core

## DXE
Driver Execution Environment

- DXE Core: MdeModulePkg/Core/Dxe
    - Various DXE/UEFI drivers now run
    - BDS is run after all DXE architectural protocols are available

## BDS
Boot Device Selection

- BDS: IntelFrameworkModulePkg/Universal/BdsDxe + IntelFrameworkModulePkg/Library/GenericBdsLib
    - OvmfPkg/Library/PlatformBdsLib guides BDS in a platform specific manner
- BDS figures out what to boot, and starts devices to enable the boot
    - BDS may be able to load the EFI shell if it is present in the system firmware file system

