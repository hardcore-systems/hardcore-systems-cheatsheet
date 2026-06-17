# 《etcd-io / etcd》高密知识图谱与速查手册 (Cheatsheet)

*   **L0 一句话本质**：通过 Raft 协议在分布式节点间维护强一致性的复制状态机，配合底层的 MVCC 多版本存储与 BoltDB，为云原生架构提供高可用、强一致的元数据协调与服务发现底座。
*   **L1 四句话逻辑**：
    1.  **Raft 复制状态机** (M1) 保证了多节点间强一致性的日志追加与状态提交，通过 ReadIndex/LeaseRead 规避脑裂期间的脏读。
    2.  **WAL 与嵌入式 BoltDB** (M2) 是底层的物理存储支柱，WAL 顺序刷盘保证日志不丢，BoltDB 的 B+ 树则维护了 KV 状态。
    3.  **MVCC 与 KeyIndex** (M3) 实现了事务隔离与历史版本追溯，通过 Revision 递增版本控制解决写写冲突并支持历史压缩。
    4.  **Lease 与 Watch 机制** (M4 & M5) 提供了动态租约与异步事件流订阅，是分布式协调、分布式锁与配置下发的神经元枢纽。
*   **L2 存储引擎数据流演进拓扑**：
    *   [用户 gRPC 请求] $\rightarrow$ [Raft 状态机日志共识] $\rightarrow$ [顺序追加 WAL] $\rightarrow$ [Raft 提交 & Apply] $\rightarrow$ [写入 MVCC 内存 KeyIndex] $\rightarrow$ [后台 Batch 提交 BoltDB] $\rightarrow$ [更新 WatchableStore 队列] $\rightarrow$ [WatchStream 多路复用事件推送] $\rightarrow$ [LeaseManager 动态清理过期键]

---

## 🌐 分布式协调与强一致系统思维滤镜 (Etcd Epistemic Filter)

- **认识论 (Epistemology) - 从不可靠网络中构建绝对确定性**：
  在分布式环境中，“网络延迟是常态，分区是必然”。etcd 并不试图消灭网络分区，而是通过 Raft 共识协议，要求任何状态变更（如日志追加）必须在“法定多数派 (Quorum)”节点上持久化成功才算完成。这种以“Quorum 多数派共识”对抗“单点物理故障”的设计，是分布式系统确定性的认识论起点。
- **时间论 (Temporal Physics) - Revision 构成绝对的时空单向流动**：
  在单机数据库中，物理时钟用于判定先后关系，但在分布式系统中，物理时钟存在不可避免的漂移与不同步。etcd 抛弃了绝对物理时间，采用全局单调递增的 `Revision` 作为整个系统内唯一的“时空流向标”。每一次写入（即使是修改同一个 Key）都会产生一个全新的 Revision，通过 Revision，etcd 得以复现任意历史时刻的数据快照，并为 Watcher 提供精确的断点续传能力。
- **生存论 (Ontology) - 租约动态绑定与生命周期的物化**：
  分布式锁、临时节点、配置发现的本质都是“活性与死亡判定”。etcd 将数据的存在状态与 `Lease (租约)` 进行了解耦与多对一动态绑定。Lease 拥有独立的 TTL 生命倒计时，一旦倒计时归零，与之绑定的所有键值将在整个复制状态机中被级联擦除。通过这种物化生命周期的方式，etcd 将复杂的锁超时与防死锁降维为简单的租约续期逻辑。
---

## ⚔️ etcd 经典架构折衷与异常防范矩阵 (etcd Architectural Trade-offs)

