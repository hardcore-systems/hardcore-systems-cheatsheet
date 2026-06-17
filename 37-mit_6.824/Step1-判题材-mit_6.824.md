# Step 1: 题材判定与卡片划分 - MIT 6.824 / distributed-systems (分布式系统工程)

## 1. 题材判定
*   **核心内幕**: 分布式共识机制（Raft 状态机）、强一致分布式存储（MapReduce 并行计算、GFS 物理副本读写一致性）、高可用分片 KV 存储迁移，以及 ZooKeeper 协调服务的 Zab 共识机制与分布式锁。
*   **设计基调**: 运用莫兰迪高雅灰蓝与灰绿调色盘（groupM1~groupM6），体现严谨的学术工程与工业实践结合，排版高密且结构层次对称。
*   **目标版面**: 双页 A4 Landscape 横向海报（extarticle + multicol + tcolorbox），底部左侧为一致性哈希/读写分离/缓存折衷设计矩阵，底部右侧为 Zone T 核心架构参数与排障调试指令。

---

## 2. 核心卡片划分 (28 张核心卡片)

### 📂 M1: MapReduce & GFS (Cards 1-4)
*   **Card 1. MapReduce 任务调度与 Shuffling 排序合并过程**：Map 端输出分区，Shuffle 将相同 Key 路由至同一 Reduce 节点，合并排序保证顺序。
*   **Card 2. MapReduce 容错与 Worker 节点故障 Failover**：Master 周期性 ping 工作节点。未完成的 Map 重新分配执行，故障节点的 intermediate data 重新生成。
*   **Card 3. GFS 架构设计：Master-Chunkserver 租约协调与多副本读写**：Chunk 默认 64MB。写流程中 Master 分配 Lease 给 Primary 副本，Primary 确定写入顺序，Secondary 写入。
*   **Card 4. GFS 追加写入（Record Append）一致性与静默损坏检测**：支持并发追加。不保证精确副本一致，但保证至少写入一次（At-Least-Once）。Chunk 校验和检测损坏。

### 📂 M2: Raft 共识 - 领导者选举 (Cards 5-8)
*   **Card 5. 随机选举超时（Randomized Election Timeout）与瓜分选票防范**：通过将超时设置在 150ms-300ms 随机区间，避免多个 Follower 同时成为 Candidate 导致瓜分选票死锁。
*   **Card 6. Raft 选举安全约束：日志新旧（Up-to-date）比对规则**：Candidate 在拉票时发送最新 Log 的 Index 和 Term，Follower 校验若拉票者 Log 比自己旧则拒绝投票。
*   **Card 7. 联合共识（Joint Consensus）与单节点成员变更**：配置变更的过渡阶段使用 $C_{\text{old,new}}$ 联合多数派；单节点变更每次只增减一个节点以确保多数派始终重叠。
*   **Card 8. 领导者心跳维持、AppendEntries 探测与 Term 单调递增**：Leader 发送空 AppendEntries 维持权威，任期号（Term）作为逻辑时钟，高 Term 覆盖低 Term 强制退位。

### 📂 M3: Raft 共识 - 日志复制与安全 (Cards 9-13)
*   **Card 9. 日志复制多数派（Quorum）判定与 Commit Index 推进**：当某条日志已被复制到多数派节点上时，Leader 推进 commitIndex，并在下一次心跳或日志同步中广播给 Follower。
*   **Card 10. 日志冲突恢复：NextIndex 回溯与快速匹配机制**：Follower 返回拒绝的 AppendEntries RPC，包含冲突位置的 Term 和首个 Index，Leader 一步跳过冲突 Term。
*   **Card 11. 领导者不能提交之前任期的日志（Raft 安全定理）**：Leader 只能通过提交当前任期的日志来间接提交之前任期的日志，防止已提交日志被覆盖。
*   **Card 12. 日志压缩快照（Snapshotting）与 InstallSnapshot 传输**：当日志过长时，节点在本地创建状态机快照并截断日志。落后过多的 Follower 通过 InstallSnapshot 追平。
*   **Card 13. Raft 线性化读取保障：ReadIndex 与 Lease Read**：ReadIndex 流程询问 Quorum 确认自身仍是 Leader 并等待 commitIndex 推进；Lease Read 利用租约时钟安全跳过 RPC。

