# Step 1: 题材判定与卡片划分 - spanner / cockroachdb (分布式数据库系统工程)

## 1. 题材判定
*   **核心内幕**: 全球分布式 SQL 数据库架构。聚焦于表 Range 动态分片分裂、Multi-Raft 共识心跳合并、租约（Leaseholder）读写分离流向、二阶段提交（2PC）与事务记录（Transaction Record）、行级写意图（Write Intent）锁、混合逻辑时钟（HLC）推进与物理时钟漂移不确定区间重试，以及串行化快照隔离（SSI）与 Read Timestamp Cache 防写偏斜。
*   **设计基调**: 使用高雅的莫兰迪灰蓝与褐红调色盘（groupM1~groupM6），突出工业级硬核分布式事务底座的设计美感，高密两栏流式排版。
*   **目标版面**: 双页 A4 Landscape 横向海报（extarticle + multicol + tcolorbox），底部左侧为分布式事务与共识折衷矩阵，底部右侧为 Zone T 核心架构参数与排障调试指令。

---

## 2. 核心卡片划分 (28 张核心卡片)

### 📂 M1: SQL 执行引擎与分布式计划分发 (Cards 1-4)
*   **Card 1. SQL 解析与分发执行：Gateway 节点的两阶段分发**：Gateway 节点解析 SQL 生成抽象语法树（AST），进行语义分析与逻辑重写，识别出查询所跨越的 Ranges 并构建物理计划。
*   **Card 2. 分布式 SQL 执行引擎（DistSQL）流式处理与 Flow 拓扑**：DistSQL 将物理计划切分为多个 Flow（由物理算子与流式 pipeline 构成），通过 Router 跨节点流式分发中间元组，避免网关全局汇总瓶颈。
*   **Card 3. 分布式 Scan 算子与主键/二级索引 KV 编解码规则**：关系行按 `Prefix/TableID/IndexID/Key` -> `Value` 编码存入底层。二级索引的 Key 编码中包含主键，支持全索引扫描与反查。
*   **Card 4. 优化器基于代价的路径选择（CBO）与统计信息维护**：利用等深直方图、关联度统计及物理网络 Locality 延迟代价，自动在 Lookup Join、Merge Join 与 Hash Join 之间做出最优选择。

### 📂 M2: Range 分片分裂与 Raft 多副本协同 (Cards 5-8)
*   **Card 5. Range 动态分片分裂（Range Splits）与合并机制**：单表按主键划分为连续 Range，默认上限 64MB。当超限或检测到写入热点时，执行 Split 生成新 Range；当 Range 收缩时触发 Merge。
*   **Card 6. Multi-Raft 共识架构：多 Raft Group 隔离与心跳合并（Coalesced Heartbeats）**：单节点管理数万个 Range 的 Raft 副本。通过对同一源-宿主机的多个 Raft 实例的心跳包执行批量合并，显著降低网络与 CPU 开销。
*   **Card 7. 副本租约（Leaseholder）机制：读写流向与脑裂防范**：区分 Raft Leader 与 Leaseholder。Leaseholder 独立处理所有读写请求，写请求由其提案发往 Raft 复制，读请求可在本地强一致执行。
*   **Card 8. 副本重平衡（Rebalancing）与 Raft 成员变更控制**：数据副本分配器根据物理资源水位线、Locality 拓扑和负载状况，生成迁移指令，使用 Raft Joint Consensus 机制平滑迁移副本。

### 📂 M3: 分布式事务与两阶段提交 (2PC) 锁意图 (Cards 9-13)
*   **Card 9. 分布式两阶段提交（2PC）原子性与事务记录（Transaction Record）**：协调节点在系统 Range 写入 `PENDING` 状态的 Transaction Record；当所有分片写入完毕，将该记录修改为 `COMMITTED`，以此确认事务提交。
*   **Card 10. 写意图（Write Intent）锁机制与行级并发冲突检测**：修改操作先写为 Write Intent（一种带事务元数据指针的临时记录，相当于排他锁）。后续读写通过查询对应事务状态确定其可见性。
*   **Card 11. 两阶段提交的优化：单阶段提交（1PC）与非冲突并发**：若事务仅修改单个 Range 内部的 Key，则协调节点跳过 2PC。直接由该 Range 的 Raft 提交并应用，将网络延迟降为单次共识开销。
*   **Card 12. 事务清理（Intent Cleanup）与异步垃圾回收**：事务最终状态确立（COMMITTED/ABORTED）后，协调节点以流式异步方式将所有 Write Intents 转换为正式物理版本（或清除），解冻并发。
*   **Card 13. 读写冲突解决机制：事务优先级与推迟重试**：冲突发生时，根据事务的 priority（结合时延与重试次数）判定。低优先级事务挂起等待，或主动回滚并以更高优先级重启。

