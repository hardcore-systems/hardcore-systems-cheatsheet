# 《linux-insides》高密知识图谱与速查手册 (Cheatsheet)

*   **L0 一句话本质**：从实模式汇编引导与内核解压自举起点出发，通过两级引导页表向保护/长模式平滑过渡，利用伙伴系统的 2 的幂次分割与 SLUB 对象复用机制管理物理内存，借由红黑树 CFS 调度器平摊 CPU 算力，并通过 VFS 文件描述符分发与 Page Cache 延迟回写屏蔽底层块设备差异的单体宏内核。
*   **L1 四句话逻辑**：
    1.  **引导自举与初始化** (M1) 经历 BIOS/UEFI 引导、解压内核、从汇编 `head_64.S` 建立临时页表过渡到 C 语言 `start_kernel()` 全局子系统初始化。
    2.  **中断陷阱与 CFS 调度** (M2) 依托硬件中断描述符表（IDT），通过 `common_interrupt` 封装寄存器上下文，并利用以红黑树为核心的 CFS 调度进程。
    3.  **伙伴系统物理分配** (M3) 将物理内存按 Zone（DMA/Normal/HighMem）管理，以 2 的幂次 Page 块为单位利用 Buddy 算法分割与归并，规避外部碎片。
    4.  **Slab 与块设备 Page Cache** (M4-M5) 在伙伴系统之上建立 SLUB 细粒度对象缓存池，通过 Page Cache 脏页环形队列及换页 LRU 算法优化磁盘 I/O。
*   **L2 存储引擎数据流演进拓扑**：
    *   [汇编引导 head_64.S] $\rightarrow$ [C 语言 start_kernel()] $\rightarrow$ [E820 内存布局探查] $\rightarrow$ [伙伴系统 Free List 初始化] $\rightarrow$ [SLUB 内核缓存池建立] $\rightarrow$ [CFS 调度器启动] $\rightarrow$ [VFS 虚拟文件系统挂载] $\rightarrow$ [Page Cache 读入文件] $\rightarrow$ [BIO 请求队列落盘]

---

## 🌐 操作系统物理内核底层机制与内存分配滤镜 (Linux Kernel Epistemic Filter)

- **认识论 (Epistemology) - 以宏内核物理统一与硬件直接操纵为设计起点**：
  Linux Kernel 认识论的基石在于“效率优先，将硬件控制权牢牢固化在统一的虚实映射地址空间中”。不同于微内核（Microkernel）通过消息传递进行服务隔离，Linux 作为宏内核（Monolithic Kernel），将进程管理、虚拟内存、文件系统和网络协议栈全部放入同一个特权级（Ring 0）地址空间中。其设计的起点是汇编级的硬控制：从引导自举（Bootstrapping）时关闭中断、切换 64 位长模式临时页表开始，到通过直接操纵 CR0/CR3/CR4 寄存器重塑 CPU 空间规则。宏内核架构换取了极高的数据通道速度与进程间通信效率。
- **一致性论 (Consistency Theory) - 页面缓存写回屏障、SLUB 分配一致与 OOM 防御**：
  在内核态运行的复杂软件，一致性面临着并发与资源的双重挤压。在内存分配层面，Buddy 系统保证了物理 Page 的不重叠分配，而 SLUB 必须维护 CPU 本地 freelist 指针的原子性以避免缓存行竞争；当内存极度匮乏时，一致性退位给生存权，内核依靠 OOM Killer 的 badness 算法，强制清理内存占比过高的受害者以保障整体系统的连续运行。在存储 I/O 层面，为了抵消磁盘极高延迟，Page Cache 使用 Radix Tree 将脏页延迟滞留内存，内核必须配合 `sysctl` writeback 写回机制和平滑 I/O 队列控制，防范突发断电造成的物理文件不一致。
- **方法论 (Methodology) - 极致物理缓存对齐、对象池复用与多队列电梯合并**：
  Linux 方法论的核心是“消除一切无谓的访存延迟与总线争用”。线性寻址的伙伴系统无法应付小内核对象（如 inode/dentry）的高频申请，方法论引入 SLUB 分配器，直接在未分配的对象头部嵌入指向下一个空闲对象的 `freelist` 指针，实现无锁的本地 CPU 分配。在 Block 层，为了平摊机械臂寻道磁头移位开销，电梯算法将杂乱的 I/O 请求（BIO）按物理扇区编号进行合并（Elevator Merge）并排入多队列调度，以极度精密的算法逻辑弥补物理硬件的延迟瓶颈。

