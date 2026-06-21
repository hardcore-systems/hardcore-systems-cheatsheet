# Scoping & Categorization Document: 《postgres-internals》 (PostgreSQL 存储内核与事务隔离内幕)

本文件定义了《postgres-internals》（PostgreSQL 存储内核与查询规划实现）的底层分析与高密拆解框架，规划 28 张核心卡片在 6 大模块及 Zone T 辅助版块中的物理映射，并确立莫兰迪色彩体系与页面布局参数。

---

## 一、 核心元数据与物理页面预算

*   **目标书籍/项目**：[postgres / postgres](https://github.com/postgres/postgres) 经典学院派开源关系型数据库
*   **拆书定位**：PostgreSQL 多进程模型与共享内存、查询分析重写与 CBO 动态规划、Heap Page 槽位与 Toast 大字段、MVCC 多版本并发与 Auto-Vacuum 回收、WAL 预写日志与两阶段提交 2PC、以及 SSI 串行快照隔离并发控制。
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
| **M1** | **进程模型与连接生命期 (Process Model)** | 石板蓝 (Slate Blue) | `#7A8B99` / `RGB(122,139,153)` | Postmaster 主进程、Backend 单连接进程、共享内存段、pg_bouncer |
| **M2** | **查询规划与执行器 (Optimizer & Executor)** | 苔绿 (Moss Green) | `#7D8F7B` / `RGB(125,143,123)` | Parser/Rewrite 规则系统、CBO 动态规划 Join 路径、Volcano 算子执行、并行 Gather |
| **M3** | **存储引擎与堆页面 (Heap & Index Storage)** | 梅玫瑰 (Plum Rose) | `#9E828A` / `RGB(158,130,138)` | Item line pointer 变长堆页、Toast 物理存储、B-Tree 页面并发分裂、GIN/GiST 索引 |
| **M4** | **事务隔离与并发控制 (ACID & MVCC)** | 陶土红 (Terracotta) | `#B58A7D` / `RGB(181,138,125)` | tuple 头部 xmin/xmax 版本链、Auto-vacuum 回收空间、事务 ID 冻结回绕、排他等锁 |
| **M5** | **预写日志与崩溃恢复 (WAL & recovery)** | 靛青 (Indigo) | `#5F7582` / `RGB(95,117,130)` | LSN 逻辑日志、Checkpointer 脏页均衡写、Hot Standby 流复制、分布式 2PC 提交 |
| **M6** | **高阶优化与诊断监控 (Optimization & Labs)** | 古董金 (Antique Gold) | `#BFA88F` / `RGB(191,168,143)` | Buffer 时钟扫描淘汰、FSM/VM 空间图、LLVM JIT 动态编译、Fdw/pg_stats 诊断 |

---

## 三、 L0 ~ L2 压缩阶梯设计

在海报页眉处放置“PostgreSQL 存储引擎第一性原理公理阶梯”，作为阅读的心智锚点：

*   **L0 一句话本质**：在多进程隔离与共享内存段的物理界限下，通过 CBO 动态规划搜索最优多表物理 Scan-Join 执行树，借由堆页面 tuple 行头 xmin/xmax 字段实现高并发快照隔离，并通过预写日志与 Auto-vacuum 回绕冻结规避强吞吐崩溃的关系型数据库系统。
*   **L1 四句话逻辑**：
    1.  **多进程共享内存架构** (M1) 采用独立的 Backend 后台执行进程隔离客户端，通过共享缓冲区段与 ProcArray 共享全局状态。
    2.  **查询分析重写与规划** (M2) 遍历 Catalog 获取直方图代价，通过 System R 动态规划生成逻辑 Path 并固化为 Volcano 算子执行树。
    3.  **Heap堆页与 Toast 分裂** (M3) 将变长元组通过行指针槽对齐 Heap 页面，超限大字段切片至 Toast 表，B-Tree 索引采用并发右键裂变。
    4.  **xmin/xmax版本与回绕规避** (M4-M5) 在行头封装多版本生命期，依赖 Auto-vacuum 回收老版本并强制 Freeze 以规避 32 位事务 ID 溢出。
*   **L2 存储引擎数据流演进拓扑**：
    *   [SQL 字符串] $\rightarrow$ [Postgres 进程分配] $\rightarrow$ [Parser/Rewrite 规则转换] $\rightarrow$ [CBO 最优物理 Join 路径选择] $\rightarrow$ [Shared Buffers 读入 Heap] $\rightarrow$ [xmin/xmax 读快照过滤] $\rightarrow$ [WAL 写入与 Checkpoint 刷盘] $\rightarrow$ [Auto-vacuum 物理垃圾回收]

---

## 四、 28 张核心卡片物理映射表

每一张卡片代表一个高密度的系统设计考点或机制，包含中文表述与专业西文术语。

### M1：进程模型与连接生命期 (M1_1 至 M1_4) - 【石板蓝】
*   **Card 1 (M1_1)**: Postmaster 与 Backend 多进程架构 (Multi-process Architecture, 每个连接分配独立工作 Backend 进程，提高健壮性)
*   **Card 2 (M1_2)**: 共享内存段物理构成与作用 (Shared Memory Layout, Shared Buffers、WAL Buffers、Lock table 及全局 ProcArray 构成)
*   **Card 3 (M1_3)**: 连接池代理层 pg_bouncer 调优 (Connection Pooling & pg_bouncer, 事务级池 Session vs Transaction，规避进程创建销毁大代价)
*   **Card 4 (M1_4)**: PostgreSQL 前后端 v3.0 通信协议 (Frontend/Backend Protocol, 报文交互，Startup Packet 认证与 Simple/Extended 查询流)

### M2：查询规划与执行器 (M2_1 至 M2_5) - 【苔绿】
*   **Card 5 (M2_1)**: Parser 词法分析与 System Catalog 检索 (Parsing & Catalog Lookup, AST 构建，语义校验关联 pg_class/pg_attribute 核心系统表)
*   **Card 6 (M2_2)**: pg_rewrite 规则系统与视图重写打平 (Rules System pg_rewrite, 视图物理展开，行级安全策略（RLS）动态插入条件)
*   **Card 7 (M2_3)**: 代价优化器 CBO Join 路径规划 (CBO Join Path Selection, 动态规划估算 `seq_page_cost/random_page_cost` 代价，截断左深树)
*   **Card 8 (M2_4)**: Volcano 算子物理执行节点控制 (Executor Operator Nodes, SeqScan/IndexScan/NestLoop/HashJoin 流式 next 递归拉取)
*   **Card 9 (M2_5)**: Parallel Query 并行执行 Gather 节点 (Parallel Query Workers, 动态共享内存（DSM）分发，并行 Scan 算子聚合处理)

### M3：存储引擎与堆页面 (M3_1 至 M3_4) - 【梅玫瑰】
*   **Card 10 (M3_1)**: Heap Page 堆页面物理结构与槽布局 (Heap Page Layout, Item Pointer 行偏移指针数组向右，Tuple 实体向左双向增长)
*   **Card 11 (M3_2)**: Toast 超限大字段切片表空间技术 (TOAST Spillage, 行记录超 2KB 自动触发切片并存入 pg_toast 附属表，免去 Heap Page 溢出)
*   **Card 12 (M3_3)**: nbtpage.c B-Tree 并发右链裂变机制 (B-Tree High-Key & Right-Link, 节点分裂时保留右向指针与 High Key，避免并发读写死锁)
*   **Card 13 (M3_4)**: GIN / GiST / BRIN 多元索引结构应用 (GIN & BRIN Indexes, GIN 倒排索引倒排项树，BRIN 块范围索引针对超大规模顺序写入)

### M4：事务隔离与并发控制 (M4_1 至 M4_4) - 【陶土红】
*   **Card 14 (M4_1)**: tuple 头部 xmin/xmax 与 MVCC 快照读 (Tuple Header xmin/xmax, Tuple 头部标记插入及删除事务 ID，结合 Read View 判断可见性)
*   **Card 15 (M4_2)**: Vacuum 垃圾元组物理空间整理 (Vacuum Space Reclamation, 清理已提交删除的 dead tuples，更新自由空间映射表 FSM，降低空闲洞)
*   **Card 16 (M4_3)**: 事务 ID 冻结与回绕防崩溃机制 (Transaction ID Wraparound & Freeze, 规避 32 位整型溢出，定期触发 Freeze 将旧 XID 标记为 Invalid)
*   **Card 17 (M4_4)**: 锁管理器多模锁加锁优先级与死锁 (Lock Manager Layout, 8种排他级别锁链，等待图 Wait-for-graph 检测环路抛出异常)

### M5：预写日志与崩溃恢复 (M5_1 至 M5_6) - 【靛青】
*   **Card 18 (M5_1)**: WAL 日志物理段结构与 LSN 逻辑映射 (WAL Segments & LSN, 连续字节流写入 WAL segment 文件，LSN 唯一标示物理物理状态变化)
*   **Card 19 (M5_2)**: Checkpointer 背景写脏页分摊算法 (Checkpointer Spreading, 定期同步内存页，使用平滑系数平摊 write I/O 峰值规避 Stall 抖动)
*   **Card 20 (M5_3)**: Crash Recovery 崩溃重做与 Hot Standby (Replay Redo & Hot Standby, 从 Checkpoint 开始重放 WAL，只读节点实时追踪 LSN 同步)
*   **Card 21 (M5_4)**: 物理/逻辑复制插槽 Streaming Replication (Replication Slots, 保留 WAL Segment 不被 Checkpoint 清理，维持主从长连接流式消费)
*   **Card 22 (M5_5)**: PostgreSQL 分布式二阶段提交 2PC (PREPARE TRANSACTION, 生成本地状态文件持久化，确保全局事务原子完成与回滚)
*   **Card 23 (M5_6)**: 可串行化快照隔离 SSI 锁因果链 (Serializable Snapshot Isolation SSI, SIREAD 谓词锁标志读依赖，检测 rw-antidependencies 环)

### M6：高阶优化与诊断监控 (M6_1 至 M6_5) - 【古董金】
*   **Card 24 (M6_1)**: Buffer Buffmgr 时钟循环页面淘汰 (Shared Buffers Clock Sweep, `usage_count` 引用记数环形扫描，大表 Scan 引入环形环缓冲)
*   **Card 25 (M6_2)**: FSM 空间图与 VM 可见性映射物理作用 (FSM & VM Maps, FSM 用于极速定位插入页，VM 用于 Skip Index-Only Scan 的 heap Page 校验)
*   **Card 26 (M6_3)**: LLVM JIT 动态表达式执行编译优化 (LLVM JIT Compile, 动态编译 Target SQL 复杂的 projection 和 filter 表达式，免去解释器开销)
*   **Card 27 (M6_4)**: Foreign Data Wrapper (FDW) 跨库联邦引擎 (Foreign Data Wrapper, 外部数据源挂载，查询物理下推（Push-down）减少网络拷贝)
*   **Card 28 (M6_5)**: pg_stat_statements 全局慢查询性能诊断 (pg_stat_statements, 获取全库核心 SQL 执行频率、均值耗时及缓冲池命中指标)

---

## 五、 辅助版块 Zone T 设计

作为 cheatsheet 底部或侧边的实用工具箱，为 PostgreSQL 内核调优提供即时参数与命令字典参考：

1.  **T1 PostgreSQL 典型参数调优对照表**：
    *   `shared_buffers` (共享内存页缓冲大小)：物理内存的 25% - 40%。
    *   `work_mem` (排序与哈希内存大小)：默认 4MB，大排序/Join 建议根据单个 SQL 评估调高，规避磁盘 I/O 穿透。
    *   `autovacuum_max_workers 3` (自动 Vacuum 进程数)：高并发写大表建议调大至 5-8，规避死元组堆积。
2.  **T2 经典性能监控与诊断 SQL 指令字典**：
    *   诊断事务死锁与锁等待排查：
        ```sql
        SELECT blocked_locks.pid     AS blocked_pid,
               blocking_locks.pid    AS blocking_pid,
               blocked_activity.query AS blocked_statement
        FROM  pg_catalog.pg_locks         blocked_locks
        JOIN pg_catalog.pg_stat_activity blocked_activity ON blocked_activity.pid = blocked_locks.pid
        JOIN pg_catalog.pg_locks         blocking_locks 
          ON blocking_locks.locktype = blocked_locks.locktype AND blocking_locks.database IS NOT DISTINCT FROM blocked_locks.database
          AND blocking_locks.relation IS NOT DISTINCT FROM blocked_locks.relation AND blocking_locks.page IS NOT DISTINCT FROM blocked_locks.page
          AND blocking_locks.tuple IS NOT DISTINCT FROM blocked_locks.tuple AND blocking_locks.virtualxid IS NOT DISTINCT FROM blocked_locks.virtualxid
          AND blocking_locks.transactionid IS NOT DISTINCT FROM blocked_locks.transactionid AND blocking_locks.classid IS NOT DISTINCT FROM blocked_locks.classid
          AND blocking_locks.objid IS NOT DISTINCT FROM blocked_locks.objid AND blocking_locks.objsubid IS NOT DISTINCT FROM blocked_locks.objsubid
          AND blocking_locks.pid != blocked_locks.pid
        JOIN pg_catalog.pg_stat_activity blocking_activity ON blocking_activity.pid = blocking_locks.pid
        WHERE NOT blocked_locks.granted;
        ```
    *   分析表膨胀率与 Dead Tuples 状态：
        ```sql
        SELECT relname, n_dead_tup, n_live_tup, last_vacuum, last_autovacuum 
        FROM pg_stat_user_tables WHERE n_dead_tup > 1000;
        ```
