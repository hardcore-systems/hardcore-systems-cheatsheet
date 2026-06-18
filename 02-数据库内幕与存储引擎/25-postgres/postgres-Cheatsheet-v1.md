# 《postgres-internals》高密知识图谱与速查手册 (Cheatsheet)

*   **L0 一句话本质**：在多进程隔离与共享内存段的物理界限下，通过 CBO 动态规划搜索最优多表物理 Scan-Join 执行树，借由堆页面 tuple 行头 xmin/xmax 字段实现高并发快照隔离，并通过预写日志与 Auto-vacuum 回绕冻结规避强吞吐崩溃的关系型数据库系统。
*   **L1 四句话逻辑**：
    1.  **多进程共享内存架构** (M1) 采用独立的 Backend 后台执行进程隔离客户端，通过共享缓冲区段与 ProcArray 共享全局状态。
    2.  **查询分析重写与规划** (M2) 遍历 Catalog 获取直方图代价，通过 System R 动态规划生成逻辑 Path 并固化为 Volcano 算子执行树。
    3.  **Heap堆页与 Toast 分裂** (M3) 将变长元组通过行指针槽对齐 Heap 页面，超限大字段切片至 Toast 表，B-Tree 索引采用并发右键裂变。
    4.  **xmin/xmax版本与回绕规避** (M4-M5) 在行头封装多版本生命期，依赖 Auto-vacuum 回收老版本并强制 Freeze 以规避 32 位事务 ID 溢出。
*   **L2 存储引擎数据流演进拓扑**：
    *   [SQL 字符串] $\rightarrow$ [Postgres 进程分配] $\rightarrow$ [Parser/Rewrite 规则转换] $\rightarrow$ [CBO 最优物理 Join 路径选择] $\rightarrow$ [Shared Buffers 读入 Heap] $\rightarrow$ [xmin/xmax 读快照过滤] $\rightarrow$ [WAL 写入与 Checkpoint 刷盘] $\rightarrow$ [Auto-vacuum 物理垃圾回收]

---

## 🌐 经典关系型数据库多进程物理滤镜 (PostgreSQL Epistemic Filter)

- **认识论 (Epistemology) - 以进程物理隔离与操作系统虚拟地址为设计起点**：
  PostgreSQL 认识论的基石在于“进程级强隔离与内存安全第一”。不同于 MySQL 的单进程多线程架构，Postgres 将每个物理连接映射为一个独立的 Backend 进程。进程间绝对的地址空间隔离，决定了它的全局缓冲池（Shared Buffers）与锁管理器必须构建在复杂的 System V 或 POSIX 共享内存段上。这一物理特性使其对单进程崩溃极具免疫力，但也带来了解析系统表、频繁创建进程和进程上下文切换的额外开销。
- **一致性论 (Consistency Theory) - tuple 级多版本生命期与 Auto-vacuum 空间回收**：
  Postgres 的 MVCC 将多版本直接写在堆页面（Heap Page）元组行头部，而不是写入专门的 Undo Log 文件。这种“原位追加新版本”的策略，使得它的更新（UPDATE）在物理上等同于“先 DELETE 再 INSERT”。因此，系统一致性的最大挑战在于物理膨胀与垃圾回收。它依靠 Auto-vacuum 守护进程不断扫描并擦除过期的 Dead Tuples，并利用 FSM 可用空间图进行空间复用，在 32 位 XID 计数溢出前通过强制 Freeze 操作重塑事务时空起点。
- **方法论 (Methodology) - 声明式 AST 向物理 CBO 动态规划的映射推演**：
  Postgres 极其注重学术严谨与优化器代价精确度。其方法论的精髓在于“利用统计信息和直方图穷尽物理 Join 可能性”。优化器从 AST 语法树生成逻辑路径，在计算单表 Scan 代价后，使用 System R 动态规划自底向上进行 Join 排列。它在内存中为多种 Scan（SeqScan, IndexScan, BitmapIndexScan）及 Join（NestLoop, MergeJoin, HashJoin）算子分配物理参数，在可串行化隔离级别下借助可串行化快照隔离（SSI）的谓词锁依赖环算法，保障了完美的执行完整性。

---

## ⚔️ 数据库并发与崩溃恢复折衷矩阵 (Database Concurrency & Recovery Trade-offs)