| 开发者直觉 (⚠) | 系统物理现实 (✗) | 架构折衷与防范金标准 (✓) |
| :--- | :--- | :--- |
| **Follower 节点读取速度快且能线性扩展** | 读 Follower 可能由于 Raft 日志复制延迟导致“脏读”；开启强一致线性读（Linearizable Read）必须向 Leader 广播心跳确认，这会增加网络 RTT。 | 默认开启 ReadIndex 强读规避写磁盘；在时钟稳定（时钟漂移低于安全阈值）环境下开启 LeaseRead 获得 0 RTT 强一致读取性能；非核心高频配置查询可选用 Serializable 弱一致读取。 |
| **Watcher 能无限接收所有历史修改事件** | 若 Watch 客户端请求的历史 Revision 已经被 etcd 底层 Compact（压缩清理），将触发 `errCompacted` 错误，导致 Watch 建立失败。 | 生产端必须监控 Compaction 进度，并在客户端 Watch 模块实现退避重试：捕获 Compact 报错后，重新全量拉取当前最新 Baseline，再基于最新 Revision 重新建立 Watch 监听。 |
| **频繁创建短生命周期的独立租约非常方便** | 每个独立租约（Lease）都会在 Lessor 的最小堆中进行排序调度。数万个独立租约会引发高额的 CPU 排序消耗以及 KeepAlive 双向流心跳网络风暴。 | 实施“租约合并与共享（Lease Sharing）”设计。将拥有相似生命周期、过期时间的成千上万个 Key 绑定到同一个全局租约 ID 上，通过一次长心跳完成集中保活。 |

---

## 🗺️ 6大核心模块与 28 张高密速查卡片 (Core Cards Map)

### M1：Raft 共识与日志复制 (Slate Blue)

#### Card 1. Raft 状态机与任期机制 (Raft State Machine & Term)
*   **状态转换物理机**：节点在 `Follower` (跟随者)、`Candidate` (候选者)、`Leader` (领导者) 三种角色间单向转换。任期 `Term` 是逻辑时钟，每次选主自增。
*   **Term 安全约束**：任何节点收到比自己小的 Term 消息直接忽略；收到比自己大的 Term 消息则立即退化为 Follower，并更新本地 Term，以保障集群逻辑时钟强一致同步。

#### Card 2. 领导者选举与随机超时 (Leader Election & Randomized Timeout)
*   **随机化避撞 (Randomized Timeout)**：`Election Timeout` 随机化设置（如 1000ms - 3000ms），避免多个 Follower 同时超时成为 Candidate 并发起选举，造成选票瓜分（Split Vote）死锁。
*   **强一致选主特权**：Candidate 必须获得超过半数节点（Quorum）的投赞成票才能当选。投票规则为：本地日志比自己新（Term 更大，或 Term 相同但日志 Index 更大）的 Candidate 才能投赞成票，保证选出的 Leader 必定包含所有已提交的日志。

#### Card 3. 日志复制与两阶段提交 (Log Replication & Commit)
*   **三阶段流水线**：Leader 接收写请求 $\rightarrow$ 本地顺序追加未提交日志 $\rightarrow$ 向所有 Follower 广播 `AppendEntries` 消息 $\rightarrow$ 收到过半数 Follower 成功应答 $\rightarrow$ 标记日志为已提交（Committed）并广播应用（Apply）至状态机。
*   **状态追踪指针**：Leader 为每个 Follower 物理维护 `MatchIndex`（该 Follower 已同步的最大日志索引）与 `NextIndex`（下一次向该 Follower 发送日志的索引），通过回退探测自动对齐分区节点日志。

#### Card 4. 网络分区与 Pre-Vote 预投票机制 (Pre-Vote Phase)
*   **旧版痛点**：网络分区的 Follower 无法收到 Leader 心跳，会不断超时自增 Term 发起选举。当网络恢复后，该节点带着极高的 Term 归队，会导致正常工作的 Leader 强行退化，引起集群剧烈抖动。
*   **Pre-Vote 准入机制**：Follower 超时后先进入 Pre-Vote 阶段，向集群发起不增加 Term 的“虚拟选举测试”。只有当 Pre-Vote 获得多数派同意后，节点才真正增加 Term 并转换为 Candidate，彻底规避了网络分区干扰健康集群的隐患。

#### Card 5. 动态变更与联合共识 (Membership Changes & Joint Consensus)
*   **配置交替风险**：直接从旧配置 $C_{old}$（如 3 节点）切换到新配置 $C_{new}$（如 5 节点）可能因配置变更消息到达节点的先后顺序不同，导致在同一索引位置产生两个独立的“多数派”从而导致脑裂。
*   **联合共识 (Joint Consensus)**：变更期间使用过渡状态 $C_{old,new}$。任何提案必须同时在 $C_{old}$ 和 $C_{new}$ 两个独立集合中均获得多数派批准，才算提交成功，随后再平滑推进至 $C_{new}$，保障成员动态热扩缩容时的绝对安全。

