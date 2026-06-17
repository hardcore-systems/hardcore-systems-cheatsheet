# mit_6.824 高密度核心卡片系统 (中文版)

## L0 ~ L2 知识阶梯

*   **L0 一句话本质**: Distributed Systems（分布式系统）的本质是在存在网络分区、包延迟和节点崩溃等不可靠硬件的物理世界里，利用共识协议（如 Raft）和状态机复制技术，构建具备线性一致性（Linearizability）和高容灾能力的虚拟单机可靠状态服务。
*   **L1 四句话逻辑**:
    1.  **大规模存储与计算并行**：利用 MapReduce 将海量数据分区以在工作节点平行计算，配合 GFS 大文件多副本物理切片存储，构筑存储与计算水平扩展基础。
    2.  **共识状态机复制与安全**：依靠 Raft 的多数派选举（Quorum Leader Election）与单向日志复制（Log Replication），确保在不超过半数节点损坏时状态机完全一致。
    3.  **分布式分片路由与配置迁移**：利用 ShardController 集中式管理分片配置版本，驱动各分片组（ShardGroup）在服务期间安全、动态地迁移数据与重定向路由。
    4.  **客户端幂等去重与去锁协调**：在业务层引入 Session 去重机制消灭网络重试带来的副作用，在控制面基于 ZooKeeper 层次节点与 Watcher 高效实现分布式锁与健康感知。
*   **L2 核心数据流转拓扑**:
    *   `Client Request` ➜ `ShardRouter Lookup Config` ➜ `Target Shard Raft Leader` ➜ `Leader Append Log locally` ➜ `AppendEntries RPC sent to Followers` ➜ `Follower logs persisted` ➜ `Follower returns Success` ➜ `Leader detects majority accepts` ➜ `Leader advances commitIndex` ➜ `ApplyMsg sent to DB State Machine` ➜ `Update Client Session Cache` ➜ `Linearizable DB State returned to Client`.

---

## ⚔️ 分布式系统高并发与可用性设计折衷矩阵 (Page 1 Bottom)

| 设计维度 | 传统冗余直觉 (⚠) | 底层系统物理真实 (★) | 核心架构折衷与演进设计 |
| :--- | :--- | :--- | :--- |
| **分布式共识机制** | 采用多写或多 Master 架构以追求所有节点并行写入的最大系统吞吐量。 | 在存在网络分区的物理世界中，多写会导致脑裂（Split-brain），造成数据严重冲突和丢失。 | **多数派主从共识 (Quorum Consensus)**: 使用 Raft 强 Leader 模式，依靠 $2F+1$ 节点组成的法定人数收敛状态。 |
| **读一致性保障** | 允许客户端直接从任意最近的本地数据库副本中读取数据，以实现最低的读延迟。 | 副本数据同步存在网络延迟，且分区可能导致客户端读到过期（Stale）的历史旧状态。 | **线性化读校验 (ReadIndex/Lease Read)**: 读请求需与多数派确认 Leader 身份，或利用时钟租约确保不读旧数据。 |
| **状态机复制一致性** | 往副本节点直接发送 SQL 字符串或带副作用的 Shell 指令来同步数据库。 | 包含非确定性因子（如 `RAND()`, `NOW()` 或并发顺序差异）会导致副本状态逐渐分叉。 | **确定性日志应用 (Deterministic Log SMR)**: 仅复制并持久化只读追加的二进制日志，在状态机中以相同顺序重放。 |
| **分片重平衡** | 动态扩容时，直接重改网关分片指向，并在后台异步对拷数据。 | 迁移中的并发写入会产生双写冲突，或由于网关与后端路由版本不一致导致脏数据写入。 | **两阶段分片迁移 (Two-Phase Shard Migration)**: ShardController 版本递增，先锁定源组只读，传输完毕后再激活新组。 |

---

## 📂 M1: MapReduce & GFS (Cards 1-4)