---

## ⚔️ 操作系统内核虚拟化、并发与物理性能折衷矩阵 (Linux Kernel Core Trade-offs)

| 开发者直觉 (⚠) | 系统物理现实 (✗) | 架构折衷与防范金标准 (✓) |
| :--- | :--- | :--- |
| **SLUB 内存分配会尽量将物理空间紧密排列，以防止内核产生空间碎片** | 为了消除 CPU 核心之间的 Cache Line 伪共享（False Sharing），SLUB 分配器在物理内存上会故意按照 CPU L1 Cache Line 大小（通常是 64 字节）对齐。这会导致小于 64 字节的对象也会占用 64 字节，产生一定的物理内存浪费。 | 采用 Cache Line 物理对齐设计。通过在 `kmem_cache_create` 中使用 `SLAB_HWCACHE_ALIGN` 标志，确保频繁访问的对象在物理上独占 Cache Line，虽然牺牲了微量的内存利用率，但极大地换取了多核并发时的 CPU 吞吐。 |
| **CFS 完全公平调度器会绝对公平地平摊 CPU 时间给所有活跃的线程** | 绝对的公平会导致严重的“线程饥饿”与高频上下文切换开销（Context Switch Overheads）。如果让大量短任务和长计算任务频繁打断切换，CPU 将把大部分时间浪费在保存恢复寄存器和刷新 MMU 页表上。 | 引入调度最小粒度 `sched_min_granularity_ns`。CFS 保证进程被调度后，必须至少连续运行此最小时间，防止因频繁计算虚拟时间 `vruntime` 产生过度切换，在公平性与切换开销之间达成折衷。 |
| **为了加速磁盘文件读取，我们应该把 vm.dirty_background_ratio 阈值设得无限高** | 脏页阈值过高会导致大量的脏数据滞留在内存中。当系统内存面临回收或主动执行 `sync` 时，海量脏页集中回写，会瞬间打满块设备 I/O 通道，引发严重的系统级 Stall 停摆。 | 合理调优写回参数。在生产高吞吐服务器上，通常建议将 `vm.dirty_background_ratio` 限制在 5%-10%，并结合 `vm.dirty_ratio` 设为 20% 左右，以保持内核 pdflush 背景回写线程高频平滑写入。 |

---

## 🗺️ 6大核心模块与 28 张高密速查卡片 (Core Cards Map)

### M1：引导自举与内核初始化 (Slate Blue)

#### Card 1. x86 引导扇区载入与实模式向长模式切换 (Boot to Long Mode)
*   **引导状态跳转**：BIOS/UEFI 读取磁盘 MBR，载入引导装载程序（如 GRUB），将内核 bzImage 读取到内存，随后跳转至 `boot/header.S` 汇编入口。
*   **64位长模式过渡**：执行 16 位实模式自检后，开启 CR0 寄存器的 PE 位切换到 32 位保护模式；随后建立两级临时物理映射页表，将 CR4 寄存器的 PAE 位置 1，修改 EFER MSR 寄存器，最终进入 64 位长模式。

#### Card 2. head_64.S 汇编初始化与 C 语言入口 (head_64.S to start_kernel)
*   **汇编段自举**：CPU 进入 64 位后，跳转到 `arch/x86/kernel/head_64.S`。在这里执行最初的汇编代码：清除 BSS 段、设置段寄存器、分配临时的内核栈（Kernel Stack）并加载正式的 GDT/IDT 指针。
*   **跳转 start_kernel**：一切就绪后，通过 `jmp` 指令飞越物理边界，进入内核真正的 C 语言初始化总入口 `start_kernel()`。

#### Card 3. setup_arch() 与 E820 内存布局探查 (setup_arch & E820)
*   **物理内存探查**：`setup_arch()` 接收来自 bootloader 的 `boot_params` 参数。通过读取 E820 物理内存表，掌握宿主机物理内存的空闲与保留边界。
*   **memblock 临时分配器**：在正式的伙伴系统建立前，使用 `memblock` 结构体进行粗粒度的物理内存管理与保留区划定，防止初始化阶段内核内存被自我覆盖。

