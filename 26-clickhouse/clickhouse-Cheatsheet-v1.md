# 《clickhouse-internals》高密知识图谱与速查手册 (Cheatsheet)

*   **L0 一句话本质**：通过列式物理存储与压缩块设计，将批量数据映射为 8192 行粒度的稀疏索引标记，利用 CPU SIMD 寄存器级向量化操作在大批量 Column 块上并行过滤与聚合，依靠异步 MergeTree 引擎及 ZooKeeper 协同保障大规模数据分片副本最终一致性的极速列存 OLAP 引擎。
*   **L1 四句话逻辑**：
    1.  **分布式查询与网络协议** (M1) 支持原生 TCP 与 HTTP，通过两阶段分布式查询在各分片 (Shard) 上并发下推算子并由 Coordination 节点汇总。
    2.  **向量化执行与 SIMD** (M2) 放弃单行 Volcano 模型，改用 Column 块作为迭代单元，用 CPU 硬件级的 SIMD 向量化指令提升单核数据计算吞吐。
    3.  **MergeTree 存储引擎** (M3) 采用 LSM-Tree 变体，将追加写入的数据段按分区 (Partition) 物理落盘，并在后台异步进行 Merge 归并排序合并。
    4.  **稀疏索引与压缩块** (M4-M6) 配合 `primary.idx` 和 mark 标记文件以 8192 行为 Mark 粒度检索，通过 Block 级列式压缩及多种编码技术极大压榨 I/O。
*   **L2 存储引擎数据流演进拓扑**：
    *   [SQL 查询请求] $\rightarrow$ [分布式解析下推] $\rightarrow$ [MinMax 分区过滤] $\rightarrow$ [Primary Key 稀疏 Mark 定位] $\rightarrow$ [Compressed Block 读取解压] $\rightarrow$ [SIMD 向量化 Filter/Aggregate] $\rightarrow$ [Shard 数据汇总] $\rightarrow$ [最终结果输出]

---

## 🌐 极速列式存储引擎物理滤镜 (ClickHouse Epistemic Filter)

- **认识论 (Epistemology) - 以列式物理对齐与 I/O 吞吐最大化为设计起点**：
  ClickHouse 认识论的基石在于“列式连续存储是加速聚合计算的终极物理手段”。不同于行式数据库将一行的所有字段连续存储，ClickHouse 将每一列的数据独立存放在专属的 `.bin` 文件中。这一物理布局决定了它的 I/O 模式：在执行分析查询时，内核仅读取涉及的列，完全规避了无关列的磁盘 I/O。内存分配同样围绕列式展开，每一列在内存中表示为连续的二进制数组，使得 CPU 能够实现极致的缓存行（Cache Line）局部性，并将数据源源不断地送入寄存器进行计算。
- **一致性论 (Consistency Theory) - 最终一致的异步 Merge 复制与分片路由**：
  在分布式环境下，ClickHouse 的一致性模型是“强最终一致性”。它的 ReplicatedMergeTree 引擎并不通过分布式两阶段提交（2PC）来强同步写入，而是依赖 ZooKeeper（或 ClickHouse Keeper）作为共享协调日志。写入操作在单节点成功落盘并生成 Part 后，在 ZK 注册写日志。其余副本通过监听 `/log` 队列，在后台以异步 pull 的方式，通过 HTTP 协议拉取缺少的 Part。在这个过程中，后台的异步 Merge 进程会合并碎片 Part，并在宏观上消弭版本差异，保障最终查询视图的一致性。
- **方法论 (Methodology) - SIMD 向量化指令集与稀疏 Mark 跨越检索**：
  ClickHouse 方法论的精髓在于“消除一切不必要的 CPU 分支预测失败与无效 I/O 扫描”。为了规避传统 Volcano 迭代器模型中每行数据调用一次虚函数的巨大开销，ClickHouse 采用“向量化执行模型”，即每次算子调用流转一个 Column Block（默认 65536 行）。通过高度优化 C++ 模板并针对 SSE4.2/AVX2/AVX512 指令集进行硬编码，利用 SIMD 寄存器单指令处理多个数值。在定位数据时，它不使用开销巨大的 Dense B-Tree 索引，而是使用稀疏主键索引，每 8192 行生成一个 Mark 标记，在查询时以 Block 粒度跨越式跳过不满足条件的行。

