# awesome_scalability High-Density Core Card System (English Version)

## L0 ~ L2 Knowledge Ladder

*   **L0 One-Sentence Essence**: awesome-scalability compiles classic architectural patterns used by big tech in high-concurrency and high-scalability evolution. Its essence is to decouple complex businesses into microservices and event-driven architectures by making services stateless and horizontally scaling them, under the trade-offs of Command Query Responsibility Segregation (CQRS), asynchronous queuing for peak shaving, and system-level resilience (bulkheads/circuit breakers) to build highly fault-tolerant and infinitely scalable distributed systems.
*   **L1 Four-Sentence Logic**:
    1.  **Stateless Services & Horizontal Scaling**: Stripping state from the application layer to achieve a completely stateless design, enabling seamless horizontal scale-out via L4/L7 load balancers while offloading state to multi-level cache and shared databases to eliminate single points of physical bottleneck.
    2.  **Responsibility Segregation & Event Sourcing**: Implementing CQRS in complex scenarios to decouple the write side (strict business invariants) from the read side (highly denormalized read-optimized query structures), persisting state as append-only event logs in an Event Store, and updating read views asynchronously.
    3.  **Asynchronous Queuing & Reliable Delivery**: Integrating high-throughput message queues to decouple producers and consumers, guaranteeing at-least-once delivery semantics supplemented by message idempotency validation on the consumer side using distributed locks.
    4.  **Bulkhead Isolation & Chaos Resilience**: Enforcing strict thread-pool or semaphore-based bulkhead isolation via sidecars in service meshes to prevent cascading failure; utilizing chaos engineering to inject faults to build system self-healing and fault tolerance.
*   **L2 Core Data Flow Topology**:
    *   `Client Command` ➜ `Gateway Router` ➜ `JWT & Rate Limiter` ➜ `Command Controller` ➜ `Command Model` ➜ `Event Store (Append Only WAL)` ➜ `Publish Event to Kafka` ➜ `Event Consumer (Projector)` ➜ `Update Query DB (Read Model)` ➜ `Client Query` ➜ `Gateway Router` ➜ `Read Service` ➜ `Redis Cache (Hit)` ➜ `Cache Miss` ➜ `Query DB (Fast Return)` ➜ `Return Client`.

---

## ⚔️ High Concurrency & Availability Architecture Trade-offs Matrix (Page 1 Bottom)

| Design Dimension | Traditional Monolithic Intuition (⚠) | Underlying System Physical Reality (★) | Core Architectural Trade-off & Evolution |
| :--- | :--- | :--- | :--- |
| **System Scaling** | Scaling up the system by upgrading hardware configurations (CPU, RAM, Disk) of a single server when bottleneck occurs. | Single-machine hardware has hard physical limits with exponentially scaling costs; single point of failure (SPOF) risks remain. | **Scale-out & Statelessness**: Split applications, externalize state, and add nodes horizontally using L4/L7 load balancers. |
| **Microservices Read/Write** | Executing read and write transactions on the same database schema to simplify the model and maintain relational normalization. | Under high concurrency, complex JOIN queries and write transactions compete for row locks, table locks, and I/O resources, causing locks. | **CQRS (Read/Write Segregation)**: Decoupling write models (strict invariants) from read views (denormalized, flat structures). |
| **Distributed Transactions** | Applying synchronous, blocking Two-Phase Commit (2PC) to enforce ACID strong consistency across microservice boundaries. | Participants lock local resources until Commit, causing deadlocks if coordinator crashes or network splits. | **Sagas (Flexible Compensation)**: Partitioning into local transactions and executing compensating actions if failure occurs. |
| **Overload Protection** | Resisting sudden traffic surges by blindly spawning threads or auto-scaling microservice instances to absorb all requests. | Sudden traffic spikes cause thread storms, exhaust database connections, and propagate cascading failures up the call tree. | **Bulkheads & Active Rate Limiting**: Assigning isolated thread pools to individual services and applying circuit breakers to fail fast. |

---