---

### M2：WAL 预写日志与存储 (Moss Green)

#### Card 6. WAL 预写日志顺序追加结构 (WAL Append-Only Structure)
*   **磁盘 I/O 物理对齐**：WAL 文件大小固定为 64MB。为消除磁盘分配指针时的文件空洞与随机 I/O 延迟，etcd 在创建 WAL 时利用 `fallocate` 系统调用在物理介质上预分配并锁定全部 64MB 连续块。
*   **内存写对齐**：写日志时以 `512 字节（扇区大小）` 进行对齐，并使用顺序追加（Append-Only）日志段，极大地利用了磁盘的顺序写性能。

#### Card 7. BoltDB/bbolt 嵌入式存储引擎 (bbolt B+ Tree Storage)
*   **物理存储底座**：etcd 采用纯 Go 实现的单文件嵌入式键值数据库 `bbolt`。内部通过 B+ 树物理页面组织数据，叶子节点存储实际的 Revision 和 KV 记录，非叶子节点存储路由索引。
*   **共享内存并发**：通过 `mmap` 技术将 BoltDB 的数据文件直接映射到用户态内存空间，规避了内核态到用户态的双重缓冲拷贝开销，支持极致的只读事务零拷贝并发。

#### Card 8. WAL 与 BoltDB 提交顺序屏障 (WAL Pre-write & BoltDB Commit Sequence)
*   **顺序金律 (Sequential Barrier)**：在处理事务时，任何键值更新必须 **先顺序写入 WAL 物理磁盘并执行 fsync 刷盘成功**，然后才能向 BoltDB 提交批量事务（Batch Commit）并应用到内存中。
*   **防崩溃隔离**：即使 BoltDB 写入过程中发生断电或操作系统崩溃，系统在重启时也可以通过重放 WAL 日志，无损重构出最新未同步到 BoltDB 的一致性数据。

#### Card 9. 快照机制与日志历史截断 (Snapshotting & WAL Truncation)
*   **内存膨胀防线**：为防止 WAL 无限膨胀撑爆磁盘，etcd 引入快照机制。定期将 BoltDB 中的当前状态机快照归档并固化到磁盘，生成快照文件。
*   **历史日志裁剪**：快照落盘后，etcd 会安全截断并回收该快照 Revision 之前的所有 WAL 日志段，只保留增量日志，实现内存开销与磁盘占用的动态平衡。

---

### M3：MVCC 多版本并发控制 (Plum Rose)

#### Card 10. Revision 全局递增序列号 (Global Revision)
*   **绝对时空座标**：etcd 维护一个 64 位全局单调递增的 Revision，代表集群的逻辑时钟。每一个修改操作（PUT、DELETE）都会导致该 Revision 自增 1。
*   **双版本标识 (Revision Struct)**：Revision 由两部分组成：`main`（外部物理写事务序列号）与 `sub`（同一个写事务/Batch 内不同操作的内部索引号，从 0 开始），构成了系统内任意键值状态的唯一索引。

#### Card 11. KeyIndex 内存 B-Tree 索引 (KeyIndex struct)
*   **内存索引大图**：BoltDB 中的键是 Revision，而用户查询的键是原始 `Key`。etcd 在内存中维护了一棵 B-Tree（KeyIndex），用于完成 `Key -> Revision` 的高速映射。
*   **Generation 机制**：每个 `keyIndex` 结构体包含一个 `generations` 数组。每个 Generation 代表该键的一次生命周期（从创建 PUT 到删除 DELETE），数组中记录了该生命周期内所有修改操作对应的历史 Revision 列表。

#### Card 12. MVCC 快照读隔离与 Range 查询 (MVCC Snapshot Isolation)
*   **只读快照免锁**：用户在读取数据时如果不指定 Revision，默认使用当前最新 Revision 进行一致性读；如果指定 Revision（如 `--rev=100`），etcd 会直接在内存 KeyIndex 中定位 Revision，再去 BoltDB 查出相应版本的 KV 值。
*   **无干涉读写并发**：由于历史版本物理保留在 BoltDB 中，因此读取操作完全不需要加锁，读请求可以在任何历史 Revision 的快照上并发执行，写请求在单独的事务中进行，彻底消除了读写冲突。

