# Scoping & Categorization Document: 《sqlite-internals》 (SQLite 存储引擎内幕拆解大图)

本文件定义了《sqlite-internals》（SQLite 存储引擎内核实现）的底层分析与高密拆解框架，规划 28 张核心卡片在 6 大模块及 Zone T 辅助版块中的物理映射，并确立莫兰迪色彩体系与页面布局参数。

---

## 一、 核心元数据与物理页面预算

*   **目标书籍/项目**：[sqlite / sqlite-internals](https://github.com/sqlite/sqlite) 经典嵌入式数据库存储引擎
*   **拆书定位**：SQLite 虚拟机 VDBE 字节码、B-Tree 页面结构与 Pager 缓存控制、Rollback Journal 事务保障、WAL 模式读写并发、以及 OS VFS 物理锁机制。
*   **物理页面预算**：**精确 2 页 A4 横向** (Landscape A4, 297mm x 210mm)
*   **LaTeX 布局引擎**：`extarticle` + `multicol` + `tcolorbox` 三列流式自适应排布，0 溢出。
*   **字体配置**：
    *   主要中文字体：`Microsoft YaHei` (微软雅黑) 或 `SimSun` (宋体)
    *   等宽字体：`Courier New` 或 `Consolas`
    *   西文字体：`Arial` 或 `Times New Roman`

---

## 二、 莫兰迪技术色彩系统 (Morandi Palette)

为每个模块定义一个低饱和度、高质感的莫兰迪色彩 Token，以在 LaTeX 海报和 HTML 互动沙盒中保持一致的视觉层级：

| 模块编码 | 模块名称 | 莫兰迪色系 | RGB / Hex 代码 | 视觉语义 |
| :--- | :--- | :--- | :--- | :--- |
| **M1** | **SQL 编译与分析树 (Lemon Parser & AST)** | 石板蓝 (Slate Blue) | `#7A8B99` / `RGB(122,139,153)` | Lemon 语法解析器、AST、代码生成器、Prepared 语句缓存 |
| **M2** | **VDBE 虚拟机与字节码 (VDBE VM & Opcode)** | 苔绿 (Moss Green) | `#7D8F7B` / `RGB(125,143,123)` | 寄存器架构虚拟机、中央解释循环、游标游走操作、Explain 解析 |
| **M3** | **B-Tree 与 Pager 页面引擎 (B-Tree & Pager)** | 梅玫瑰 (Plum Rose) | `#9E828A` / `RGB(158,130,138)` | 叶子与内部节点结构、溢出页指针、PCache 页面缓存、空闲页整理 |
| **M4** | **回滚日志机制 (Rollback Journal)** | 陶土红 (Terracotta) | `#B58A7D` / `RGB(181,138,125)` | `-journal` 日志结构、双阶段事务回滚、物理锁定状态变迁 |
| **M5** | **WAL 模式与并发控制 (WAL Mode & Concurrency)** | 靛青 (Indigo) | `#5F7582` / `RGB(95,117,130)` | `-wal` 与 `-shm` 物理结构、读写并发流、Checkpointer 脏页合并 |
| **M6** | **OS 抽象层与 VFS 锁 (VFS & OS Locks)** | 古董金 (Antique Gold) | `#BFA88F` / `RGB(191,168,143)` | `sqlite3_vfs` 接口抽象、Windows/Unix 锁差异、mmap 内存映射、加密拓展 |

---

## 三、 L0 ~ L2 压缩阶梯设计

在海报页眉处放置“SQLite 存储引擎第一性原理公理阶梯”，作为阅读的心智锚点：

*   **L0 一句话本质**：在单文件物理隔离的限界下，通过声明式 SQL 编译为 VDBE 寄存器字节码流，借由 B-Tree 页与 Pager 缓存抽象 I/O，并利用 Journal 或 WAL 模式下的 VFS OS 锁定机制提供跨进程 ACID 强一致性的轻量级嵌入式关系数据库。
*   **L1 四句话逻辑**：
    1.  **词法解析与代码生成** (M1) 采用定制 Lemon 生成器规避传统 Yacc 重入锁，将 AST 结构优化扁平化并一键生成 VDBE 机器指令。
    2.  **VDBE 寄存器解释机** (M2) 维护游标与伪汇编操作码（Opcode），逐行读取 B-Tree 节点以规避传统 Volcano 模型的函数调用堆栈开销。
    3.  **B-Tree 与 Pager 缓存控制** (M3) 实现自适应溢出页对齐与 PCache 缓存锁定，物理封装异构块大小，向虚拟机提供平坦的键值查询。
    4.  **原子回滚与 WAL 共享内存** (M4 & M5) 配合 VFS OS 文件锁保护数据安全性，实现从独占写锁向 WAL 读写并发及 Checkpoint 的物理演进。
*   **L2 存储引擎数据流演进拓扑**：
    *   [SQL 字符串] $\rightarrow$ [Lemon 语法解析 AST] $\rightarrow$ [Prepared VDBE 字节码缓存] $\rightarrow$ [VDBE 解释器 Loop] $\rightarrow$ [VFS 文件读取锁申请] $\rightarrow$ [PCache 缓冲页映射] $\rightarrow$ [B-Tree Page 槽位定位] $\rightarrow$ [WAL 追写记录] $\rightarrow$ [Checkpoint 定期同步回源数据库]

---

## 四、 28 张核心卡片物理映射表

每一张卡片代表一个高密度的系统设计考点或机制，包含中文表述与专业西文术语。

### M1：SQL 编译与分析树 (M1_1 至 M1_4) - 【石板蓝】
*   **Card 1 (M1_1)**: Tokenizer 与 Lemon 语法解析器 (Lemon Parser Generator, LALR(1) 语法生成器, 线程安全且支持更小的内存堆栈 footprint)
*   **Card 2 (M1_2)**: 代码生成器与查询重写优化 (Code Generator & Query Flattening, AST 物理拍平, 子查询重写合并, 规避冗余逻辑计划生成)
*   **Card 3 (M1_3)**: 索引选择与代价评估模型 (SQLite Cost-Based Index Selector, 计算扫描行数开销, 覆盖索引探测与 Expression Indexes 表达式索引)
*   **Card 4 (M1_4)**: Prepared Statement 缓存机制 (Prepared Statement Cache, 使用 `sqlite3_prepare_v2` 将 SQL 预编译为 VDBE 程序并绑定变量复用)

### M2：VDBE 虚拟机与寄存器机 (M2_1 至 M2_5) - 【苔绿】
*   **Card 5 (M2_1)**: VDBE 核心架构与操作码循环 (VDBE Architecture & Opcode Loop, 寄存器式 VM 代替栈式 VM, 巨型 switch-case 的 `sqlite3_step` 解释循环)
*   **Card 6 (M2_2)**: VDBE 寄存器类型与动态存储 (VDBE Registers & Dynamic Typing, Mem 结构体支持 NULL/INTEGER/REAL/TEXT/BLOB 动态切换及指针复用)
*   **Card 7 (M2_3)**: 游标寻址与 B-Tree 交互操作 (Cursor Operations, `OpenRead/OpenWrite` 分配句柄, `SeekRowid` 索引寻址, `Column` 按列提取)
*   **Card 8 (M2_4)**: 虚拟机子程序与协程调度 (Bytecode Subroutines & Yield, 利用 `Yield/Gosub` 指令实现子流转逻辑, 支持 GROUP BY 与排序暂存区)
*   **Card 9 (M2_5)**: VDBE 字节码诊断与计划分析 (Explain & Explain Query Plan, 使用 `EXPLAIN` 前缀输出 VDBE 机器伪指令, 评估索引匹配度)

### M3：B-Tree 与 Pager 页面引擎 (M3_1 至 M3_4) - 【梅玫瑰】
*   **Card 10 (M3_1)**: B-Tree 物理页面布局 (B-Tree Page Layout, 内部节点存 Key-PageID 单元, 叶子节点存 Key-Data 单元, 槽偏移数组尾向增长)
*   **Card 11 (M3_2)**: 溢出页链表组织 (Overflow Pages, 单单元 Payload 超过阈值时将溢出部分写入链表, 保持 B-Tree 主页面高度与检索扇出率)
*   **Card 12 (M3_3)**: Pager 逻辑层与页面缓存管理 (Pager Layer & PCache, 内存缓冲页的状态管理 (Clean/Dirty/Pinned), 控制页面读写隔离)
*   **Card 13 (M3_4)**: Free-list 空闲页链表与 VACUUM 整理 (Free-list & Compaction, 删除行释放页挂载至 Free-list 头部, `VACUUM` 重排全库文件除碎片)

### M4：回滚日志机制 (M4_1 至 M4_4) - 【陶土红】
*   **Card 14 (M4_1)**: `-journal` 回滚日志物理结构 (Rollback Journal Layout, 日志头部记录原数据库大小, 按页大小保存修改前的原始页面数据)
*   **Card 15 (M4_2)**: 回滚模式事务提交与崩溃恢复 (Rollback Mode Commit & Recovery, 写入 Journal $\rightarrow$ fsync $\rightarrow$ 修改 DB $\rightarrow$ fsync $\rightarrow$ 截断/删除 Journal)
*   **Card 16 (M4_3)**: 回滚日志状态的锁变迁 (Lock Escalation in Rollback Mode, 事务的锁状态从 SHARED $\rightarrow$ RESERVED $\rightarrow$ PENDING $\rightarrow$ EXCLUSIVE 升级变迁)
*   **Card 17 (M4_4)**: 独占写冲突与多进程锁死 (Exclusive Write Concurrency Limit, 回滚模式提交期间 EXCLUSIVE 锁独占文件, 强行阻塞所有进程的读/写)

### M5：WAL 模式与并发控制 (M5_1 至 M5_6) - 【靛青】
*   **Card 18 (M5_1)**: `-wal` 日志文件物理结构 (WAL Log Layout, 包含 WAL Header 及若干 WAL Frame (包含 24 字节 Frame Header 和物理数据页))
*   **Card 19 (M5_2)**: `-shm` 共享内存索引文件 (WAL Shared Memory Index, 利用 `mmap` 映射共享内存文件, 将 WAL 页号建立哈希索引加速读查询)
*   **Card 20 (M5_3)**: WAL 读写并发非阻塞控制 (WAL Read/Write Concurrency, 写操作仅向 WAL 追加, 读操作并发结合主库和 `-shm` 读最新快照)
*   **Card 21 (M5_4)**: Checkpoint 定期脏页回源机制 (Checkpointing, 将 WAL 中最新修改帧按顺序回写至主数据库 `.db` 文件, 并重置 WAL 写入指针)
*   **Card 22 (M5_5)**: WAL 物理写屏障与 fsync 规则 (WAL fsync Rules, 日志写入后 fsync, 检查点回源后主库 fsync, 双重屏障保障 ACID 耐久性)
*   **Card 23 (M5_6)**: 追尾避免与 WAL 重写重置 (WAL Wraparound, 当所有活跃读事务的快照点越过当前 WAL 头部时, 允许安全清空重写 WAL 文件)

### M6：OS 抽象层与 VFS 锁 (M6_1 至 M6_5) - 【古董金】
*   **Card 24 (M6_1)**: VFS 虚拟文件系统抽象层 (VFS Abstraction Layer, `sqlite3_vfs` 定义操作系统文件操作接口, 方便移植嵌入式与内存文件系统)
*   **Card 25 (M6_2)**: Windows 与 Unix 锁定原语物理映射 (Win32 vs POSIX File Locking, Unix 使用 `fcntl` 建议锁, Windows 使用 `LockFileEx` 强制锁的差异)
*   **Card 26 (M6_3)**: 多线程与多进程锁隔离屏障 (Thread vs Process Locks, 内存中的互斥锁 Mutex 用于隔离多线程, 磁盘文件字节锁用于隔离多进程)
*   **Card 27 (M6_4)**: mmap 内存映射物理 I/O (mmap I/O Optimization, 直接将部分数据库文件地址空间映射至用户态, 绕过 OS 缓冲拷贝实现零拷贝读取)
*   **Card 28 (M6_5)**: 加密扩展与物理页安全性加密 (SQLite Encryption Extension SEE, 保持明文头部偏移, 对 B-Tree Page 主体进行异或与 AES 扇区加密)

---

## 五、 辅助版块 Zone T 设计

作为 cheatsheet 底部或侧边的实用工具箱，为系统工程诊断提供即时参数与运维视图参考：

1.  **T1 SQLite 典型参数调优对照表**：
    *   `PRAGMA journal_mode = WAL;` (切换 WAL 模式)：大幅提升写性能和读写并发能力。
    *   `PRAGMA synchronous = NORMAL;` (同步级别)：在 WAL 模式下可配置为 NORMAL，仅在 Checkpoint 时 fsync，大幅规避写卡顿。
    *   `PRAGMA cache_size = -2000;` (页面缓存大小)：负数表示以 KB 为单位（此处约 2MB 缓存），正数表示页数。
2.  **T2 经典性能监控与诊断 PRAGMA 指令字典**：
    *   获取编译后字节码：`EXPLAIN SELECT * FROM users WHERE id = 100;`
    *   分析索引查询计划：`EXPLAIN QUERY PLAN SELECT * FROM orders JOIN users ON orders.uid = users.id;`
    *   完整性健康校验：`PRAGMA integrity_check;`
