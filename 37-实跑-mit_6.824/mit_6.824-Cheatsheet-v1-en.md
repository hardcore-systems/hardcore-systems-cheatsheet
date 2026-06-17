# mit_6.824 High-Density Core Card System (English Version)

## L0 ~ L2 Knowledge Ladder

*   **L0 One-Sentence Essence**: Distributed Systems engineering is the science of building reliable virtual single-image stateful services out of unreliable physical hardware (unstable networks, package drops, and machine crashes) using consensus protocols (like Raft) and State Machine Replication (SMR).
*   **L1 Four-Sentence Logic**:
    1.  **Parallel Computation & Storage**: Parallelizing huge datasets using MapReduce partition shuffling and scaling storage through GFS multi-replica file chunking.
    2.  **State Machine Replication**: Replicating log entries deterministic sequences through Raft Quorum leader elections and single-directional log replication to tolerate up to $F$ failures with $2F+1$ nodes.
    3.  **Config-driven Sharding & Migration**: Balancing load dynamically across shard groups under versioned configuration changes orchestrated by a centralized ShardController.
    4.  **Idempotency & Coordination**: Eliminating network retry side-effects using Client Session sequence numbers and managing cluster health/locking using ZooKeeper ephemeral nodes and watches.
*   **L2 Core Data Flow Topology**:
    *   `Client Request` ➜ `ShardRouter Lookup Config` ➜ `Target Shard Raft Leader` ➜ `Leader Append Log locally` ➜ `AppendEntries RPC sent to Followers` ➜ `Follower logs persisted` ➜ `Follower returns Success` ➜ `Leader detects majority accepts` ➜ `Leader advances commitIndex` ➜ `ApplyMsg sent to DB State Machine` ➜ `Update Client Session Cache` ➜ `Linearizable DB State returned to Client`.

---

## ⚔️ High Concurrency & Availability Architecture Trade-offs Matrix (Page 1 Bottom)

| Design Dimension | Traditional Redundancy Intuition (⚠) | Underlying System Physical Reality (★) | Core Architectural Trade-off & Evolution |
| :--- | :--- | :--- | :--- |
| **Consensus Protocol** | Utilizing multi-write or multi-master databases to maximize parallel write throughput. | Network partitions cause split-brain, leading to unresolvable data divergence and silent loss. | **Quorum-based Consensus**: Enforcing a single-active Leader Raft topology, committing entries via majority consensus. |
| **Consistency Guarantee** | Reading directly from any local replica node to minimize query latencies. | Replicas can lag behind the leader, causing clients to read stale or partitioned data. | **Linearizable Reads**: Forcing leaders to verify quorum authority via heartbeats or clocks before responding. |
| **SMR Consistency** | Syncing replicas by sending raw SQL queries or shell commands with side effects. | Non-deterministic variables (e.g. `NOW()`, `RAND()`, or execution orders) lead to replica state drifts. | **Deterministic Log Application**: Replicating only append-only, ordered log entries applied in identical order. |
| **Shard Rebalancing** | Adjusting router definitions instantly and copying shard files asynchronously under load. | Concurrent updates during migrations trigger write conflicts or dirty routing state mismatches. | **Two-Phase Shard Migration**: Increasing configuration version inside ShardController, locking source shard first. |

---

## 📂 M1: MapReduce & GFS (Cards 1-4)

#### Card 1. MapReduce Task Scheduling and Shuffling Mechanics
*   **Technical Mechanism**: MapReduce execution separates into Map and Reduce phases. The Map phase reads inputs, processes records, and writes intermediate `(Key, Value)` pairs into memory buffers. These are spilled to local disk partitioned by key hashes. The **Shuffle** phase pulls partitions from Map nodes across the network, executes an External Merge Sort to group values by Key, and passes them to Reduce functions.
*   **Application Strategy**: Configure large file merge buffers (e.g., 100MB) during shuffling to minimize disk thrashing and speed up merges.