#### Card 4. init_IRQ() 与内核早期页表接管 (init_IRQ & Kernel Page Tables)
*   **中断机制初始化**：`init_IRQ()` 初始化中断描述符表（IDT），写入内核正式的软硬件中断门（Interrupt Gates）。
*   **接管早期页表**：销毁 bootloader 建立的临时物理页表，切换到由内核编译期静态定义的 `swapper_pg_dir` 页表，开启内核虚实地址空间映射，为进程分配奠定基础。

---

### M2：中断、异常与 CFS 进程调度 (Moss Green)

#### Card 5. IDT 中断描述符表与硬件中断物理压栈 (IDT & Interrupt Stack)
*   **IDT 物理构造**：IDT 是内核维护的 256 项数组，每项（Gate Descriptor）包含 64 位的中断服务例程（ISR）入口虚拟地址及特权级。
*   **CPU 硬件压栈**：当中断发生时，CPU 硬件自动暂停执行，将当前 SS、RSP、RFLAGS、CS、RIP 压入当前进程的内核栈，读取 IDT 对应项，并跳转至通用中断入口 `common_interrupt`。

#### Card 6. syscall 硬件指令与系统调用表分发 (syscall & sys_call_table)
*   **硬件指令特权转换**：x86_64 放弃了传统的 `int 0x80` 软件中断，改用 `syscall` 指令。CPU 读取 `IA32_LSTAR` MSR 寄存器，瞬间将 RIP 设为 `entry_SYSCALL_64` 并切换特权级。
*   **系统调用表分发**：在汇编入口保存寄存器后，根据 RAX 中的调用号，检索 `sys_call_table` 函数指针数组，跳转执行具体的内核服务。

#### Card 7. 软中断 (softirq) 与工作队列 (workqueue) 底半部 (Bottom Halves)
*   **中断上下文快速让出**：为了防止硬中断阻塞网卡或磁盘，内核将中断处理分为两半。顶半部只做最少寄存器操作，然后挂起软中断。
*   **下半部异步处理**：
    1.  **softirq (软中断)**：在硬中断返回前或 `ksoftirqd` 进程中执行，不能阻塞休眠，适合网卡收包。
    2.  **workqueue (工作队列)**：由内核背景线程执行，运行在进程上下文，可以安全执行 I/O 或睡眠阻塞。

#### Card 8. CFS 完全公平调度器核心虚拟运行时间更新 (CFS vruntime Update)
*   **虚拟时间轴递增**：CFS 调度器通过 `vruntime` 衡量进程获得的 CPU 资源。进程运行一段时间后，虚拟时间增加：$\Delta vruntime = \Delta physical\_time \times \frac{NICE\_0\_LOAD}{weight}$。
*   **红黑树动态排序**：CFS 使用红黑树 `rb_root_cached` 挂载所有 runnable 进程，以 `vruntime` 为键值排序。每次调度通过 $O(1)$ 的时间复杂度快速取走最左侧叶子节点（`rb_leftmost`）执行。

#### Card 9. switch_to() 上下文切换汇编内核栈重定向 (switch_to Context Switch)
*   **硬件寄存器重定向**：调度程序决定切换进程时，调用 `switch_to()`。
*   **汇编重写栈指针**：利用汇编修改 RSP 寄存器，将其重定向到目标进程的 `thread.sp`（内核栈顶）。由于所有局部变量和 CPU 寄存器都在栈中，完成栈指针切换即完成了 CPU 上下文的物理转移。

---

### M3：物理内存管理与伙伴系统 (Plum Rose)

#### Card 10. NUMA 架构、物理 Node 与内存 Zone 划分 (NUMA Zones)
*   **NUMA 物理节点**：多路 CPU 下，物理内存被划分为多个 Node（结构为 `pg_data_t`），靠近各自 CPU 以减少跨总线访存开销。
*   **Zone 物理分层**：每个 Node 内部根据硬件限制划分为：
    1.  **ZONE_DMA**：物理内存前 16MB，用于兼容老式 DMA 硬件。
    2.  **ZONE_NORMAL**：16MB 至 4GB，内核可以直接映射访问的主物理区域。
    3.  **ZONE_HIGHMEM**：4GB 以上（32位系统下无法直接映射的区域，64位下已基本不存在）。

