# Step1-判题材-rocket_chip.md: Rocket Chip RISC-V 参数化芯片生成器与 TileLink 一致性总线审计大纲

本审计项目聚焦于 RISC-V 社区最著名的 SoC 生成框架 `Rocket Chip`。我们将通过 28 张高密度核心知识卡片，深度解构 Chisel 硬件描述语言与参数配置系统、Rocket Core 5级经典处理器流水线、物理内存保护（PMP）与页表转换（MMU/TLB）、数据与指令高速缓存的物理设计、TileLink 总线协议（A-E 五通道模型）与多核缓存一致性（MSI 协议）维护、动态生成 MMIO/中断控制器（PLIC）、FIRRTL 硬件编译器降级、以及硬件级调试模块（JTAG/DM）和物理综合后端。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: 参数化生成器与 Chisel 核心** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - Chisel 硬件语言建模、Cake 蛋糕模式配置系统、Rocket Core 处理器流水线、FPU 与 MMU 虚拟转换、指令高速缓存（I$）、非阻塞数据高速缓存（D$）。
*   **M2: TileLink 总线与缓存一致性** (Cards 7–12) - `#6B8272` (Muted Sage)
    - TileLink 五通道协议、MSI 一致性协议维护、L1-L2 核心互连总线、总线协议适配与变换、交叉开关与仲裁逻辑、外设内存映射（MMIO）控制器。
*   **M3: SoC 集成与系统调试** (Cards 13–19) - `#9C6666` (Tea Red)
    - 硬件调试模块（JTAG/DM）、引导加载与 Reset 向量、高级中断控制器（PLIC）、FIRRTL 中端编译器、Verilog RTL 仿真发射、时钟与电源管理、物理后端 SRAM 宏映射。
*   **M4: 核心复合扩展机制** (Cards 20–24) - `#7A7A7A` (Iron Grey)
    - RoCC 自定义协处理器接口、硬件性能监控计数器、PMP 物理内存保护过滤、多核广播一致性集线器、直接内存存取（DMA）总线代理。
*   **M5: 动态配置与仿真** (Cards 25–28) - `#9A825A` (Dusty Gold)
    - 生成器编译入口（Main）、参数化 Cache 特性、仿真环境与 Harness 绑定、多 Tile 异构配置。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    Rocket Chip 的本质是一个基于 Scala/Chisel 的参数化 SoC 芯片生成器，它在编译期通过外交关系（Diplomacy）协议协商各模块的物理参数，生成支持 TileLink 缓存一致性的多核 RISC-V RTL 代码及片上系统结构。
*   **L1 四句话逻辑**：
    1. **外交驱动参数协商**：在生成电路前，通过“外交学”（Diplomacy）在有向无环图上自动传递和协商总线位宽、地址映射及中断连接等物理参数。
    2. **MSI 双端一致性维护**：采用 TileLink 协议中 A (Request), B (Probe), C (Release), D (Response), E (Grant) 5个独立物理通道，支持内核与 L2 Cache 之间的多向无锁缓存行一致性转换。
    3. **非阻塞缓存与 MSHR 合并**：数据 Cache 使用未命中状态持有寄存器（MSHR）合并相同地址的访存未命中，保障流水线在发生 Cache Miss 时仍可继续向后执行非冲突指令。
    4. **硬隔离保护与协处理扩展**：在流水线最末端硬编 PMP（物理内存保护）模块过滤非法物理地址，同时提供 RoCC 接口以插桩方式无缝扩展专用加速器（如 ML 模块）。
*   **L2 核心数据流转拓扑**：
    `Chisel 源代码` ➜ `Scala 运行实例化 (Diplomacy 参数协商)` ➜ `AST 生成` ➜ `FIRRTL 降级编译 (RTL 优化)` ➜ `Emit Verilog RTL 源码` ➜ `SoC 重启` ➜ `CPU 发起访存 (D$)` ➜ `Cache Miss` ➜ `TileLink 通道 A 发送 Read Request` ➜ `总线仲裁接入 L2` ➜ `L2 回应 Grant (通道 D)` ➜ `CPU 更新 MSHR 并恢复流水线`

