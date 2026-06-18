# RISC-V 指令集体系结构规范高密卡片系统设计大图

## 1. 28 张卡片依赖拓扑图 (Mermaid)

```mermaid
graph TD
    %% Module 1: ISA Axioms & Pipeline Base
    C1[Card 1: RISC-V 架构公理] --> C2[Card 2: 寄存器与 ABI 规范]
    C2 --> C3[Card 3: 经典五级流水线]
    C3 --> C4[Card 4: CSRs 访问控制]
    C4 --> C5[Card 5: 特权模式与 Trap 路径]

    %% Module 2: Pipeline Hazards & Speculation
    C3 --> C6[Card 6: 流水线冒险与锁步]
    C6 --> C7[Card 7: 数据旁路转发 Bypass]
    C6 --> C8[Card 8: 控制冒险与冲刷]
    C8 --> C9[Card 9: 分支预测器 BHT/BTB]
    C5 --> C10[Card 10: 精确异常捕捉]
    C6 --> C10

    %% Module 3: Out-of-Order (OoO) Execution
    C3 --> C11[Card 11: 乱序执行与 Tomasulo]
    C11 --> C12[Card 12: 寄存器重命名 PRF]
    C12 --> C13[Card 13: 发射队列与动态调度]
    C13 --> C14[Card 14: ROB 顺序提交]
    C14 --> C15[Card 15: 访存队列与 Store-to-Load]

    %% Module 4: Cache Hierarchy & Coherence
    C15 --> C16[Card 16: 高速缓存 Cache 结构]
    C16 --> C17[Card 17: Cache 写策略与一致性]
    C17 --> C18[Card 18: MESI 缓存一致性状态机]
    C3 --> C19[Card 19: 地址翻译与 TLB Walk]
    C19 --> C16

    %% Module 5: Memory Models & Sync
    C18 --> C20[Card 20: 内存一致性物理意义]
    C20 --> C21[Card 21: RVWMO 弱内存模型]
    C21 --> C22[Card 22: FENCE/FENCE.I 屏障]
    C22 --> C23[Card 23: 原子指令 AMO]
    C23 --> C24[Card 24: LR/SC 原语监视器]

    %% Module 6: RVV & Debugging
    C1 --> C25[Card 25: 向量扩展 RVV 本质]
    C25 --> C26[Card 26: 向量 vsetvli 参数配置]
    C3 --> C27[Card 27: 性能计数器 HPM]
    C27 --> C28[Card 28: QEMU/Spike 仿真模拟]

    %% Cross-Module Links
    C10 --> C14
    C9 --> C14
    C15 --> C20
```

---

## 2. 经典 RISC-V 处理器硬件与模拟器源码物理路径映射

为了印证 28 张核心卡片中涉及的处理器微架构设计，我们将其映射至经典的开源 RISC-V 处理器项目 [chipsalliance/rocket-chip](https://github.com/chipsalliance/rocket-chip)（顺序流水线标杆）、[riscv-boom/riscv-boom](https://github.com/riscv-boom/riscv-boom)（乱序执行标杆）以及 [riscv-software-src/riscv-isa-sim (Spike)](https://github.com/riscv-software-src/riscv-isa-sim)（指令集模拟器金丝雀）的物理源码路径中：

### M1-M2: 五级流水线、特权模式、CSR 与冒险旁路
*   **Rocket Chip 顺序流水线控制流**: [rocket-chip/src/main/scala/rocket/IDecode.scala](file:///D:/riscv-cores/rocket-chip/src/main/scala/rocket/IDecode.scala) (指令译码映射表)
*   **CSR 寄存器控制与陷入跳转**: [rocket-chip/src/main/scala/rocket/CSR.scala](file:///D:/riscv-cores/rocket-chip/src/main/scala/rocket/CSR.scala) (xepc/xcause 陷入逻辑处理)
*   **数据旁路转发与流水线锁步**: [rocket-chip/src/main/scala/rocket/Rocket.scala](file:///D:/riscv-cores/rocket-chip/src/main/scala/rocket/Rocket.scala) (五级顺序流水线数据前推逻辑)
*   **分支预测器 (BHT/BTB)**: [rocket-chip/src/main/scala/rocket/BTB.scala](file:///D:/riscv-cores/rocket-chip/src/main/scala/rocket/BTB.scala) (两比特饱和计数器与目标地址缓冲实现)

### M3: 乱序执行、重命名、发射与重排序缓冲区 (BOOM 乱序核)
*   **寄存器重命名与映射表**: [riscv-boom/src/main/scala/exu/rename/rename.scala](file:///D:/riscv-cores/riscv-boom/src/main/scala/exu/rename/rename.scala) (消除伪相关的重命名逻辑)
*   **发射队列与动态调度**: [riscv-boom/src/main/scala/exu/issue-unit.scala](file:///D:/riscv-cores/riscv-boom/src/main/scala/exu/issue-unit.scala) (唤醒 Wakeup 与选择 Select 逻辑)
*   **重排序缓冲区 (ROB) 顺序提交**: [riscv-boom/src/main/scala/exu/rob.scala](file:///D:/riscv-cores/riscv-boom/src/main/scala/exu/rob.scala) (实现精确异常的顺序 Commit 缓冲)
*   **加载/存储队列 (LSU) 与访存前推**: [riscv-boom/src/main/scala/exu/lsu/lsu.scala](file:///D:/riscv-cores/riscv-boom/src/main/scala/exu/lsu/lsu.scala) (Store-to-Load Forwarding)

### M4-M5: MESI 缓存一致性、虚存翻译与内存屏障
*   **Cache 控制与一致性协议**: [rocket-chip/src/main/scala/tilelink/Metadata.scala](file:///D:/riscv-cores/rocket-chip/src/main/scala/tilelink/Metadata.scala) (缓存行 MESI 状态定义)
*   **硬件页表遍历 (Page Table Walker)**: [rocket-chip/src/main/scala/rocket/PTW.scala](file:///D:/riscv-cores/rocket-chip/src/main/scala/rocket/PTW.scala) (硬件地址翻译翻译机理)
*   **FENCE 与原子指令同步原语**: [rocket-chip/src/main/scala/rocket/ALU.scala](file:///D:/riscv-cores/rocket-chip/src/main/scala/rocket/ALU.scala) (AMO 原子操作硬件执行)

### M6: 向量扩展 (RVV) 与 Spike 仿真器
*   **Spike 向量指令集执行定义**: [riscv-isa-sim/riscv/insns/](file:///D:/riscv-cores/riscv-isa-sim/riscv/insns) 
    *   `vsetvli.cc` (元素宽度 SEW 与倍率 LMUL 参数配置)
    *   `vadd_vv.cc` (向量加法并行流水线仿真)
