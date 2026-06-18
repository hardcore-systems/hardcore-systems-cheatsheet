# linux-insides Internals Scoping & Card Planning Document

This document defines the scoping and plan for candidate **torvalds / linux-insides** (Linux Kernel Initialization & Internals). It contains the 6 modules, L0~L2 ladders, and plans the 28 cards.

---

## 1. 题材判研 (Subject Evaluation)

Linux Kernel internals represents the pinnacle of OS engineering. Analyzing it through `linux-insides` reveals booting sequence (head_64.S, start_kernel), memory zone definitions, Buddy page allocation, SLUB object recycling, the CFS scheduler tree, page cache eviction, and VFS block device drivers.

---

## 2. L0~L2 核心本质与高层逻辑 (L0~L2 Core Ladders)

*   **L0 一句话本质**：从实模式汇编引导与内核解压自举起点出发，通过两级引导页表向保护/长模式平滑过渡，利用伙伴系统的 2 的幂次分割与 SLUB 对象复用机制管理物理内存，借由红黑树 CFS 调度器平摊 CPU 算力，并通过 VFS 文件描述符分发与 Page Cache 延迟回写屏蔽底层块设备差异的单体宏内核。
*   **L1 四句话逻辑**：
    1.  **引导自举与初始化** (M1) 经历 BIOS/UEFI 引导、解压内核、从汇编 `head_64.S` 建立临时页表过渡到 C 语言 `start_kernel()` 全局子系统初始化。
    2.  **中断陷阱与 CFS 调度** (M2) 依托硬件中断描述符表（IDT），通过 `common_interrupt` 封装寄存器上下文，并利用以红黑树为核心的 CFS 调度进程。
    3.  **伙伴系统物理分配** (M3) 将物理内存按 Zone（DMA/Normal/HighMem）管理，以 2 的幂次 Page 块为单位利用 Buddy 算法分割与归并，规避外部碎片。
    4.  **Slab 与块设备 Page Cache** (M4-M5) 在伙伴系统之上建立 SLUB 细粒度对象缓存池，通过 Page Cache 脏页环形队列及换页 LRU 算法优化磁盘 I/O。

---

## 3. 6大模块与 28 张卡片规划 (6 Modules & 28 Cards Plan)

### M1：引导自举与内核初始化 (Booting & Kernel Initialization)
*   **Card 1. 实模式向保护模式与长模式切换**：引导扇区装载、CR0 寄存器 PE 位置位及 64 位长模式临时页表跳转物理流。
*   **Card 2. head_64.S 汇编自举与 start_kernel 入口**：内核自解压后，执行汇编入口，清空 BSS 并跳转至 C 语言 `init/main.c` 初始化大本营。
*   **Card 3. setup_arch() 体系结构特定初始化**：解析 boot_params、探查物理内存布局（E820 物理映射表）及保留内存（memblock）管理。
*   **Card 4. init_IRQ() 与初期内存页表接管**：建立正式的内核页表（Kernel Page Tables）并接管中断向量，切换系统运行模式。

### M2：中断、异常与 CFS 进程调度 (Interrupts & CFS Scheduler)
*   **Card 5. IDT 中断描述符表与硬件陷阱响应**：内核如何利用 `gate_desc` 在共享内存构建 IDT，并由 CPU 硬件自动压栈跳转处理。
*   **Card 6. system_call 中断门与系统调用过渡**：`sysenter`/`syscall` 指令调用、MSR 寄存器寻址及系统调用表（sys_call_table）索引。
*   **Card 7. 软中断 (softirq) 与工作队列 (workqueue) 底半部**：下半部（Bottom Half）的异步处理，软中断中断上下文与工作队列进程上下文切换。
*   **Card 8. CFS 完全公平调度器核心 vruntime 逻辑**：`update_curr()` 虚拟时间更新公式、权重衰减及红黑树最小节点调度调度。
*   **Card 9. 进程上下文切换 switch_to() 与 ThreadInfo**：`switch_to()` 汇编宏、x86 寄存器压栈与内核 `thread_info` 物理地址转换。

