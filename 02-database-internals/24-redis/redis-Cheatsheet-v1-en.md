# 《redis-internals》 High-Density Knowledge Map and Reference Guide (Cheatsheet)

*   **L0 One-Line Essence**: An in-memory key-value database engine that executes operations via a single-threaded event loop, maps datasets to compact CPU-cache-aligned representations, and ensures persistence and clustering via COW snapshots and Gossip shard routing.
*   **L1 Four-Line Logic**:
    1.  **Single-Threaded Event Loop** (M1) offloads socket I/O but executes command logic sequentially on `ae.c`, bypassing thread context switching and lock overheads.
    2.  **Compact Dynamic Structures** (M2) leverage SDS, Ziplist/Listpack, and Skiplists to maximize cache hits and perform progressive hashing via double hash tables.
    3.  **Copy-on-Write and Append-Only Logs** (M3) split background writes from main processes using child process page table sharing (`fork()`) and sequential WAL execution.
    4.  **Approximated Eviction and Gossip Clustering** (M4-M6) use randomized key sampling to replace global LRU locks and map datasets across 16384 slots without central metadata nodes.
*   **L2 Storage Engine Data Flow Topology**:
    *   [RESP Client Connection] $\rightarrow$ [ae.c Event Loop Listener] $\rightarrow$ [SDS Query Buffer Load] $\rightarrow$ [Dict Command Hash Match] $\rightarrow$ [Memory Address Seek & Progressive Rehash] $\rightarrow$ [AOF Append / RDB COW Fork] $\rightarrow$ [Master Backlog Broadcast] $\rightarrow$ [Cluster Slot Redirect]

---

## 🌐 Redis Memory Engine Physical Epistemic Filter

- **Epistemology - Memory Latency and CPU Cache Alignment as First Principles**:
  Redis operates on the principle that memory access is five orders of magnitude faster than disk and network I/O. Therefore, disk-based structures (like B-Trees and buffer caches) are discarded. Redis builds SDS strings, Skiplists, and Listpacks directly in virtual memory, focusing on cache line alignment (L1/L2) and reducing heap allocations and padding overheads.
- **Consistency Theory - Loose Persistence Trade-offs and Lock-free Replication**:
  In high-performance memory scenarios, frequent synchronous disk operations (like per-write `fsync`) create bottleneck gates. Redis uses asynchronous every-second AOF updates and RDB snapshots to accept minor data loss. Master nodes replicate updates via circular backlogs and asynchronous command streams, relying on Gossip epochs and Sentinel quorums to handle splits and failovers.
- **Methodology - Single-Threaded Core Loop and CPU Task Offloading**:
  Instead of using complex locking architectures across multiple threads, Redis relies on a single-threaded core loop in `ae.c` to parse and write keys. To prevent blocking, Redis offloads heavy operations: RDB generation uses OS copy-on-write page table separation, memory reclamation runs on background thread pools (`bio.c` via UNLINK), and socket I/O is managed by worker threads in Redis 6.0.

---

## ⚔️ Database Concurrency & Recovery Trade-offs Matrix

| Developer Intuition (⚠) | System Physical Reality (✗) | Architect's Golden Rule (✓) |
| :--- | :--- | :--- |
| **Large single instances are easier to manage and scale than sharded setups** | Instances exceeding 32GB experience latency spikes during RDB forks due to page table copying. Network synchandshake transfers can saturate link bandwidth. | Limit single instance footprints to 4GB-8GB. Use Redis Cluster Gossip slots or proxy sharding to scale out, and offload socket processing using Threaded I/O. |
| **Using simple expire commands is a perfect way to reclaim memory for cold keys** | Deleting millions of keys at the exact same second stalls the event loop during passive cleanup scans, resulting in latency spikes. | Inject random time jitter to expire values to smooth deletions across time, and rely on Master delete broadcasts to maintain replica consistency. |
| **Everysec AOF logging provides reliable transaction safety with zero CPU cost** | If disk write latency spikes, the main thread blocking on `write` system calls waits for previous fsync operations to finish, stall-locking database access. | Enable `no-appendfsync-on-rewrite` to suspend disk writes during background AOF rewrites, reducing write conflict latency under high disk load. |

