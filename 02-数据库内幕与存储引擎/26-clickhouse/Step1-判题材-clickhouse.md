# ClickHouse Internals Scoping & Card Planning Document

This document defines the scoping and plan for candidate **ClickHouse / ClickHouse** (OLAP Database Internals). It contains the 6 modules, L0~L2 ladders, and plans the 28 cards.

---

## 1. 题材判研 (Subject Evaluation)

ClickHouse is the gold standard for column-oriented OLAP databases. It is highly complex, utilizing low-level C++ templates, SIMD instruction sets, vectorization, sparse indexing, and partition merging. Its internals represent a high-value domain for cheatsheet compilation.

---

## 2. L0~L2 核心本质与高层逻辑 (L0~L2 Core Ladders)

*   **L0 一句话本质**：通过列式物理存储与压缩块设计，将批量数据映射为 8192 行粒度的稀疏索引标记，利用 CPU SIMD 寄存器级向量化操作在大批量 Column 块上并行过滤与聚合，依靠异步 MergeTree 引擎及 ZooKeeper 协同保障大规模数据分片副本最终一致性的极速列存 OLAP 引擎。
*   **L1 四句话逻辑**：
    1.  **分布式查询与网络协议** (M1) 支持原生 TCP 与 HTTP，通过两阶段分布式查询在各分片 (Shard) 上并发下推算子并由 Coordination 节点汇总。
    2.  **向量化执行与 SIMD** (M2) 放弃单行 Volcano 模型，改用 Column 块作为迭代单元，用 CPU 硬件级的 SIMD 向量化指令提升单核数据计算吞吐。
    3.  **MergeTree 存储引擎** (M3) 采用 LSM-Tree 变体，将追加写入的数据段按分区 (Partition) 物理落盘，并在后台异步进行 Merge 归并排序合并。
    4.  **稀疏索引与压缩块** (M4-M6) 配合 `primary.idx` 和 mark 标记文件以 8192 行为 Mark 粒度检索，通过 Block 级列式压缩及多种编码技术极大压榨 I/O。

---

## 3. 6大模块与 28 张卡片规划 (6 Modules & 28 Cards Plan)

### M1：分布式查询与网络 (Connection & Distributed Queries)
*   **Card 1. TCP/HTTP 连接与原生协议**：ClickHouse 物理连接层，如何解析报文并分配查询上下文。
*   **Card 2. 分布式查询双阶段执行**：Coordinator 节点解析分布式表，分发执行树至 Remote Shards 的物理流程。
*   **Card 3. Cluster 与 Shard 拓扑路由**：通过配置中 Cluster 的 weight 与 random 路由，下发节点与本地表的映射。
*   **Card 4. 外部表透明挂载与 FDW 下推**：ClickHouse-mysql/odbc/file FDW 将外部数据拉取与算子物理下推。

### M2：向量化执行与 SIMD (Vectorized Execution & SIMD)
*   **Card 5. Column 物理表示与 Memory 对齐**：列数据在内存中以连续 Array/Vector 表示的物理对齐细节。
*   **Card 6. Vectorized Execution 向量化循环**：放弃 Volcano，按块（Block）在 for 循环中流式流转数据。
*   **Card 7. CPU SIMD 指令集硬加速**：ClickHouse 如何自动或显式使用 SSE4.2/AVX2/AVX512 指令实现向量过滤。
*   **Card 8. Projection / Materialized View 动态更新**：投影列与物化视图在主表 Merge 写入时的同步刷写。
*   **Card 9. 内存管理 Memory Tracker 与 OOM 防范**：ClickHouse 严格的线程级与查询级内存上限检测机制。

### M3：MergeTree 存储引擎 (MergeTree Storage Engine)
*   **Card 10. MergeTree 物理目录与 Part 结构**：`data/db/table/` 目录下分区的 Part 数据文件构成（bin/mrk/txt）。
*   **Card 11. 异步后台 Merge 归并合并算法**：后台 Merge 线程如何挑选 Parts 并执行归并排序物理重构。
*   **Card 12. TTL 声明周期过期与物理删除**：行级与列级 TTL 定义，在 Merge 阶段如何清除过期数据。
*   **Card 13. Mutation 机制与异步非原子修改**：ClickHouse 的 UPDATE/DELETE 的 Mutation 机制与物理合并开销。

### M4：稀疏索引与分区键 (Sparse Index & Partitions)
*   **Card 14. Primary Key 稀疏索引设计**：`primary.idx` 中以 index_granularity (默认 8192 行) 采样建立索引。
*   **Card 15. Mark 标记文件与数据定位**：`.mrk` 文件建立主键 Mark 到 `.bin` 数据文件物理偏移量的双向指针映射。
*   **Card 16. MinMax 索引与分区键过滤**：`minmax_[part].idx` 用分区键对 Part 级进行初过滤。
*   **Card 17. Skip Index 跳数索引设计与优化**：`minmax` / `set` / `bloom_filter` 二级跳数索引的作用域与粒度过滤。

### M5：副本复制与 ZooKeeper 协同 (Replication & ZooKeeper Coordination)
*   **Card 18. ReplicatedMergeTree 架构本质**：如何通过 Replicated 表引擎在 ZooKeeper 注册副本节点。
*   **Card 19. Replica 间日志同步与 ZooKeeper 事务**：`/clickhouse/tables/` 路径下 log 队列的同步监听。
*   **Card 20. Queue 处理与 Parts 下载流水线**：备副本如何解析 replication queue 并通过 HTTP 从主副本拉取 Part。
*   **Card 21. 分布式 DDL ON CLUSTER 物理过程**：ON CLUSTER 语句在 ZooKeeper `/query_log` 注册与各副本消费。
*   **Card 22. Keeper (ClickHouse Keeper) 代替 ZK**：RAFT 协议的 CH Keeper 替代传统 ZooKeeper 的一致性。
*   **Card 23. 分区合并冲突与裂脑恢复**：ZooKeeper 丢失连接或网络分区时，副本变 Read-only 的防脑裂策略。

### M6：数据物理布局与压缩编码 (Block Layout & Compression)
*   **Card 24. Chunk 数据块列式物理对齐**：Block 由 Column 块加 Header 元数据组成，作为 I/O 读写物理单元。
*   **Card 25. Compression Block 压缩块逻辑**：`.bin` 内部是由压缩头、未压缩长度、已压缩字节构成的 block 链。
*   **Card 26. LZ4 / ZSTD 编码与参数调优**：适合 OLAP 大宽表的 LZ4 (低延迟) 与 ZSTD (高压缩比) 的物理边界。
*   **Card 27. Specialized Codecs 物理编码**：`DoubleDelta` / `T64` / `Gorilla` 编码对数值与时序字段物理压缩。
*   **Card 28. ClickHouse 并发写冲突与批量刷入调优**：频繁单行写入导致 MergeTree 产生 `Too many parts` 崩溃防范。