## 📂 M1: Scalability Principles & Patterns (Cards 1-4)

#### Card 1. Vertical Scaling (Scale-up) vs. Horizontal Scaling (Scale-out) Physical Limits
*   **Technical Mechanism**: Scale-up upgrades single-server hardware (e.g., 128-core CPU, 2TB RAM) to increase throughput, bound by Moore's Law and bus bandwidth limits with non-linear costs. Scale-out builds shared-nothing clusters using commodity servers, distributing traffic via consistent hashing or hash slots.
*   **Application Strategy**: Use scale-up in early stages for quick optimization, and transition to scale-out shared-nothing architectures when database write limit or high availability requirements hit threshold.

#### Card 2. Stateless Services Shared-nothing Design and Session Externalization
*   **Technical Mechanism**: Application nodes are forced to store no client context (e.g., HTTP Session). Session state is offloaded to Redis clusters or database, and clients carry self-contained JWT tokens. L7 load balancers (Nginx/LVS) route requests round-robin, allowing arbitrary node scaling.
*   **Application Strategy**: Decoupling compute logic from data state is the fundamental prerequisite for microservices to perform sub-second Horizontal Pod Autoscaling (HPA).

#### Card 3. Shared-Nothing Architecture Isolation and Scalability in Distributed Compute
*   **Technical Mechanism**: In a Shared-Nothing architecture, each node possesses its own CPU, memory, and disk. Nodes communicate solely through high-speed networks, eliminating global locking bottlenecks and single points of contention.
*   **Application Strategy**: Modern distributed databases (e.g., Cassandra, ClickHouse Cluster) apply this model to achieve near-linear throughput scaling curves.

#### Card 4. Edge Computing & CDN Dynamic/Static Content Geolocation Routing
*   **Technical Mechanism**: Static resources (HTML, CSS, images) are cached at edge nodes, using Anycast IP and GeoDNS to route requests to the nearest edge. For dynamic data, edge compute workers (e.g., Cloudflare Workers) intercept requests for pre-auth and cache checking.
*   **Application Strategy**: At least 80% of read-only requests should be served at the edge layer under global high-traffic scenarios to protect the primary data center.

---

## 📂 M2: Database Partitioning & Routing (Cards 5-9)

#### Card 5. Read/Write Separation Replication Lag and "Read-Your-Own-Writes" Strategy
*   **Technical Mechanism**: Write transactions execute on Master while reads are routed to Slaves using binary log replication. Under high write loads, replication lag can reach seconds, causing clients to read stale data on Slaves immediately after a write.
*   **Application Strategy**: Enforce master routing for critical operations (e.g., reading order details right after checkout), or tag writes with timestamps (TSO) to delay Slave reads until Slave catches up.

#### Card 6. Horizontal Sharding and Shard Key Selection Pitfalls
*   **Technical Mechanism**: Sharding splits a table horizontally across multiple databases. The Shard Key determines routing. A poor Shard Key (e.g., partitioning by tenant ID when one tenant owns 90% of data) causes severe data skew and hot spot nodes.
*   **Application Strategy**: Choose shard keys with high cardinality and even distribution (e.g., `user_id`), and implement virtual buckets to distribute extreme peaks.

#### Card 7. Eliminating Distributed Cross-Shard JOINs: Denormalization vs. Application-level Joins
*   **Technical Mechanism**: JOIN operations across physical databases are impossible post-sharding. Resolution paths: 1) Denormalization: store redundant usernames directly in order table; 2) Application-level joins: middleware queries sub-tables and merges results via Hash Join in RAM.
*   **Application Strategy**: Prohibit cross-database JOINs in core OLTP databases, offloading complex multi-dimensional search to Elasticsearch.

#### Card 8. Distributed Global Primary Key: Snowflake Clock Drift Resolution
*   **Technical Mechanism**: Snowflake generates 64-bit unique IDs (Timestamp + Machine ID + Sequence). Since it relies on system time, NTP clock rollback causes ID collision.
*   **Application Strategy**: Capture time offsets. In case of minor drifts, spin-wait; for major rollbacks, reject writes or fallback to a backup machine ID or Raft-backed coordinator.

