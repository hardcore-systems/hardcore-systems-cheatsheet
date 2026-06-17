# Step1-判题材-cuda_programming.md: GPGPU 并行计算与 CUDA 编程模型设计大纲

本审计项目聚焦于高性能并行计算的核心基础设施——NVIDIA CUDA。我们将通过 28 张核心知识卡片，深度剖析软硬件拓扑架构，设计 L0-L2 的 J-Ladder（阶梯递进）理论模型，并规划双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

本速查海报分为六大核心主题模块，采用精选的莫兰迪配色体系进行区域划分与视觉区分：

*   **M1: CUDA 编程模型与网格拓扑** (Cards 1–5) - `#4B5F7A` (Slate Blue)
    - 聚焦网格（Grid）、线程块（Block）和线程（Thread）的软件多维映射，以及最基础的核函数（Kernel）启动机制与资源限制。
*   **M2: GPU 硬件架构与 Warp 调度机理** (Cards 6–10) - `#6B8272` (Muted Sage)
    - 剖析流多处理器（SM）的物理构成、SIMT 架构核心、Warp 线程束物理调度、Warp Divergence（分支分歧）及延迟隐藏。
*   **M3: CUDA 存储分级与合并访存优化** (Cards 11–15) - `#9C6666` (Tea Red)
    - 详解 Global Memory 物理合并访存（Coalesced）、Shared Memory 的 32-Bank 组织、Bank 冲突避免、Constant/Texture 缓存及寄存器溢出（Spill）。
*   **M4: 块内同步、协作组与原子操作** (Cards 16–19) - `#7A7A7A` (Iron Grey)
    - 剖析屏障同步（__syncthreads）、Warp Shuffle（线程束洗牌原语）、协作组（Cooperative Groups）的组内集体通信以及硬件原子操作（Atomic）。
*   **M5: 异步流、异构重叠与统一内存** (Cards 20–24) - `#9A825A` (Dusty Gold)
    - 深入分析非阻塞流（Stream）并发、流水线式计算与访存重叠（Overlap）、统一内存（UM）缺页异常迁移及 Page-locked Pinned Memory 数据高速拷贝。
*   **M6: 张量核心 (Tensor Cores) 与高频性能算子优化** (Cards 25–28) - `#755B77` (Muted Grape)
    - 覆盖硬件 Tensor Cores 的矩阵乘累加（MMA）原语、动态并行（Dynamic Parallelism）、高频经典算法算子（Reduce/Scan/GEMM）优化与 Nsight 性能瓶颈诊断。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    GPU 并行计算的本质是通过超大规模轻量级线程（Threads）在三维网格（Grid-Block-Thread）拓扑下，利用 SIMT（单指令多线程）硬件架构与高宽带显存（HBM/GDDR）实现高吞吐的数据并行吞吐与张量计算加速。
*   **L1 四句话逻辑**：
    1. **网格拓扑映射**：软件逻辑层将计算任务抽象为 Grid 和 Block，由千万级轻量级线程在三维坐标系下通过 `threadIdx` 与 `blockIdx` 确定物理数据的一维/二维映射。
    2. **SIMT 硬件调度**：硬件执行层将线程块 Block 映射到流多处理器 SM（Streaming Multiprocessor），以 32 个线程的线程束 Warp 为基本物理调度单元在 SIMT 架构下执行。
    3. **高吞吐存储层级**：构建由线程私有寄存器、Block 共享内存（Shared Memory）、全局显存（Global Memory）构成的多级高吞吐存储层次，并使用合并访存（Coalesced Memory Access）与共享内存无 Bank 冲突（Bank Conflict）来最大化有效带宽。
    4. **异步流水与并发**：通过 `__syncthreads()` 执行 Block 内线程同步，利用流（Stream）与图（Graphs）技术实现异构 CPU-GPU 数据拷贝与计算在时间上的重叠（Overlap）并发。
*   **L2 核心数据流转拓扑**：
    `CPU 主存数据` ➜ (PCIe / NVLink 拷贝 HtoD) ➜ `GPU 全局显存 (Global Memory)` ➜ (网格加载 Grid Launch) ➜ `SM 缓存 / 寄存器` ➜ (Shared Memory 协同块内 Reduction) ➜ `Warp 寄存器进行算术计算 (FMA / Tensor Core)` ➜ (Shared Memory / Global Memory 写回) ➜ (PCIe / NVLink 拷贝 DtoH) ➜ `CPU 主存结果`