### M3：物理内存管理与伙伴系统 (Physical Memory & Buddy System)
*   **Card 10. NUMA 架构、Node 与内存 Zone 划分**：`pg_data_t` 结构体、Zone_DMA、Zone_Normal、Zone_HighMem 的物理划分界限。
*   **Card 11. Page 页帧结构体 struct page 物理元数据**：40字节 `struct page` 如何映射全局物理内存，以及映射计数（_refcount, _mapcount）。
*   **Card 12. 伙伴系统 (Buddy System) 2的幂次分配算法**：`free_area` 链表数组、伙伴块物理地址异或（XOR）比对及分裂与合并流程。
*   **Card 13. 内存碎片整理 (Migration & Compaction)**：异步 kcompactd 线程如何通过移动 active 页面到一端，回收连续大块物理内存。

### M4：Slab / SLUB 对象分配器 (Slab/SLUB Allocator)
*   **Card 14. Slab 分配器设计本质与对象复用**：规避伙伴系统 4KB 页粒度浪费，针对小内核对象（如 inode, task_struct）进行高速缓存复用。
*   **Card 15. kmem_cache 与 slab 逻辑分层结构**：kmem_cache 结构、`kmem_cache_cpu`（本地 CPU 缓存）与 `kmem_cache_node`（NUMA 共享节点）分层。
*   **Card 16. SLUB 分配器无锁分配 freelist 指针**：SLUB 如何通过直接在空闲对象头部嵌入下一空闲指针，实现 CPU 本地 freelist 的无锁快速分配。
*   **Card 17. kmalloc() 底层实现与 slab 动态退回**：`kmalloc` 如何根据申请字节大小（如 kmalloc-128, kmalloc-256）动态路由到指定大小的 slab 缓存。

### M5：Page Cache 页面缓存与内存收回 (Page Cache & Page Reclaim)
*   **Card 18. Page Cache 物理结构与 address_space 映射**：文件 Inode 的 `i_mapping` 物理指针、基数树（Radix Tree/xarray）对 Page 块的索引。
*   **Card 19. 延迟写回 pdflush / dirty page 限制阈值**：内核 `writeback` 背景线程根据脏页占比执行异步 IO 回写的控制时钟。
*   **Card 20. 物理页面收回 (Page Reclaim) 与 PFRA 算法**：物理页框回收算法、活动链表（Active List）与非活动链表（Inactive List）双向流动。
*   **Card 21. LRU 页面置换、二次机会与 swappiness**：如何通过两轮扫描确认 Page 的 Ref 标志，以及根据 swappiness 权重决定换出匿名页或文件页。
*   **Card 22. OOM-Killer 评分机制与内存危机防御**：`badness()` 评分公式，根据进程内存占比及 `oom_score_adj` 自动挑出受害者杀死。
*   **Card 23. 内存页换出 Swap 空间管理**：Swap 分区物理结构、`swap_map` 引用计数以及 Swap 页换入换出缺页中断。

### M6：虚拟文件系统与块设备 I/O (VFS & Block Device I/O)
*   **Card 24. VFS 四大核心对象元数据 (Dentry & Inode)**：`super_block`、`inode`（索引节点）、`dentry`（目录项）、`file`（打开文件描述）物理关联。
*   **Card 25. file_operations 虚表与驱动文件挂载**：设备驱动初始化、file_operations 虚表重定向分发实现虚拟文件操作。
*   **Card 26. BIO (Block I/O) 物理结构与请求块封装**：`struct bio` 列式结构、`bio_vec` 物理页切片及 scatter-gather DMA 地址列表。
*   **Card 27. I/O 调度器算法与电梯调度合并**：电梯合并算法（Elevator Merge）、BFQ（多队列公平调度）以及 Kyber 调度器的读写队列调度。
*   **Card 28. Page Fault 缺页中断与文件内存映射 (mmap)**：`mmap` 创建虚拟地址 VMA 区间，访问时触发缺页异常（Page Fault）读取文件页填入 PTE。
