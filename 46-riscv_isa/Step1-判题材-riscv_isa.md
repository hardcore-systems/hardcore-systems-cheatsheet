# RISC-V 指令集体系结构规范高密卡片系统题材判定

## 1. 核心模块与 28 张卡片大纲

### M1: 指令集架构 (ISA) 公理、特权级与流水线基础 (Cards 1-5)
*   **Card 1**: RISC-V 体系架构公理：规整性、模块化（I 基础指令，M/A/F/D/C 扩展）与极简 Load-Store 架构设计哲学。
*   **Card 2**: 物理寄存器状态与 ABI 调用规范：通用寄存器（x0-x31）硬件零寄存器机制、ABI 寄存器别名（如 ra, sp, gp, tp, t0-t6, a0-a7, s0-s11）与参数传递规则。
*   **Card 3**: 经典五级流水线架构：取指 (IF)、译码 (ID)、执行 (EX)、访存 (MEM)、写回 (WB) 级硬件结构，CPU 周期（Clock Cycles）物理推进模型。
*   **Card 4**: 控制与状态寄存器 (CSRs) 访问：CSR 专属地址空间、特权模式读写控制指令（csrrw, csrrs, csrrc, csrrwi, csrrsi, csrrci）与物理隔离。
*   **Card 5**: 多特权模式切换与陷入 (Trap) 硬件处理路径：User (U)、Supervisor (S)、Machine (M) 模式，xepc, xcause, xtvec 寄存器流转与陷入返回（mret/sret）过程。

### M2: 流水线冒险、旁路分支与异常挂起 (Cards 6-10)
*   **Card 6**: 流水线冒险 (Hazards) 与硬件锁步：结构冒险（同一硬件冲突）、数据冒险（RAW, WAR, WAW 相关）成因，硬件插入气泡（Bubbles/Stalls）对执行流的锁步挂起。
*   **Card 7**: 数据旁路转发 (Bypass Forwarding) 机制：ALU 到 ALU、MEM 到 ALU、ALU 到 MEM 的多路选择器前推流，解决 RAW 冒险的硬件实现。
*   **Card 8**: 控制冒险 (Control Hazards) 与流水线冲刷：分支指令条件跳转冲突开销、指令清空 (Pipeline Flush) 与分支延迟槽机制。
*   **Card 9**: 分支预测器 (Branch Predictors) 硬件实现：分支目标缓冲 (BTB) 地址记录、分支历史表 (BHT) 两比特饱和计数状态机（00-11）、gshare 全局关联分支预测算法。
*   **Card 10**: 精确异常 (Precise Exception) 硬件捕获：中断/异常发生时被中断指令及后续流的撤销、精确现场记录（mepc/mcause）与恢复。

### M3: 乱序执行 (OoO) 处理器微架构与超标量流水线 (Cards 11-15)
*   **Card 11**: 乱序执行 (Out-of-Order Execution) 本质：Tomasulo 算法实现原理、指令保留站 (Reservation Stations) 硬件结构与通用数据总线 (CDB) 广播唤醒。
*   **Card 12**: 寄存器重命名 (Register Renaming) 机制：消除反相关 (WAR) 与输出相关 (WAW) 的伪相关，物理寄存器堆 (PRF) 到架构寄存器映射表 (Map Table) 的空闲列表（Free List）分配。
*   **Card 13**: 指令发射与动态调度队列：发射队列 (Issue Queue) 缓冲，唤醒 (Wakeup) 依赖解析与选择 (Select) 指令分发算子执行逻辑。
*   **Card 14**: 重排序缓冲区 (Reorder Buffer, ROB)：乱序执行与顺序提交 (In-order Commit) 的物理流转，支持精确异常与分支预测失败投机回滚。
*   **Card 15**: 访存队列冲突消解：存储缓冲区 (Store Buffer) 写入暂存、加载队列 (Load Queue) 乱序读取，写转读前推 (Store-to-Load Forwarding) 冲突解决。

### M4: 多级缓存一致性协议与硬件维护 (Cards 16-19)
*   **Card 16**: 多级高速缓存 (L1/L2 Cache) 硬件结构：直接映射、组相联（Set-associative）寻址、Cache Line（Tag, Data, Valid/Dirty bit）构成与 LRU/LFU 替换策略。
*   **Card 17**: 高速缓存写策略与一致性瓶颈：写回 (Write-Back) 脏数据延迟落盘、写直达 (Write-Through) 实时更新，多核心私有 L1 缓存的数据冲突痛点。
*   **Card 18**: 经典 MESI 缓存一致性状态机：Modified (修改)、Exclusive (独占)、Shared (共享)、Invalid (失效) 状态转换、总线监听 (Bus Snooping) 与广播失效信号。
*   **Card 19**: 硬件地址翻译与 TLB 缓存未命中：多级页表机制（SV39/SV48）、物理内存保护 (PMP) 检查、页表遍历器 (Page Table Walker) 硬件重填。