#### Card 1. MapReduce 任务调度与 Shuffling 排序合并过程
*   **技术机制**: MapReduce 执行分为 **Map** 和 **Reduce** 两阶段。Map 端读取数据切片，输出中间状态的 `(Key, Value)` 键值对，缓存在内存并按哈希输出到本地磁盘的分区中；**Shuffle** 阶段，Reduce 节点通过网络从各 Map 节点拉取对应分区的 Key，在本地执行外排（External Sort）将相同 Key 的所有 Value 聚合并排好序，最后送入 Reduce 函数执行。
*   **应用策略**: 在 Shuffle 阶段配置高效的文件合并缓冲区大小（如 100MB），减少磁盘 I/O 读写频次。

#### Card 2. MapReduce 容错与 Worker 节点故障 Failover
*   **技术机制**: Master 节点周期性通过心跳 ping 每一个 Worker。若某个 Map Worker 在限定时间内未回应，Master 将该 Worker 标记为故障，并将其上**已完成**和**未完成**的任务全部重置为 `Idle` 状态；因为 Map 任务输出缓存在本地磁盘，机器挂了中间数据就丢了，必须重新调度。而已完成的 Reduce 任务因输出写到了 GFS 全局分布式存储上，无需重新执行。
*   **Application Strategy**: Master 应保持任务进度的元数据幂等，重新分配任务时仅改变状态指针，防止重复产出脏数据。

#### Card 3. GFS 架构设计：Master-Chunkserver 租约协调与多副本读写
*   **技术机制**: GFS 由单个 **Master** 和多个 **Chunkserver** 组成。大文件被分割为 64MB 的 Chunk。写入时，Master 授予其中一个副本 **Lease（租约）** 使其成为 Primary，并由其决定当前 Chunk 的写入顺序。写数据先以流水线形式推送到所有副本的内存中，再由 Primary 发起写入指令，促使各副本落盘。
*   **应用策略**: Master 利用 Lease 减少了控制面压力，避免了每次写操作都由 Master 串行介入排序。

#### Card 4. GFS 追加写入（Record Append）一致性与静默损坏检测
*   **技术机制**: GFS 保证并发追加写入的 **At-Least-Once（至少一次）** 语义。当写入失败重试时，可能导致部分 Chunk 副本中产生空洞或重复数据。GFS Master 通过副本检测来回收无效数据。Chunkserver 本地维护 Checksum（校验和），在读取或空闲时校验数据，一旦发现静默损坏（Silent Corruption）则上报 Master 重新拉取正常副本覆盖。
*   **应用策略**: 应用层必须在设计时容忍记录的重复，通过在记录中写入唯一流水号并在读取后进行去重。

---

## 📂 M2: Raft 共识 - 领导者选举 (Cards 5-8)

#### Card 5. 随机选举超时（Randomized Election Timeout）与瓜分选票防范
*   **技术机制**: 当 Follower 失去 Leader 心跳时，会增加自身的任期号并转为 Candidate。若所有 Follower 同时超时，会同时发起选举，导致选票被均分（Split Vote）而无法产生 Leader。Raft 将选举超时时间随机化限制在 `[T, 2T]` 区间（通常是 `150ms-300ms`），确保在下一次选举时，率先超时的节点能够拿到多数派选票。
*   **应用策略**: 随机超时区间必须明显大于节点间的网络往返时延（RTT），否则 Candidate 尚未拉完票就会再次发生超时。

#### Card 6. Raft 选举安全约束：日志新旧（Up-to-date）比对规则
*   **技术机制**: Raft 保证已提交的日志绝对不会丢失。Follower 在接收到 RequestVote RPC 时，会对比 Candidate 的日志。只有满足以下条件的 Candidate，Follower 才会给它投票：Candidate 的最后一条日志的 Term 大于 Follower ；或者 Term 相同但 Candidate 的 Log Index 大于等于 Follower。
*   **应用策略**: 此规则限制了只有包含全部已提交日志的节点才有可能成为 Leader，从而省去了像 Paxos 那样在选举后从其他节点恢复已提交日志的步骤。