#### Card 2. MapReduce Fault Tolerance and Worker Failover
*   **Technical Mechanism**: The Master periodically pings workers via heartbeats. If a Map worker fails to respond within a timeout, the Master resets its completed and in-progress tasks to `Idle`. Completed Map tasks must be re-run because their intermediate outputs are stored on the failed worker's local disk. Completed Reduce tasks do not require re-run since their outputs are committed to GFS.
*   **Application Strategy**: Ensure Master progress metadata changes are idempotent, shifting task pointers without producing duplicate output files.

#### Card 3. GFS Architecture: Master-Chunkserver Lease Coordination
*   **Technical Mechanism**: GFS consists of a single Master and multiple Chunkservers. Large files are split into 64MB Chunks. During writes, the Master grants a **Lease** to a Primary replica chunk. The Primary determines the serial write order. Data is pushed to all replicas' memory in a pipeline, and then the Primary issues write commands to commit to disk.
*   **Application Strategy**: Master chunk leases offload consensus management from the control plane, eliminating per-write Master serialization overhead.

#### Card 4. GFS Record Append Consistency and Silent Data Corruption
*   **Technical Mechanism**: GFS guarantees **At-Least-Once** write consistency for concurrent appends. If a write fails midway, retries introduce padding holes or duplicate entries on some replicas. GFS master sweeps these differences. Chunkservers compute Checksums locally, comparing them during idle cycles to report silent disk corruptions and trigger replica recovery.
*   **Application Strategy**: The application layer must handle duplicate records, tagging records with unique IDs and de-duplicating on the client side.

---

## 📂 M2: Raft Consensus - Leader Election (Cards 5-8)

#### Card 5. Randomized Election Timeouts and Split Vote Prevention
*   **Technical Mechanism**: When a Follower loses heartbeats, it increments its Term and becomes a Candidate. If all Followers timeout at once, they split votes, causing no leader to emerge. Raft randomizes election timeouts between `[T, 2T]` (typically `150ms-300ms`), ensuring the first node to timeout collects quorum votes before others.
*   **Application Strategy**: Set randomized timeout intervals significantly larger than the network RTT to allow Candidates time to gather votes.

#### Card 6. Raft Election Safety: Log Up-to-date Comparison Rules
*   **Technical Mechanism**: Raft guarantees committed logs are never lost. Followers reject Candidate votes unless the Candidate's log is at least as up-to-date as theirs. The Follower compares: Candidate's last log term (higher is newer), or if terms match, last log index (longer index is newer):
    $$\text{GrantVote} = (T_{\text{last,c}} > T_{\text{last,f}}) \lor \left((T_{\text{last,c}} == T_{\text{last,f}}) \land (I_{\text{last,c}} \ge I_{\text{last,f}})\right)$$
*   **Application Strategy**: This rule ensures only nodes possessing all committed logs can lead, eliminating the need to sync historical logs post-election.

#### Card 7. Joint Consensus and Single-Node Configuration Membership Changes
*   **Technical Mechanism**: Modifying configuration configurations directly (e.g., from 3 to 5 nodes) can cause two leaders to be elected simultaneously. Raft's Joint Consensus uses an intermediate configuration $C_{\text{old,new}}$ requiring majorities from both configurations. Single-node changes simplify this by altering only one node at a time, ensuring majorities overlap.
*   **Application Strategy**: Implement single-node configuration changes for migrations, running sequential additions/removals to prevent split elections.

#### Card 8. Leader Heartbeats, AppendEntries Probes, and Term Logical Clock
*   **Technical Mechanism**: Leaders send periodic empty `AppendEntries` RPCs to act as Heartbeats, resetting Followers' election timers. The Term acts as a logical clock. If a Leader receives an RPC response with a higher Term, it immediately steps down to a Follower. If a Follower receives a low Term RPC, it rejects it.
*   **Application Strategy**: Term comparisons serve as a lock-free authority check, resolving conflicts caused by partitioned or delayed leaders.

