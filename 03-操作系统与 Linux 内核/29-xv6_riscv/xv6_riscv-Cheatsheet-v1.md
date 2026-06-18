# 《xv6-riscv》 RISC-V 操作系统内核实现与同步机制高密 Cheatsheet

*   **L0 一句话本质**：xv6-riscv 是一个教学级 RISC-V 操作系统内核，通过精简的 C 语言与汇编代码，展示了进程特权级隔离（U/S 模式）、三级页表虚拟内存、硬中断路由、基于 Write-Ahead Logging (WAL) 的磁盘日志事务，以及经典 UNIX 管道同步的核心实现。
*   **L1 四句话逻辑**：
    1.  **特权切换与中断陷入**：进程在用户态（U Mode）运行时通过 `ecall` 触发软中断陷入，跳转至 Trampoline 页（物理与虚拟地址均固定在虚拟空间顶端），保存 CPU 寄存器至 Trapframe 后切入内核态（S Mode）的 `usertrap` 执行。
    2.  **双端页表与内存隔离**：每一个进程均独享一个三级（SV39）用户页表，而内核共享一张内核页表。内核在进程切换时，通过修改 `satp` 寄存器加载不同进程的页表基地址并执行 `sfence.vma` 刷新 TLB，确保内存强隔离。
    3.  **基于自旋锁与条件变量的并发控制**：采用自旋锁（Spinlock，依赖 RISC-V 原子指令 `amoswap`）保护临界资源；通过 `sleep` 和 `wakeup` 原语在共享的进程等待通道上挂起或唤醒进程，避免死锁与竞态。
    4.  **两级缓冲与崩溃一致性日志**：文件系统在磁盘扇区与内存 inode 之间建立两级块缓存（Buffer Cache），并通过内存日志区域（Journal）记录事务操作，所有的磁盘元数据写操作先顺序写入日志区，在 `commit` 时再刷入实际磁盘块，保证系统崩溃后的数据一致性。
*   **L2 存储与流转拓扑**：
    *   用户调用 (`write`) ➜ 文件描述符层 (`filewrite`) ➜ 管道层 (`pipewrite`/唤醒读进程) 或 Inode层 (`writei`) ➜ 日志层 (`write_log`) ➜ 块缓存层 (`bwrite`) ➜ 磁盘驱动层 (`virtio_disk_rw`) ➜ 硬件物理磁盘。

---

## 📂 页面一：引导初始化、中断陷入与页表虚存管理

### M1: 引导初始化与进程调度

#### Card 1. RISC-V 引导入口与多核休眠控制 (`kernel/entry.S` / `kernel/start.c`)
*   **引导物理入口**：QEMU 加载内核后，所有 CPU 核心 (Hart) 均跳转至 `_entry` (entry.S)。
*   **核心栈指针建立**：通过汇编计算 `sp = stack0 + (hartid * 4096)`，为每个核心预留 4KB 的内核栈空间。
*   **M Mode ➜ S Mode 切换**：在 `start.c` 中，将机器状态寄存器 `mstatus` 的 MPP 位配置为 Supervisor 模式，并将 `mret` 的跳转地址 `mepc` 指向 `main()`。同时配置 CLINT 定时器，开启时钟中断路由。

#### Card 2. 进程控制块 PCB 布局与进程分配 (`struct proc` / `kernel/proc.c`)
*   **PCB 数据结构**：`struct proc` 定义在 `proc.h` 中，管理进程状态 (`state`：UNUSED, SLEEPING, RUNNABLE, RUNNING, ZOMBIE)、页面根地址 (`pagetable`)、内核栈地址 (`kstack`)、Trap 上下文 (`trapframe`) 及打开的文件描述符表。
*   **进程初始化**：`allocproc()` 扫描全局 `proc` 数组，分配 PID，使用 `kalloc()` 申请物理页用于存放 `trapframe`；并在进程虚拟空间最高端映射 `TRAMPOLINE` 与 `TRAPFRAME`；最后在内核栈底伪造一个 `context` 结构体，将返回地址 `ra` 指向 `forkret()`，以兼容调度切换。