---

## 🗺️ 6 Core Modules and 28 High-Density Cards (Core Cards Map)

### M1: Core Event Loop & Networking (Slate Blue)

#### Card 1. ae.c Single-Threaded Event Loop
*   **Event Merging**: The core loop in `aeProcessEvents` coordinates both network events (socket reads/writes) and timer tasks (e.g., key expiration, clustering heartbeat).
*   **Zero Locking**: By running entirely on a single thread, Redis avoids thread switching, mutex locking, and cache line invalidations caused by thread scheduling.

#### Card 2. I/O Multiplexing Abstraction
*   **OS Interface Wrapper**: Redis abstracts operating system non-blocking select interfaces via conditional compiles (`ae_select.c`, `ae_epoll.c`, `ae_kqueue.c`).
*   **Platform Selection**: Selects the fastest interface (epoll on Linux, kqueue on BSD) to support $O(1)$ event callbacks on thousands of sockets.

#### Card 3. Client Connection Buffers
*   **Buffer Controls**: Client connection descriptors allocate a 16KB query read buffer that expands to a 1GB safety limit, dropping connections exceeding this limit.
*   **Write Response Buffers**: Outbound payloads are formatted into client static buffers or appended to queue link lists before being flushed during socket write events.

#### Card 4. RESP Protocol Parser
*   **Binary-Safe Format**: RESP (Redis Serialization Protocol) uses prefix codes (e.g., `+` for strings, `$` for bulk strings, `:` for integers) to parse streams.
*   **Low Overhead**: The parser splits data payloads without JSON/XML parser allocations, ensuring safe execution with binary strings using length headers.

---

### M2: Dynamic Memory Data Structures (Moss Green)

#### Card 5. SDS (Simple Dynamic String)
*   **Length Trackers**: SDS structures store current length `len`, allocated capacity `alloc`, and header metadata `flags` to provide $O(1)$ length operations.
*   **Overflow Protection**: SDS allocation monitors bounds before write operations, expanding capacity (doubling if under 1MB, adding 1MB if over) to handle binary payloads.

#### Card 6. Dict Progressive Rehash
*   **Incremental Hashing**: Redis dictionaries manage hash collisions using two tables, `ht[0]` and `ht[1]`. If table size exceeds limits, `ht[1]` is allocated at double capacity.
*   **Rehash Index Tracker**: Commands (e.g., add, find, delete) migrate buckets step-by-step from `ht[0]` to `ht[1]`, tracking progress via `rehashidx` to avoid CPU blocks.

#### Card 7. Ziplist & Listpack Memory Alignment
*   **Contiguous Layout**: Ziplists store elements sequentially in a single memory block (`[prev_entry_len][encoding][entry_data]`) to avoid pointer memory overheads.
*   **Cascade Update Prevention**: Newer versions use Listpacks. Listpacks store element lengths as suffixes rather than prefixes, preventing cascade updates during insertions.

#### Card 8. Dict & Skiplist Integration
*   **ZSET Dual Layout**: Sorted sets (ZSET) integrate a hash table (`dict`) and a skiplist (`zskiplist`) to support different query patterns.
*   **Range Scan Acceleration**: The skiplist supports $O(\log N)$ range searches and sorting, while the hash table provides $O(1)$ key-to-score lookups.

#### Card 9. Quicklist & Intset
*   **Pointer Optimization**: A `quicklist` is a double-linked list where each node is a compact `ziplist`, reducing memory fragmentation.
*   **Integer Array Compression**: An `intset` stores integers in a compact sorted array, automatically upgrading elements (int16 $\rightarrow$ int32 $\rightarrow$ int64) to save memory.