---

## 🗂️ 28 张核心知识卡片大纲
1. Chisel 硬件描述语言：在 Scala 面向对象框架下进行高阶硬件电路生成。
2. 蛋糕模式与配置参数化：利用 Scala Traits 的 Cake 模式及 Parameters 上下文传递硬件参数。
3. Rocket Core 五级流水线：顺序执行、IF/ID/EX/MEM/WB 级流转与数据旁路。
4. FPU 与 MMU 虚拟转换：IEEE-754 浮点硬加速及 SV39 级三级页表硬件 Walk 与 TLB。
5. 指令 Cache 与 ITim 物理结构：单端口 RAM、非阻塞指令预取与指令紧凑紧邻存储（ITim）。
6. 数据 Cache 与 MSHR 结构：两级元数据校验、未命中状态寄存器（MSHR）非阻塞设计。
7. TileLink 总线五通道模型：A (请求)、B (探测)、C (释放)、D (响应)、E (授权) 物理通道解耦。
8. 缓存一致性 MSI 协议转换：宿主内核 Cache MSI 状态机变迁、Probe 和 Release 消息路由。
9. 系统总线（SystemBus）拓扑：Subsystem 内系统总线、外设总线与控制总线互联。
10. 总线适配器与协议转换器：TileLink 到 AXI4/APB 转换、位宽平移与时钟异步跨桥。
11. 交叉开关（Crossbar）与总线仲裁：定优先级与轮询仲裁逻辑、总线防死锁拓扑。
12. 内存映射 I/O (MMIO) 控制器：外设地址空间自动规划、寄存器映射路由生成。
13. 硬件调试模块与 JTAG 协议：Debug Module (DM) 规范、DMI 寄存器读取与运行时暂停控制。
14. BootROM 与 Reset 引导逻辑：CPU 复位跳转地址设计、BootROM 内置固件与设备树（DTB）注入。
15. 平台级中断控制器 PLIC 与 CLINT：中断分发仲裁树、中断优先级屏蔽、定时器中断（CLINT）。
16. FIRRTL 硬件中端编译器：Chisel 汇编代码到 FIRRTL AST、多级优化与死逻辑裁减。
17. Verilog 代码发射与 RTL 仿真：Verilog 源码吐出、Verilator C++ 测试套与测试载荷自动装载。
18. 时钟域划分与低功耗控制：时钟分频器单元、异步复位同步释放器、时钟门控（Clock Gating）。
19. 物理综合后端与 SRAM 宏映射：前端寄存器代码到工艺库 SRAM 硬件宏、物理布线约束。
20. RoCC 自定义协处理器接口：Rocket 协处理器总线、自定义指令编码（Custom0-3）与状态同步。
21. 硬件性能监控计数器 HPM：事件发生选择器、周期/指令/访存命中率硬件计数。
22. 物理内存保护 PMP 过滤：PMP 寄存器配置、地址区间权限硬比对与非法读写中断。
23. 多核广播一致性 Hub：在无 L2 directory 时通过广播代理维护 L1 Cache 之间的一致性。
24. 直接内存存取 DMA 代理：前端总线 DMA 写入、虚实地址转换与总线一致性探测。
25. 生成器入口驱动 Main.scala：编译主程序配置读取、RTL 构建目录输出命令驱动。
26. 参数化 Cache 大小ASSOCIATIVITY配置：参数表配置容量、组相联度自动解构为硬件寻址译码器。
27. C++ 仿真测试平台 Harness 绑定：C++ 物理线缆仿真、Memory 虚拟通道桥接与波形吐出。
28. 异构多核 Subsystem 配置：多 Tile 混合部署（Big-Little 架构）与统一 TileLink 总线接入。