#### Card 3. 轮询式进程调度循环机制 (`scheduler()` / `kernel/proc.c`)
*   **调度主循环**：每个 CPU 核心运行一个独立的 `scheduler()` 线程（从不返回）。它循环扫描全局进程数组 `proc`，寻找状态为 `RUNNABLE` 的进程。
*   **切换锁屏障**：获取目标进程的 `p->lock`，将其状态变更为 `RUNNING`，并将当前 CPU 的运行指针 `c->proc` 指向该进程。
*   **上下文切换**：调用 `swtch(&c->context, &p->context)`，将 CPU 寄存器切换为目标进程的内核寄存器；当进程因时间片用尽（`yield()`）、等待 I/O（`sleep()`）或主动退出时，再次获取 `p->lock` 并调用 `sched()` 回切至 CPU 的 `scheduler` 线程。

#### Card 4. 上下文切换汇编解析与寄存器保存 (`kernel/swtch.S` / `kernel/proc.c`)
*   **swtch 汇编特征**：`swtch` 是用纯汇编编写的非特权函数，只保存 RISC-V 的被调用者保存寄存器（Callee-saved registers：`ra`, `sp`, `s0 - s11`）。
*   **切换过程**：
    1. 保存当前 `ra` (返回地址) 与 `sp` (栈指针) 以及 `s0 - s11` 至 `old` 的 `context` 结构体中。
    2. 从 `new` 的 `context` 结构体中恢复 `ra`, `sp` 和 `s0 - s11` 寄存器。
    3. 执行 `ret` 指令跳转，此时 CPU 实际上在执行 `new` 线程的下一条汇编指令。

---

### M2: 中断陷入与系统调用

#### Card 5. 硬件特权模式与 Trap 向量设置 (`stvec` / `kernel/trap.c`)
*   **特权级强隔离**：用户程序运行在 U 模式（User），内核运行在 S 模式（Supervisor）。当发生系统调用、硬件中断或异常时，硬件会自动将 CPU 提升至 S 模式。
*   **Trap 向量设置**：内核在引导进程或切回用户态前，写入 `stvec` 寄存器为 `uservec` 的虚拟地址（汇编入口）。
*   **硬件响应机制**：Trap 发生时，硬件自动将用户 PC 写入 `sepc`，将 Trap 原因写入 `scause`，将附加参数（如缺页地址）写入 `stval`，并将 CPU 的模式从 U 改为 S，最后跳转至 `stvec` 执行。

#### Card 6. 汇编 Trampoline 陷入蹦床页分析 (`kernel/trampoline.S`)
*   **蹦床页设计原理**：`TRAMPOLINE` 物理页同时映射在内核与用户页表的最高端虚拟地址处（同为 `0x3FFFFFF000`）。这保证了在修改 `satp`（切换页表）时，PC 所指向的指令序列依然有效且连续。
*   **uservec 执行流**：
    1. 通过 `csrrw` 交换 `a0` 与 `sscratch`，将当前进程的 `trapframe` 指针读入 `a0`，释放用户寄存器。
    2. 将用户所有通用寄存器（`ra`, `sp`, `gp`, `tp`, `t0-t6`, `a1-a7`, `s0-s11`）顺序存入 `trapframe`。
    3. 从 `trapframe` 中读取内核页表基址 `kernel_satp`、内核栈指针 `kernel_sp`、以及内核 `usertrap()` 地址。
    4. 执行 `csrw satp, t0` 切换至内核页表，执行 `sfence.vma` 刷新 TLB，跳转至 `usertrap()`。

#### Card 7. Trapframe 进程中断上下文页结构 (`struct trapframe` / `kernel/proc.h`)
*   **结构定义**：每个进程拥有一个 `trapframe` 物理页，在用户空间下映射在 `TRAPFRAME` 虚拟地址（仅次于 `TRAMPOLINE`）。
*   **核心字段**：
    *   `kernel_satp`：指向内核页表根节点。
    *   `kernel_sp`：该进程对应的内核栈顶指针。
    *   `kernel_trap`：内核陷入入口函数 `usertrap` 的地址。
    *   `epc`：保存中断发生时的用户 PC。
    *   `regs[32]`：保存所有用户通用寄存器。

#### Card 8. 用户态陷入处理与系统调用路由 (`usertrap()` / `kernel/trap.c`)
*   **陷入中枢路由**：`usertrap()` 首先关闭 CPU 中断，将 `stvec` 变更为 `kernelvec`（处理内核态陷入的向量）。
*   **系统调用路由**：如果 `scause == 8`（表明是系统调用 `ecall`）：
    1. 判断进程是否被杀，将 `p->trapframe->epc` 增加 4（指向 `ecall` 下一条指令，以防陷入死循环）。
    2. 开启中断，以便处理其他高优先级硬中断。
    3. 调用 `syscall()` 根据 `trapframe->a7` 中的系统调用号进行查表分发。
    4. 将返回值写入 `p->trapframe->a0`。
    5. 调用 `usertrapret()` 恢复用户寄存器并切回 U 模式。