| 开发者直觉 (⚠) | 系统物理现实 (✗) | 架构折衷与防范金标准 (✓) |
| :--- | :--- | :--- |
| **work_mem 默认大小是极佳的通用配置，不需要根据 SQL 特殊调优** | Postgres 的 Sort 和 HashJoin 完全受 work_mem 内存大小限制。如果排序数据超出 work_mem，会溢出写入磁盘临时文件，导致 I/O 穿透且查询速度断崖下降。 | 用 `EXPLAIN ANALYZE` 监控 SQL 实际使用排序内存。在需要执行大排序、大数据量 Join 的会话中，临时调大 `SET work_mem = '128MB';`，使算子在内存中完成。 |
| **Auto-vacuum 会占用大量磁盘 I/O 吞吐，应该在写入负载高峰期将其关闭** | 关闭 Auto-vacuum 会导致大量死元组（dead tuples）堆积引发表膨胀。更致命的是，这会消耗 XID 计数器，一旦超 20 亿限制，Postgres 会为了防止回绕强制只读停机。 | 绝不允许关闭 Auto-vacuum。调优 `autovacuum_vacuum_cost_delay` 与 `limit` 参数以平摊其 I/O 负荷，并在生产环境中合理设置 Freeze 阈值，以提前安全回绕事务 ID。 |
| **Postgres 拥有完善的 MVCC 快照隔离，写写读读完全互不冲突，默认可串行化** | 可重复读隔离级别下，Postgres 的 MVCC 无法识别跨表或多对象的因果写倾斜（Write Skew）异常，这会导致违反全局一致性约束（如值班医生均下线）。 | 在核心防重业务中，显式将事务设置为串行化隔离 `SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;`，使内核自动启用 SSI 谓词锁定，并在回滚时加入自动重试。 |

---

## 🗺️ 6大核心模块与 28 张高密速查卡片 (Core Cards Map)

### M1：进程模型与连接生命期 (Slate Blue)

#### Card 1. Postmaster 与 Backend 多进程架构 (Multi-process Architecture)
*   **多进程强隔离隔离**：PostgreSQL 采用多进程架构。主进程 `Postmaster` 监听端口，在客户端连入时调用 `fork()` 派生一个独立的 `Backend` 工作进程处理该连接的所有事务。
*   **故障单点隔离**：单个连接进程发生内存越界或挂起崩溃时，仅需终止该后台进程，其他连接不受任何物理影响，保证了系统的极高容错性。

#### Card 2. 共享内存段物理构成与作用 (Shared Memory Layout)
*   **物理共享页表段**：多进程依靠物理共享内存（Shared Memory）通信。
*   **共享控制区**：共享内存主要划分为 `Shared Buffers`（共享页面缓冲区）、`WAL Buffers`（预写日志缓冲区）、全局锁管理器（Lock Table）、事务状态记录（Commit Log, CLOG）以及维护全局进程状态的 `ProcArray`。

#### Card 3. 连接池代理层 pg_bouncer 调优 (Connection Pooling)
*   **进程创建高昂开销**：由于 Postgres 每个连接都是 OS 进程，创建和销毁连接需要进行繁重的页表拷贝与系统调用。
*   **事务级连接池复用**：生产中必须配置 `pg_bouncer`。其事务级池（Transaction Mode）能在事务结束后立即释放 Backend 给其他客户端复用，规避进程上限导致 CPU 耗尽在上下文切换中。

#### Card 4. PostgreSQL 前后端 v3.0 通信协议 (Frontend/Backend Protocol)
*   **网络报文规范**：PG 采用 Frontend/Backend 3.0 协议。连接握手时发送 Startup Message，包含库名与认证。
*   **两类查询通信**：
    1.  **Simple Query**：发送 SQL 文本，返回 RowDescription 和若干 DataRow，最后以 ReadyForQuery 闭环。
    2.  **Extended Query**：通过 Prepare、Bind、Execute 三阶段报文，支持二进制传输参数。

---

### M2：关系查询处理器与优化器 (Moss Green)

#### Card 5. Parser 词法分析与 System Catalog 检索 (Parsing & Catalog Lookup)
*   **AST 语法树构建**：Parser 对输入的 SQL 文本进行分词与词法分析，生成抽象语法树（AST）。
*   **Catalog 语义校验**：Analyzer 遍历 AST，检索 System Catalog 系统表（如 `pg_class`、`pg_attribute`），校验表名、列名和数据类型是否存在，输出 Query 结构。

#### Card 6. pg_rewrite 规则系统与视图重写打平 (Rules System pg_rewrite)
*   **宏展开与替换规则**：Rewriter 接收 Query 结构，利用存储在 `pg_rewrite` 系统表中的规则，将针对视图（View）的查询进行扁平化宏重写。
*   **RLS 行安全插入**：在此阶段处理行级安全策略（Row-Level Security, RLS），动态在 AST 的 Where 条件中追加当前登录用户的筛选子句。