---

### M3: Persistence Mechanisms (Plum Rose)

#### Card 10. RDB Snapshotting & Copy-on-Write
*   **Page Table Sharing**: Executing `BGSAVE` forks a child process sharing the parent's page tables to write database state to an RDB file.
*   **COW Page Split**: When the parent process writes to a page, the OS triggers a copy-on-write fault to clone the page, keeping the child's snapshot consistent.

#### Card 11. AOF Logging & Fsync Policies
*   **Log Buffer Serialization**: Database writes append RESP commands to an `aof_buf` buffer before flushing to disk.
*   **Fsync Configurations**:
    1.  `always`: Syncs on every event loop iteration; highly secure but bottlenecked by disk speed.
    2.  `everysec`: Async background thread syncs every second; balances write speed and safety (Standard).
    3.  `no`: Relies on OS write flushes; prone to data loss.

#### Card 12. AOF BGREWRITEAOF Rewrite
*   **Size Reduction**: When AOF files exceed limits, `BGREWRITEAOF` creates a child process to write the current memory state into a new file.
*   **Rewrite Buffer Tracking**: Active writes are appended to an `aof_rewrite_buf` buffer, which is merged into the new file by the main thread after rewriting.

#### Card 13. Hybrid Persistence Recovery
*   **RDB & AOF Merging**: Hybrid persistence writes the current dataset as a binary RDB snapshot at the header of the AOF file, appending subsequent writes as RESP commands.
*   **Recovery Speed**: Bypasses parsing overhead during initial startup phases while retaining incremental write logs.

---

### M4: Memory Management & Eviction (Terracotta)

#### Card 14. jemalloc Allocator & Fragmentation
*   **Arena Management**: Redis uses the jemalloc allocator to allocate fixed size slots, reducing memory page fragmentation.
*   **Fragmentation Ratio**: The `mem_fragmentation_ratio` in `INFO memory` monitors RSS versus allocated bytes; ratios exceeding 1.5 suggest fragmentation.

#### Card 15. robj Object Layer
*   **Abstraction Wrappers**: All database entries are wrapped in a `redisObject` header specifying data `type` and underlying memory `encoding`.
*   **Metadata Offsets**: Stores a `refcount` for shared memory integer reuse and a 24-bit `lru` field tracking access times or LFU counters.

#### Card 16. Approximated LRU Eviction
*   **Avoid Global Locks**: Avoids the locking overhead of global double-linked lists by replacing them with randomized sampling.
*   **Randomized Sampling**: When memory exceeds thresholds, Redis samples $N$ keys (default 5) and evicts the oldest key based on its `lru` offset.

#### Card 17. UNLINK Async Freeing
*   **Main Thread Block Prevention**: Traditional `DEL` operations on large collections block the main thread while freeing memory pages.
*   **Background bio Threads**: The `UNLINK` command unlinks the key from the dictionary instantly, passing the memory pointer to `bio.c` threads for async deletion.

---

### M5: High Availability & Replication (Indigo)

#### Card 18. Replication Backlog Buffer & PSYNC
*   **Replication Offset**: Master and replicas track replication progress using data stream offsets.
*   **Circular Replication Backlog**: Replicas disconnected from the master send their offsets. If the offset falls within the master's circular `repl_backlog_buffer`, it performs a `PSYNC` incremental sync.

#### Card 19. Sentinel Failover Consensus
*   **Three-Step Failover**:
    1.  **sdown**: A single Sentinel marks the master offline after heartbeats time out.
    2.  **odown**: Once a quorum of Sentinels confirms the status, the master is marked offline.
    3.  **Raft Leader Election**: Sentinels elect a leader via Raft to promote a replica and broadcast the new configuration.

#### Card 20. Cluster Gossip Protocol
*   **Cluster Auto-Discovery**: Nodes in a Redis Cluster communicate directly without central registration nodes.
*   **Gossip Messages**: Nodes exchange `PING/PONG` packets containing slot ownership mappings and node metadata to converge cluster state.