#### Card 9. PLIC 与 UART 硬件中断分发机制 (`kernel/plic.c` / `kernel/uart.c`)
*   **硬中断控制器**：平台级中断控制器 PLIC 负责路由外部硬件中断（如磁盘、UART）。
*   **中断声明与完成**：
    *   当 CPU 捕获硬件外部中断时（`scause == 9` 或 `8` 且硬中断位生效），调用 `devintr()`。
    *   `devintr()` 通过向 PLIC Claim 寄存器读取数据来“认领”中断，获取中断源 ID。
    *   若 ID 对应 UART0（串口），则调用 `uartintr()` 处理字符输入输出；若对应 VIRTIO0，则调用磁盘驱动。
    *   处理完成后，向 PLIC Complete 寄存器写入该 ID，以告知 PLIC 中断处理完毕。

---

### M3: 虚拟内存与页表映射

#### Card 10. SV39 三级页表结构与 Page Walk 地址翻译 (`walk()` / `kernel/vm.c`)
*   **SV39 规范**：39位虚拟地址划分为三级：L2（9位）、L1（9位）、L0（9位）索引，以及偏移量 Offset（12位）。
*   **Page Walk 模拟**：`walk(pagetable, va, alloc)` 模拟 CPU 硬件 MMU 的多级翻译：
    1. 获取每一级页表对应的页表项 (PTE)。
    2. 若 PTE 标志位 `PTE_V`（有效）为 0，说明该级页表未分配。若 `alloc` 参数为真，则使用 `kalloc()` 申请物理页并写入 PTE；否则直接返回 NULL。
    3. 提取 PTE 中的物理页帧号 (PPN)，乘以 4096 得到下一级页表的物理基地址，继续向下遍历。
    4. 最终返回 L0 页表中对应 `va` 的 PTE 指针。

#### Card 11. 内核地址空间静态映射与设备内存 (`kvminit()` / `kernel/vm.c`)
*   **静态映射布局**：内核页表 `kernel_pagetable` 在引导时通过 `kvminit()` 静态建立：
    *   设备寄存器：将 `UART0` (0x10000000) 和 `PLIC` (0x0C000000) 映射到相同的虚拟地址（等值映射，方便读写寄存器）。
    *   物理内存：将 `KERNBASE` (0x80000000) 到 `PHYSTOP` (0x86400000) 进行等值映射，并配置读、写、执行权限。
    *   Trampoline 页：将物理页 `trampoline` 映射到虚拟地址最高端 `TRAMPOLINE`（内核与用户页表映射到同一物理页）。

#### Card 12. 用户虚拟地址空间布局与页分配 (`uvm.c` / `kernel/vm.c`)
*   **用户空间布局**：从 0 开始。底部为代码段与数据段，接着是单页用户栈，栈底映射了一个不可读写的“保护页”（Guard Page，未设置 `PTE_U`）以防止栈溢出，上方是进程堆空间（由 `sbrk` 扩展）。
*   **物理页分配**：
    *   `uvmalloc(pagetable, oldsz, newsz)`：在新区间内以 4KB 粒度循环调用 `kalloc()` 申请物理页，并调用 `mappages()` 建立映射，标志位设为 `PTE_U | PTE_W | PTE_R | PTE_V`。
    *   `uvmdealloc(pagetable, oldsz, newsz)`：释放该区间的物理页并清除对应的页表映射项 PTE。

#### Card 13. 写时复制 (COW) 缺页异常机制原理 (`copy-on-write` 优化)
*   **延迟拷贝优化**：`fork()` 时，内核不复制父进程的物理页，而是调用 `uvmcopy()` 将子进程页表项映射到父进程相同的物理页。
*   **标志位变更**：将父子进程对应 PTE 的 `PTE_W`（可写）标志清除，并设置自定义的 `PTE_COW` 标志。
*   **异常处理流程**：当任一进程尝试写入该页时，硬件触发 Store Page Fault 异常（`scause == 15`），控制权转移至 `usertrap()`：
    1. 判断发生缺页的虚拟地址是否设置了 `PTE_COW`。
    2. 检查物理页的引用计数（使用全局引用数组跟踪）。
    3. 若计数 > 1，则 `kalloc()` 申请一个新物理页，拷贝原页内容，并将新物理页映射给当前进程，重新配置为可写（`PTE_W`）；将原物理页的计数减 1。
    4. 若计数 == 1，直接将该页的 `PTE_W` 置 1，清除 `PTE_COW`。
    5. 减小 `sepc` 重新执行写操作指令。

