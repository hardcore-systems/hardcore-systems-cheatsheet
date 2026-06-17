# Go Runtime 核心原理高密卡片系统题材判定

## 1. 核心模块与 28 张卡片大纲

### M1: Goroutine GMP 调度器模型与网络轮询器 (Cards 1-5)
*   **Card 1**: G、M、P 核心结构体物理定义与角色分工 (`runtime.g`、`runtime.m`、`runtime.p`)。
*   **Card 2**: 调度主循环的核心决策链 (`schedule()`, `findrunnable()`, 本地队列 vs 全局队列 vs 轮询器)。
*   **Card 3**: 任务工作窃取机制 (Work Stealing) 与空闲线程休眠自旋判定。
*   **Card 4**: 网络轮询器 (Netpoller) 基于 epoll/kqueue 的非阻塞网络 I/O 汇流机制。
*   **Card 5**: Sysmon 监控线程的协作/非协作抢占调度机制 (`sysmon` 扫描周期与 `SIGURG` 信号抢占)。

### M2: 堆内存分配机制与逃逸分析 (Cards 6-10)
*   **Card 6**: 内存分配三层物理架构 (`mcache`、`mcentral`、`mheap`) 与本地缓存锁消除。
*   **Card 7**: `mspan` 物理内存块划分、Size Classes 细粒度尺寸设计与对象分配流程。
*   **Card 8**: 微小对象 Tiny Allocator 压缩分配存储与空间利用率优化。
*   **Card 9**: 逃逸分析 (Escape Analysis) 静态编译器逃逸场景判定与标量生命周期分析。
*   **Card 10**: 协程物理栈伸缩机制（分段栈的历史局限、连续栈拷贝 `copystack` 机制）。

### M3: 并发三色标记 GC 引擎 (Cards 11-15)
*   **Card 11**: 三色标记状态机（黑-已扫描、灰-待扫描、白-待回收）与 GC 四阶段状态转换流程。
*   **Card 12**: Dijkstra 插入写屏障与 Yuasa 删除写屏障结合的 Go 混合写屏障 (Hybrid Write Barrier) 底层保障。
*   **Card 13**: 用户辅助标记机制 (GC Assist) 对内存分配速率的自适应约束。
*   **Card 14**: 并发清理阶段 (Sweep) 的惰性扫描 (`lazy sweep`) 与内存页向 OS 释放。
*   **Card 15**: GC Pacer 自反馈调优触发水位线动态反馈计算公式。

### M4: Channels 管道与同步原语 (Cards 16-20)
*   **Card 16**: Channel 底层 `hchan` 物理布局、环形缓冲区控制及锁争抢。
*   **Card 17**: Channel 阻塞读写的等待队列管理 (`sudog` 重定向、无锁直接拷贝优化)。
*   **Card 18**: Select 多路复用非阻塞模式的编译展开与随机洗牌锁表 (`selectgo`)。
*   **Card 19**: Mutex 互斥锁状态位映射、正常模式自旋与饥饿模式强制唤醒的判定模型。
*   **Card 20**: WaitGroup（计数与信号量池）与 Once（原子锁与双重校验）物理同步底座。

### M5: Maps 扩容与接口动态派发 (Cards 21-24)
*   **Card 21**: Map 底层 `hmap`、`bmap` (bucket) 内存排布、哈希冲突拉链与溢出桶分配。
*   **Card 22**: Map 渐进式扩容模式：翻倍扩容与等量扩容的 evacuate 槽位渐进式迁移机制。
*   **Card 23**: 空接口 `eface` 与非空接口 `iface` 底层物理结构区别 (`itab` 与类型元数据描述)。
*   **Card 24**: 接口动态分发虚函数调用开销与编译器去虚拟化 (Devirtualization) 调优。

