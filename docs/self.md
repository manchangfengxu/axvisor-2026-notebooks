# 原理：

OVMF的复位向量需要放在地址特定位置：0xFFFFFFF0 （即 4GB 物理地址空间的顶端，倒数第 16 个字节的地方）

- 复位向量指cpu通电/复位后第一个寻找的指令，物理机上通过硬件方式自动锁死了cpu寄存器的值
- 指令：jmp SEC_STAGE_ENTRY  ; 弹跳到固件中真正的 SEC 阶段入口

1. 需要大致规划好启动配置是什么。

2. 同时梳理目前代码的启动链路，让我们的配置能够识别加载。
- 这是否意味着我们要为当前读取配置逻辑加一个模块还是能复用？
- 配置最主要的是能够将 复位指令/OVMF镜像 的地址空间映射好
- 其次是告诉axvisor模拟硬件行为时，给cpu寄存器的值配置成什么

3. vcpu如何模拟复位向量过程
- VMCS（Virtual Machine Control Structure，虚拟机控制结构）
- 相应时刻，模拟硬件行为在struct::多个寄存器上去写入值
- vcpu运行前，相应内容的GPA应映射到了HPA。



设置复位向量地址，让vcpu启动去读取那条地址，开始启动OVMF的第一步。

# 进入OVMF阶段发生的异常
- OVMF 在早期阶段执行到 guest_rip=0x82997a 附近时，先触发了一个 #BR 硬件异常。
- CPU 随后尝试通过 IDT 投递该异常，但在读取/使用 selector 0x10 的门描述符时又触发了 #GP，导致异常处理链路失败,
- 最终 VMX 以 exception/nmi exit 形式把这个故障暴露出来。
## 判断
- #BR 不是 reset-vector 问题了
- #BR 也不是 OVMF_CODE 没加载的问题了
- 它已经进入 OVMF 早期执行后的异常处理阶段
- 现在真正要查的是：
  a. RIP=0x82997a 对应 OVMF 里哪段代码
  b. 当时为什么会触发 #BR
  c. IDT[0x05] 为什么会走到 selector 0x10
  d. 0x10 在当前状态下为什么会再触发 #GP





# note
实际答案是：AxVisor 不是全局只能 smp1，但当前 x86_64 OVMF 路线基本只能把 smp1 当作可信工作目标。

  原因分三层：

  1. 配置层支持多 vCPU
    - VM config 有 cpu_num。
    - VM 创建路径会按 cpu_num 创建多个 vCPU。
    - x86 CPUID 代码也已经用 current_vm_vcpu_num() 返回 guest logical CPU count。
    - 所以不是框架上完全只支持 smp1。
  2. 当前 OVMF x86 配置只有 smp1
    - 目前主线配置是：
  ovmf-x86_64-qemu-smp1.toml
    - 没有现成可信的：
  ovmf-x86_64-qemu-smp2.toml
  ovmf-x86_64-qemu-smp4.toml
    - 文档和 bring-up 路线也都围绕 smp1。
  3. x86 vLAPIC / IPI / AP startup 还不完整
    - current_vm_active_vcpus() 后端还是 todo!。
    - x86_vlapic 里关键 SMP 行为还没实现完整：
        - self IPI
      - interrupt injection
      - NMI injection
      - INIT/SIPI AP startup
      - broadcast destination / active vCPU mask
    - 这些正是 OVMF 多核启动会依赖的路径。

  所以更准确的说法是：

  AxVisor 有多 vCPU 框架；
  但当前 x86_64 OVMF guest 的可验证 bring-up 路线是 smp1；
  smp2/smp4 现在大概率会卡在 APIC/IPI/AP startup 未实现处。

  这也解释了我们当前为什么一直用：

  ovmf-x86_64-qemu-smp1.toml

  而不是直接切 smp4 去逼 OVMF 走更多 MP 路径。

  如果后续要验证 x2APIC / AP startup，更稳的顺序是：

  1. 先让 smp1 OVMF 完整过 PEI/DXE；
  2. 补齐 x2APIC MSR path；
  3. 再做 smp2/smp4 配置；
  4. 最后实现和验证 INIT/SIPI、IPI delivery、active vCPU mask。