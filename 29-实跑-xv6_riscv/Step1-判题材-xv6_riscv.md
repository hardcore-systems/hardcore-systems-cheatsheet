# Step 1: 判题材 - mit-pdos / xv6-riscv

## 1. 题材深度分析与判定

`mit-pdos/xv6-riscv` 是 MIT 经典教学级操作系统内核，基于 RISC-V 64位架构重构。虽然代码量极小（仅数千行），但它完整实现了 UNIXv6 的核心设计理念，是理解现代操作系统底层原理的“工业级”微型样本。
本项目包含硬件特权态切换、三级页表地址空间隔离、无锁/自旋锁同步机制、磁盘块事务日志（WAL）以及虚拟文件系统。信息密度极高，完全符合“硬核技术”题材。

## 2. 核心架构与 6 大模块划分

为契合 A4 双页 Landscape 布局，我们将 `xv6-riscv` 划分为 6 个核心技术模块，每个模块对应一个莫兰迪色块（M1 ~ M6）：

*   **M1: 引导初始化与进程调度** (Slate Blue - `#7A8B99`)
    *   RISC-V 汇编入口、内核启动、`userinit`、进程状态转换、上下文切换汇编、CFS简化调度器（轮询式调度循环）。
*   **M2: 中断陷入与系统调用** (Moss Green - `#7D8F7B`)
    *   RISC-V 特权态（U/S Mode）转换、Trampoline 与 Trapframe 页机制、内核/用户 Trap 路由、系统调用表分发、UART 与 PLIC 硬中断驱动。
*   **M3: 虚拟内存与页表映射** (Plum Rose - `#9E828A`)
    *   SV39 页面管理、三级页表结构、内核虚空布局与用户虚空布局、Page Walk 地址翻译、写时复制（COW）缺页异常。
*   **M4: 自旋锁与多核同步** (Terracotta - `#B58A7D`)
    *   自旋锁自旋等待机制（依赖 RISC-V 原子指令 `amoswap`）、睡眠锁（Sleeplock）、睡眠与唤醒机制（`sleep` & `wakeup` 竞态防护）、多核共享变量死锁防御。
*   **M5: 块缓存与崩溃一致性日志** (Indigo - `#5F7582`)
    *   块缓存管理（bcache 双向链表与 LRU 回收）、日志事务控制（`begin_op`/`end_op`）、WAL 物理盘写日志、日志 commit 与 install、故障恢复逻辑。
*   **M6: 文件系统与 IPC 管道** (Antique Gold - `#BFA88F`)
    *   Inode 分层（磁盘 inode 与内存 inode）、目录项解析、文件描述符表、管道环形缓冲区（Pipe Sync 同步机制）、系统调用执行。

---

## 3. L0 ~ L2 知识阶梯

### L0 一句话本质
xv6-riscv 是一个教学级 RISC-V 操作系统内核，通过精简的 C 语言与汇编代码，展示了进程特权级隔离（U/S 模式）、三级页表虚拟内存、硬中断路由、基于 Write-Ahead Logging (WAL) 的磁盘日志事务，以及经典 UNIX 管道同步的核心实现。

### L1 四句话逻辑
1.  **特权切换与中断陷入**：进程在用户态（U Mode）运行时通过 `ecall` 触发软中断陷入，跳转至 Trampoline 页（物理与虚拟地址均固定在虚拟空间顶端），保存 CPU 寄存器至 Trapframe 后切入内核态（S Mode）的 `usertrap` 执行。
2.  **双端页表与内存隔离**：每一个进程均独享一个三级（SV39）用户页表，而内核共享一张内核页表。内核在进程切换时，通过修改 `satp` 寄存器加载不同进程的页表基地址并执行 `sfence.vma` 刷新 TLB，确保内存强隔离。
3.  **基于自旋锁与条件变量的并发控制**：采用自旋锁（Spinlock，依赖 RISC-V 原子指令 `amoswap`）保护临界资源；通过 `sleep` 和 `wakeup` 原语在共享的进程等待通道上挂起或唤醒进程，避免死锁与竞态。
4.  **两级缓冲与崩溃一致性日志**：文件系统在磁盘扇区与内存 inode 之间建立两级块缓存（Buffer Cache），并通过内存日志区域（Journal）记录事务操作，所有的磁盘元数据写操作先顺序写入日志区，在 `commit` 时再刷入实际磁盘块，保证系统崩溃后的数据一致性。

