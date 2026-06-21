# 系统性能调优与诊断高密卡片系统题材判定

## 1. 核心模块与 28 张卡片大纲

### M1: 系统性能分析方法论与指标体系 (Cards 1-5)
*   **Card 1**: 性能定义三要素：延时 (Latency)、吞吐量 (Throughput) 与效率 (Efficiency) 物理关系模型。
*   **Card 2**: 经典性能分析方法论：USE 方法（利用率 Utilization、饱和度 Saturation、错误率 Errors）对系统物理资源的诊断流程。
*   **Card 3**: 经典微服务分析方法论：RED 方法（请求速率 Rate、错误数 Errors、延时 Duration）对应用层性能的评测。
*   **Card 4**: 系统资源与负载特性：队列理论模型、响应时间与并发量对数增长曲线。
*   **Card 5**: 性能观测开销判定：观测器效应 (Observer Effect)、探针插装损耗与采样频率折衷。

### M2: CPU 性能瓶颈、调度延迟与火焰图剖析 (Cards 6-10)
*   **Card 6**: CPU 物理架构观测：硬件性能计数器 (PMC)、指令缓存/数据缓存命中、超标量执行与 NUMA 亲和性。
*   **Card 7**: 操作系统 CPU 调度与负载：负载均值 (Load Average) 定义缺陷、运行队列 (Run Queue) 长度与调度器延迟 (Scheduling Latency) 诊断。
*   **Card 8**: CPU 性能剖析模式：基于时间中断的栈采样 (Stack Sampling) 物理机理。
*   **Card 9**: 火焰图 (Flame Graph) 渲染规范：Y 轴调用栈深度、X 轴函数占 CPU 样本宽度、合并相同调用栈帧的解析机理。
*   **Card 10**: 核心 CPU 诊断工具链：`perf` 采样指令集、`mpstat` 多核状态分解、`pidstat` 线程级上下文切换 (`cswch/s`) 评估。

### M3: 虚拟内存管理、缺页异常与 Swap 性能诊断 (Cards 11-15)
*   **Card 11**: 虚拟内存映射体系：MMU 页表结构、TLB 快表命中的硬件转换以及 Huge Pages 巨页优化。
*   **Card 12**: 内核物理内存分配：伙伴系统 (Buddy System) 物理分裂合并、Slab/Slub 局部对象缓存。
*   **Card 13**: 内存回收与交换机制：页面缓存 (Page Cache) 回收、匿名内存与 Swap 交换区映射、脏页冲刷策略。
*   **Card 14**: 缺页异常 (Page Fault) 底层机理：次要缺页 (Minor Page Fault) 页表填充与主要缺页 (Major Page Fault) 磁盘 I/O 阻塞。
*   **Card 15**: 内存诊断工具链：`vmstat` 核心指标拆解、`free` 统计定义、`/proc/meminfo` 与 OOM Killer 杀进程评分权重算法。

### M4: 磁盘 I/O 调度、文件系统缓存与 VFS 寻址 (Cards 16-20)
*   **Card 16**: 磁盘 I/O 物理特性：顺序 vs 随机 I/O、磁盘排队时间与服务时间对数变化。
*   **Card 17**: 虚拟文件系统 (VFS)：Inode 与 Directory Entry 物理定义、文件系统挂载映射。
*   **Card 18**: 文件系统缓存体系：Page Cache、Buffer Cache 合并机理、读预读 (Read-Ahead) 与写回 (Write-Back) 策略。
*   **Card 19**: 块设备 I/O 调度器：Deadline、BFQ、Kyber、none 等调度算法对吞吐和延时的折衷控制。
*   **Card 20**: 磁盘 I/O 诊断工具链：`iostat` 核心吞吐延时分解、`biolatency` 统计分布、`blktrace` I/O 生命周期追踪。

### M5: 网络协议栈吞吐优化与 Socket 队列缓冲区 (Cards 21-24)
*   **Card 21**: 网络接收与发送路径：硬中断 (Hard IRQ)、软中断 (Soft IRQ / `ksoftirqd`)、NAPI 轮询与网卡 Ring Buffer 队列收发。
*   **Card 22**: 套接字 (Socket) 缓冲区：`rmem`/`wmem` 套接字缓冲区物理大小限制、TCP 发送与接收窗口动态对齐。
*   **Card 23**: TCP 协议栈调优指标：三次握手半连接队列 (syn queue) 与全连接队列 (accept queue) 溢出判定、拥塞控制算法 (Cubic vs BBR)。
*   **Card 24**: 网络性能诊断工具链：`netstat` 协议栈错误计数器、`ss` 实时连接队列查看、`tcpdump` 包捕获开销风险判定。