#### Card 11. struct page 物理页帧元数据映射机制 (struct page Layout)
*   **全局页描述符数组**：内核在初始化时，为系统内所有 4KB 物理页框分配一个 `struct page` 描述符，集中存放在 `mem_map` 数组中。
*   **映射计数控制**：`struct page` 不包含任何文件数据，仅用 `_refcount`（内核引用计数）和 `_mapcount`（用户态页表映射次数）控制页面的生命周期。当 `_refcount` 归 0 时，页面被释放回伙伴系统。

#### Card 12. 伙伴系统 (Buddy System) 2的幂次物理分割算法 (Buddy Allocator)
*   **幂次空闲链表数组**：伙伴系统维护一个从 0 到 10（即 1 页到 1024 页）的 `free_area` 链表数组。
*   **异或比对与拆分归并**：
    *   **分配**：申请 Order 级页块时，若对应链表为空，则向更高阶链表申请，并将得到的大块对半“拆分（Split）”。
    *   **释放**：释放页块时，计算伙伴块地址 $Partner = Block \oplus (Order \ll PAGE\_SHIFT)$，若伙伴块也空闲，则将其“合并（Merge）”并提升到上一级 Order 链表。

#### Card 13. 内存异步整理 (Migration & Compaction) 机制 (Memory Compaction)
*   **内存碎片化瘫痪**：长时间运行后，频繁分配释放会导致物理内存中充斥着散落的已分配页，无法找到 Order-9（2MB）等连续大块页。
*   **双向扫描平移整理**：`kcompactd` 线程从 Zone 两端向中间扫描：一端扫描可移动的 active 页面，另一端扫描空闲块。将 active 页面整体平移复制到空闲块中，并在页表中重定向，物理拼接出大片连续的空闲空间。

---

### M4：Slab / SLUB 对象分配器 (Terracotta)

#### Card 14. Slab 分配器设计思想与内核对象复用 (Slab Allocator Concept)
*   **规避伙伴系统粗粒度**：由于伙伴系统的最小单位是 4KB 页，频繁申请几十字节的内核对象（如 `file` 描述符）会导致巨大的内存内部碎片。
*   **对象池生命周期缓存**：Slab 分配器在物理页内划分固定大小的“对象池”。对象在释放时并不退回伙伴系统，而是保留其构造初始化状态，仅标记为“可用”，下次申请时直接返回，实现了极致的 CPU 局部性。

#### Card 15. kmem_cache 层级结构与 CPU 局部性锁优化 (kmem_cache Hierarchies)
*   **CPU 本地无锁快速路径**：`kmem_cache` 结构包含 `kmem_cache_cpu` 结构，提供单 CPU 的无锁快速通道，可以直接从本地 CPU 缓存的对象链表中分发对象。
*   **Node 共享慢速路径**：当本地缓存耗尽时，通过慢速路径（Slow Path）锁定 NUMA 节点特定的 `kmem_cache_node` 共享链表，批量补充对象到本地 CPU 缓存。

#### Card 16. SLUB 分配器 inline freelist 指针构造 (SLUB freelist Offset)
*   **零额外开销 freelist**：SLUB 分配器对传统 Slab 进行精简，去除了复杂的外部管理数据。
*   **空闲链表内嵌**：它将指向下一个空闲对象的指针（`freelist` 指针）直接写入空闲对象自身的物理内存首地址（Offset）中。分配时直接提取该地址，释放时将当前对象首地址写入前一空闲对象头，实现了零内存开销的链表维护。

#### Card 17. kmalloc() 动态动态大小路由机制 (kmalloc Implementation)
*   **动态大小预分配**：`kmalloc()` 专用于分配任意大小的连续内核内存。
*   **Slab 静态池路由**：内核在初始化时，静态预先创建了一组不同尺寸的 kmem_cache（如 `kmalloc-32`, `kmalloc-64`, `kmalloc-128`, `kmalloc-2048`）。应用调用 `kmalloc(100)` 时，内核自动向上取整路由到 `kmalloc-128` 缓存池分发。

---