#### Card 9. Multi-Region Active-Active Database Replication and Conflict Resolution
*   **Technical Mechanism**: Under active-active multi-region writes, dual-replication suffers from network latency, causing conflicts. Resolution algorithms: Last-Write-Wins (LWW, synchronized clocks), Vector Clocks (causal history tracking), or CRDTs.
*   **Application Strategy**: Minimize concurrent modifications on the same entity. Route transactions cell-based (Cell-based Routing) so that a user's write is locked to a single data center.

---

## 📂 M3: Event-Driven & CQRS Architectures (Cards 10-13)

#### Card 10. CQRS Read/Write Model Physical Isolation and Eventual Consistency Window
*   **Technical Mechanism**: The Command side enforces transactions and validation rules; the Query side is customized for page views. They use separate databases, with Commands writing to Event Store, which publishes domain events to asynchronously project read-optimized structures.
*   **Application Strategy**: Ideal for systems with highly skewed read-to-write ratios (e.g., 100:1) and dynamic read views, accepting a brief eventual consistency delay.

#### Card 11. Event Sourcing Log-Structured Append-only Event Streams and State Rebuild Replays
*   **Technical Mechanism**: State changes are persisted as an append-only stream of event logs. Current state is reconstructed by replaying all historical events from baseline. Snapshots are taken periodically to optimize initialization time.
*   **Application Strategy**: Provides built-in audit trails and time-travel debugging, matching the requirements of financial transactions and DDD.

#### Card 12. Event-Driven Architecture (EDA) Publish-Subscribe Model and Microservice Decoupling
*   **Technical Mechanism**: Microservices interact asynchronously via Event Bus rather than blocking HTTP/RPC calls. Upstream publishers broadcast events without knowledge of consumers; downstream services subscribe independently.
*   **Application Strategy**: Reduces compile-time and runtime topology dependencies, preventing single-service outages from propagating, thus maximizing system scalability.

#### Card 13. Domain-Driven Design (DDD) Bounded Context Boundaries and Event Storming
*   **Technical Mechanism**: Subdivides complex systems into autonomous Bounded Contexts where domain models remain consistent. Bounded contexts are identified during "Event Storming" workshops, mapping commands and domain events to microservice boundaries.
*   **Application Strategy**: Prevents microservices from devolving into a "distributed monolith" caused by split-by-table schemas, ensuring high cohesion.

---

# Page 2 (Distributed, Async, Rate Limiting & Resiliency)

## 📂 M4: Asynchronous Messaging & Queues (Cards 14-18)

#### Card 14. Message Queues Peak Shaving and Traffic Shaping in High-concurrency Bursts
*   **Technical Mechanism**: Sudden spikes of incoming requests are transformed into events at the gateway layer and written directly to high-throughput message queue partitions (e.g., Kafka). Downstream consumers pull events at a steady, manageable rate, protecting databases.
*   **Application Strategy**: Traded instantaneous response times for system durability by buffer-storing peak traffic.

#### Card 15. At-most-once, At-least-once, and Exactly-once Semantics and Transactional Messaging
*   **Technical Mechanism**: At-most-once (no retries, loss allowed); At-least-once (retries and ACKs, no loss but duplicates allowed); Exactly-once (At-least-once + idempotent consumers or two-phase transactional message commits).
*   **Application Strategy**: Standardize on At-least-once delivery due to lower performance overhead, relying on consumer-side idempotency to simulate exactly-once.

#### Card 16. Consumer Idempotency: Unique Message IDs, Redis SETNX, and DB Constraints
*   **Technical Mechanism**: The publisher attaches a unique `message_id` to each message. The consumer acquires a lock in Redis via `SETNX` or inserts the ID into a de-duplication table before executing business logic. If it fails, the message is ignored and ACKed.
*   **Application Strategy**: Critical safety guard for payment processing, billing, and inventory reduction to prevent double-charging due to network retries.