#### Card 13. 历史版本压缩与空间整理 (Compaction & DB Defragmentation)
*   **空间回收原理**：MVCC 随着写入增长会导致 BoltDB 文件不断增大。用户需定期触发 `Compaction`，物理删除指定 Revision 之前的所有历史版本数据。
*   **碎片重整 (Defragmentation)**：压缩只是在 BoltDB 树中释放了对应的 Key，并没有将物理磁盘空间归还给操作系统。需要通过 `etcdctl defrag` 指令对底层 BoltDB 进行页面重新排布与紧凑化，重建索引以物理回收磁盘碎片。

---

### M4：Lease 租约与生存期管理 (Terracotta)

#### Card 14. Lease 租约管理器物理结构 (LeaseManager)
*   **最小堆时间轮**：etcd 内部使用 `LeaseManager` 统一管理所有的租约（Lease）。为在 $O(1)$ 时间复杂度内感知到期租约，内部使用 **最小堆 (Min-Heap) 优先队列** 按照租约的绝对到期时间（Expiration Time）排序。
*   **高速 ID 映射**：同时维护一个以 `Lease ID` 为 Key 的哈希表，提供对租约续约与删除的快速检索映射。

#### Card 15. KeepAlive 续约与 gRPC 双向流 (Lease KeepAlive)
*   **长连接复用**：如果客户端为每个 Key 独立发送 HTTP 续约心跳，会导致网络连结和 CPU 负荷暴增。etcd 采用 gRPC 双向流（Bidirectional Stream）复用技术。
*   **网络流控优化**：客户端和 etcd 之间通过单条 gRPC 长连接上的双向流发送续约心跳（KeepAliveRequest/Response），实现了低延迟、低带宽占用的高频租约维护。

#### Card 16. 租约撤销与键联级清理 (Lease Revocation)
*   **到期物理销毁**：Lease 最小堆到期轮询线程检测到租期到期（TTL <= 0）时，会触发清理事务。
*   **联级删除 (Cascade Delete)**：从租约的关联键列表（Keys Associated）中提取出所有绑定的键名，生成对应的 Delete 事务写入 Raft 复制状态机，并在应用到状态机时从内存 KeyIndex 和 BoltDB 中物理清除这些 Key，向所有 Watcher 广播删除事件。

#### Card 17. 租约共享与可扩展性设计 (Lease Sharing)
*   **海量临时节点消胀**：在 K8s 中，可能有数万个 Pod 需要绑定生存期租约。如果为每个 Pod 独立申请 Lease，集群将充斥着大量的心跳。
*   **共享心跳池**：etcd 推荐采用“共享租约”设计，即创建一个全局的租约（如 60s TTL），让几千个不同的临时 Key 绑定到这个相同的 Lease ID。它们共享单次续期心跳，极大地降低了系统管理生命周期的总体开销。

#### Card 18. 基于租约的分布式锁 (Distributed Locks via Lease)
*   **抢占与保活并发**：利用创建 Key 的原子性（PUT Transaction）配合 Lease 实现分布式锁。锁的持有者通过创建绑定 Lease 的临时 Key 来声明锁的物理占用。
*   **心跳看门狗 (Watchdog)**：锁持有者在后台启动心跳看门狗线程，定期通过 KeepAlive 刷新 Lease 时间。一旦持有者进程发生死锁或宕机，Lease 自动到期，锁 Key 级联自动释放，完美防止集群产生死锁。

---

### M5：Watch 异步事件流 (Indigo)

#### Card 19. Watcher 与 WatchStream 订阅管道 (Watcher & WatchStream)
*   **事件分发管道**：客户端通过创建 `Watcher` 监听特定键或前缀。这些监听器通过 gRPC 多路复用聚合在同一个网络通道 `WatchStream` 上，实现低能耗事件流推送。
*   **响应语义**：当 etcd 状态机应用写事务时，所产生的 KV 变更记录（包括 Action、Key、Value、PreKV）都会被转换为 Watch Event 发送给相应的 Stream。