#### Card 21. Hash Slots & ASK/MOVED Redirects
*   **16384 Slots Mappings**: The cluster routes keys across 16384 logical slots.
*   **Redirections**: If a target slot is located on another node, the node returns a `MOVED` redirect. During migrations, it returns an `ASK` redirect.

#### Card 22. Transactions & WATCH Limits
*   **No rollback Support**: Redis transactions (`MULTI/EXEC`) group commands but execute all remaining instructions even if one fails, lacking ACID rollback support.
*   **WATCH CAS Locks**: The `WATCH` command monitors keys, aborting the transaction if another client modifies the watched key before execution.

#### Card 23. Lua Script Atomicity
*   **VM Isolation**: Redis executes Lua scripts inside an embedded Lua interpreter.
*   **Exclusive Execution**: While a script executes, the event loop blocks all other client requests, ensuring atomic operations.

---

### M6: Advanced Internals & Performance (Antique Gold)

#### Card 24. Threaded I/O Socket Offloading
*   **I/O Thread Pool**: Redis 6.0 offloads read and write system calls to helper threads to maximize networking performance.
*   **Single-Threaded Core**: Worker threads parse client packet buffers and write responses, while the core database operations remain single-threaded.

#### Card 25. Active Defragmentation
*   **Dynamic Heap Compaction**: Background active defragmentation sweeps memory allocations without service restarts.
*   **Data Pointer Redirection**: Relocates memory blocks to contiguous regions and updates dictionary data pointers to free fragmented pages.

#### Card 26. Client-Side Caching
*   **RESP3 Tracking**: Replicas use RESP3 tracking to cache values directly in client memory.
*   **Invalidation Alerts**: When a tracked key is modified, Redis broadcasts `Invalidation` packets to clients to clear local caches.

#### Card 27. Slowlog & Latency Doctor
*   **Execution Timing**: The `slowlog` records commands whose execution times in the core loop exceed thresholds, ignoring network delays.
*   **Diagnostics**: The `LATENCY DOCTOR` command diagnoses issues (e.g., fork latency, AOF write stalls) to identify performance bottlenecks.

#### Card 28. Modules API Extension
*   **Dynamic Library Integration**: The Modules API allows developers to load compiled C/C++ dynamic libraries into Redis.
*   **Native Execution**: Modules access internal memory structures directly to support custom types and optimizations without Lua overhead.

---

## 🔬 Zone T: Database Performance & Diagnostic Labs

### T1 Redis Typical Tuning Parameters

| Parameter / redis.conf Directive | Recommended Value | Tuning Description |
| :--- | :--- | :--- |
| `maxmemory <bytes>` | `50% - 70% of RAM` | Limits instance memory. Retain at least 30% for AOF/RDB fork processes. |
| `hash-max-ziplist-entries` | `512` | Controls when data structures transition. Below this limit, ziplists/listpacks compress memory up to 5x. |
| `repl-backlog-size` | `10 MB - 100 MB` | Size of the replication backlog. Increase in high-throughput environments to prevent full resyncs on disconnect. |
| `activedefrag` | `yes` | Enables active memory defragmentation to reclaim fragmented heap space without server restarts. |

### T2 Database Maintenance Commands

*   **1. Memory & Fragmentation Diagnostics**
    ```shell
    -- Displays memory usage metrics, focusing on used_memory_rss and mem_fragmentation_ratio
    redis-cli INFO memory
    ```
*   **2. Replication Offset Alignment**
    ```shell
    -- Displays replica state and offset metrics to analyze backlog window status
    redis-cli INFO replication
    ```
*   **3. Slow Command Collection**
    ```shell
    -- Collects the last 10 commands exceeding slowlog-log-slower-than execution thresholds
    redis-cli SLOWLOG GET 10
    ```