---

## 📂 M3: Raft Consensus - Log Replication & Safety (Cards 9-13)

#### Card 9. Log Replication Quorum and Commit Index Progression
*   **Technical Mechanism**: When a Leader receives client commands, it appends them locally and broadcasts `AppendEntries` RPCs. Once a majority of nodes (Quorum) confirm writing the log entry, the Leader increments its `commitIndex`, applies the entry to its state machine, and returns success to the client.
*   **Application Strategy**: Run odd-numbered cluster sizes (3, 5, or 7 nodes) to maximize fault tolerance relative to cost (3 nodes tolerate 1 crash, 4 nodes still tolerate only 1).

#### Card 10. Log Discrepancy Recovery: Fast NextIndex Backtracking
*   **Technical Mechanism**: Follower logs can drift due to crashes or partitions. Leaders track `nextIndex[i]` and `matchIndex[i]`. If a Follower rejects log sync, the Leader decrements `nextIndex`. For faster recovery, Followers return `ConflictTerm` and `ConflictFirstIndex`. The Leader skips matching terms, aligning indexes in fewer round trips.
*   **Application Strategy**: Enable fast log backtracking in high-latency environments to quickly align nodes returning from outages.

#### Card 11. Leader Commitment Limits for Prior Terms (Raft Safety Property)
*   **Technical Mechanism**: If a Leader commits log entries from prior terms by counting replicas, a new leader might overwrite these entries later. Raft safety dictates a **Leader can only commit log entries from prior terms by committing an entry from its current term** via majority count.
*   **Application Strategy**: Strictly enforce term matching during commitIndex updates to prevent overwriting seemingly committed history.

#### Card 12. Log Snapshotting and InstallSnapshot State Transfers
*   **Technical Mechanism**: To prevent logs from exhausting disk space, nodes periodically snapshot their state machines, discarding committed logs. The snapshot stores the `LastIncludedIndex` and `LastIncludedTerm`. Lagging followers whose logs are already discarded are synced via `InstallSnapshot` RPCs.
*   **Application Strategy**: Use Copy-On-Write (COW) snapshotting to dump state machines asynchronously without blocking incoming writes.

#### Card 13. Linearizable Reads: ReadIndex and Lease Read
*   **Technical Mechanism**: Reading directly from a Leader can return stale data if the Leader is partitioned. **ReadIndex**: Leaders record their current `commitIndex`, verify their status by querying a majority of nodes, and respond once their state machines catch up. **Lease Read**: Leaders maintain a time-bound lease, serving reads locally without RPCs.
*   **Application Strategy**: Use ReadIndex for strict linearizability, and Lease Read for high read-throughput if clocks are reliable.

---

## 📂 M4: Sharded Key-Value Stores (Cards 14-17)

#### Card 14. ShardController Versioning and Balance Shards Rebalancing
*   **Technical Mechanism**: The ShardController tracks sharding assignments (e.g., 10 shards mapped to ShardGroups). When ShardGroups join or leave, the controller increments its Configuration Version and re-balances shard assignments, minimizing data movements while ensuring even distribution.
*   **Application Strategy**: Ensure the rebalancing algorithm is deterministic, allowing all replicas to compute identical shard routing layouts.

#### Card 15. Cross-Shard Transaction Routing and Client Redirections
*   **Technical Mechanism**: Clients query the ShardController for the latest shard routing map. When writing a Key, the client hashes it to locate the ShardGroup. If the shard moved to another group and the client sends it to the old one, the server returns `ErrWrongGroup`, prompting the client to refresh its config.
*   **Application Strategy**: Implement client-side configuration caching and retry backoff policies to handle transient routing errors gracefully.