#### Card 7. 代价优化器 CBO Join 路径规划 (CBO Join Path Selection)
*   **代价公式判定**：CBO（基于代价优化器）基于单表 Scan 代价，计算生成物理 Path 的代价。
*   **System R 剪枝算法**：采用 System R 动态规划算法自底向上搜索多表 Join 的连接顺序。通过配置 `seq_page_cost` 和 `random_page_cost` 指导优化器在 IndexScan 与 SeqScan 之间做出最优代价决策，自动裁剪低效左深树。

#### Card 8. Volcano 算子物理执行节点控制 (Executor Operator Nodes)
*   **火山迭代树式调用**：执行器（Executor）接收物理 Plan 树。每个算子（如 HashJoin, SeqScan）均实现统一接口。
*   **自下而上流式拉取**：顶级节点递归调用子算子的 `ExecProcNode()`。子算子按需流式拉取一行记录（Tuple）并返回，将内存占用限制在单行级别，实现了计算流式直通。

#### Card 9. Parallel Query 并行执行 Gather 节点 (Parallel Query Workers)
*   **并行计算网格分发**：当优化器判断全表扫描代价过高时，会启动 Parallel Plan。
*   **DSM 内存数据聚合**：主 Backend 进程启动 `Gather` 算子，派生出多个 Parallel Worker 进程，利用动态共享内存（DSM）段分配并行 Scan 页，各 Worker 独立计算并由 Gather 汇总，大幅释放多核 CPU 算力。

---

### M3：存储引擎与堆页面 (Plum Rose)

#### Card 10. Heap Page 堆页面物理结构与槽布局 (Heap Page Layout)
*   **槽位指针双向对齐**：PG 默认页大小为 8KB。页面头部包含 PageHeaderData 结构，包含页元数据。
*   **堆行数据反向增长**：行的行指针（ItemId/Line Pointer，4 字节）从页头向右顺序增长；实际的 Tuple 元组数据则从页尾向左反向增长。删除元组时只需将对应 ItemId 标为 Unused 并在页面整理（Page Repair）时平移元组。

#### Card 11. Toast 超限大字段切片表空间技术 (TOAST Spillage)
*   **堆页大小物理限制**：由于 8KB 堆页限制，Postgres 规定单行元组大小不得超过约 2KB。
*   **附属切片行外存储**：当字段（如大文本 TEXT）过大时，触发 TOAST 机制。将数据压缩后切割成若干个不超过 2KB 的 Chunk，物理存放到专属的 Toast 表中，主 Heap 页面仅保留 18 字节的 OID 指针。

#### Card 12. nbtpage.c B-Tree 并发右链分裂机制 (B-Tree High-Key & Right-Link)
*   **无锁读并发分裂**：Postgres 的 B-Tree 索引页面分裂采用 Lehman & Yao 算法。
*   **右链接与 High Key**：分裂时并不立即加锁父节点，而是先在页头部写入 `Right-Link` 指针指向分裂出的新右页，并记录 `High Key`。读进程如果发现目标 Key 大于 High Key，可以直接沿 Right-Link 读向右页，避免了全局加锁阻塞。

#### Card 13. GIN / GiST / BRIN 多元索引结构应用 (GIN & BRIN Indexes)
*   **异构数据查询优化**：
    1.  **GIN (倒排索引)**：针对数组、JSONB 和全文检索，内部为每个元素维护一个 B-Tree 索引项，指向所有包含该元素的行。
    2.  **BRIN (块范围索引)**：针对时间序列等顺序写入大表，仅记录每 128 页（默认）的最大最小值，索引大小只有 B-Tree 万分之一，极大减少了 I/O 扫描。

---

### M4：事务隔离与并发控制 (Terracotta)

#### Card 14. tuple 头部 xmin/xmax 与 MVCC 快照读 (Tuple Header xmin/xmax)
*   **行头生命周期字段**：每个堆元组头部包含 `t_xmin`（插入该元组的事务 ID）和 `t_xmax`（删除或更新该元组的事务 ID）。
*   **读视图可见性校验**：只读事务在启动时获取当前活动事务快照（Snapshot Data），包含活跃 XID 列表。读取元组时，根据行头的 xmin/xmax 判定该元组对当前快照已提交还是未提交，实现无锁快照读。

#### Card 15. Vacuum 垃圾元组物理空间整理 (Vacuum Space Reclamation)
*   **死元组占位问题**：由于 UPDATE 是在 Heap 页追加新版本并设置老版本 xmax，长时间运行会导致大量 Dead Tuples 堆积引发表膨胀。
*   **原地擦除空闲整理**：`VACUUM` 扫描堆页面，物理擦除无用 Dead Tuples，并更新 Free Space Map (FSM)，供后续写入新 Tuple 复用。但它不会物理缩容文件大小（除非使用排他的 `VACUUM FULL`）。