### 📂 M4: 分片键值存储 (Cards 14-17)
*   **Card 14. ShardController 配置版本化与一致性哈希/负载均衡算法**：ShardController 管理配置版本号。当 ShardGroup 增减时，重新分配 Shard 保持迁移量最小。
*   **Card 15. 跨分片事务路由与客户端请求重定向机制**：客户端缓存配置，向 target group 发送请求。若收到 `ErrWrongGroup`，客户端更新配置并重定向。
*   **Card 16. 动态分片迁移（Shard Migration）的拉取式 vs 推送式状态传输**：新配置生效时，目标组从源组拉取（或源组推送）特定 Shard 的 KV 数据。
*   **Card 17. 分片迁移并发控制：传输期间客户端读写的拦截隔离**：迁移过程中，特定 Shard 标记为 `Migrating` 或 `Invalid`。此时对应 Shard 的读写被阻塞或拒绝。

### 📂 M5: Raft KV 存储引擎内幕 (Cards 18-21)
*   **Card 18. 客户端请求去重与幂等处理：Session ID 与序列号**：通过跟踪 `ClientID` 和单调递增的 `SeqNum`，在状态机中缓存最近一次回复，消除网络重试导致的重复应用。
*   **Card 19. 读写请求提交流程与 Raft Apply 异步事件循环**：写请求发往 Raft 层，在 `applyCh` 中被异步消费，匹配 ClientID 与 SeqNum 后应用到内存数据库中。
*   **Card 20. 状态机内存泄露防范与 Channel 关闭优雅清理**：避免内存对象无节制增长。在服务停止或 Leader 退位时，优雅关闭 `applyCh` 并释放资源。
*   **Card 21. 快照触发阈值与内存数据库写时复制（COW）防阻塞**：当 Raft 状态大小超过阈值时，克隆当前数据库副本，使用 COW 技术后台异步刷盘以防阻塞前台读写。

### 📂 M6: ZooKeeper 分布式协调 (Cards 22-28)
*   **Card 22. ZooKeeper 层次命名空间与临时节点（Ephemeral Nodes）**：树形目录结构，临时节点的生命周期与客户端会话绑定，一旦会话超时自动删除。
*   **Card 23. ZooKeeper 监听机制（Watcher）的一击即溃与通知机制**：Watcher 是单次触发的（One-time trigger）。当数据变化时向客户端发通知，之后自动失效，需重新注册。
*   **Card 24. Zab 共识协议：崩溃恢复与原子广播**：Zab 核心为 Leader 崩溃恢复（选举任期最大的节点）与原子广播（使用两阶段提交提交提议，无回滚）。
*   **Card 25. 分布式锁实现：利用 ZooKeeper 顺序节点与 Watch 避免惊群**：创建临时顺序节点（Ephemeral Sequential），只监听比自己小 1 的节点以规避惊群效应。
*   **Card 26. 成员发现服务与 Master 主备选举协调模式**：集群节点启动时在 `/members` 下创建临时节点，Master 抢占创建 `/master`，其余节点 Watch 以实现热备切换。
*   **Card 27. 脑裂防护防御线：隔离令牌（Fencing Token）与世代（Epoch）隔离**：客户端请求携带由协调器递增的 Epoch 编号，后端数据库拒绝携带旧 Epoch 令牌的过期写入。
*   **Card 28. 会话维持、心跳协商与 Session 丢失自愈恢复**：客户端周期性发送 ping 以保活。超时未收到心跳则判定会话断开。客户端可重连其他服务器尝试恢复 Session。

---

## 3. L0 ~ L2 知识阶梯

*   **L0 一句话本质**：分布式系统工程的本质是利用共识协议与状态机复制技术，在充满网络延迟、分区及节点崩溃的物理现实中，构建具备线性一致性与高容灾能力的统一状态服务。
*   **L1 四句话逻辑**：
    1.  **分治并行计算与容灾持久**：通过 MapReduce 平行解构海量计算与 GFS 大文件切片副本化，实现基础存储与计算的横向水平扩展。
    2.  **共识状态机复制与安全保护**：依靠 Raft 的多数派领导者选举和单向日志复制，构建高可用复制状态机，在多数派在线时保证安全性与活性。
    3.  **动态分片配置与迁移重置**：利用 ShardController 版本配置驱动分片迁移与路由重定向，实现大规模集群负载的动态平滑伸缩。
    4.  **客户端去重与去锁协调**：在应用层配合 Session 幂等消除网络重试副作用，在控制面基于 ZooKeeper 临时顺序节点高效实现无惊群分布式锁与高可用协调。
*   **L2 核心数据流转拓扑**：
    *   `Client write` ➜ `Locate shard` ➜ `RPC to Leader` ➜ `Leader log append` ➜ `AppendEntries RPC` ➜ `Follower logs` ➜ `Follower accept` ➜ `Leader count majority` ➜ `Leader advance commitIndex` ➜ `Apply to DB state machine` ➜ `Cache ClientSeq response` ➜ `Client read` ➜ `ReadIndex check quorum` ➜ `Return linearizable DB state`.