#### Card 17. Queue Congestion Backpressure and Dynamic Producer Velocity Control
*   **Technical Mechanism**: When downstream consumer latency rises, queue managers throttle TCP receive windows or issue pause commands to upstream producers, blocking write calls or throwing overflow exceptions.
*   **Application Strategy**: Avoids out-of-memory crashes on queue brokers during consumer blockages by pushing pressure back to ingestion points.

#### Card 18. Dead Letter Queue (DLQ) Message Redelivery Thresholds and Fault Isolation
*   **Technical Mechanism**: When a message repeatedly fails processing or hits retry thresholds (e.g., 16 retries), it is automatically redirected to a Dead Letter Queue (DLQ) to prevent malformed payloads from blocking the partition processing pipeline.
*   **Application Strategy**: Set high-priority alerts on DLQ size; an increasing DLQ intake rate typically reveals upstream schema contract violations.

---

## 📂 M5: Resilience & Service Mesh (Cards 19-23)

#### Card 19. Circuit Breaker Three-state State Machine, Sliding Windows, and Cool-down
*   **Technical Mechanism**: Tracks downstream invocation error rate.
    *   **Closed**: Normal routing.
    *   **Open**: Triggers if failure rate exceeds threshold, instantly intercepting requests with local fallback.
    *   **Half-Open**: Activates after a cool-down delay, routing probe traffic. Reverts to Closed on success, or Open on failure.
*   **Application Strategy**: Essential fault-isolation mechanism. Enable circuit breakers for non-critical downstream dependencies (e.g., user recommendations) to fail fast.

#### Card 20. Token Bucket vs. Leaky Bucket Rate Limiting Algorithms for Burst Traffic
*   **Technical Mechanism**: Leaky Bucket processes requests at a constant, fixed rate, smoothing traffic but causing queues and rejecting bursts. Token Bucket adds tokens at a fixed rate, allowing requests to consume accumulated tokens instantly to support bursts.
*   **Application Strategy**: Use Token Bucket at API gateways to allow bursty user actions, and Leaky Bucket for rate-limiting calls to sensitive third-party APIs.

#### Card 21. Bulkhead Pattern Thread Pool Isolation, Semaphore Limiters, and Cascading Outages
*   **Technical Mechanism**: Inspired by Titanic's watertight bulkheads. Sharing a single global thread pool across all microservice calls allows a single slow downstream service to exhaust the entire thread pool. Bulkheads allocate isolated thread pools or semaphores for each service.
*   **Application Strategy**: Industry-standard pattern (e.g., Hystrix/Resilience4j). Ensures that a degraded external dependency does not exhaust resources for healthy services.

#### Card 22. Service Mesh Sidecar Pattern, Control Plane, and Data Plane Decoupling
*   **Technical Mechanism**: Decouples network routing, mTLS, circuit breaking, and telemetry from application code into a dedicated Sidecar proxy process (e.g., Envoy) running alongside the service. A central control plane (e.g., Istio) distributes configurations.
*   **Application Strategy**: Standardizes platform-wide traffic management and security policies, removing library dependency locking in multi-language environments.

#### Card 23. Timeout Budgets and Exponential Backoff with Jitter for Anti-Thundering Herd
*   **Technical Mechanism**: Retries must expand backoff intervals exponentially (e.g., 1s, 2s, 4s, 8s...) and inject a randomized "Jitter" delay to prevent clients from synchronized thundering herd spikes.
*   **Application Strategy**: Enforce randomized jitter across all retry attempts in distributed clients to protect recovering servers from collapse.

---

## 📂 M6: Big Tech Case Studies (Cards 24-28)

#### Card 24. Netflix Chaos Engineering Chaos Monkey Fault Injection and System Immunity
*   **Technical Mechanism**: The philosophy of proactively injecting failures into production environment to verify resilience. Chaos Monkey shuts down virtual machine instances or injects latency during business hours, checking if systems self-heal.
*   **Application Strategy**: Uncovers hidden architectural vulnerabilities that would normally go unnoticed until peak traffic events, driving engineers to implement robust failovers.