### L2 核心数据流转拓扑
```
[User App] 
    │ (ecall)
    ▼
[Trampoline & Trapframe] (寄存器保存 / U➜S)
    │
    ▼
[VFS / File Descriptors] (file.c / sysfile.c)
    │
    ├──────────────────────────────┐ (pipe.c)
    ▼ (fs.c)                       ▼
[Inode Cache (icache)]         [Pipe Ring Buffer]
    │                              │ (sleep/wakeup)
    ▼ (log.c)                      ▼
[Logging Transactions (Journal)] [Reader Process]
    │ (Write-Ahead Log)
    ▼ (bio.c)
[Buffer Cache (bcache)] (LRU List)
    │
    ▼ (virtio_disk.c)
[VirtIO Disk Driver]
    │
    ▼ (Hardware Disk)
[Physical Disk Block]
```

---

## 4. 28张卡片大纲规划

### Page 1 (引导、中断与虚拟内存)

#### M1: 引导初始化与进程调度 (Cards 1-4)
*   **Card 1**: RISC-V 引导入口与多核休眠控制 (`kernel/entry.S` / `kernel/start.c`)
*   **Card 2**: 进程控制块 PCB 布局与进程分配 (`struct proc` / `kernel/proc.c`)
*   **Card 3**: 轮询式进程调度循环机制 (`scheduler()` / `kernel/proc.c`)
*   **Card 4**: 上下文切换汇编解析与寄存器保存 (`swtch.S` / `kernel/proc.c`)

#### M2: 中断陷入与系统调用 (Cards 5-9)
*   **Card 5**: 硬件特权模式与 Trap 向量设置 (`stvec` / `kernel/trap.c`)
*   **Card 6**: 汇编 Trampoline 陷入蹦床页分析 (`kernel/trampoline.S`)
*   **Card 7**: Trapframe 进程中断上下文页 structure (`struct trapframe` / `kernel/proc.h`)
*   **Card 8**: 用户态陷入处理与系统调用路由 (`usertrap()` / `kernel/trap.c`)
*   **Card 9**: PLIC 与 UART 硬件中断分发机制 (`kernel/plic.c` / `kernel/uart.c`)

#### M3: 虚拟内存与页表映射 (Cards 10-13)
*   **Card 10**: SV39 三级页表结构与 Page Walk 地址翻译 (`walk()` / `kernel/vm.c`)
*   **Card 11**: 内核地址空间静态映射与设备内存 (`kvminit()` / `kernel/vm.c`)
*   **Card 12**: 用户虚拟地址空间布局与页分配 (`uvm.c` / `kernel/vm.c`)
*   **Card 13**: 写时复制 (COW) 缺页异常机制原理 (`copy-on-write` 优化)

---

### Page 2 (锁同步、存储事务与文件系统)

#### M4: 自旋锁与多核同步 (Cards 14-17)
*   **Card 14**: 自旋锁实现与 RISC-V 原子汇编指令 (`amoswap` / `kernel/spinlock.c`)
*   **Card 15**: 睡眠锁设计与多核进程独占保护 (`sleeplock` / `kernel/sleeplock.c`)
*   **Card 16**: 睡眠与唤醒管道等待同步机制 (`sleep()` & `wakeup()` / `kernel/proc.c`)
*   **Card 17**: 进程退出与僵尸状态回收机制 (`exit()` & `wait()` / `kernel/proc.c`)

#### M5: 块缓存与崩溃一致性日志 (Cards 18-23)
*   **Card 18**: 双向循环链表块缓存与 LRU 替换算法 (`bcache` / `kernel/bio.c`)
*   **Card 19**: 日志写前记录（WAL）事务机制模型 (`begin_op` / `kernel/log.c`)
*   **Card 20**: 日志物理块顺序持久化机制 (`write_log` / `kernel/log.c`)
*   **Card 21**: 日志提交与崩溃原子保障 (`commit` / `kernel/log.c`)
*   **Card 22**: 内存日志块安装至数据区块流程 (`install_trans` / `kernel/log.c`)
*   **Card 23**: 系统崩溃热重启与日志重放恢复 (`recover_from_log` / `kernel/log.c`)

#### M6: 文件系统与 IPC 管道 (Cards 24-28)
*   **Card 24**: 磁盘 Inode 布局与内存 Inode 高速缓存 (`icache` / `kernel/fs.c`)
*   **Card 25**: 目录查找与硬链接操作实现 (`dirlookup` & `dirlink` / `kernel/fs.c`)
*   **Card 26**: File descriptor 分配与文件表抽象 (`struct file` / `kernel/file.c`)
*   **Card 27**: IPC 管道环形缓冲区与同步读写 (`pipe.c` / `kernel/pipe.c`)
*   **Card 28**: 文件系统调用层与内核边界安全校验 (`argraw` / `kernel/sysfile.c`)