### M6: 内核跟踪、eBPF 系统诊断与动态探针语法 (Cards 25-28)
*   **Card 25**: 内核可观测性演进：静态跟踪点 (Tracepoints) 稳定性与动态探针 (Kprobes/Uprobes) 零侵入内存修改机制。
*   **Card 26**: eBPF (Extended Berkeley Packet Filter) 物理架构：沙盒虚拟机、寄存器规范、内核安全验证器 (Verifier) 及 BPF Map 高效状态共享。
*   **Card 27**: eBPF 性能调优工具集 (BCC)：`execsnoop` 进程追踪、`opensnoop` 文件监听、`tcplife` 网络诊断的实现原理。
*   **Card 28**: `bpftrace` 动态探针 DSL 语法：探针格式映射（如 `kprobe:sys_write`）、过滤条件与哈希关联数组。

---

## 2. 莫兰迪色系设计配置 (Morandi Palette)
为了实现视觉设计上的风格化与一致性，系统性能调优与诊断的 cheatsheet 与 HTML 知识大图采用以下精选莫兰迪色系：

*   **M1**: 莫兰迪蓝灰色 (`#4B5F7A` / Slate Blue) - 性能分析方法论与指标
*   **M2**: 莫兰迪绿灰色 (`#6B8272` / Muted Sage) - CPU 调度与火焰图
*   **M3**: 莫兰迪铁灰色 (`#7A7A7A` / Iron Grey) - 内存管理与缺页异常
*   **M4**: 莫兰迪茶红色 (`#9C6666` / Tea Red) - 磁盘 I/O 调度与 VFS 缓存
*   **M5**: 莫兰迪金黄色 (`#9A825A` / Dusty Gold) - 网络协议栈与套接字队列
*   **M6**: 莫兰迪紫灰色 (`#755B77` / Muted Grape) - eBPF 追踪与动态探针
*   **L0-L2 梯级底色**: 暗夜黑 (`#0B0F19` / Dark Charcoal) 与 炭灰色 (`#111827` / Carbon)

---

## 3. 梯级架构层级 (J-Ladder)
*   **L0 (一句话本质)**: 系统性能诊断的本质是以 USE 方法为代表的方法论为指引，通过硬件计数器、内核跟踪点及 eBPF 探针对 CPU、内存、存储和网络资源的利用率、饱和度进行无损观测与瓶颈定位。
*   **L1 (四句话逻辑)**:
    1.  **资源方法与指标定位**: 采用 USE 方法自底向上分析物理硬件，结合 RED 方法自顶向下分解应用性能，通过饱和度指标捕捉资源瓶颈。
    2.  **调度火焰与采样剖析**: 通过时间中断定期采样调用栈，在火焰图中合并热点帧进行热点可视化，排查调度延迟与锁竞争气泡。
    3.  **两级缺页与缓存读写**: 物理内存通过伙伴系统和 Slab 细粒度管理，缺页异常区分次要（映射页表）和主要（磁盘 I/O 阻塞），读写依靠 Page Cache 预读与写回机制提速。
    4.  **软硬中断与动态跟踪**: 网络数据收发在硬/软中断与 NAPI 间流转，系统诊断通过 eBPF 安全虚拟机和 Kprobe/Tracepoint 动态无损捕获内核事件。
*   **L2 (核心数据流转拓扑)**:
    *   `Application Request` ➜ `RED Rate check` ➜ `CPU instruction execute` ➜ `PMC cache miss check` ➜ `Run Queue queuing` ➜ `Scheduling Latency` ➜ `Memory access` ➜ `TLB Hit? No` ➜ `Page Table walk` ➜ `Unmapped? Yes` ➜ `Page Fault (Minor)` ➜ `Mapped to Physical Page` ➜ `Disk swap needed? Yes` ➜ `Page Fault (Major)` ➜ `VFS read()` ➜ `Page Cache check` ➜ `Cache Miss ➜ Disk I/O Queue` ➜ `I/O Scheduler (BFQ)` ➜ `Disk read` ➜ `iostat latency detection` ➜ `Packet arrive` ➜ `NIC Ring Buffer` ➜ `Hard IRQ` ➜ `Soft IRQ (ksoftirqd)` ➜ `Socket Buffer Queue (rmem)` ➜ `eBPF Kprobe hook` ➜ `BPF Map update` ➜ `bpftrace collect` ➜ `Flame Graph rendering`.