---

## ⚔️ 操作系统内核虚拟化、并发与物理性能折衷防范矩阵

| 开发者直觉 (⚠) | 物理硬核限界 (⚡) | 架构设计折衷方案 (🛠) |
| :--- | :--- | :--- |
| **全量拷贝**：认为 `fork()` 必须立刻完整拷贝所有用户物理页以保证内存隔离。 | **物理内存带宽极限**：全页拷贝会导致内存分配雪崩，大量消耗 CPU 时钟周期。 | **写时复制 (COW)**：读共享、写时按需分裂页，折衷了物理内存空间与进程创建时延。 |
| **细粒度多级锁**：为了提高并发度，对每一级页表项 PTE 甚至 walk 的每一次跳转加锁。 | **多核锁竞争与死锁**：在频繁虚拟地址翻译中，高密锁竞争会导致流水线完全阻塞。 | **大粗粒度内核锁 + 进程私有页表**：各进程独享私有用户页表，仅在全局 `kalloc` 时加页表锁。 |
| **实时同步磁盘**：每一次文件 `write()` 系统调用必须即时同步刷入物理磁盘扇区。 | **I/O 设备物理写延迟**：VirtIO 磁盘写入时延比内存慢 5 个数量级。 | **块缓存 (bcache) + WAL 日志事务**：内存写日志后立即返回，由后台事务批量 install 数据。 |
| **无限递归陷入**：用户态陷入可任意嵌套，内核栈可任意增长以支持复杂调用。 | **内核栈物理空间受限**：xv6 每个进程的内核栈物理大小被强限制为 4KB（1 页）。 | **严格非嵌套陷入 + 蹦床页单栈指针**：跳转陷入时强制重置 `sp` 到 `trapframe->kernel_sp` 顶端。 |

---

## 📂 页面二：并发控制、多核锁同步、日志事务与文件系统

### M4: 自旋锁与多核同步

#### Card 14. 自旋锁实现与 RISC-V 原子汇编指令 (`amoswap` / `kernel/spinlock.c`)
*   **多核竞争锁**：自旋锁 `acquire()` 使用 RISC-V 原子指令 `amoswap.w.aq` 进行无锁自旋：
    ```c
    while(__sync_lock_test_and_set(&lk->locked, 1) != 0);
    ```
    该指令在硬件级别保证读取并写入 `lk->locked = 1` 为一个不可分割的原子操作。
*   **死锁防范**：`acquire` 必须首先关闭当前 CPU 核心的硬件中断 (`push_off()`)，以防止当前核心的硬中断处理程序尝试获取同一把自旋锁而导致当前核无限自旋（自我死锁）。

#### Card 15. 睡眠锁设计与多核进程独占保护 (`sleeplock` / `kernel/sleeplock.c`)
*   **长等待防范**：自旋锁不适于等待长时 I/O 任务，因为持有自旋锁的 CPU 无法让出（不允许在持有自旋锁时调用 `sleep` 或被调度切换，否则会导致其他 CPU 自旋超时）。
*   **睡眠锁结构**：`struct sleeplock` 内部封装了一把自旋锁 `lk`。
*   **睡眠锁行为**：
    *   `acquiresleep(lk)` 获取其自旋锁，若 `lk->locked` 为真，则释放自旋锁并调用 `sleep(lk, &lk->lk)` 让出 CPU。
    *   `releasesleep(lk)` 重置 `lk->locked = 0`，并调用 `wakeup(lk)` 唤醒等待该锁的所有进程。这允许进程在持有睡眠锁时让出 CPU 并进行磁盘 I/O。

#### Card 16. 睡眠与唤醒管道等待同步机制 (`sleep()` & `wakeup()` / `kernel/proc.c`)
*   **条件变量模拟**：`sleep(chan, lk)` 让进程在指定的虚拟等待通道（`chan`，一般为一个内存指针）上进入休眠。
*   **原子释放切换**：为防范 Lost Wakeup 竞态（即在释放锁与进入休眠的间隙，其他核触发了 wakeup 导致唤醒信号丢失）：
    1. 必须传入保护临界资源的锁 `lk`。
    2. 先获取进程控制块的自旋锁 `p->lock`。
    3. 释放传入的 `lk`。
    4. 将进程状态变更为 `SLEEPING`，记录 `p->chan = chan`。
    5. 调用 `sched()` 执行上下文切换，让出 CPU。