### 📂 M4: 混合逻辑时钟 (HLC) 与 TrueTime API 一致性保障 (Cards 14-17)
*   **Card 14. 混合逻辑时钟（HLC）物理与逻辑复合公式**：HLC 时间戳由物理读数 $l.i$ 与逻辑计数 $c.i$ 构成。即 $HLC_i = (l.i, c.i)$。当物理漂移低于偏差时，累加逻辑计数以保证事件的偏序关系。
*   **Card 15. 时钟偏移不确定性（Clock Offset Uncertainty）与不确定度读取重试**：若读取快照时间落入时钟偏移不确定性区间 $[T_{read}, T_{read} + \text{max\_offset}]$，事务必须等待或执行 Read Restart 重试，确保线性一致性。
*   **Card 16. HLC 逻辑时钟的 Tick 与 Update 事件推进协议**：本地事件触发推进 $l.i$ 并清零 $c.i$；接收消息时，比对本地与消息的 HLC 时间戳，取大者推进，保证全集群 HLC 严格单调且随物理时间收敛。
*   **Card 17. 跨地域只读副本（Follower Reads）与 HLC 时间戳对齐**：允许客户端携带一个小于当前 $HLC - \text{max\_offset}$ 的历史快照时间戳，直接在本地 Follower 节点读取，消除跨洲际往返延迟。

### 📂 M5: MVCC 多版本控制与并发冲突防御 (Cards 18-21)
*   **Card 18. MVCC 物理存储布局与多版本 Key 编解码**：在底层的 Pebble/RocksDB 中，存储键编码为 `UserKey/Timestamp`，Value 存储真实数据。读请求优先获取最近时间戳版本，旧版本后台 GC。
*   **Card 19. 读屏障（Read Barrier）与快照隔离级别（SI）实现**：读事务持有时钟戳 $T_{read}$。任何修改如果时间戳大于 $T_{read}$，对该事务皆不可见；读屏障强制过滤写意图锁，提供稳定一致快照。
*   **Card 20. 串行化隔离级别（SSI）与读写冲突依赖环防范**：通过追踪 `Read Timestamp Cache`。当写事务试图更新被高时间戳读过的 Key 时，强制使该写事务重启以防写偏斜（Write Skew）。
*   **Card 21. MVCC 历史版本物理回收（GC TTL）与 Compaction 清理**：由后台定时器驱动，回收超过保留期限（TTL）的旧多版本数据，并在底层引擎执行 SSTable Compaction 时物理擦除以释放空间。

### 📂 M6: 副本租约 (Leaseholder) 与集群重平衡机制 (Cards 22-28)
*   **Card 22. Gossip 协议集群成员拓扑自动发现与状态广播**：节点间利用 Gossip 协议周期性同步节点活跃状态、所在数据中心（Locality）、分片磁盘水位及网络连接矩阵延迟。
*   **Card 23. Node/Rack/Datacenter 物理感知与容灾放置策略**：根据 Locality 配置，副本分配器（Allocator）将三副本分别放置在不同机架（Rack）或不同机房，确保在单点或区域故障时多数派在线。
*   **Card 24. 动态数据迁移与副本数平滑在线调整**：在不停止读写的情况下修改表的 Zones 副本策略，底层 Allocator 自动通过 Raft 副本变更追加新节点并异步追平日志，最终下线旧副本。
*   **Card 25. 物理 Schema 变更：非阻塞表结构在线升级**：通过将 schema 划分为多步过渡状态（Delete-only -> Write-only -> Public），在后台进行历史数据回填，避免长锁表与写中断。
*   **Card 26. 慢节点（Stuck Nodes）隔离与 Lease 租约自动流转**：当 Leaseholder 发生硬件或网络劣化导致 QPS 响应暴跌，其他副本在租约心跳（Lease Heartbeat）超时后主动抢占 Lease。
*   **Card 27. 热点分片（Hotspot）自动散列分发与限流保护**：如果检测到个别 Range 读写吞吐（QPS）偏离均值，立即触发热点分裂（Hot Split），并跨物理节点调度 Leaseholder 分散压力。
*   **Card 28. 时间同步心跳探活与时钟崩溃自动熔断隔离**：节点通过 NTP 维持时间同步。若检测到自身物理时钟与多数派节点的偏差超过允许的上限（如 200ms），节点自动 Panic 宕机下线。

---

## 3. L0 ~ L2 知识阶梯

*   **L0 一句话本质**：分布式 SQL 数据库的本质是利用 Multi-Raft 进行水平 Range 分片共识复制，基于混合逻辑时钟（HLC）和 MVCC 实现分布式事务一致性与快照隔离，使得无共享架构集群呈现单机关系型 SQL 数据库的体验。
*   **L1 四句话逻辑**：
    1.  **数据分治与共识分发**：将表自动划分为 64MB 的 Range 分片，每个 Range 绑定 Multi-Raft Group 并依靠租约（Leaseholder）拦截读写。
    2.  **分布式事务与锁意图**：通过二阶段提交（2PC）修改单条原子事务记录，配合行级写意图（Write Intent）锁实现全局 ACID。
    3.  **混合时钟与冲突自愈**：利用 HLC 逻辑推进与不确定性区间等待协议，破除物理时钟漂移，实现快照隔离与线性一致性。
    4.  **动态扩容与在线升级**：通过 Gossip 成员发现与代价值 Allocator 动态搬迁 Range，实现零停机的物理容灾与非阻塞 Schema 变更。
*   **L2 核心数据流转拓扑**：
    *   `Client SQL` ➜ `Gateway Parse` ➜ `Lookup Route Range` ➜ `Locate Leaseholder` ➜ `Raft Command` ➜ `Write Intent` ➜ `Create PENDING Transaction Record` ➜ `Raft Consensus Commit` ➜ `Update Transaction Record to COMMITTED` ➜ `Async Intent Cleanup` ➜ `Return OK`.