#### Card 7. 联合共识（Joint Consensus）与单节点成员变更
*   **技术机制**: 变更集群配置时（如从 3 节点扩容到 5 节点），如果直接应用新配置，可能因为节点配置更新时间不一致导致同时选出两个 Leader。Raft 联合共识通过引入过渡状态 $C_{\text{old,new}}$，要求所有日志必须在旧多数派和新多数派中均获得通过。而单节点变更算法限定一次只能加入或退出一个节点，使多数派始终重叠，简化了配置变更流。
*   **应用策略**: 工业实现通常选择单节点变更，通过多次迭代完成大规模拓扑升级，避免了联合共识两阶段配置变更的复杂性。

#### Card 8. 领导者心跳维持、AppendEntries 探测与 Term 单调递增
*   **技术机制**: Leader 通过周期性发送空 `AppendEntries` RPC 来做 Heartbeat，重置 Followers 的选举计时器。Term 是单调递增的逻辑时钟。如果 Leader 收到一个更高的 Term 响应，必须立刻退位并转为 Follower；如果 Follower 收到 Term 小于自己当前任期的 RPC，则直接拒绝。
*   **应用策略**: 任期号作为分布式系统的防线，能够强行平息由于长延迟网络包乱序导致的“老 Leader”僭越。

---

## 📂 M3: Raft 共识 - 日志复制与安全 (Cards 9-13)

#### Card 9. 日志复制多数派（Quorum）判定与 Commit Index 推进
*   **技术机制**: Leader 收到客户端的写命令后，先追加到本地日志，并向 Followers 广播 `AppendEntries`。当 Leader 收到超过半数节点（Quorum）的成功写入确认时，会将 `commitIndex` 递增，随后将日志应用到本地状态机（Apply），并向客户端返回成功。
*   **应用策略**: 保证集群节点数为奇数（如 3, 5, 7），以最大化容灾性价比（3 节点可坏 1 台，4 节点依然只能坏 1 台）。

#### Card 10. 日志冲突恢复：NextIndex 回溯与快速匹配机制
*   **技术机制**: Follower 日志可能因宕机或网络分区落后或分叉。Leader 维护 `nextIndex[i]`（发送的下一索引）和 `matchIndex[i]`（已匹配的最大索引）。当 Follower 拒绝日志同步时，Leader 递减 `nextIndex`。为了加速，Follower 可返回拒绝时的 `ConflictTerm` 和该任期内的第一个 Index。 Leader 收到后直接跳过该任期的所有冲突项，大幅缩短回溯时钟。
*   **应用策略**: 在网络抖动频发的环境下，启用快速匹配算法是压缩故障节点重连后数据恢复时间的关键。

#### Card 11. 领导者不能提交之前任期的日志（Raft 安全定理）
*   **技术机制**: 如果 Leader 试图通过计算副本数来提交之前旧任期的日志，可能会因为在未复制到多数派之前宕机，导致旧日志在未来被新 Leader 覆盖。Raft 规定：**Leader 只能通过提交当前任期的日志，间接将之前任期的日志顺带提交**。
*   **应用策略**: 在实现状态机应用时，必须严格校验当前提交日志的 Term 是否与 Leader 的 currentTerm 一致，否则拒绝推进 commitIndex。

#### Card 12. 日志压缩快照（Snapshotting）与 InstallSnapshot 传输
*   **技术机制**: 为了防止日志无限增长撑爆磁盘，节点会周期性地对状态机进行快照处理，丢弃已提交的日志，保留快照中的 LastIncludedIndex 和 LastIncludedTerm。若某个 Follower 落后太多，其所需的日志已被 Leader 快照清理，Leader 会调用 `InstallSnapshot` RPC 直接将快照发送给 Follower 以完成对齐。
*   **应用策略**: 采用写时复制（COW）机制在后台异步生成快照，避免快照操作阻塞主线程的客户端写入。