#### Card 16. Dynamic Shard Migration: Pull-based vs Push-based State Transfer
*   **Technical Mechanism**: When a configuration change shifts Shard X from Group A to Group B, Group B must copy Shard X's data before serving reads. Group B pulls shard states by requesting snapshots and incremental WALs from Group A, or Group A pushes the database records directly.
*   **Application Strategy**: Prefer pull-based migration to let the receiving node regulate traffic rates according to its local resource consumption.

#### Card 17. Shard Migration Concurrency: Isolation and Request Blocking
*   **Technical Mechanism**: To prevent data inconsistency during migration, Shard X's keys are isolated. Group A rejects writes to Shard X once migration starts, and Group B rejects reads until it has received and committed Shard X's complete state. The shard is briefly locked.
*   **Application Strategy**: Keep shard data volumes small to reduce transfer times, keeping locked durations within milliseconds.

---

## 📂 M5: Raft KV Store Internals (Cards 18-21)

#### Card 18. Client Request Deduplication: Session IDs and Monotonic Sequence Numbers
*   **Technical Mechanism**: Under network disruptions, clients retry requests. To avoid duplicate state machine updates, clients acquire a unique `ClientID` and tag writes with a monotonic `SeqNum`. The state machine caches each client's last `SeqNum` and response, returning it directly for retries.
*   **Application Strategy**: Set a session expiration timer to garbage collect inactive Client IDs, preventing memory leakage over time.

#### Card 19. Write Request Lifecycles and Raft Apply Event Loops
*   **Technical Mechanism**: The KV server processes writes by: 1. Calling Raft `Start()` to append a log. 2. Listening to an asynchronous `applyCh`. 3. Applying committed log messages from `applyCh` to the database, caching sequence numbers. 4. Notifying the handler thread at the log index to return the response.
*   **Application Strategy**: Multi-channel event dispatching can scale state machine applications, processing post-commit writes concurrently.

#### Card 20. Raft Term Changes, Client Channel Cleanups, and Memory Leak Prevention
*   **Technical Mechanism**: During Leader changes, logs appended by the old leader might be overwritten by the new leader. Client threads waiting on specific log index channels will never receive apply notifications. The server must detect term changes, close pending channels, and force clients to retry.
*   **Application Strategy**: Clean up all waiting index channels whenever the server steps down or the term increments, preventing blocked goroutine leaks.

#### Card 21. Snapshot Triggering Thresholds and Copy-On-Write (COW) Memory Safety
*   **Technical Mechanism**: Dumping memory databases to disk requires consistent views. Locking the database during snapshotting blocks writes. Unix `fork()` exploits copy-on-write memory pages: the parent continues writing while the child dumps memory to disk, keeping snapshotting non-blocking.
*   **Application Strategy**: Monitor physical memory usage; COW can cause memory footprint spikes if write volumes are high during snapshotting.

---

## 📂 M6: ZooKeeper Distributed Coordination (Cards 22-28)

#### Card 22. ZooKeeper Hierarchical Namespace and Ephemeral Nodes
*   **Technical Mechanism**: ZooKeeper stores data in a tree structure of znodes. Ephemeral znodes are tied to the client's session lifecycle. If the client crashes and its heartbeat timer expires, ZooKeeper closes the session and deletes all its ephemeral znodes automatically.
*   **Application Strategy**: Use ephemeral nodes for service discovery registration, ensuring dead instances disappear from directory catalogs automatically.

#### Card 23. ZooKeeper Watcher Mechanics and One-time Trigger Constraints
*   **Technical Mechanism**: Clients can register Watchers on znodes. When a znode is created, updated, or deleted, ZooKeeper sends a watcher notification. Watchers are one-time triggers; once fired, they are removed. Clients must re-register Watchers if they require continuous monitoring.
*   **Application Strategy**: Re-register watchers immediately upon receiving notification, acknowledging a brief race window between notifications and reads.