---

## ⚔️ 列式 OLAP 架构设计与执行折衷矩阵 (OLAP Storage & Execution Trade-offs)

| 开发者直觉 (⚠) | 系统物理现实 (✗) | 架构折衷与防范金标准 (✓) |
| :--- | :--- | :--- |
| **主键索引应该像 MySQL 一样越精准越好，且主键建得越多越方便多条件检索** | ClickHouse 采用稀疏索引（Sparse Index）。如果主键字段过多或基数过高，会导致内存中 `primary.idx` 物理膨胀，且稀疏索引无法精确定位单行，反而造成无用的全 Granule I/O 扫描。 | 仅对经常作为 Where 过滤条件且物理有序的低/中基数字段建主键。多条件检索使用二级“跳数索引”（Skip Index）或物化视图（Materialized View）进行物理重组加速。 |
| **像传统数据库那样，采用高并发高频次单行 INSERT 来实时更新数据** | 每次写入都会在 MergeTree 中生成一个独立的物理 Part。如果高频写入，后台 Merge 进程来不及归并合并，会导致 Part 数量暴增，触发 `Too many parts in all data parts` 写入限流甚至崩溃。 | 必须在客户端或使用 Buffer 表进行批量聚合写入。建议单次写入批次不少于 20000 行，或维持写入频率在每秒不超过 1 次，以给后台 Merge 归并留出充分的物理调度窗口。 |
| **通过频繁的 UPDATE 和 DELETE 语句来实时修改和删除数据状态** | MergeTree 的 UPDATE/DELETE 本质上是通过 Mutation 机制异步重写整个 Part 数据块。这会引发极高的磁盘 I/O 穿透和 CPU 重新编码开销，导致查询性能短时间内雪崩。 | 采用 `ReplacingMergeTree` 或 `CollapsingMergeTree` 引擎，通过写入标记行（Sign）或版本号，在后台 Merge 时自动物理去重，或者通过逻辑 Filter 规避物理重写。 |

---

## 🗺️ 6大核心模块与 28 张高密速查卡片 (Core Cards Map)

### M1：分布式查询与网络 (Slate Blue)

#### Card 1. TCP/HTTP 原生协议与报文交互 (Network Protocol)
*   **双重协议网关**：ClickHouse 同时支持高效率的原生 TCP 协议（默认 9000 端口）和兼容性极佳的 HTTP 协议（默认 8123 端口）。
*   **块级报文序列化**：原生 TCP 采用自研二进制协议，数据报文按 Block（列式块）进行紧凑序列化传输，最大程度压榨网卡带宽。

#### Card 2. 两阶段分布式查询执行下推 (Distributed Query Executor)
*   **分布式 AST 分发**：分布式表（Distributed Engine）接收查询后，解析为分布式执行计划。
*   **本地并行汇总**：第一阶段将 Query 重写，下推（Pushdown）至各 Shard 节点并行执行本地计算；第二阶段由 Coordinator 节点拉取各 Shard 的中间聚合结果（如 Hash 槽段）进行全局归并。

#### Card 3. Shard / Replica 拓扑路由选择 (Topology Routing)
*   **逻辑分片与物理副本**：Cluster 在配置中定义分片（Shard）和副本（Replica）。
*   **动态路由探测**：写入分布式表时，根据 `sharding_key` 哈希值进行路由分配。查询时，Coordinator 自动探测副本延迟，优先路由至延迟最低的 Active Replica 节点。

#### Card 4. ClickHouse FDW 外部表引擎挂载 (Foreign Data Wrapper)
*   **异构联邦挂载**：支持 `MySQL`、`PostgreSQL`、`ODBC`、`HDFS` 等外部引擎。
*   **算子物理下推**：在执行联邦查询时，CH 优化器解析 Where 过滤条件和 Limit 限制，将其转化为 SQL 下推到远端数据库执行，避免全表拉取。

---