#### Card 16. 事务 ID 冻结与回绕防崩溃机制 (Transaction ID Wraparound & Freeze)
*   **32位计数器溢出风险**：事务 ID（XID）是 32 位循环整型。两个 XID 大小比较通过模 $2^{32}$ 运算，一旦超过 20 亿差值，系统判定新老事务逻辑颠倒，发生“回绕（Wraparound）”。
*   **页冰冻机制**：Auto-vacuum 定期对旧页触发 Freeze 扫描，将行头 xmin 替换为已冻结标记 `FrozenTransactionId`（即 2），使其在物理上永远对所有新事务可见，消除了回绕崩溃风险。

#### Card 17. 锁管理器多模锁加锁优先级与死锁 (Lock Manager Layout)
*   **多模等锁哈希表**：锁管理器维护全局哈希表。支持 8 种锁模式（如 RowShare, AccessExclusive），锁模式之间有详细的互斥冲突矩阵。
*   **有向图死锁检测**：一旦 Backend 申请锁被阻塞，进入 Wait 队列。后台线程启动 `DeadlockTimeout`，构建锁等待有向图。一旦检测到环路，强制回滚（Abort）当前事务以释放锁。

---

### M5：预写日志与崩溃恢复 (Indigo)

#### Card 18. WAL 日志物理段结构与 LSN 逻辑映射 (WAL Segments & LSN)
*   **段文件追加刷盘**：WAL 记录包含对数据的物理修改增量，连续追加写入 16MB 大小的 WAL segment 文件中。
*   **LSN 字节流偏移**：每一条 WAL 记录拥有一个 64 位无符号整型 LSN（Log Sequence Number），表示该记录在 WAL 字节流中的绝对偏移量，是崩溃恢复和流复制的物理时间轴。

#### Card 19. Checkpointer 背景写脏页分摊算法 (Checkpointer Spreading)
*   **Checkpoint 停顿停摆**：如果 Checkpointer 瞬时将内存所有脏页写入磁盘，会导致宿主机 I/O 被打满发生严重 Stall 抖动。
*   **平滑写系数平摊**：Postgres 引入 `checkpoint_completion_target`（默认 0.9）。Checkpointer 计算上次 checkpoint 到现在的时间差，将脏页写入平摊到 90% 的周期内执行，平滑了磁盘 I/O。

#### Card 20. Crash Recovery 崩溃重做与 Hot Standby (Replay Redo & Hot Standby)
*   **前向扫描重放历史**：系统崩溃重启后，Startup 进程读取控制文件，找到最后一个 Checkpoint，开始正向扫描 WAL 日志并重放 Redo 修改，直到 LSN 追平。
*   **只读流复制同步**：Hot Standby 备库启动只读服务，通过不断接收主库发送的 WAL 日志并物理重放，实现与主库的毫秒级数据同步。

#### Card 21. 物理/逻辑复制插槽 Streaming Replication (Replication Slots)
*   **备机延迟导致的日志清理**：传统的 Streaming Replication，若备机断开，主机可能会在 Checkpoint 期间将备机需要的旧 WAL 文件物理清理掉。
*   **复制插槽锁定 LSN**：引入 Replication Slot。插槽在主机上锁定备机当前的最小读取 LSN，强制主机保留该 LSN 之后的所有 WAL segment 文件，直到备机重新连入同步。

#### Card 22. PostgreSQL 分布式二阶段提交 2PC (PREPARE TRANSACTION)
*   **跨节点状态对齐**：支持分布式 2PC 提交。
*   **本地状态文件固化**：执行 `PREPARE TRANSACTION 'xid'` 时，Backend 进程将该事务的锁状态和修改的 CLOG 强制写入本地 `pg_twophase` 目录的物理状态文件中。此时即使进程重启，锁和状态依旧保留，直到后续收到 `COMMIT PREPARED` 广播。

#### Card 23. 可串行化快照隔离 SSI 锁因果链 (Serializable Snapshot Isolation SSI)
*   **写倾斜无锁检测**：SSI 在执行时不加任何物理阻塞锁，依靠读写反依赖关系（rw-antidependencies）判定写倾斜风险。
*   **因果依赖有向环**：内核在共享内存中维护 `SIREAD` 虚拟锁。一旦发现 $T1 \xrightarrow{rw} T2 \xrightarrow{rw} T3$ 这种双重读写反依赖闭环，判定发生串行化异常，强制 Abort 并回滚正在提交的事务。

---