### M5: 内存一致性模型 (Memory Consistency) 与同步原语 (Cards 20-24)
*   **Card 20**: 内存一致性模型 (Memory Consistency) 物理意义：顺序一致性 (Sequential Consistency) 理想多核内存流、硬件缓存与写缓冲区引发的重排违背。
*   **Card 21**: RISC-V 弱内存模型 (RVWMO)：Read-Read, Read-Write, Write-Write, Write-Read 四种硬件访存指令乱序重排规则（RVWMO 宽松语义）。
*   **Card 22**: 栅栏同步指令 (FENCE / FENCE.I)：I-Cache 与 D-Cache 一致性同步（FENCE.I）、FENCE 指令对 PI/PO/PR/PW (前趋/后继读写) 屏障的全局排序。
*   **Card 23**: 原子内存操作指令 (A 扩展 AMO)：`amoadd`, `amoswap`, `amoand` 等指令硬件实现，总线锁定与 Cache 一致性锁锁定机制。
*   **Card 24**: 链接加载/条件存储 (LR/SC) 原语：`lr.w` (Load-Reserved) 硬件监视器 (Reservation Set) 注册、`sc.w` (Store-Conditional) 检测原子写入判定。

### M6: 向量扩展 (RVV) 与处理器性能调试 (Cards 25-28)
*   **Card 25**: 向量扩展 (Vector Extension) 物理本质：SIMD 硬件并发，RVV 向量长度无关 (Vector Length Agnostic, VLA) 设计哲学，向量寄存器（v0-v31）物理重组。
*   **Card 26**: 向量状态配置与执行模式：`vsetvli` 配置指令，元素宽度 (SEW) 与向量倍率因子 (LMUL) 硬件参数的动态调配机理。
*   **Card 27**: 硬件性能计数器与调试接口：硬件性能监视器 (HPM)、指令与周期计数器（cycle, time, instret）读取，JTAG 与 Boundary Scan 调试规范。
*   **Card 28**: 虚拟机与指令集模拟器原理：QEMU 动态翻译 (TCB/TB) 原理，Spike 经典 RISC-V 金丝雀参考仿真器工作流。

---

## 2. 莫兰迪色系设计配置 (Morandi Palette)
为了实现视觉设计上的风格化与一致性，cheatsheet 与 HTML 知识大图采用以下精选莫兰迪色系：

*   **M1**: 莫兰迪蓝灰色 (`#4B5F7A` / Slate Blue) - 指令集架构 (ISA) 公理、特权级与流水线基础
*   **M2**: 莫兰迪绿灰色 (`#6B8272` / Muted Sage) - 流水线冒险、旁路分支与异常挂起
*   **M3**: 莫兰迪茶红色 (`#9C6666` / Tea Red) - 乱序执行 (OoO) 处理器微架构与超标量流水线
*   **M4**: 莫兰迪铁灰色 (`#7A7A7A` / Iron Grey) - 多级缓存一致性协议与硬件维护
*   **M5**: 莫兰迪金黄色 (`#9A825A` / Dusty Gold) - 内存一致性模型与同步原语
*   **M6**: 莫兰迪紫灰色 (`#755B77` / Muted Grape) - 向量扩展 (RVV) 与处理器性能调试
*   **L0-L2 梯级底色**: 暗夜黑 (`#0B0F19` / Dark Charcoal) 与 炭灰色 (`#111827` / Carbon)

---

## 3. 梯级架构层级 (J-Ladder)
*   **L0 (一句话本质)**: RISC-V 体系架构的本质是通过模块化、规整化的指令集定义，在微架构上支持从简捷的顺序流水线旁路转发到高度复杂的乱序执行（Tomasulo 算法、ROB 顺序提交）处理器，并在多核硬件上通过监听协议（MESI）与弱内存模型（RVWMO）屏障指令保障共享数据一致性。
*   **L1 (四句话逻辑)**:
    1.  **规整规约与特权控制**: RISC-V 提供精简的通用寄存器和受 CSR 控制的多特权级陷入（Trap）处理通道，支持指令和异常硬件级精确捕获。
    2.  **锁步转发与分支猜测**: 五级流水线通过数据旁路前推（Bypass Forwarding）消解读写冒险，结合动态分支预测器（BHT/BTB）缩减控制流跳转开销。
    3.  **重命名重排与顺序提交**: 乱序执行利用 Tomasulo 保留站和寄存器重命名打破假相关，通过重排序缓冲区（ROB）在提交阶段恢复程序执行流的顺序性。
    4.  **缓存监听与弱序一致**: 硬件通过 MESI 协议总线监听维护多级 Cache 状态，RVWMO 弱序模型配合 FENCE 栅栏与 LR/SC 原语完成低损耗多核强同步。
*   **L2 (核心数据流转拓扑)**:
    *   `PC fetch` ➜ `Instruction Cache read` ➜ `Decode (ID)` ➜ `Register Rename (Map Table lookup)` ➜ `Dispatch to Reservation Station (RS)` ➜ `Wakeup & Select` ➜ `Read operands from Physical Register File (PRF)` ➜ `ALU Execute` ➜ `Data Hazard? No, Bypass Forwarding activated` ➜ `Memory Load/Store Buffer queue` ➜ `L1 Data Cache look up` ➜ `Cache Miss? Yes ➜ TLB virtual to physical walk` ➜ `MESI Bus Request` ➜ `State Invalid ➜ Broadcast Read Invalid` ➜ `Fill L1 Cache Line ➜ State Modified` ➜ `Store-to-Load Forwarding` ➜ `Writeback result to ROB` ➜ `ROB in-order commit` ➜ `Commit to Architectural Register File`.
