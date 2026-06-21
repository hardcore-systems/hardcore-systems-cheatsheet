# system_design_primer High-Density Core Card System (English Edition)

## L0 ~ L2 Knowledge Ladder

*   **L0 One-Sentence Essence**: The System Design Primer outlines the evolutionary patterns from monolithic to scale architectures under physical network bounds and CAP constraints, leveraging load balancers, multi-level caching, consensus, and asynchronous queues to balance consistency and availability.
*   **L1 Four-Sentence Logic**:
    1.  **Traffic Partitioning & Consistent Hashing**: Traffic is dynamically offloaded via DNS and L4/L7 load balancers using consistent hashing rings, where virtual nodes ensure that node changes cause only a fractional $1/N$ data migration.
    2.  **Multi-Level Caching & Write Policies**: Heat maps are intercepted at CDN and distributed cache layers, where read paths implement Cache-aside and write paths balance write latency vs consistency (Write-through vs Write-behind).
    3.  **Database Replication & Sharding Key**: Databases employ read/write separation with master-slave replication, scaling out write capacity via horizontal sharding keys, utilizing the Snowflake algorithm to prevent distributed primary key collisions.
    4.  **Consensus Protocols & Resiliency Fallback**: Distributed state is copied across nodes using Raft consensus, while microservices manage burst traffic via Token Bucket rate limits, circuit breakers, and feature toggles to prevent cascade failure.
*   **L2 Core Data Flow Topology**:
    *   `Downstream Request` ➜ `Anycast DNS / CDN Edge` ➜ `L4 LB (LVS)` ➜ `L7 Proxy (Envoy)` ➜ `Token Bucket Rate Limiting` ➜ `Cache-aside Cache Hit (Return)` ➜ `Cache Miss` ➜ `Slave DB Read` ➜ `Update Cache` ➜ `Write Request` ➜ `Write-behind Async Queue` ➜ `Master DB Write` ➜ `Binlog Replication` ➜ `Slave DB Update`.

---

## ⚔️ High Concurrency & Availability Architecture Trade-off Matrix (Page 1 Bottom)

| Design Dimension | Monolithic Intuition (⚠) | Physical System Reality (★) | Architectural Compromise & Evolutionary Design |
| :--- | :--- | :--- | :--- |
| **Distributed Routing** | Distribute requests among target nodes using simple modulo hashing (`hash(key) % N`). | Node scaling or node failures alter `N`, causing cache keys to invalidate globally, triggering database collapse. | **Consistent Hash Ring**: Distributes nodes on a circular ring using virtual nodes, restricting data migration to $1/N$ on change. |
| **Cache Updates** | Update database records first and simultaneously overwrite the corresponding cache data. | Interleaved read/write requests under heavy concurrent loads lead to race conditions, leaving stale data in cache. | **Cache-aside (Invalidate)**: Writes update the DB and directly invalidate the old cache entry, forcing a fresh read from DB next. |
| **High Concurrency Writes** | Force every client write transaction to block until database disk I/O persists successfully. | Disk I/O speeds are constrained by hardware limits; blocking writes causes lock queues and latency spikes under load. | **Write-behind (Async Queue)**: Writes commit to memory cache first and return; a background task asynchronously batch-syncs to DB. |
| **Distributed Key Gen** | Use auto-increment columns (e.g. `AUTO_INCREMENT`) on each SQL instance to generate primary keys. | Auto-increment primary keys collide across split database shards, preventing record merging and synchronization. | **Snowflake (No-Lock Gen)**: Generates 64-bit Long IDs based on timestamps, node IDs, and sequence IDs, producing 4096 sorted IDs/ms. |
| **Burst Traffic Peaks** | Spin up new worker threads or coroutines dynamically to process each incoming request immediately. | Sudden traffic surges (e.g. flash sales) trigger thread storming, saturating memory and crashing database connections. | **Asynchronous Message Queue**: Buffers peak traffic spikes in durable partitions, allowing worker nodes to pull tasks at safe rates. |

---

## 📂 M1: Load Balancing & Consistent Hashing (Cards 1-4)

#### Card 1. DNS Routing, Anycast IP & CDN Scheduling
*   **Mechanism**: The first line of defense in traffic partition. Anycast permits multiple physical routing nodes to announce the same IP address, directing client packets to the nearest node via BGP. This works alongside GeoDNS to resolve requests to nearest CDN edge servers, serving static files directly at network edges and reducing RTT.
*   **Strategy**: In global multi-datacenter deployments, Anycast and CDN geo-routing are critical to distribute high-capacity traffic.