### M6：高阶优化与调试监控 (Antique Gold)

#### Card 24. Buffer Buffmgr 时钟循环页面淘汰 (Shared Buffers Clock Sweep)
*   **引用自减淘汰指针**：Postgres 的共享内存缓冲池使用 `usage_count`（使用计数，0~5）来管理淘汰。
*   **大表 Scan 环形缓冲**：时钟指针循环检查页面，非 0 页面使用计数减 1，为 0 且未被 Pin 锁定的页面被直接驱逐。为了防止大表全表扫描污染整个缓冲池，PG 引入 Bulk Read 环形缓冲区机制，仅在固定的少量页面内循环替换。

#### Card 25. FSM 空间图与 VM 可见性映射物理作用 (FSM & VM Maps)
*   **自由空间定位**：`FSM` (Free Space Map) 文件记录堆页面的可用字节档位，写入时可微秒级定位到哪个物理页有空闲插入元组，避免全文件顺序查找。
*   **可见性物理标记**：`VM` (Visibility Map) 标记哪些页面的元组对于当前所有事务均完全可见。Auto-vacuum 可以直接跳过已全部可见的页面扫描，且 Index-Only Scan 可以通过 VM 直接判断，无需二次读取 Heap Page 验证。

#### Card 26. LLVM JIT 动态表达式执行编译优化 (LLVM JIT Compile)
*   **复杂 SQL 解释开销**：当 SQL 包含多层 Projection、Filter 表达式计算时，传统的解释器在 CPU 级有大量的分支预测失败和循环跳转开销。
*   **LLVM 运行时机器码生成**：Postgres 集成 LLVM。在规划期动态编译这些表达式生成本地 CPU 机器码指令，在执行阶段直接由 CPU 闭环执行，性能提升数倍。

#### Card 27. Foreign Data Wrapper (FDW) 跨库联邦引擎 (Foreign Data Wrapper)
*   **异构库表透明挂载**：Postgres 符合 SQL/MED 标准。利用 FDW 可以将 MySQL, Oracle, MongoDB 等外部表挂载为本地虚拟表。
*   **优化器算子下推**：在执行查询时，CBO 能够分析外部数据源的特性，将 Where 过滤、Limit 以及部分的 Join 算子物理下推到远端数据库执行，最小化网络传输消耗。

#### Card 28. pg_stat_statements 全局慢查询性能诊断 (pg_stat_statements)
*   **查询执行统计库**：PG 官方提供的性能诊断扩展。能够将全库执行的 SQL 进行参数泛化归类。
*   **诊断分析视图**：实时收集每种 SQL 的调用次数（calls）、总执行时间、共享缓冲池命率和 I/O 读写时长，是定位慢 SQL、分析缓冲命中瓶颈的终极诊断利器。

---

## 🔬 辅助版块 Zone T：生产诊断与调试字典 (Zone T)

### T1 PostgreSQL 典型参数调优对照表

| 配置参数 / postgresql.conf | 经典建议数值 | 物理调优语义 |
| :--- | :--- | :--- |
| `shared_buffers` | `25% - 40% 物理内存` | 共享页面缓冲区大小。不建议设置过高（如 80%），因为 Postgres 极其依赖操作系统页缓存（Double Buffering）实现高效读取。 |
| `work_mem` | `4MB - 64MB (按会话调优)` | 排序与哈希算子排序大小。为防止大量并发导致 OOM，全局默认不宜过大。需要大排序的复杂 SQL 可在事务内临时调高。 |
| `maintenance_work_mem` | `128MB - 1GB` | 创建索引和 VACUUM 维护操作专用的物理内存空间。调大此参数能显著缩短大规模索引创建时间。 |
| `autovacuum_max_workers` | `3 - 8 (大写入调大)` | 自动 Vacuum 的工作并发进程数。高频写场景下建议调高，否则单进程清理无法跟上 Dead Tuples 生成速度。 |

### T2 经典性能监控与诊断 SQL 指令字典

*   **1. 诊断实时事务死锁与锁等待排查**
    ```sql
    -- 查看当前被阻塞的进程、阻塞源进程 PID 以及被阻塞的 SQL 语句文本，方便执行 pg_terminate_backend 杀掉阻断源
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
*   **2. 分析全局用户表的膨胀与 Dead Tuples 状态**
    ```sql
    -- 获取 Dead Tuples 数量最多的前 10 张表，排查 autovacuum 是否落后或失效
    SELECT relname, n_dead_tup, n_live_tup, last_vacuum, last_autovacuum 
    FROM pg_stat_user_tables 
    ORDER BY n_dead_tup DESC LIMIT 10;
    ```