*   **唤醒路由**：`wakeup(chan)` 获取每个进程的 `p->lock`，若 `p->state == SLEEPING && p->chan == chan`，则将进程状态修改为 `RUNNABLE`。

#### Card 17. 进程退出与僵尸状态回收机制 (`exit()` & `wait()` / `kernel/proc.c`)
*   **资源回收两阶段**：
    *   第一阶段 (`exit(status)`)：进程自己关闭打开的文件描述符，解除对当前目录 inode 的引用。若子进程的父进程已退出，将其祖先重新托孤给 `initproc` (PID=1)；将自身状态修改为 `ZOMBIE`，唤醒父进程，调用 `sched()` 永久让出 CPU。
    *   第二阶段 (`wait(addr)`)：父进程扫描全局进程表。若找到状态为 `ZOMBIE` 的子进程，父进程调用 `kfree()` 释放其对应的 `trapframe` 页，调用 `freepagetable()` 销毁其用户页表及所有物理页映射，回收 PID，清空进程槽。

---

### M5: 块缓存与崩溃一致性日志

#### Card 18. 双向循环链表块缓存与 LRU 替换算法 (`bcache` / `kernel/bio.c`)
*   **块缓存池结构**：内核在内存中维护了一个固定大小的块缓存池 `bcache`（通常为 30 个 `struct buf`），由一把全局自旋锁 `bcache.lock` 保护。
*   **双向循环链表与 LRU**：`bcache.head` 串联了所有缓存块。
    *   **缓存命中**：`bget()` 遍历链表，若找到对应扇区（`dev` 和 `blockno`）的缓存块，增加其引用计数 `refcnt` 并返回。
    *   **缓存未命中与 LRU 替换**：从未被引用的缓存块（`refcnt == 0`）中，从后向前（Least Recently Used）扫描链表，找到最久未被使用的空闲块，重新指派为目标扇区，增加 `refcnt` 并返回。
    *   **回收控制**：使用完毕后，调用 `brelse()` 将该块移动到链表头部（最先被搜索的位置），以维持 LRU 顺序。

#### Card 19. 日志写前记录（WAL）事务机制模型 (`begin_op` / `kernel/log.c`)
*   **Write-Ahead Logging**：任何文件元数据（如 inode、位图、目录块）的写操作，必须先记录在磁盘的专属日志区（Journal），然后再写入实际的文件数据区。
*   **并发事务合并**：`begin_op()` 开启一个文件修改事务。
    *   若当前有提交（commit）正在进行，或当前事务累积的写入块数超过了日志区的最大容纳量，当前线程将被 `sleep` 挂起。
    *   若安全通过，则增加 `log.outstanding` 计数。这允许多个系统调用（如并发创建多个文件）合并在同一个事务中进行，从而大幅减少磁盘 I/O。

#### Card 20. 日志物理块顺序持久化机制 (`write_log` / `kernel/log.c`)
*   **顺序写入设计**：在 `end_op()` 判定当前没有其他活跃的事务在写入（`log.outstanding == 0`）时，启动提交逻辑。
*   **日志块持久化**：`write_log()` 顺序迭代内存日志块列表：
    1. 从内存 bcache 中读取被修改的数据块。
    2. 调用驱动的 `bwrite()` 将其顺序写入磁盘的日志块区域（Log Blocks，通常从扇区 2 开始）。
    3. 在内存的日志头结构 `log.lh` 中记录这些块的实际目标扇区号。

#### Card 21. 日志提交与崩溃原子保障 (`commit` / `kernel/log.c`)
*   **崩溃原子分界线**：在所有数据块成功写入磁盘日志区后，内核调用 `write_head()` 来提交事务。
*   **提交原理**：将内存中的日志头 `log.lh`（包含已写入日志块的总数 `n`，以及每个块的目标扇区映射表）写入磁盘日志区第 1 个扇区（Log Header Block）。
*   **提交标志**：只有当 Log Header 在磁盘上成功写入且 `n > 0` 时，事务才算真正提交完成。如果在 Log Header 写入成功前系统断电，重启后会视该事务未发生；如果在 Log Header 写入成功后断电，重启后内核会根据 Log Header 的记录进行重放恢复。

