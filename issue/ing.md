# mod0
1. pflash 设备模型
2. OVMF_CODE 只读保护
3. OVMF_VARS 可写持久化
4. 变量区写入行为
5. firmware window 的访问属性
6. 可能还要支持 OVMF 对 flash 命令/状态寄存器的访问

# mod1
AxVisor 当前还不支持 string/repeat I/O exit，所以先停在：

- `VMX unsupported IO-Exit: ... port: 0x511`

这里没有扩大 AxVisor 的 string I/O 支持，而是继续采用最小改动原则，改 OVMF 的 probe 代码。

位置：`edk2/OvmfPkg/Library/QemuFwCfgLib/QemuFwCfgPei.c`

把：

- `IoReadFifo8(FW_CFG_IO_DATA, ...)`

改成了逐字节：

- `IoRead8(FW_CFG_IO_DATA)`

这样就绕过了当前 VMX 对 `rep insb` 的限制，让 PEI 里的 fw_cfg 继续往下走。