#### Card 2. L4 Load Balancing vs L7 Proxying
*   **Mechanism**: L4 Load Balancers (like LVS DR mode) modify MAC addresses (DR) or IP headers (NAT) at the transport layer without parsing application data, handling millions of concurrent connections. L7 Load Balancers (like Envoy/Nginx) parse HTTP body and headers, enabling cookie stickiness, path-based routing, and SSL termination.
*   **Strategy**: Deploy LVS at the perimeter for packet-level routing, distribution, and multiplexing across downstream Envoy/Nginx clusters for L7 filtering.

#### Card 3. Consistent Hashing Ring Topology, Clockwise Routing & Scaling
*   **Mechanism**: Consistent hashing uses a virtual ring containing $2^{32}$ slots. Nodes and keys are mapped using the same hashing algorithm (e.g., MD5). Keys traverse clockwise to route to the first encountered node. Deleting a node only affects data immediately counter-clockwise to it on the ring.
*   **Strategy**: Apply to distributed caching pools (e.g., Memcached) to ensure cache hit ratios stay above 90% during node scaling.

#### Card 4. Virtual Nodes: Balancing Data Distribution & Preventing Hotspots
*   **Mechanism**: Hashing rings with few physical nodes can suffer from uneven distribution (data hot spots). Virtual Nodes (e.g., `ServerA#1`, `ServerA#2`) map virtual instances of physical nodes around the ring, balancing data distributions and accommodating node hardware differences.
*   **Strategy**: Cassandra and DynamoDB use virtual nodes to distribute write partitions symmetrically across clusters.

---

## 📂 M2: Multi-Level Caching & Consistency Strategies (Cards 5-9)

#### Card 5. Multi-Level Caching: Client, Gateway, Local & Distributed Cache
*   **Mechanism**: Layered caching works by mapping memory access speeds: browser cache (HTTP headers), CDN (edge cache), Nginx/Varnish (reverse proxy cache), application local caches (Guava/Ehcache), and distributed caches (Redis). Latency drops exponentially at higher layers.
*   **Strategy**: Utilize a funnel model to filter 95%+ of read queries at edge or memory cache layers, preventing backend DB exhaustion.

#### Card 6. Cache-Aside Strategy: Concurrency & Inconsistency Mitigation
*   **Mechanism**: Read: Check cache; if miss, query DB and populate cache. Write: Update DB and then delete (invalidate) the cache entry. Updating the cache directly on write causes write races; deleting cache first can cause a concurrent read to fetch stale DB data and write it back.
*   **Strategy**: To prevent stale cache states during complex write races, implement delayed double-deletion or subscribe to DB binary logs to invalidate asynchronously.

#### Card 7. Write-Through vs Write-Behind: Consistency & Latency Trade-offs
*   **Mechanism**: Write-through synchronous writes: applications write to cache, cache writes to DB, and both must succeed before returning. Write-behind (Write-back) asynchronous writes: application writes to cache and returns; a background task consolidates and flushes queue updates to DB.
*   **Strategy**: Choose Write-behind for high-throughput write scenarios (e.g., logging, metrics) where eventual consistency is acceptable, accepting the risk of loss on crash.

#### Card 8. Cache Eviction Algorithms: LRU & LFU Implementation Complexities
*   **Mechanism**: LRU (Least Recently Used) tracks recency with a hash table + doubly-linked list, evicting the list tail in $O(1)$ time. LFU (Least Frequently Used) counts access frequencies, grouping elements into frequency buckets to evict low-use entries, protecting against scan pollution.
*   **Strategy**: Redis uses an approximate LRU algorithm by randomly sampling keys (default: 5) and evicting the oldest, saving significant memory.

#### Card 9. Mitigating Cache Penetration, Breakdown & Avalanche
*   **Mechanism**: Cache Penetration (non-existent keys): filter queries via Bloom Filters. Cache Breakdown (hot key expiry): restrict DB queries to a single worker thread using locks (e.g., Redis `SETNX`) while others wait. Cache Avalanche (simultaneous key expiry): inject random time jitters into key TTLs.
*   **Strategy**: Implement these protection mechanisms to protect critical database layers from traffic spikes and malicious exploits.

---

## 📂 M3: DB Read/Write Split & Sharding (Cards 10-13)

#### Card 10. Master-Slave Replication: Sync, Semi-Sync, Async & Lag
*   **Mechanism**: Master database handles writes and outputs binary logs; slave replicas pull and replay logs. Async replication returns immediately; sync replication blocks until all replicas write logs; semi-sync replication returns once at least one replica acknowledges log receipt.
*   **Strategy**: To prevent reading stale data due to replica lag, route critical read-after-write requests to master or track transactional state.

