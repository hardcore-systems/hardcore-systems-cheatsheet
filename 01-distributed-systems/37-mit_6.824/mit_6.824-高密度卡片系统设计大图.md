# mit_6.824-高密度卡片系统设计大图 (分布式系统工程)

本大图展示了 `MIT 6.824 / distributed-systems` 的 28 张核心卡片的物理依赖与控制流走向，并对应了核心共识算法、哈希分片、去重与分布式锁的数理推导模型。

## 1. 卡片依赖拓扑 (Mermaid Diagram)

```mermaid
graph TD
    %% M1: MapReduce & GFS
    C1["Card 1: MapReduce Shuffle"] --> C2["Card 2: MR Worker Failover"]
    C3["Card 3: GFS Master Leases"] --> C4["Card 4: GFS Record Append"]
    
    %% M2: Raft Leader Election
    C5["Card 5: Randomized Timeout"] --> C8["Card 8: Heartbeats & AppendEntries"]
    C6["Card 6: Election Up-to-date Safety"] --> C8
    C7["Card 7: Joint Consensus & Membership"] --> C8
    
    %% M3: Raft Log Replication & Safety
    C8 --> C9["Card 9: Quorum Log Replication"]
    C9 --> C10["Card 10: Fast NextIndex Recovery"]
    C9 --> C11["Card 11: Commit Safety Term Limit"]
    C9 --> C12["Card 12: Log Snapshotting"]
    C11 --> C13["Card 13: ReadIndex & Lease Read"]
    
    %% M4: Sharded Key-Value Stores
    C14["Card 14: ShardController Balance"] --> C15["Card 15: Transaction Routing"]
    C15 --> C16["Card 16: Shard State Transfer"]
    C16 --> C17["Card 17: Shard Lock Concurrency"]
    
    %% M5: Raft KV Store Internals
    C13 --> C18["Card 18: Client Deduplication Seq"]
    C18 --> C19["Card 19: Apply Loop Event Loop"]
    C19 --> C20["Card 20: Channel Cleanups"]
    C12 --> C21["Card 21: COW Snapshots"]
    
    %% M6: ZooKeeper Distributed Coordination
    C22["Card 22: ZK Ephemeral Nodes"] --> C24["Card 24: Zab Protocol Recovery"]
    C23["Card 23: ZK Watcher Trigger"] --> C25["Card 25: Non-Thundering Lock"]
    C22 --> C26["Card 26: Master Election Pattern"]
    C26 --> C27["Card 27: Fencing Token Epoch"]
    C22 --> C28["Card 28: Session Expiry Recovery"]

    %% Style Classes
    classDef m1 fill:#7A8B99,stroke:#333,stroke-width:1px,color:#fff;
    classDef m2 fill:#7D8F7B,stroke:#333,stroke-width:1px,color:#fff;
    classDef m3 fill:#9E828A,stroke:#333,stroke-width:1px,color:#fff;
    classDef m4 fill:#B58A7D,stroke:#333,stroke-width:1px,color:#fff;
    classDef m5 fill:#5F7582,stroke:#333,stroke-width:1px,color:#fff;
    classDef m6 fill:#BFA88F,stroke:#333,stroke-width:1px,color:#fff;
    
    class C1,C2,C3,C4 m1;
    class C5,C6,C7,C8 m2;
    class C9,C10,C11,C12,C13 m3;
    class C14,C15,C16,C17 m4;
    class C18,C19,C20,C21 m5;
    class C22,C23,C24,C25,C26,C27,C28 m6;
```

---

## 2. 核心算法与物理公式映射

*   **Raft 选举安全定理（Election Safety）之日志新旧对比规则**：
    *   在 RequestVote RPC 中，Follower 判断 Candidate 的日志是否至少与自己一样新：
        *   若 Candidate 的最后一条日志的 Term 较大，则为真。
        *   若 Term 相同，且 Candidate 的最后一条日志的 Index 较大或相等，则为真。
        *   数学表达式：
            $$\text{GrantVote} = (T_{\text{last,c}} > T_{\text{last,f}}) \lor \left((T_{\text{last,c}} == T_{\text{last,f}}) \land (I_{\text{last,c}} \ge I_{\text{last,f}})\right)$$

*   **Raft 日志快速回溯（Fast Recovery）之冲突索引跳过**：
    *   当 Follower 拒绝 Leader 的日志对齐请求时，回传三个关键字段：
        *   `ConflictTerm`：冲突位置 Follower 的任期号。
        *   `ConflictFirstIndex`：Follower 中拥有 `ConflictTerm` 的第一条日志的索引。
        *   `ConflictLen`：Follower 的日志总长度（当 Leader 发送的日志越界时）。
    *   Leader 处理逻辑：
        *   若 Leader 的日志中没有任期为 `ConflictTerm` 的日志，则直接设置 `nextIndex = ConflictFirstIndex`。
        *   若 Leader 有该任期的日志，则设置 `nextIndex` 为 Leader 中该任期日志的最后一条的下一索引。

*   **幂等客户端序列号去重与线性化读校验**：
    *   在状态机层面，对每个客户端请求缓存最后一次的返回状态：
        $$\text{State}_{\text{apply}}(C_i, S_k) = \begin{cases} \text{ApplyAndCache}(\text{Request}), & \text{if } S_k > \text{CachedSeq}(C_i) \\ \text{ReturnCachedResponse}(C_i), & \text{if } S_k == \text{CachedSeq}(C_i) \\ \text{DropRequest}, & \text{if } S_k < \text{CachedSeq}(C_i) \end{cases}$$