---

## 🗂️ 28 张核心知识卡片大纲

1.  **Grid-Block-Thread 层次模型**：软件逻辑对数据的多维划分，三维索引定位机制。
2.  **Kernel 设备端运行生命周期**：主机（Host）控制与设备（Device）异步核函数启动机理。
3.  **流多处理器 (SM) 物理微架构**：SP (CUDA Cores)、DP Cores、SFU (特殊函数单元) 与寄存器堆 (Register File)。
4.  **SIMT 架构与 Warp 调度逻辑**：Warp（32线程）作为最小发射和硬件调度基本单元。
5.  **Warp Divergence (分支分歧) 规避**：分支条件不同导致硬件指令串行化执行与掩码寄存器（Active Mask）。
6.  **GPU 延迟隐藏与占用率 (Occupancy)**：Active Warps 硬件上下文零开销高速切换，规避流水线气泡。
7.  **GPU 物理存储分级层次**：寄存器、局部内存（Local）、共享内存（Shared）、全局显存（Global）的带宽与延迟差异。
8.  **Global Memory 合并访存 (Coalesced Access)**：内存对齐、合并交易（Memory Transactions）及吞吐瓶颈。
9.  **Shared Memory 与 32-Bank 组织**：片上超高速 SRAM 寻址物理机制，Bank 划分与循环映射。
10. **Bank Conflict (银行冲突) 与广播**：Stride 跨步访问导致的串行化延迟、1-way/n-way 冲突及无冲突广播（Broadcast）。
11. **寄存器溢出 (Register Spilling) 的代价**：编译期寄存器压力超过硬件限制导致的 Local Memory (隐式显存) 慢速退化。
12. **Constant/Texture Memory 常量只读缓存**：Warp 级单数据广播访问与二维纹理滤波硬件加速。
13. **L1 / L2 硬件级缓存调优**：L1 与 Shared Memory 的大小划分配置及统一 L2 缓存访问模式。
14. **统一内存 (Unified Memory, UM) 页面迁移**：CPU 与 GPU 间基于硬件缺页中断的 Page Migration 机制。
15. **屏障同步与内存栅栏 (Barrier & Fence)**：`__syncthreads()` 的执行流同步限制与 `__threadfence()` 的存储可见性保证。
16. **协作组 (Cooperative Groups) 通信机制**：跨 Block 线程组网同步，构建网格级（Grid-level）协同原语。
17. **Warp Shuffle (洗牌) 高速片上交换**：Warp 内部寄存器直接数据交换（`__shfl_sync` / `__shfl_xor_sync`）脱离共享内存依赖。
18. **GPU 原子操作与硬件 ALU 支持**：原子加减（AtomicAdd）、CAS（比较并交换）以及 L2 缓存控制器处的硬件原子支持。
19. **CUDA Streams (并发流) 异步控制**：默认流（Stream 0）的阻塞特性与非阻塞自定义多流（Non-blocking streams）。
20. **CPU-GPU 异步操作与 Event 监控**：核函数异步非阻塞返回与 `cudaEventRecord` / `cudaEventSynchronize` 时序控制。
21. **CUDA Graphs 静态执行拓扑**：利用计算图在 Host 端捕获 Kernel 并一键提交，彻底消除 CPU 端的核函数启动 Overhead。
22. **Page-locked (Pinned) 内存与 DMA 传输**：固定物理内存（Pinned Host Memory）提升主机-设备间 PCIe 总线传输带宽。
23. **Multi-GPU P2P 直接互联与 NVLink**：多显卡直接 PCIe 通信与 NVLink 总线高带宽点对点网格互联（Peer-to-Peer）。
24. **Tensor Cores 深度学习硬加速**：混精度矩阵乘累加（16x16x16 MMA）硬件指令架构及硬件算术逻辑。
25. **CUDA 动态并行 (Dynamic Parallelism)**：设备端 Kernel 动态自我嵌套启动，支持稀疏网格与细粒度自适应计算。
26. **GPU 性能优化黄金法则 (Optimization Matrix)**：访存受限（Memory-bound）与计算受限（Compute-bound）性能转换指南。
27. **GPGPU 经典并行算子设计模式**：Reduce (规约)、Scan (前缀和)、Histogram (直方图) 的无冲突设计。
28. **Nsight Compute & Systems 诊断体系**：通过 Roofline 模型分析延迟、利用率（Occupancy）与硬件 stall 原因。