### M5：Page Cache 页面缓存与内存收回 (Page Cache & Page Reclaim)

#### Card 18. Page Cache 物理结构与 xarray / 基数树索引 (Page Cache Layout)
*   **文件页内存镜像**：Page Cache 是文件数据在物理内存中的映射缓存。
*   **树状物理索引**：每个 Inode 的 `address_space` 结构包含一个基数树（在现代内核中为高效的 `xarray`）来维护其 Page Cache。通过文件内部的 Page 偏移量作为键值，可在 $O(1)$ 的时间复杂度内迅速定位内存中的物理 `struct page` 指针。

#### Card 19. pdflush 背景写回线程与脏页控制阈值 (Page Writeback Control)
*   **脏页延迟滞留**：文件修改写入 Page Cache 后被标记为 Dirty（脏页），不会立即触发慢速磁盘 I/O。
*   **异步刷盘控制**：内核背景线程（如 `wb_workfn` / 老版 `pdflush`）定期被唤醒。当脏页占比达到 `vm.dirty_background_ratio` 时，启动异步写回；当达到临界阈值 `vm.dirty_ratio` 时，强行阻塞用户进程的写操作，执行同步写回。

#### Card 20. 物理页框回收算法 (PFRA) 与双向 LRU 链表 (Page Reclaim PFRA)
*   **空闲内存告急收回**：当物理内存水位低于低安全水位（low watermark）时，`kswapd` 守护进程启动物理页框回收。
*   **活动状态双向迁移**：Zone 维护 `Active` 和 `Inactive` 两个 LRU（最近最少使用）页面链表。经常访问的页面留在 Active 链表；长时间未被访问的页面向下迁移至 Inactive 链表，作为回收的优先候选者。

#### Card 21. 两轮 LRU 置换扫描与 swappiness 换出分配 (LRU & Swappiness)
*   **二次机会引用校验**：收回 Inactive 链表时，内核清除页表项中的 Referenced 标志位。如果在下一次扫描前该页又被访问，则将其重新移回 Active 链表；否则，安全回收。
*   **Swappiness 回收倾向**：`swappiness` 值（0-100）作为权重。值越高，内核越倾向于回收匿名内存页（Anonymous Memory）并写入 Swap 分区；值越低，越倾向于直接回收干净的文件页（File Pages）。

#### Card 22. OOM-Killer badness() 积分评分公式 (OOM Killer Scoring)
*   **内存崩溃自我救赎**：当系统彻底耗尽内存且无法收回任何物理页时，触发 OOM-Killer。
*   **积分公式与惩罚**：内核根据 `badness()` 算法计算每个进程的得分：$Points = \frac{RSS\_Memory}{Total\_System\_Memory} \times 1000$。在此基础上，叠加进程运行时间惩罚，并加上 `oom_score_adj` 的用户调整偏置，挑选最高得分者物理杀死以释放空间。

#### Card 23. Swap 空间映射管理与缺页换入 (Swap Page Management)
*   **物理页向磁盘转移**：被换出的匿名内存页被写入指定的磁盘 Swap 分区。
*   **swap_map 引用跟踪**：内核使用 `swap_map` 数组记录 Swap 分区中每个 4KB 槽位（Swap Slot）的进程引用计数。当进程重新访问该页时，触发 Page Fault 缺页中断，内核分配新物理页并从 Swap 磁盘读入。

---

### M6：虚拟文件系统与块设备 I/O (VFS & Block Device I/O)

#### Card 24. VFS 四大核心对象拓扑关系 (VFS Core Objects)
*   **四大物理抽象元数据**：
    1.  **super_block**：描述文件系统全局元数据（如块大小，挂载点）。
    2.  **inode**：代表具体的物理文件实体，包含文件物理块指针。
    3.  **dentry**：表示目录树中的一个节点（文件名与 Inode 的绑定），加速路径寻址。
    4.  **file**：代表进程打开的一个文件实例，包含当前的读取读写偏移。

#### Card 25. file_operations 虚表与设备文件分发驱动 (file_operations VFS)
*   **多态多路文件操作接口**：`struct file` 内置指向 `struct file_operations` 的指针虚表。
*   **驱动挂载映射**：在执行 `read()` 系统调用时，内核并不关心具体底层细节，而是执行 `file->f_op->read()`。对于 EXT4，这指向 `ext4_file_read`；对于字符设备（如 `/dev/null`），它被驱动程序接管直接指向设备操作函数。