#### Card 22. 内存日志块安装至数据区块流程 (`install_trans` / `kernel/log.c`)
*   **数据落盘**：在事务成功提交（即 Log Header 物理刷盘完成）后，内核执行 `install_trans()`。
*   **数据复制**：遍历磁盘日志区的每一个有效块，将其从日志区读取出来，写入磁盘实际的真实数据扇区（Home Blocks）。
*   **缓存同步**：拷贝完成后，修改 bcache 中对应缓存块的 `dirty` 标志，使内存缓存能够及时同步最新写入的物理块数据。

#### Card 23. 系统崩溃热重启与日志重放恢复 (`recover_from_log` / `kernel/log.c`)
*   **开机恢复引导**：在系统启动并初始化磁盘驱动后，内核在 `initlog()` 中首先调用 `recover_from_log()`。
*   **重放判断与执行**：
    1. 读取磁盘上的 Log Header Block。
    2. 若头部记录的日志块数 `n > 0`，说明上次关机/断电前，事务已经成功提交，但未完成 install 写入真实数据区。
    3. 内核循环读取日志块，写入真实的数据扇区（即再次执行 `install_trans()` 过程）。
    4. 重放完毕后，将 Log Header 中的 `n` 清零并写回磁盘，以避免下次开机时重复恢复。

---

### M6: 文件系统与 IPC 管道

#### Card 24. 磁盘 Inode 布局与内存 Inode 高速缓存 (`icache` / `kernel/fs.c`)
*   **磁盘 Inode (`struct dinode`)**：在磁盘上占据固定空间，包含文件类型 (`type`)、硬链接数 (`nlink`)、文件大小 (`size`) 以及 12 个直接块和 1 个一级间接块索引（支持最大 $12 + 1024 = 1036$ 块，约 512KB 大小）。
*   **内存 Inode (`struct inode`)**：是磁盘 inode 在内存中的高速缓存副本。
*   **生命周期控制**：
    *   `iget(dev, inum)`：如果对应的 inode 已经在 `icache` 链表中，则增加其引用计数 `ref`；否则从磁盘读取并加载到内存缓存中。
    *   `iput(ip)`：减少 `ref`。当 `ref == 0` 且 `nlink == 0`（文件已被彻底删除）时，释放该 inode 占用的所有磁盘块，清空磁盘 inode。

#### Card 25. 目录查找与硬链接操作实现 (`dirlookup` & `dirlink` / `kernel/fs.c`)
*   **目录项查找**：目录也是一种文件，其内容是一组 `struct dirent`（目录项，包含 `inum` 和 `name`）。
    *   `dirlookup(dp, name, poff)`：循环读取目录文件内容，如果找到匹配的 `name`，返回对应的 inode 缓存指针 `iget(dp->dev, de.inum)`。
*   **硬链接创建**：`dirlink(dp, name, inum)` 向目录文件追加写入一个新的 `struct dirent`。创建硬链接（如 `link` 系统调用）时，只是向目标目录中写入一个新目录项，并将目标文件的 `ip->nlink` 自增 1，而不需要复制实际的文件数据。

#### Card 26. 文件描述符分配与文件表抽象 (`struct file` / `kernel/file.c`)
*   **三层文件抽象**：
    1. **进程层**：`p->ofile[fd]` 保存指向文件表项的指针，`fd` 即为数组索引。
    2. **系统层**：全局文件表 `ftable` 包含所有打开的文件项 `struct file`，管理读写偏移量 `off` 以及文件引用计数 `ref`。
    3. **物理层**：`struct file` 可以抽象为 INODE（普通文件/设备）或 PIPE（管道）。
*   **分配与关闭**：`filealloc()` 遍历 `ftable` 分配空闲文件项。`fileclose()` 减少其引用计数 `ref`。若引用计数归零，则根据其类型调用 `iput()` 释放 inode 或调用 `pipeclose()` 关闭管道。