### M2：向量化执行与 SIMD (Moss Green)

#### Card 5. ColumnVector 内存物理连续对齐 (Column Vector Layout)
*   **内存数组连续性**：列式数据在内存中表示为 `ColumnVector` 结构。内部使用一片连续的 C++ 原始内存空间存储同一列的数据。
*   **无指针跳转开销**：消除行式结构中对象指针的随机内存跳转，完美对齐 CPU L1/L2 缓存行，实现微秒级的内存寻址。

#### Card 6. Vectorized Loop 块迭代机制 (Vectorized Iteration)
*   **废弃虚函数循环**：完全废弃行式 Volcano 算子中每行数据都进行虚函数调用的高能耗设计。
*   **大块流式直通**：算子间流转的单元为 `Chunk`（默认 65536 行 Column 数组）。在算子内部使用极简的紧凑 for 循环处理整个数组，大幅降低 CPU 分支预测失败率。

#### Card 7. CPU SIMD 指令集硬件加速 (SIMD Optimizations)
*   **并行向量流水线**：CH 代码深度集成了 SSE4.2、AVX2、AVX512 指令集。
*   **硬件级并行过滤**：在执行如 `WHERE col > 10` 时，编译器将循环展开，利用 SIMD 寄存器单条指令同时对比 8/16 个数值，吞吐量直达硬件物理极限。

#### Card 8. Projection 投影与 Materialized View 同步刷写 (Projections & MV)
*   **投影列物理冗余**：`Projection` 允许在主表内对部分列按不同排序键物理冗余存储。
*   **物化视图流式触发**：Materialized View 本质上是写触发器。当数据 Part 写入主表时，自动流式触发计算并写入关联的物化视图表，保障原子可见。

#### Card 9. Memory Tracker 线程级内存追踪 (Memory Tracker)
*   **层级内存计数器**：ClickHouse 在共享内存中为每个查询线程分配 `MemoryTracker`。
*   **精确 OOM 拦截**：实时追踪查询分配的字节数。一旦超过 `max_memory_usage` 阈值，立即抛出内存超限异常终止查询，保护宿主机不被 OS OOM-Killer 杀死。

---

### M3：MergeTree 存储引擎 (Plum Rose)

#### Card 10. MergeTree Part 物理目录与文件结构 (Part Directory Layout)
*   **分区物理硬隔离**：每个写入批次生成一个以 `PartitionID_MinBlock_MaxBlock_Level` 命名的物理 Part 目录。
*   **列物理文件打散**：目录内每一列分别拥有 `[column].bin`（压缩数据）和 `[column].mrk2`（Mark 标记指针）文件，以及全局 `primary.idx` 稀疏索引。

#### Card 11. 异步后台 Merge 归并合并流程 (Merge Task Pipeline)
*   **碎片 Part 后台归并**：后台 Merge 线程池定期扫描表目录，将属于同一分区的多个小 Part 挑选出来。
*   **归并排序物理去重**：执行流式归并排序（External Merge Sort），将老 Parts 合并为包含更大 Block 区间的新 Part，并将老 Parts 物理删除。

#### Card 12. Row/Column TTL 声明周期物理擦除 (TTL Cleaning)
*   **到期物理回收**：支持表级、列级及行级 TTL 定义。
*   **合并时无感清除**：过期标记记录在 Part 的 `ttl.txt` 中。后台 Merge 进程合并数据块时，物理擦除过期行，或将过期列数据清空重置，实现零开销物理回收。

#### Card 13. Mutation 异步非原子修改机制 (Mutations)
*   **非事务性重写**：UPDATE/DELETE 被视为 `Mutation` 操作（通过 ALTER 语句触发）。
*   **克隆 Part 重写**：ClickHouse 不支持行级事务锁，它通过复制原 Part 目录，在后台流式读取、过滤/修改并重新写入新 bin 文件的形式实现修改，代价极高。

---

### M4：稀疏索引与分区键 (Terracotta)