#### Card 24. Zab Protocol: Recovery and Atomic Broadcast Phases
*   **Technical Mechanism**: Zab (ZooKeeper Atomic Broadcast) drives ZooKeeper consensus. It functions in: **Recovery** phase (elects leader with highest zxid and aligns follower histories) and **Broadcast** phase (FIFO two-phase commit logs, committing once a majority of nodes acknowledge).
*   **Application Strategy**: ZXID formats pack the Epoch and Counter. This enforces logical age checks, blocking old leaders from broadcasting proposals.

#### Card 25. Ephemeral Sequential Locks: Non-Thundering Distributed Locking
*   **Technical Mechanism**: Clients create EPHEMERAL_SEQUENTIAL znodes under `/lock` (e.g., `/lock/lock-0000001`). The client with the lowest sequence number holds the lock. Other clients watch only the znode with the sequence immediately preceding theirs, preventing thundering herds.
*   **Application Strategy**: Check lock status on reconnection, ensuring new session sequences do not leave zombie lock reservations behind.

#### Card 26. Active-Passive Master Coordination and Node Failover Patterns
*   **Technical Mechanism**: Multiple controllers attempt to create an ephemeral znode `/master`. The first creator becomes Active Master, while others register Watchers on `/master`. If the Active Master crashes, the znode disappears, and the watch fires, triggering a new election.
*   **Application Strategy**: Ephemeral nodes ensure active-passive master synchronization, binding master states to active sessions.

#### Card 27. Split-Brain Mitigation: Fencing Tokens and Epoch Verifications
*   **Technical Mechanism**: If an Active Master hangs during GC, ZooKeeper session timeouts expire, delete `/master`, and elect a standby. Once GC completes, the old Master might attempt database writes. Fencing tokens (increasing epochs) reject writes carrying older epoch tags.
*   **Application Strategy**: Pass fencing epochs to storage nodes (e.g., GFS Master), ensuring backend writes are verified against stale master updates.

#### Card 28. Session Keep-Alives, TCP Reconnections, and Timeout Recovery
*   **Technical Mechanism**: Clients maintain sessions with ZooKeeper via periodic heartbeats. If connection drops, the client reconnects to other cluster nodes. If reconnection succeeds before session timeout, ephemeral nodes persist; otherwise, state machines must rebuild registrations.
*   **Application Strategy**: Optimize session timeout values to avoid frequent master failovers from network drops, while keeping down detection delay reasonable.

---

## ⚔️ MIT 6.824 Core Parameters & Distributed Diagnostics Dictionary (Page 2 Bottom)

### T1: MIT 6.824 Core Parameters
*   **`raft-election-timeout-range: 150ms-300ms`**: Randomized timeout range for leader elections. This interval minimizes split vote rates and guarantees sub-second failover recovery.
*   **`raft-heartbeat-interval: 50ms`**: Leader heartbeat send rate. Ensures at least 3 heartbeat packets are sent within the minimum election timeout (150ms), preventing false timeouts.
*   **`zookeeper-session-timeout: 3000ms`**: Maximum allowed session heartbeat loss duration in ZooKeeper, triggering ephemeral node cleanup and watch notifications.
*   **`gfs-chunk-size: 64MB`**: Size of GFS chunk files. Minimizes metadata memory consumption on Master, optimize sequential read throughputs.

### T2: System-level Troubleshooting & Diagnostics
*   Querying the active role, term, and committed log index of a local Raft service node to diagnose lags:
    `curl -s http://localhost:8080/raft/stats | jq '.role, .currentTerm, .commitIndex'`
*   Simulating a network partition on a specific host to test split-brain consensus safety:
    `iptables -A INPUT -s 192.168.1.3 -j DROP; iptables -A OUTPUT -d 192.168.1.3 -j DROP`
*   Sending four-letter commands to a ZooKeeper node to retrieve current mode (Leader vs Follower) status:
    `echo stat | nc localhost 2181 | grep Mode`
*   Displaying network queue drop rates and packet drops on server interface to debug Raft packet loss:
    `tc -s qdisc show dev eth0 | grep -E "dropped|overlimits"`