#### Card 11. Database Sharding & Shard Key Selection
*   **Mechanism**: Sharding divides a table horizontally across database instances. A Shard Key (e.g., `user_id`) must have high cardinality and distribute queries evenly. Querying without a shard key forces the middleware to broadcast queries to all database partitions (Scatter-Gather).
*   **Strategy**: Evaluate shard keys to match access query paths, creating secondary indices or syncing data to Elasticsearch for multi-dimensional lookups.

#### Card 12. Distributed Global ID: Snowflake Collision Resolution
*   **Mechanism**: Snowflake generates 64-bit unique IDs: 1 sign bit (0), 41 bits timestamp (ms), 10 bits machine identifier (5 bits datacenter + 5 bits worker), and 12 bits sequence number. Clock drift verification and sequence increment ensure lock-free generation.
*   **Strategy**: Snowflake IDs are naturally ordered over time, improving B+ Tree insert performance in database clustering indexes compared to UUIDs.

#### Card 13. Database Federation & Denormalization
*   **Mechanism**: Database Federation splits a large database vertically by domain schema (e.g., User DB, Order DB). Denormalization replicates redundant attributes directly inside records, eliminating costly table JOIN operations during read queries.
*   **Strategy**: Rely on denormalization to optimize read pathways, using binary log sync tools (e.g., Canal) to maintain consistency.

---

# Page 2 (分布式、异步、限流与容灾)

## 📂 M4: Distributed Coordination & Consensus (Cards 14-18)

#### Card 14. CAP Theorem Limits, BASE Model & PACELC Trade-off Matrix
*   **Mechanism**: CAP theorem states that under network partitions (P), a system must trade off between consistency (C) and availability (A). PACELC extends this: even without partition (E), systems must choose between latency (L) and consistency (C). BASE guides AP architectures.
*   **Strategy**: Choose CP (strong consistency) for transactional systems and AP (eventual consistency) for social feeds or点赞 services.

#### Card 15. Decentralized Gossip Protocol in Cluster Synchronization
*   **Mechanism**: Peer-to-peer decentralization. Nodes select $k$ random neighbors periodically to exchange peer metadata and state tables. State updates propagate exponentially across the network, achieving eventual consistency and high fault tolerance.
*   **Strategy**: Used in Cassandra cluster state management and Redis Cluster topological replication to handle node joins and failures.

#### Card 16. Two-Phase Commit (2PC): Coordinator SPOF & Deadlock
*   **Mechanism**: 2PC relies on Prepare (coordinator asks CanCommit, participants execute local transactions and lock resources) and Commit (coordinator broadcasts DoCommit). If the coordinator crashes mid-commit, participants remain locked, causing resource blockages.
*   **Strategy**: Avoid 2PC in high-scale systems where network latencies are unpredictable, preferring saga or transactional outbox patterns.

#### Card 17. Three-Phase Commit (3PC): Timeout & PreCommit State
*   **Mechanism**: 3PC introduces a PreCommit phase and timeout settings. After CanCommit, participants enter PreCommit. If a participant fails to receive a commit message from the coordinator within a timeout, it commits automatically.
*   **Strategy**: While timeouts prevent permanent lock blockages, they can lead to split-brain data discrepancies if the coordinator actually issued an abort.

#### Card 18. Raft Consensus: Leader Election & Log Replication
*   **Mechanism**: Raft defines three node states: Follower, Candidate, and Leader. Followers transition to Candidates and request votes upon election timeouts. Once elected, the Leader replication stream logs, committing updates only after a majority of replicas write.
*   **Strategy**: Implement Raft for storing configuration metadata and orchestrating clustering nodes (e.g., etcd, Kubernetes control plane).

---

## 📂 M5: Async Comm & MQ Flow Control (Cards 19-23)

#### Card 19. Message Queue for Traffic Shaving & Load Levelling
*   **Mechanism**: Message queues act as temporary shock absorbers. Sudden inbound transaction spikes are written to queue partitions, while downstream workers pull messages at a consistent, safe processing rate, shielding database layers.
*   **Strategy**: Implement queues (Kafka/RabbitMQ) to decouple microservices and guarantee system survival during marketing campaigns.

#### Card 20. Message Delivery Guarantees: At-Most-Once, At-Least-Once & Exactly-Once
*   **Mechanism**: At-most-once (no retries, packets may drop); At-least-once (retries and ACKs ensure packet delivery but allow duplicates); Exactly-once (combines idempotent producers with transactional write protocols, causing high latency).
*   **Strategy**: Standardize on At-least-once delivery for robust transmission, routing duplicate checking to consumer application layers.

#### Card 21. Consumer Idempotence: Unique Message ID & Optimistic Locking
*   **Mechanism**: Consumer idempotency is key to handling At-least-once duplicates. Each message contains a unique UUID. Consumers lock this UUID via Redis `SETNX` or write to a database unique constraint, dropping duplicate processing attempts.
*   **Strategy**: Enforce unique transaction keys and optimistic locks on consumer updates to prevent double-charging or duplicate writes.