#### Card 14. Primary Key 稀疏索引与 Granule 采样 (Primary Sparse Index)
*   **稀疏采样存储**：`primary.idx` 并不记录每行数据，而是以 `index_granularity`（默认 8192 行）为一个 Granule 进行主键采样，仅记录采样点的键值。
*   **常驻内存常备检索**：由于是稀疏采样，即使表有数十亿行，主键索引文件也只有几兆字节，可完美常驻内存。

#### Card 15. Mark 标记文件双向偏移量指针 (.mrk Files)
*   **索引与 bin 物理桥梁**：`.mrk` 文件将 `primary.idx` 中的 Mark 序号与 `.bin` 数据文件中的物理偏移量建立关联。
*   **定位解压边界**：每一项包含两个物理偏移：一是压缩块（Compressed Block）在 `.bin` 文件中的起始字节，二是解压后数据在该块内的起始字节。

#### Card 16. MinMax 分区索引树过滤 (MinMax Pruning)
*   **分区边界快速裁剪**：主表按 Partition Key 分区。每个 Part 目录中保存 `minmax_[partition_key].idx`。
*   **第一级排除扫描**：查询规划阶段，优化器根据 Where 条件对比各 Part 的 MinMax 范围，快速剔除不相关的 Part 目录，避免无效读盘。

#### Card 17. Skip Index 跳数索引定位优化 (Skip Indexes)
*   **二级过滤加速器**：支持建在非主键列上的跳数索引（如 `minmax`、`set`、`bloom_filter`）。
*   **跳过 Granules 扫描**：跳数索引按指定的 `granularity`（如每 5 个 granules）记录特征值。查询时如果不满足条件，直接跳过对应的 5 * 8192 行数据不予解压读取。

---

### M5：副本复制与 ZooKeeper 协同 (Indigo)

#### Card 18. ReplicatedMergeTree 注册机制 (Replicated Engine)
*   **ZK 元数据注册**：Replicated 引擎创建表时，在 ZooKeeper 注册 `/clickhouse/tables/[uuid]/` 共享路径。
*   **副本元数据一致**：所有 Replica 节点共享相同的分区结构元数据与列定义。

#### Card 19. Replica 间物理写入 Log 协同 (Replication Log Sync)
*   **操作日志追加**：副本执行 INSERT 产生新 Part 后，向 ZK 的 `/log/` 节点追加一条 `GET` 类型的操作日志。
*   **订阅通知派发**：其他 Replica 节点通过 Watch 机制监听该节点变更，将操作拉取到本地的 `replication_queue` 队列。

#### Card 20. Replication Queue 与 Parts 异步拉取 (Replication Queue)
*   **下载任务流水线**：副本解析 `replication_queue` 中的任务。
*   **HTTP 直连拉取**：若发现缺失某些 Parts，并不通过 ZK 传输，而是通过本地配置的端口，直接通过 HTTP 协议从含有该 Part 的主副本拉取 bin 压缩数据，然后本地加载并上报 ZK。

#### Card 21. 分布式 ON CLUSTER DDL 工作流 (DDLWorker)
*   **全局任务分发**：执行 `DDL ON CLUSTER` 时，主 Backend 将 DDL 文本序列化写入 ZK 的 `/query_log/` 节点。
*   **各节点异步监听消费**：每个副本节点的 `DDLWorker` 线程监听到新任务后，在本地执行该 DDL，并将执行结果写回 ZK 对应子节点，供 Coordinator 收集汇总。

#### Card 22. ClickHouse Keeper 替代 ZooKeeper 实现 (CH Keeper)
*   **消除外部依赖开销**：ClickHouse 内置 `CH Keeper`。
*   **RAFT 协议闭环**：采用 C++ 实现的 RAFT 共识算法，完全兼容 ZooKeeper 协议接口，但在内存开销、读写吞吐和连接稳定性上显著优于 Java 编写的 ZK。

#### Card 23. ZK 断连副本 Read-Only 安全降级 (ZK Connection Loss)
*   **会话超时防脑裂**：当副本与 ZooKeeper 集群断开连接，且会话超时（Session Timeout）时。
*   **自动写拦截降级**：该副本会自动将自身状态降级为 `Read-only`，拦截一切 INSERT 写入，防止脑裂导致多副本数据永久分叉。