#### Card 13. Raft 线性化读取保障：ReadIndex 与 Lease Read
*   **技术机制**: 客户端直接读取 Leader 可能会因为 Leader 已被分区（脑裂）读到旧数据（Stale Read）。**ReadIndex**：Leader 在处理读请求时，记录当前 `commitIndex`，并向多数派发送心跳确认自己仍是合法 Leader，当确认成功且状态机至少应用到该 `commitIndex` 时，方可执行读取。**Lease Read**：Leader 维持一个比 Follower 选举超时短的时钟租约，在租约内无需网络交互直接服务读取。
*   **应用策略**: 对于高一致性系统采用 ReadIndex，对于吞吐量敏感系统可启用 Lease Read 但需注意时钟漂移风险。

---

## 📂 M4: 分片键值存储 (Cards 14-17)

#### Card 14. ShardController 配置版本化与负载均衡重平衡算法
*   **技术机制**: ShardController 管理集群中数据的分片（Shard，通常为 10 个）与 ShardGroup 之间的映射关系。每次 ShardGroup 发生增减（Join/Leave）时，ShardController 递增配置版本号（Configuration Version），并运行重平衡算法，使 10 个分片尽可能均匀分摊在存活的 ShardGroup 上，同时使分片迁移的数据搬迁量最小。
*   **应用策略**: 迁移映射计算必须是确定性的，以保证所有副本计算出的映射关系完全一致。

#### Card 15. 跨分片事务路由与客户端请求重定向机制
*   **技术机制**: 客户端启动时从 ShardController 拉取最新配置。当读写 Key 时，计算 `hash(Key) % N` 找到对应的 ShardID 及其所在的 ShardGroup。如果目标 ShardGroup 发生变更但客户端未更新配置，请求发送后会被目标服务器拒绝并返回 `ErrWrongGroup`。此时客户端会从 Controller 拉取最新版本配置并重试。
*   **应用策略**: 在客户端 SDK 内置退避重试与配置拉取机制，避免频繁抛出异常导致应用层逻辑中断。

#### Card 16. 动态分片迁移（Shard Migration）的拉取式 vs 推送式状态传输
*   **技术机制**: 当检测到配置版本更新（例如 Shard X 从 Group A 移到 Group B），Group B 必须在本地激活 Shard X 前，通过 RPC 从 Group A 传输该分片的所有数据。可以由 Group B 主动发起拉取（Pull）请求，读取 Group A 的 Snapshot 和增量 WAL；或者由 Group A 检测到新配置后，主动向 Group B 进行推送（Push）传输。
*   **应用策略**: 优先选用 Pull 模式，由接收方根据自身内存与磁盘资源情况控制并发流量，防止网络打满。

#### Card 17. 分片迁移并发控制：传输期间客户端读写的拦截隔离
*   **技术机制**: 为了防止迁移过程中的数据不一致，迁移是分阶段执行的。在源 Group A 未将 Shard X 的数据完整传输给 Group B 且 Group B 未确认接收前，Group A 对 Shard X 的写请求必须返回错误，且 Group B 在未确认迁移成功前也不能提供对 Shard X 的任何读写服务。此时该分片处于短暂的锁定/不可服务状态。
*   **应用策略**: 拆细分片粒度，降低单分片的数据大小，以将迁移服务中断时间缩短到毫秒级。

---

## 📂 M5: Raft KV 存储引擎内幕 (Cards 18-21)

#### Card 18. 客户端请求去重与幂等处理：Session ID 与序列号
*   **技术机制**: 在不可靠网络下，客户端发出的写请求可能因网络丢包或连接中断重试。为了防止状态机重复应用该指令，客户端在启动时申请唯一的 `ClientID`，并为每次写入分配单调递增的序列号 `SeqNum`。KV 状态机内部缓存每个 ClientID 的最新 `SeqNum` 和对应的响应。当收到重复的 `SeqNum` 时，直接返回缓存结果。
*   **应用策略**: 限制缓存大小，在状态机内定期淘汰处于非活动状态的客户端 Session，避免内存泄漏。

#### Card 19. 读写请求提交流程与 Raft Apply 异步事件循环
*   **技术机制**: KV 服务的处理逻辑是：1. 收到请求，调用 Raft `Start()` 提交日志。2. 异步监听 `applyCh` channel。3. Raft 达成共识后，后台循环从 `applyCh` 读取日志，将其应用到 KV 内存数据库，并缓存序列号。4. 唤醒对应日志索引上挂起的等待线程，向客户端返回响应。
*   **应用策略**: 使用哈希 Channel 分发机制，用多线程处理 apply 后续的状态机落地逻辑以提升单机吞吐。