#### Card 22. Backpressure Mechanism & Producer Rate Control
*   **Mechanism**: If consumers slow down and queue depth hits safety thresholds, backpressure triggers. The broker suspends socket reads or transmits flow-control frames, halting the producer's write operations to protect system stability.
*   **Strategy**: Leverage backpressure flow controls to protect message brokers from memory starvation and disk exhaustion under load.

#### Card 23. Dead Letter Queue (DLQ): Delivery Timing & Isolation
*   **Mechanism**: If a message fails decoding or triggers persistent errors, the broker isolates it to a Dead Letter Queue (DLQ) after exceeding retry limits, preventing bad payloads from blocking the head of queue.
*   **Strategy**: Set up alerts on DLQ write volumes to detect code regression issues or api integration changes on upstream systems.

---

## 📂 M6: System Elasticity Design: Circuit Breaker, Rate Limiting & Avalanche Prevention (Cards 24-28)

#### Card 24. Circuit Breaker: Three-State State Machine Transitions
*   **Mechanism**: Circuit breakers isolate faulty services. States: **Close** (requests pass, error rate monitored); **Open** (error rate exceeds limits, calls bypass downstream to return fallback values); **Half-Open** (limited probes pass; close circuit on success, revert to Open on failure).
*   **Strategy**: Wrap microservice client calls and external integrations with circuit breakers to establish fault boundaries.

#### Card 25. Token Bucket vs Leaky Bucket Rate Limiting
*   **Mechanism**: Leaky Bucket processes packets at a constant rate, smoothing traffic but causing latency under bursts. Token Bucket adds tokens at a fixed rate, allowing clients to consume accumulated tokens to handle burst traffic.
*   **Strategy**: Apply Leaky Buckets to strictly constrained backend databases, and Token Buckets for consumer-facing public API gateways.

#### Card 26. Bulkhead Pattern: Thread Pool & Semaphore Isolation
*   **Mechanism**: Resource isolation. Sharing a single global thread pool means a slow downstream dependency A can occupy all worker threads, starving requests for dependencies B and C. Bulkheads assign dedicated thread pools to each service.
*   **Strategy**: Implement bulkheads in API gateways to contain downstream crashes and preserve core operations.

#### Card 27. Graceful Degradation & Feature Toggles
*   **Mechanism**: Trade minor feature experiences for system survival. Graceful degradation drops non-essential services (e.g., recommendations) via feature toggles, freeing up CPU and DB capacity for checkout processes.
*   **Strategy**: Schedule load tests to verify degradation playbooks and categorize toggle tiers (Level 1-3) before peak traffic campaigns.

#### Card 28. High Availability Evaluation: MTBF, MTTR & SLA
*   **Mechanism**: Availability is defined as $\frac{\text{MTBF}}{\text{MTBF} + \text{MTTR}}$, where MTBF is Mean Time Between Failures and MTTR is Mean Time To Repair. Reaching 99.99% availability (annual downtime < 52.56 minutes) requires minimizing MTTR.
*   **Strategy**: Focus on lowering MTTR through automated rollbacks, robust health checks, and detailed telemetry dashboards.

---

## ⚔️ System Design Parameters & OS Isolations Dict (Page 2 Bottom)

### T1: System Design Core Parameters
*   **`consistent-hash-replicas: <integer>`**: Virtual nodes count per physical host. Standardizing on 100~300 nodes minimizes hash ring data skew.
*   **`token-bucket-capacity / fill-rate`**: Capacity controls max allowed burst packets, while fill rate defines steady QPS limits.
*   **`circuit-breaker-failure-rate-threshold: <percentage>`**: The failure threshold percentage (e.g., 50%) to transition the breaker state to Open.
*   **`cache-max-memory / eviction-policy`**: Max cache memory limit and eviction policy (e.g., volatile-lru, volatile-lfu, volatile-ttl).

### T2: System-level Troubleshooting, Diagnostics & Benchmark Commands
*   Inspect CDN edge cache hit headers (verify `X-Cache` and `Age` response values):
    `curl -I -s https://example.com/static/logo.png`
*   Execute high-concurrency benchmarks to test load balancers and rate limiters:
    `wrk -t12 -c400 -d30s --latency http://127.0.0.1:8080/api/v1/resource`
*   Query master-slave replication sync latency and replication process status:
    `mysql -e "SHOW SLAVE STATUS\G" | grep "Seconds_Behind_Master"`
*   Verify Kafka consumer group lag to decide if backpressure scaling is needed:
    `kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --describe --group my-consumer-group`