#### Card 20. WatchableStore 同步与未同步队列 (WatchableStore)
*   **双队列流量防抖**：`WatchableStore` 是 Watch 机制在内存中的核心实现。内部管理着两个监听器队列：`synced`（已同步队列，关注最新数据变化）与 `unsynced`（未同步队列，正在追溯历史 Revision 数据的监听器）。
*   **队列滚动**：当监听器追赶完所有历史 Revision 后，会从 `unsynced` 自动滑入 `synced` 队列，加入共享内存通知路径。

#### Card 21. 事件缓冲区与背压流控 (Event Buffer & Flow Control)
*   **内存耗尽防御**：当客户端网络较慢或消费能力不足时，事件会在 etcd 内存中积压。每个 Watcher 拥有固定大小的事件环形缓冲区。
*   **背压降级策略**：如果缓冲区溢出，etcd 会强行关闭该 Watcher，并返回 `errCompacted` 状态，触发慢消费者降级，要求客户端在恢复网络后主动发起重新监听。

#### Card 22. Watch 事件合并与去重机制 (Event Consolidation)
*   **网络吞吐瘦身**：当系统处于极高密度的写压力下，某个键在几毫秒内可能被更新数十次。如果将每次更新都单独发送，会造成网络拥堵。
*   **只发最新态**：etcd 在处理 WatchableStore 推送时，会尝试在缓存层（Event Buffer）对同一个键在同一个事务（main Revision）内的多次修改事件进行去重与物理合并，只推送该事务最终生效的最终态，消减事件网络风暴。

#### Card 23. 基于历史 Revision 的断点续传 (Watch Resume)
*   **防抖重连能力**：由于网络抖动导致 Watcher 断开时，客户端可以在重新建立 Watch 时指定断开瞬间的 `Revision`（如 `Watch(key, WithRev(last_received_rev + 1))`）。
*   **未同步滑行**：etcd 接收到该请求后，会将 Watcher 挂入 `unsynced` 队列，从 BoltDB 的历史 Revision 记录中扫描拉取该 Rev 之后的所有变动，增量补发完毕后再并入 synced 队列，确保数据流不丢失、不重复。

---

### M6：集群变更与强一致读 (Antique Gold)

#### Card 24. gRPC v3 强协议架构设计 (gRPC Protocol v3)
*   **协议栈跃迁**：etcd v3 彻底废弃了 v2 繁冗的 HTTP/1.x + JSON 通信方式，改用基于 HTTP/2 的 `gRPC` + `Protocol Buffers` 强契约交互。
*   **系统收益**：网络层借由 HTTP/2 的多路复用彻底消除了连接握手开销；反序列化层利用二进制 Protobuf 减少了 CPU 开销与内存分配，整体吞吐提升数倍。

#### Card 25. Linearizable 线性一致读与 ReadIndex (Linearizable Read)
*   **规避状态机死角**：如果 Follower 直接响应读请求，可能因同步延迟导致读到旧数据（脏读）。线性一致读要求读到的数据绝对是全局最新的。
*   **ReadIndex 机制**：Follower 收到读请求后，向 Leader 索要当前最新的已提交索引 `ReadIndex`。Leader 随即向 Follower 广播一次心跳，确认自己仍是合法 Leader（Quorum 同意）。随后 Follower 等待本地状态机应用指针 `ApplyIndex` 追上该 `ReadIndex`，便可安全在本地读取并返回，避免了昂贵的 Raft 日志写盘逻辑。

#### Card 26. LeaseRead 租约强一致读优化 (LeaseRead)
*   **规避网络心跳确认**：虽然 ReadIndex 避免了读请求写磁盘，但依然需要在节点间发送一轮心跳确认 Leader 身份，产生了一次网络往返延迟（RTT）。
*   **时钟置信租期**：`LeaseRead` 在物理时钟漂移在极小安全阈值内的前提下，利用 Leader 的 Raft 租约有效期进行判断。只要 Leader 确信自己本地的租约有效期仍未过期（在此期间不可能产生新 Leader），就可以直接跳过网络广播确认步骤，本地直接读取返回，将 RTT 降为 0。