### M6: Defer/Panic 与 系统接口/Cgo (Cards 25-28)
*   **Card 25**: Defer 的三种编译编译路径（堆上分配、栈上分配、开放编码 Open-coded Defer 零分配）。
*   **Card 26**: Panic 递归嵌套调用链 (`_panic` 链表与 `_defer` 链表的执行交互)。
*   **Card 27**: 协作式系统调用包装 (Syscall)：M 从 P 的解绑、脱离运行期与进入/退出机制 (`entersyscall` / `exitsyscall`)。
*   **Card 28**: Cgo 调用的性能屏障、协程栈向 C 语言物理线程栈的双向切换与运行时开销。

---

## 2. 莫兰迪色系设计配置 (Morandi Palette)
为了实现视觉设计上的风格化与一致性，Go Runtime 的 cheatsheet 与 HTML 知识大图采用以下精选莫兰迪色系：

*   **M1**: 莫兰迪蓝灰色 (`#4B5F7A` / Slate Blue) - GMP 调度与网络
*   **M2**: 莫兰迪绿灰色 (`#6B8272` / Muted Sage) - 内存分配与栈
*   **M3**: 莫兰迪铁灰色 (`#7A7A7A` / Iron Grey) - 并发三色 GC
*   **M4**: 莫兰迪茶红色 (`#9C6666` / Tea Red) - Channel 与同步原语
*   **M5**: 莫兰迪金黄色 (`#9A825A` / Dusty Gold) - Map 扩容与接口
*   **M6**: 莫兰迪紫灰色 (`#755B77` / Muted Grape) - Defer/Panic 与 Cgo
*   **L0-L2 梯级底色**: 暗夜黑 (`#0B0F19` / Dark Charcoal) 与 炭灰色 (`#111827` / Carbon)

---

## 3. 梯级架构层级 (J-Ladder)
*   **L0 (一句话本质)**: Go Runtime 的本质是通过 GMP 协程调度模型、非分代并发三色标记清除 GC、以及 tcmalloc 衍生内存分配器，在用户态低成本支持数百万高并发任务周转的自托管轻量级语言运行时。
*   **L1 (四句话逻辑)**:
    1.  **GMP 队列与工作窃取**: 用户态协程 (G) 复用物理线程 (M) 进行执行，通过逻辑处理器 (P) 绑定和 Work Stealing 算法消除单队列争抢瓶颈。
    2.  **细粒度分配与逃逸栈缩**: 逃逸分析将局部对象静态划分，不逃逸对象分配于动态伸缩的协程连续物理栈，逃逸对象由 mcache 三层架构进行快速无锁堆分配。
    3.  **三色写障与用户协同**: 垃圾回收使用并发三色标记，通过混合写屏障强行置灰变动引用以确保标记正确性，并通过 GC Assist 让分配过快的协程参与标记以防 OOM。
    4.  **管道通道与接口去虚**: 管道以原子锁和 sudog 阻塞链表保证同步安全；动态接口通过 iface/eface 维护反射，通过编译器 inline 与去虚拟化最大化减少间接调用开销。
*   **L2 (核心数据流转拓扑)**:
    *   `Goroutine Created` ➜ `P Local Runqueue` ➜ `Work Stealing if empty` ➜ `GMP Scheduler (schedule)` ➜ `M executes G` ➜ `Channel Send (hchan locked)` ➜ `Buffer full` ➜ `G parks on sudog` ➜ `M releases P to new G` ➜ `Network I/O Ready` ➜ `Netpoller detects` ➜ `Wake up G on Netpoll` ➜ `G returns to P runq` ➜ `Object Allocation` ➜ `Escape Analysis check` ➜ `Escaped? Yes ➜ Heap Alloc (mcache)` ➜ `Tricolor GC starts` ➜ `Roots scanned (Grey)` ➜ `Tricolor Mark` ➜ `User writes pointer` ➜ `Hybrid Write Barrier fires` ➜ `Force grey object` ➜ `GC sweeps white objects` ➜ `Memory Page returns to OS`.