#### Card 27. IPC 管道环形缓冲区与同步读写 (`pipe.c` / `kernel/pipe.c`)
*   **单页内存队列**：`struct pipe` 包含一个大小为 512 字节的环形缓冲区 `data`，以及读写计数指针 `nread` 与 `nwrite`。
*   **Pipe 读写同步**：
    *   `pipewrite(pipe, addr, n)`：向环形缓冲区写入数据。若缓冲区已满（`nwrite == nread + PIPESIZE`），调用 `wakeup(&pipe->nread)` 唤醒读进程，并调用 `sleep(&pipe->nwrite, &pipe->lock)` 挂起写进程。
    *   `piperead(pipe, addr, n)`：从环形缓冲区读取数据。若缓冲区为空（`nread == nwrite`），调用 `wakeup(&pipe->nwrite)` 唤醒写进程，并调用 `sleep(&pipe->nread, &pipe->lock)` 挂起读进程。

#### Card 28. 文件系统调用层与内核边界安全校验 (`argraw` / `kernel/sysfile.c`)
*   **安全防御边界**：由于用户进程可以通过寄存器传入任意指针，内核在执行系统调用前必须对参数进行严格校验。
*   **参数提取与越界校验**：
    *   `argint(n, &val)` / `argaddr(n, &val)`：从 `p->trapframe->a0 - a5` 中读取用户传入的整数或地址。
    *   `argstr(n, buf, max)`：从用户空间拷贝以 `\0` 结尾的字符串。内部使用 `fetchstr(addr, buf, max)` 逐字节读取，如果读取地址越界（超出进程虚拟内存空间 `p->sz`），则立即拒绝并返回 -1，从而防止恶意指针溢出或内核内存泄漏。

---

## 🔬 Zone T: xv6-riscv 操作系统底层诊断与运行期寄存器字典

### T1: RISC-V 核心控制寄存器与 CPU 架构参数对照表

*   `sstatus` (Supervisor Status Register)：控制内核特权级状态。SIE 位控制 S 模式中断使能；SPP 位记录 Trap 发生前 CPU 处于 U 模式还是 S 模式。
*   `satp` (Supervisor Address Translation and Promotion Register)：控制页表根节点。物理地址 PPN 写入低 44 位，高 4 位写入 MODE（SV39 模式下 MODE=8）以开启虚拟地址翻译。
*   `sepc` (Supervisor Exception Program Counter)：保存 Trap 发生时的异常指令 PC。在 `sret` 指令执行时，CPU 会自动跳转回 `sepc` 所指向的地址。
*   `scause` (Supervisor Exception Cause Register)：中断/异常原因寄存器。最高位 1 表示硬中断，0 表示异常；低位记录中断号（例如：`8` 代表用户系统调用，`15` 代表写内存缺页异常，`1` 代表时钟中断）。
*   `stval` (Supervisor Trap Value Register)：保存 Trap 相关的辅助数据。例如发生缺页异常时，`stval` 会保存导致缺页的虚拟地址；指令非法时，保存非法的指令机器码。
*   `sie` / `sip` (Supervisor Interrupt Enable / Pending Registers)：S 模式中断使能与挂起状态寄存器。`sie` 的 STIE 位控制时钟中断使能，SEIE 控制外部硬中断使能。

### T2: xv6-riscv 内核调试与 QEMU / GDB 诊断指令字典

```bash
# 1. 启动仿真与 GDB 监听 (在 QEMU 中挂起，等待 GDB 连接)
qemu-system-riscv64 -machine virt -bios none -kernel kernel/kernel -m 128M -smp 3 -nographic -drive file=fs.img,directory=.,format=raw -device virtio-blk-device,drive=x0,bus=virtio-mmio-bus.0 -S -gdb tcp::26000

# 2. 启动 GDB 调试器并建立物理连接
riscv64-unknown-elf-gdb -ex "target remote localhost:26000" -ex "symbol-file kernel/kernel"

# 3. 监控 RISC-V 页表 satp 寄存器与 CPU 寄存器状态 (在 GDB 终端)
info registers satp sstatus sepc scause stval

# 4. 追踪内核地址空间翻译 (在 QEMU 控制台, 输入 Ctrl+A ➜ C 进入控制台)
info mem           # 输出当前 CPU 在 S 模式下的三级页表虚拟映射表及物理基址
info registers     # 打印 QEMU 物理 CPU 核心当前所有控制寄存器

# 5. 断点断在陷入中枢 usertrap()
break usertrap
break syscall
print p->pid
print/x p->trapframe->a7   # 打印当前系统调用号

# 6. 追踪上下文切换汇编执行 (在 GDB 终端单步跟踪 swtch.S)
layout asm
stepi
x/10gx &p->context         # 打印进程 context 中保存的被调用者保存寄存器值
```