#### Card 27. 客户端智能负载均衡与容灾 (Client Load Balancing)
*   **去中心化寻址**：etcd 客户端集成了集群端点动态发现（Endpoints Discovery）与轮询重试逻辑。
*   **自动漂移**：客户端初始化时只需填入一个 Endpoint，便会自动调用 `Cluster` 接口获取所有节点列表并缓存在内存中。当当前连接节点发生硬件宕机或断网时，客户端能够秒级感知并透明重试漂移到其他可用节点，保持服务高可用。

#### Card 28. Etcd 网关与代理转发模式 (Etcd Gateway & L4 Proxy)
*   **物理架构解耦**：当客户端节点成千上万（如海量边缘物理节点）时，直连 etcd 集群会导致 etcd 节点上的 TCP 端口连接数暴增，耗尽系统文件描述符（FD）。
*   **边缘流量代理**：部署 `etcd gateway` 或 `etcd grpc-proxy` 作为四层无状态反向代理。网关汇聚边缘客户端的 gRPC 连接，并通过少量、复用的长连接与核心 etcd 集群交互，起到了连接数防火墙的作用。

---

## 🔬 辅助版块 Zone T：生产诊断与调试字典 (Zone T)

### T1 etcd 关键配置调优阈值表

| 配置参数 | 默认数值 | 生产建议数值 | 调优物理语义 |
| :--- | :--- | :--- | :--- |
| `--max-request-bytes` | `1.5 MB` | `1.5 MB - 10 MB` | 限制单个 RPC 包的大小，防止超大 KV 写入导致内存波动和 Raft 链路拥堵。 |
| `--quota-backend-bytes` | `2.0 GB` | `2.0 GB - 8.0 GB` | 数据库配额极限。达到此值会触发 `mvcc: database space exceeded` 错误，写请求挂起，必须进行历史压缩与碎片重整。 |
| `--heartbeat-interval` | `100 ms` | `100 ms - 250 ms` | Raft 心跳发送周期。网络较差的跨机房部署应适当调大以防频繁误判选主。 |
| `--election-timeout` | `1000 ms` | `1000 ms - 3000 ms` | 选主超时周期。必须为心跳周期的 5 - 10 倍以防发生脑裂与反复重选。 |

### T2 etcdctl 生产调试高频指令字典

*   **1. 读写与历史版本查询**
    ```bash
    # 顺序写入一个键值并指定租约
    etcdctl put /prod/config/db_host "10.0.0.5" --lease=12345678abcdef
    # 获取键值最新版本
    etcdctl get /prod/config/db_host
    # 获取指定历史 Revision 时的快照值
    etcdctl get /prod/config/db_host --rev=45
    # 物理范围查询键值前缀
    etcdctl get /prod/config/ --prefix
    ```
*   **2. Watch 异步事件通道监听**
    ```bash
    # 实时监听特定键的前缀，以十六进制格式输出事件
    etcdctl watch /prod/config/ --prefix --hex
    # 模拟网络闪断恢复后，从 Revision 50 开始进行增量断点续传监听
    etcdctl watch /prod/config/ --prefix --rev=50
    ```
*   **3. Lease 租约保活与级联**
    ```bash
    # 创建一个 60 秒生存期的租约，返回唯一的 Lease ID
    etcdctl lease grant 60
    # 在后台持续以双向流保活特定租约防止过期
    etcdctl lease keepalive 12345678abcdef
    # 强制撤销租约，级联物理擦除所有绑定此租约的 KV 键
    etcdctl lease revoke 12345678abcdef
    ```
*   **4. 集群健康度与空间整理**
    ```bash
    # 打印集群所有端点的健康度、磁盘大小、Revision 状态与同步指数
    etcdctl endpoint status --write-out=table
    # 物理压缩删除 Revision 200 之前的历史版本数据
    etcdctl compact 200
    # 物理收缩 BoltDB，释放空闲碎片页面归还给宿主机操作系统
    etcdctl defrag
    # 物理保存当前时间点的一致性 BoltDB 快照文件
    etcdctl snapshot save /var/backups/etcd-snapshot.db
    ```