#### Card 20. 状态机内存泄露防范与 Channel 关闭优雅清理
*   **技术机制**: 在 Raft 换届（Leader Change）时，之前挂起等待 `applyCh` 的请求线程由于新 Leader 覆盖了未提交的日志，将永远无法等到对应 Index 的 apply 信号。此时必须清理状态机中挂起的通知 Channel（发送超时或关闭事件），唤醒客户端让其进行重试，否则挂起的 Channel 和协程将导致严重的内存泄露。
*   **应用策略**: 在检测到 Leader 角色转变或 Term 变更时，主动遍历等待队列并关闭所有活跃的通知 Channel。

#### Card 21. 快照触发阈值与内存数据库写时复制（COW）防阻塞
*   **技术机制**: 内存数据库（如 Go Map 或 LevelDB）在生成快照时需要对内存数据进行完全读取。如果快照过程中有并发写操作，会产生竞态。通过在操作系统层面使用 `fork()` 产生子进程，利用操作系统的写时复制（COW）机制在子进程中进行安全的快照刷盘，主进程继续响应写请求，避免了全库加锁。
*   **应用策略**: 监控物理内存占用水位，COW 过程会因为大量并发写导致内存双倍膨胀，因此快照阈值要留有余量。

---

## 📂 M6: ZooKeeper 分布式协调 (Cards 22-28)

#### Card 22. ZooKeeper 层次命名空间与临时节点（Ephemeral Nodes）
*   **技术机制**: ZooKeeper 采用类似 Unix 文件系统的树形命名空间。节点称为 **Znode**。其中临时节点（Ephemeral Node）的生命周期与客户端的会话（Session）强绑定。一旦客户端宕机，心跳超时导致会话关闭，ZooKeeper 会自动删除该客户端创建的所有临时节点。
*   **应用策略**: 临时节点被广泛应用于分布式服务注册与发现（服务节点挂了自动从列表中剔除）。

#### Card 23. ZooKeeper 监听机制（Watcher）的一击即溃与通知机制
*   **技术机制**: ZooKeeper 允许客户端在特定的 Znode 上注册 **Watcher**。当 Znode 发生变化（如更新或删除）时，ZooKeeper 会向客户端发送一次性的异步通知。该通知是一次性触发的（One-time Trigger），一旦触发，Watcher 立即失效。如果客户端需要持续监听，必须在收到通知后重新注册。
*   **应用策略**: 客户端在收到 Watcher 通知后，拉取数据的同时重新注册 Watcher，注意两步骤间存在毫秒级空白盲区。

#### Card 24. Zab 共识协议：崩溃恢复与原子广播
*   **技术机制**: Zab 是专门为 ZooKeeper 设计的崩溃恢复模式协议。工作在两个阶段：**崩溃恢复**阶段（系统无 Leader 或 Leader 挂掉，Zab 选举出拥有最大 Proposal ID (zxid) 的节点为 Leader，并在多数派中完成状态同步对齐）；**原子广播**阶段（Leader 接收写请求，使用类似两阶段提交的 FIFO 广播协议写入 Followers，在收到多数派 Acknowledge 后提交）。
*   **应用策略**: 崩溃恢复是 Zab 保证脑裂收敛的关键，ZXID 包含 Epoch（世代）和 Counter，高 Epoch 拥有绝对权威。

#### Card 25. 分布式锁实现：利用 ZooKeeper 顺序节点与 Watch 避免惊群
*   **技术机制**: 在 `/lock` 目录下创建临时顺序节点（EPHEMERAL_SEQUENTIAL），如 `/lock/lock-0000001`。客户端获取 `/lock` 下所有子节点，如果自己的节点序号最小，则成功获取锁。如果不是最小，则仅在比自己小 1 的节点上注册 Watcher 监听删除事件。这样释放锁时只会唤醒一个等待者，规避了惊群效应（Thundering Herd）。
*   **应用策略**: 必须在本地维护节点路径。当网络重连后，需校验之前创建的节点是否遗留，防止产生僵尸节点。

