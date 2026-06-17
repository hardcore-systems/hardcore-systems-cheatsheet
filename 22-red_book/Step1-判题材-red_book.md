# Scoping & Categorization Document: 《architecture-of-a-database-system》 (数据库系统架构设计大图)

本文件定义了《architecture-of-a-database-system》(简称 The Red Book) 的底层分析与高密拆解框架，规划 28 张核心卡片在 6 大模块及 Zone T 辅助版块中的物理映射，并确立莫兰迪色彩体系与页面布局参数。

---

## 一、 核心元数据与物理页面预算

*   **目标书籍/项目**：[architecture-of-a-database-system](https://github.com/josephmhellerstein/db-book) 经典“红书”——数据库系统架构
*   **拆书定位**：数据库核心架构与底层机制版（专注于进程线程并发模型、关系查询处理器与优化器、存储引擎管理器与 Buffer Pool 调度、SS2PL 并发锁控制、ARIES 崩溃恢复协议、以及 Shared-Nothing 并行分布式架构）
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
| **M1** | **进程与线程模型 (Process Models)** | 石板蓝 (Slate Blue) | `#7A8B99` / `RGB(122,139,153)` | 进程/线程模型、工作线程池调度、共享内存并发控制 |
| **M2** | **关系查询处理器 (Query Processor)** | 苔绿 (Moss Green) | `#7D8F7B` / `RGB(125,143,123)` | 语法解析、CBO 代价优化器、Volcano 迭代器、向量化执行 |
| **M3** | **存储管理器与缓冲 (Storage & Buffer)** | 梅玫瑰 (Plum Rose) | `#9E828A` / `RGB(158,130,138)` | 槽页存储结构、Buffer Pool Clock 淘汰、行存与列存物理对比 |
| **M4** | **事务与并发控制 (Concurrency Control)** | 陶土红 (Terracotta) | `#B58A7D` / `RGB(181,138,125)` | 严格两阶段锁 SS2PL、锁管理器与等待图死锁、MVCC 隔离机制 |
| **M5** | **预写日志与恢复 (ARIES Recovery)** | 靛青 (Indigo) | `#5F7582` / `RGB(95,117,130)` | WAL 协议、ARIES 恢复（分析、重做、撤销三阶段）、模糊检查点 |
| **M6** | **并行与分布式数据库 (Distributed DB)** | 古董金 (Antique Gold) | `#BFA88F` / `RGB(191,168,143)` | 共享磁盘/内存/无共享架构、分布式二阶段提交 2PC、分布式 Join |

---

## 三、 L0 ~ L2 压缩阶梯设计

在海报页眉处放置“数据库架构第一性原理公理阶梯”，作为阅读的心智锚点：

*   **L0 一句话本质**：在非易失性存储与易失性内存的物理限界下，通过并发控制与预写日志保证 ACID 强一致约束，并借由查询优化器与存储引擎最大化 I/O 与 CPU 吞吐的关系型计算系统。
*   **L1 四句话逻辑**：
    1.  **并发工作流调度** (M1) 通过进程/线程池模型降低上下文切换开销，配合逻辑锁（Latch/Mutex）隔离多线程内存竞态。
    2.  **查询处理器优化** (M2) 建立逻辑计划并利用 CBO 代价模型搜索生成最优物理执行树，通过 Volcano 迭代器或向量化流式计算。
    3.  **缓冲池 Clock 淘汰与槽页** (M3) 将物理磁盘页组织为变长槽页并在缓存中 Pin/Unpin 锁页，平滑屏蔽随机 I/O 延迟。
    4.  **SS2PL锁并发与 ARIES 恢复** (M4 & M5) 构成事务的一致性防线，通过锁等待图打破死锁，利用 WAL 日志三阶段分析回滚恢复崩溃状态。
*   **L2 知识大图**：
    *   [M1: 工作线程调度] -> [M2: SQL解析/CBO优化器/Volcano迭代器] -> [M3: 缓冲池/槽页存储] -> [M4: SS2PL锁管理器/MVCC/事务] -> [M5: WAL日志/ARIES恢复] -> [M6: Shared-Nothing/分布式2PC]

---

## 四、 28 张核心卡片物理映射表

每一张卡片代表一个高密度的系统设计考点或机制，包含中文表述与专业西文术语。

### M1：进程与线程模型 (M1_1 至 M1_4) - 【石板蓝】
*   **Card 1 (M1_1)**: 每连接单进程模型 (Process-per-Connection Model, PostgreSQL 多进程共享内存与操作系统页表开销)
*   **Card 2 (M1_2)**: 每连接单线程模型 (Thread-per-Connection Model, MySQL 一线程一连接模型、线程上下文切换与堆栈碎片)
*   **Card 3 (M1_3)**: 工作线程池调度模型 (Worker Pool Scheduling, 线程池规避线程爆满、异步多路复用网络请求)
*   **Card 4 (M1_4)**: 线程级锁与内存保护屏障 (Latch vs Mutex, 读写锁与排他锁对内存缓冲池及索引页的轻量级保护)

### M2：关系查询处理器 (M2_1 至 M2_5) - 【苔绿】
*   **Card 5 (M2_1)**: SQL 解析与查询重写 (SQL Parsing & Rewriting, AST 语法树生成、语义校验与视图扁平化重写)
*   **Card 6 (M2_2)**: 基于代价的查询优化器 (Cost-Based Optimizer, 统计信息收集、单表 Scan 算子代价估算机制)
*   **Card 7 (M2_3)**: System R 优化路径决策模型 (System R Dynamic Programming Join Plan, 规避笛卡尔积的左深树与多表 Join 连接排序)
*   **Card 8 (M2_4)**: Volcano 火山迭代器执行模型 (Volcano Iterator Model, 算子流式 `open-next-close` 物理调用链约束)
*   **Card 9 (M2_5)**: 向量化执行与 JIT 即时编译 (Vectorized Execution vs JIT, 批处理数组循环与动态机器码生成消除虚函数调用)

### M3：存储管理器与缓冲 (M3_1 至 M3_4) - 【梅玫瑰】
*   **Card 10 (M3_1)**: 变长记录槽页存储布局 (Slotted-Page Storage Layout, 页尾数据向页头增长、页头偏移数组物理对齐)
*   **Card 11 (M3_2)**: 缓冲池 Clock 页面淘汰算法 (Clock Replacement Policy, 使用使用位与指针循环替代严格 LRU 链表锁竞争)
*   **Card 12 (M3_3)**: 行式存储 (NSM) 与列式存储 (DSM) (Row-Store vs Column-Store, OLTP 短事务点查与 OLAP 大吞吐聚集检索物理折衷)
*   **Card 13 (M3_4)**: 缓冲池异步预读与双缓冲 (Prefetching & Double Buffering, 底层 I/O 线程预先载入相邻页与 OS 缓存绕过)

### M4：事务与并发控制 (M4_1 至 M4_5) - 【陶土红】
*   **Card 14 (M4_1)**: 严格两阶段锁协议 (SS2PL) (Strict Two-Phase Locking, 锁增长期与锁收缩期, 事务提交前强行持有排他锁防脏读)
*   **Card 15 (M4_2)**: 锁管理器物理哈希表结构 (Lock Manager Layout, 资源 ID 哈希至锁头部、锁模式与授予/等待队列双向链表)
*   **Card 16 (M4_3)**: 锁升级与死锁环路检测 (Lock Escalation & Deadlock, 细粒度行锁汇聚升级为表锁、等待图有向环路搜索)
*   **Card 17 (M4_4)**: MVCC 多版本链与快照隔离 (MVCC Version Chain, Undo Log 构建物理版本链、活动事务 Read View 视图)
*   **Card 18 (M4_5)**: 乐观并发控制 (OCC) 验证机制 (OCC Validation, 读-校验-写三阶段, 校验只读集与写集合冲突判定)

### M5：预写日志与恢复 (M5_1 至 M5_5) - 【靛青】
*   **Card 19 (M5_1)**: Write-Ahead Logging (WAL) 预写协议 (WAL Protocol, 物理修改先刷日志、事务提交必须 fsync 日志页强保)
*   **Card 20 (M5_2)**: ARIES 恢复分析阶段逻辑 (ARIES Analysis Phase, 扫描日志重建脏页表 DPT 与活跃事务表 TT 获取恢复起点)
*   **Card 21 (M5_3)**: ARIES 恢复重做阶段逻辑 (ARIES Redo Phase, 历史日志物理重放, 无论事务是否提交强行追平磁盘数据状态)
*   **Card 22 (M5_4)**: ARIES 恢复撤销阶段逻辑 (ARIES Undo Phase, 逆序反向执行未提交事务的回滚, 写入 Compensation Log Record 补偿日志)
*   **Card 23 (M5_5)**: 模糊检查点机制原理 (Fuzzy Checkpointing, 定期写入脏页表与活跃事务表元数据, 不阻塞物理写操作)

### M6：并行与分布式数据库 (M6_1 至 M6_5) - 【古董金】
*   **Card 24 (M6_1)**: 共享内存架构物理限制 (Shared-Memory Architecture, 对称多处理器 SMP 总线竞争、内存一致性硬件瓶颈)
*   **Card 25 (M6_2)**: 共享磁盘架构锁协调代价 (Shared-Disk Architecture, 物理节点共享 SAN 存储、节点间缓存一致性协议分布式锁开销)
*   **Card 26 (M6_3)**: 无共享架构水平扩展原理 (Shared-Nothing Architecture, 节点完全独立仅通过网络进行分区通信、强水平伸缩性)
*   **Card 27 (M6_4)**: 分布式二阶段提交协议 (Two-Phase Commit, 协调者与参与者 CanCommit 问询与 DoCommit 物理广播提交)
*   **Card 28 (M6_5)**: 分布式查询数据重分布与 Join (Data Redistribution & Distributed Join, 广播 Join 与重分区 Hash Join 抉择)

---

## 二、 辅助版块 Zone T 设计

作为 cheatsheet 底部或侧边的实用工具箱，为系统工程诊断提供即时数值与参数参考：

1.  **T1 关系型数据库典型参数调优对照表**：
    *   `shared_buffers` (共享内存缓冲区大小)：推荐为物理内存的 25% - 40%。
    *   `max_connections` (最大连接数)：过多连接导致线程上下文切换，需配置连接池。
    *   `lock_timeout` (锁超时阈值)：防止全局死锁阻塞的长事务超时保护设定。
2.  **T2 经典性能监控 SQL 常用视图对照**：
    *   PostgreSQL 活动状态监控：`SELECT pid, query, state FROM pg_stat_activity WHERE state != 'idle';`
    *   PostgreSQL 锁竞争排查：`SELECT pid, locktype, mode, granted FROM pg_locks;`
    *   MySQL InnoDB 引擎内幕诊断：`SHOW ENGINE INNODB STATUS;`