#### Card 26. struct bio 物理结构与 bio_vec 页切片向量 (struct bio Layout)
*   **块设备 I/O 物理单元**：VFS 发起数据传输时，将文件页翻译为 `struct bio` 结构，它是块设备 I/O 的核心载体。
*   **聚合分散 DMA 映射**：`struct bio` 包含 `bio_vec` 数组，每个元素描述一个内存物理 Page、块内偏移和大小。DMA 控制器利用此数组执行分散-聚集（Scatter-Gather）I/O，直接将多个不连续物理页刷入磁盘。

#### Card 27. I/O 请求队列合并与调度算法 (I/O Scheduler queue)
*   **磁盘合并防抖动**：BIO 块进入请求队列 `request_queue`。调度器首先执行电梯合并（Elevator Merge），将相邻扇区（Sector）的多个 BIO 合并为单个大 Request。
*   **现代多队列调度**：现代内核引入 blk-mq（多队列）框架，为每个 CPU 核心分配专属的软件队列，结合 BFQ（多队列公平调度）或 Kyber，消除了块设备全局大锁的竞争。

#### Card 28. mmap 物理文件映射与 Page Fault 零拷贝读取 (mmap & Page Fault)
*   **虚拟内存区间建立**：`mmap` 创建一个新的虚拟内存区间（`struct vm_area_struct`），将其与目标文件 `i_mapping` 绑定，但在页表中不分配物理页。
*   **缺页异常零拷贝**：当程序首次读取该内存地址时，CPU 触发缺页异常（Page Fault）。内核捕捉该异常，从磁盘读取对应文件页填入 Page Cache，直接将该 Page 的物理地址写入进程页表，实现了零拷贝直接读写。

---

## 🔬 辅助版块 Zone T：生产诊断与调试字典 (Zone T)

### T1 Linux Kernel 操作系统核心内核参数调优对照表

| 配置参数 / /etc/sysctl.conf | 经典建议数值 | 物理调优语义 |
| :--- | :--- | :--- |
| `kernel.sched_latency_ns` | `12000000 - 24000000` | CFS 调度延迟。保证红黑树中所有 runnable 进程在多少纳秒内至少被调度一次，调小能提升桌面交互响应，调大能增加计算密集型吞吐。 |
| `kernel.sched_wakeup_granularity_ns` | `2000000 - 4000000` | 唤醒粒度。控制新进程唤醒时，其 vruntime 比当前进程小多少才能强占当前 CPU。调大能防止频繁唤醒引起的上下文切换抖动。 |
| `vm.dirty_background_ratio` | `5 - 10` | 系统脏页占物理内存的百分比。达到此阈值时，背景回写线程开始异步将内存脏页刷盘。 |
| `vm.dirty_ratio` | `20 - 30` | 脏页占物理内存的最大百分比。达到此阈值时，操作系统将强制阻塞用户的 write 写入操作，进行同步同步刷盘，避免 I/O 被打满。 |

### T2 Linux Kernel 底层追踪与慢查询诊断 CLI 指令字典

*   **1. 诊断全局 Slab 缓存对象实时分配与开销**
    ```bash
    # 打印系统内各个 slab 缓存（如 dentry, inode, kmalloc-128）的活跃对象数、大小和内存占用，排查内核内存泄漏
    slabtop -o -s a
    ```
*   **2. 使用 Ftrace 追踪内核指定函数的执行耗时**
    ```bash
    # 启用 ftrace function_graph 追踪器，追踪 mm/page_alloc.c 中的伙伴系统分配入口 __alloc_pages
    echo function_graph > /sys/kernel/debug/tracing/current_tracer
    echo __alloc_pages > /sys/kernel/debug/tracing/set_graph_function
    cat /sys/kernel/debug/tracing/trace | head -n 40
    ```
*   **3. 实时分析 CPU 核心上的热点内核函数调用百分比**
    ```bash
    # 实时抓取内核态 CPU 热点函数调用（采样率 99Hz），方便排查系统调用过多或锁自旋导致的 CPU 飙高瓶颈
    perf top -g
    ```