---

### M6：数据物理布局与压缩编码 (Antique Gold)

#### Card 24. Chunk 与 Block 内存物理容器 (Memory Chunk)
*   **纯粹列容器**：`Chunk` 是 ClickHouse 执行引擎在内存中传输的物理容器，包含一组 `Column` 列向量和行数。
*   **Block 头定义元数据**：`Block` 是 Chunk 的超集，追加了每一列的名称、类型和 OID 等 Header 元信息，作为输入输出的逻辑边界。

#### Card 25. Compressed Block 物理压缩链结构 (Compression Block)
*   **段落块式结构**：`.bin` 文件物理上由多个连续的 Compressed Block 拼接而成。
*   **块头长度定义**：每个 Compressed Block 头包含 1 字节压缩算法标识、4 字节压缩后长度、4 字节解压后长度。解压时以整个 Block 为最小物理单位载入内存。

#### Card 26. LZ4 与 ZSTD 压缩算法选型折衷 (LZ4 vs ZSTD Codecs)
*   **高吞吐低延迟选 LZ4**：默认压缩算法，解压速度极快（数 GB/s），适合高频热数据查询，但压缩比一般。
*   **高压缩选 ZSTD**：适合冷数据归档，压缩比极高，能节省 30% 以上磁盘空间，但解压和压缩时 CPU 开销较大。

#### Card 27. Specialized Codecs 物理列级编码 (Specialized Codecs)
*   **类型定制编码**：
    1.  **DoubleDelta / Gorilla**：针对时间戳、递增 ID 或时序浮点数，仅存储差值的差值，压缩比呈指数级提升。
    2.  **T64**：针对整型，通过旋转矩阵去除高位冗余的 0 占位符，极适合低基数字段。

#### Card 28. Too Many Parts 写入限流抖动保护 (Too Many Parts Protection)
*   **写缓冲崩溃保护**：如果 MergeTree 分区内的 Parts 数量超过 `parts_to_delay_insert`（默认 150），写入会故意 Stall 延迟；超 `parts_to_throw_insert`（默认 300）则直接报错。
*   **防止磁盘被写死**：这一强限制是为了防止磁盘碎片过多，强迫客户端控制写入频次。

---

## 🔬 辅助版块 Zone T：生产诊断与调试字典 (Zone T)

### T1 ClickHouse 典型参数调优对照表

| 配置参数 / users.xml & config.xml | 经典建议数值 | 物理调优语义 |
| :--- | :--- | :--- |
| `max_threads` | `物理 CPU 核心数` | 单个查询使用的最大并行线程数。OLAP 场景建议设满以压榨多核算力。 |
| `max_memory_usage` | `总内存的 70% - 80%` | 单个 Query 在单节点能使用的最大内存。超出则强制 Abort，防止单 SQL 拉垮整机。 |
| `max_bytes_before_external_group_by` | `max_memory_usage 的 50%` | 内存超限前，启用磁盘临时文件分块 GroupBy，避免大聚合查询直接触发 OOM。 |
| `background_pool_size` | `16 - 32 (大写入调大)` | 后台执行 Merge 归并合并操作的线程池大小。写吞吐量大时应调高此参数。 |

### T2 经典性能监控与诊断 SQL 指令字典

*   **1. 诊断表 Partition 物理 Part 分区健康状态**
    ```sql
    -- 实时查看库中各表的 Active/Inactive Parts 数量、磁盘物理大小以及碎片行数，排查 too many parts 异常
    SELECT partition, name, active, disk_name, bytes_on_disk, rows 
    FROM system.parts 
    WHERE database = 'default' AND table = 'my_table' 
    ORDER BY partition, name;
    ```
*   **2. 查看当前正在运行的高能耗慢查询内存占用**
    ```sql
    -- 获取当前集群正在执行的 Query 列表，按实时内存分配排序，方便执行 KILL QUERY ON CLUSTER 终止慢查询
    SELECT query_id, user, elapsed, memory_usage, read_rows, read_bytes, query 
    FROM system.processes 
    ORDER BY memory_usage DESC LIMIT 10;
    ```
