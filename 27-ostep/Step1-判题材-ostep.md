# OSTEP Internals Scoping & Card Planning Document

This document defines the scoping and plan for candidate **OSTEP / Operating Systems: Three Easy Pieces** (Operating Systems Internals). It contains the 6 modules, L0~L2 ladders, and plans the 28 cards.

---

## 1. 题材判研 (Subject Evaluation)

OSTEP is the world-renowned textbook and resource on modern operating systems. It covers three main pieces: Virtualization (CPU/Memory), Concurrency, and Persistence. It is dense with architectural mechanisms like TLB translation, multi-level paging, thread scheduling (MLFQ, CFS), locks/semaphores, RAID levels, logging/journaling, and NFS.

---

## 2. L0~L2 核心本质与高层逻辑 (L0~L2 Core Ladders)

*   **L0 一句话本质**：在硬件特权级保护与时钟中断强占的边界下，通过虚拟地址空间多级页表映射与 TLB 高速缓存隐藏物理内存，利用互斥锁与条件变量解决多线程并发共享竞态，并依靠预写日志与数据一致性扫描保障持久化存储介质安全的数据管理系统。
*   **L1 四句话逻辑**：
    1.  **CPU 虚拟化与进程调度** (M1) 依赖时钟中断进行上下文切换（Context Switch），通过多级反馈队列（MLFQ）及完全公平调度（CFS）平摊 CPU 时间。
    2.  **内存虚拟化与地址转换** (M2) 在 MMU 硬件协助下，通过两级/多级页表将进程虚拟空间转换为物理内存，利用 TLB 缓存页表项（PTE）加速访问。
    3.  **多线程并发与同步锁** (M3) 借助硬件原子指令（Test-and-Set、Compare-and-Swap）构建锁（Lock）与条件变量，防范死锁并维护因果并发序。
    4.  **存储 I/O 与文件一致性** (M4-M5) 借由日志系统（Journaling）及写屏障（Barrier）防御断电崩溃，在 RAID 纠错分片及分布式 NFS 体系下提供一致的 VFS 抽象。

---

## 3. 6大模块与 28 张卡片规划 (6 Modules & 28 Cards Plan)

### M1：CPU 虚拟化与进程调度 (CPU Virtualization & Scheduling)
*   **Card 1. 进程状态机与上下文切换**：内核如何利用 `struct proc` 维护寄存器状态并执行 `switch()` 切换。
*   **Card 2. 受限直接执行 (LDE) 特权转换**：用户态（User Mode）通过 trap 陷入内核态（Kernel Mode）的特权隔离机制。
*   **Card 3. 多级反馈队列 (MLFQ) 调度算法**：如何通过优先级衰减、时间片消耗及 Periodic Priority Boost 防范饥饿。
*   **Card 4. 比例份额与完全公平调度 (CFS)**：彩票调度（Lottery Scheduling）与 Linux CFS 中 `vruntime` 计算与红黑树排序。

### M2：内存虚拟化与分段分页 (Memory Virtualization & Paging)
*   **Card 5. 地址转换 (Address Translation) 物理机制**：基址寄存器（Base）与界限寄存器（Bounds）在硬件级的地址翻译。
*   **Card 6. 空闲空间管理与分配策略**：内存分配器（Allocator）在空闲链表上的 First-Fit, Best-Fit, Worst-Fit 算法。
*   **Card 7. 分页 (Paging) 与页表物理结构**：线性页表、页目录项（PDE）、页表项（PTE）物理字段定义。
*   **Card 8. 旁路转换缓冲 (TLB) 硬件缓存**：TLB 缓存行命中与失效（TLB Miss）硬件/软件处理逻辑及 ASID 进程标识。
*   **Card 9. 多级页表与物理内存节省**：多级页表如何通过置空未映射页表物理页，从而精简页表本身占用的开销。

### M3：并发控制与同步原语 (Concurrency & Locks)
*   **Card 10. 互斥锁 (Mutex) 与硬件原子指令**：Test-and-Set, Compare-and-Swap 以及自旋锁（Spinlock）的物理边界。
*   **Card 11. 条件变量 (Condition Variables) 与信号量**：`pthread_cond_wait`/`signal` 生产者消费者模型与 Semaphore 计数同步。
*   **Card 12. 锁竞争、睡眠队列与 Futex 机制**：如何使用 Linux futex 睡眠队列避免自旋导致的高 CPU 空耗。
*   **Card 13. 并发死锁 (Deadlock) 四大必要条件**：互斥、占有并等待、非抢占、循环等待，以及死锁预防与避免（银行家算法）。

### M4：存储设备与文件系统布局 (Storage & File Systems)
*   **Card 14. 磁盘 I/O 驱动与中断机制**：DMA（直接内存访问）、I/O 寄存器轮询与硬件中断数据传输边界。
*   **Card 15. RAID 冗余阵列物理分片与纠错**：RAID 0 (条带)、RAID 1 (镜像)、RAID 4/5 (奇偶校验) 读写吞吐与容灾折衷。
*   **Card 16. VSFS 文件系统物理布局与数据块**：超级块（Superblock）、Inode 位图、Data 位图、Inode 表及 Data Blocks 物理划分。
*   **Card 17. 文件路径解析与目录检索数据流**：打开 `/foo/bar.txt` 过程中，内核遍历目录 Block 并读取 Inode 的物理步骤。

### M5：文件系统一致性与日志机制 (File System Consistency & Journaling)
*   **Card 18. 断电崩溃 (Crash) 与一致性异常**：数据块、Inode、位图写入先后不一导致的物理不一致危害。
*   **Card 19. FSCK 文件系统一致性扫描修复**：FSCK 如何在重启后扫描 Inode 引用和位图并强制同步（效率低下问题）。
*   **Card 20. Write-Ahead Journaling 预写日志物理流程**：日志超级块、Journal Write、Commit Block 及 Checkpoint 四步法。
*   **Card 21. 数据日志 (Data) 与元数据日志 (Metadata) 模式**：Metadata Journaling 仅记录元数据以优化写放大。
*   **Card 22. 写屏障 (Write Barriers) 与磁盘缓存重排**：如何通过硬件 Barrier 保证日志在数据落盘前强制刷盘。
*   **Card 23. 日志结构文件系统 (LFS) 顺序追加**：LFS 如何将所有修改聚合成 segment 顺序写入，以适配机械盘极速写入。

### M6：分布式系统与高阶存储 (Distributed Systems & Advanced Storage)
*   **Card 24. 网络文件系统 (NFS) 协议与无状态设计**：NFS v2 协议中 File Handle 与服务器端无状态（Stateless）高容错逻辑。
*   **Card 25. NFS 客户端缓存一致性与超时校验**：Attribute Cache 与 Close-to-Open 一致性同步的时序妥协。
*   **Card 26. 闪存固态硬盘 (SSD) 物理布局**：SSD 页（Page）读写与块（Block）擦除（Erase）的物理特性约束。
*   **Card 27. 闪存翻译层 (FTL) 与垃圾回收**：逻辑地址映射、段级磨损均衡（Wear Leveling）与写放大因子（WAF）优化。
*   **Card 28. 虚拟文件系统 (VFS) 统一抽象层**：VFS 如何使用 `file_operations` 接口屏蔽 EXT4、NFS、sysfs 的底层差异。