#### Card 25. Twitter Timeline Push vs. Pull Write-diffusion Trade-offs
*   **Technical Mechanism**: Resolves high-concurrency read/write scaling:
    *   **Pull (Read Diffusion)**: Writes go to the author's outbox only; followers merge outboxes at query time. High read latency.
    *   **Push (Write Diffusion)**: Writes fanout asynchronously to all followers' home timelines. Low read latency, but celebrity tweets trigger write storms. Use hybrid push/pull based on follower count.
*   **Application Strategy**: Hybrid fanout routing is mandatory for massive social graphs where follower distributions exhibit power-law curves.

#### Card 26. Amazon Dynamo Masterless Replication, Vector Clocks, and Read/Write Quorums
*   **Technical Mechanism**: The foundational paper for decentralized AP databases. Features masterless architecture with vector clocks to track update causality and resolve conflicts. Consistency is governed by Quorum parameters: $N$ (replicas), $R$ (read nodes), $W$ (write nodes). If $R + W > N$, strong read consistency is guaranteed.
*   **Application Strategy**: Match globally distributed write-heavy systems (e.g., shopping carts) that require absolute availability, tuning Quorums to trade speed for consistency.

#### Card 27. Google Spanner Atomic Clocks + GPS TrueTime API for Global Serializable Transactions
*   **Technical Mechanism**: Achieves global serializable transactions by implementing TrueTime API, utilizing GPS receivers and atomic clocks in each data center. TrueTime guarantees a bounded clock error window $[t.\text{earliest}, t.\text{latest}]$ (typically <7ms), allowingMVCC serializability via Commit Wait.
*   **Application Strategy**: The state-of-the-art solution for globally distributed RDBMS, solving the long-standing distributed transaction consistency-latency dilemma.

#### Card 28. Uber Saga Pattern for Distributed Saga Orchestrator and Compensation
*   **Technical Mechanism**: In long-running microservice transactions where 2PC is too expensive, Saga splits the transaction into local steps $T_1, T_2, \dots, T_n$. If step $T_k$ fails, a central Orchestrator triggers compensating actions $C_{k-1}, \dots, C_1$ in reverse order to rollback state.
*   **Application Strategy**: Apply to multi-stage asynchronous workflows (e.g., booking-charging-matching rides) to avoid locking physical database rows.

---

## ⚔️ awesome-scalability Core Parameters & Troubleshooting Dictionary (Page 2 Bottom)

### T1: awesome-scalability Core Parameters
*   **`cell-based-routing: <boolean>`**: Cell-based partition routing to isolate specific user traffic completely inside a self-contained cell block.
*   **`vector-clock-max-versions: 10`**: The truncation depth for version lists in vector clocks to prevent metadata inflation.
*   **`circuit-breaker-ring-buffer-size: 100`**: Ring buffer size to compute circuit-breaker failure rates, balancing responsiveness and noise rejection.
*   **`read-write-quorum: N=3, R=2, W=2`**: Majority read-write quorum settings ensuring $R+W > N$.

### T2: System-level Troubleshooting & Diagnostics
*   Using wrk to load test endpoint throughput and latency percentiles under high concurrency:
    `wrk -t8 -c200 -d30s --latency http://127.0.0.1:8080/api/v1/orders`
*   Querying active sockets in CLOSE_WAIT status to diagnose thread-pool exhaustion on downstream services:
    `ss -ant | grep -c "CLOSE-WAIT"`
*   Describing consumer groups lag in Apache Kafka to check if backpressure or partition scaling is needed:
    `kafka-consumer-groups.sh --bootstrap-server 127.0.0.1:9092 --describe --group order-group`
*   Profiling reverse proxy TTFB and connect times via curl formatting parameters:
    `curl -o /dev/null -s -w "Connect: %{time_connect} TTFB: %{time_starttransfer} Total: %{time_total}\n" http://127.0.0.1:8080/health`