#### Card 26. 成员发现服务与 Master 主备选举协调模式
*   **技术机制**: 实现主备高可用：多个节点在 ZooKeeper 上尝试创建相同的临时节点 `/master`。谁先创建成功谁就是 Active Master。其余节点创建失败，并转而在 `/master` 上注册 Watcher。当 Active Master 挂掉时，临时节点消失，其余节点收到通知后再次发起抢占创建请求，实现秒级的主备自动切换。
*   **应用策略**: 使用临时节点作为锁载体，通过节点存续与心跳机制，确保全局始终只有一个活动 Master 在运行。

#### Card 27. 脑裂防护防御线：隔离令牌（Fencing Token）与世代（Epoch）隔离
*   **技术机制**: 当 Master 遭遇 Full GC 产生长停顿时，ZooKeeper 会判定其会话超时并将其 `/master` 节点删除，备机抢占成为新 Master。当原 Master GC 结束后恢复，它仍然认为自己是 Leader 并尝试进行写入。隔离机制要求每次 Master 变更时递增 **Epoch（世代）**。后端数据库（如 GFS 或 DB）记录最新 Epoch，拒绝任何携带旧 Epoch 令牌的请求写入。
*   **应用策略**: 任何往底层存储发起的写请求，必须强绑定 Epoch 验证，将其作为微服务边界外调用的防御护城河。

#### Card 28. 会话维持、心跳协商与 Session 丢失自愈恢复
*   **技术机制**: 客户端与 ZooKeeper 集群建立 TCP 连接，并协商超时时间。客户端通过周期性发送心跳包保活会话。如果连接断开，客户端会在超时时间内尝试连接集群中的其他节点。如果在限定时间内成功重建连接，会话（及临时节点）仍可保留；如果超时，则会话被完全销毁，状态机必须感知并执行自愈清理。
*   **应用策略**: 慎重设置 Session Timeout 参数。太小会导致由于网络偶发抖动引发频繁的主备切换，太大则会推迟故障感知。

---

## ⚔️ MIT 6.824 核心架构参数与分布式排障字典 (Page 2 Bottom)

### T1: MIT 6.824 核心架构参数
*   **`raft-election-timeout-range: 150ms-300ms`**：Raft 领导者选举随机超时建议范围。该范围能有效降低瓜分选票概率，同时保证系统在 1 秒内完成故障收敛。
*   **`raft-heartbeat-interval: 50ms`**：领导者心跳探测包的发送间隔。应确保在一个最小选举超时（150ms）内至少发送 3 次心跳，防止偶发丢包引发退位。
*   **`zookeeper-session-timeout: 3000ms`**：ZooKeeper 客户端会话超时门限。超时判定后，集群会注销对应的临时节点并触发 Watch 广播。
*   **`gfs-chunk-size: 64MB`**：GFS 数据块物理大小。该大尺寸能大幅降低 Master 元数据内存占用，且有利于大数据的顺序流式读取。

### T2: 系统级分布式排障与诊断指令
*   查询当前 Raft 节点的运行状态、当前任期以及提交日志指针，诊断是否有节点落后：
    `curl -s http://localhost:8080/raft/stats | jq '.role, .currentTerm, .commitIndex'`
*   使用 iptables 模拟双向网络分区（脑裂），将节点 3 从 1, 2, 4, 5 的局域网中切离：
    `iptables -A INPUT -s 192.168.1.3 -j DROP; iptables -A OUTPUT -d 192.168.1.3 -j DROP`
*   使用 ZK 的四字指令诊断服务状态与连接状态，并过滤出当前节点是 Leader 还是 Follower：
    `echo stat | nc localhost 2181 | grep Mode`
*   诊断特定网卡由于缓冲区溢出导致的数据包静默丢包率，排查 Raft 同步异常：
    `tc -s qdisc show dev eth0 | grep -E "dropped|overlimits"`
