# Developer Roadmap & Full-Stack System Topology Cheatsheet

## J-Ladder Hierarchical Model

### L0 One-Line Essence
The essence of the developer technical roadmap lies in establishing a vertical, topological knowledge matrix from low-level physical kernels and networks, through middle-tier databases and front-end rendering engines, to cloud-native orchestrations, ensuring the scalable delivery of high-availability systems.

### L1 Four-Sentence Logic
1. **Low-level OS Support**: Applications run atop OS kernel schedulers, managing hardware access via virtual memory address mapping and multiplexed I/O models.
2. **High-Concurrency Backend**: By linking transactional stores, cache buffers, message queues, and distributed state machines, server systems sustain heavy loads.
3. **Front-end State Engineering**: User interfaces are dynamically rendered by combining browser paint layout loops, virtual DOM diff engines, and safe cross-origin bounds.
4. **Cloud-Native Automation**: Decoupled deployment states are managed through Linux process sandboxing (Namespace/Cgroup) and Kubernetes reconciliation loops.

### L2 Core Data Flow
`User Browser Interaction` ➜ `V-DOM Diff Local Re-render` ➜ `CDN Edge Cache` ➜ `API Gateway Rate Limit` ➜ `Kubernetes Pod Microservice Pool` ➜ `Raft Distributed Consensus` ➜ `Redis Cache / MySQL Store`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: Process, Thread & Goroutine Scheduling
*   **Theory**: Execution units of computing. Processes have private virtual address spaces; Threads are the smallest schedulable OS units, sharing process-level resources; Goroutines run in user space, scheduled by user-level runtimes, avoiding kernel-space transitions.
*   **Details**: Thread context switches save/restore CPU registers and invalidate TLB caches. Goroutines use cooperative or pre-emptive scheduler loops, saving a few registers on thin user stacks, making switches 10-100x lighter than threads.
*   **Trade-off**: User-space coroutines cannot natively exploit multi-core CPUs without OS-level thread mapping. Executing blocking system calls inside a coroutine blocks the underlying OS thread, causing starvation unless bound to non-blocking APIs (e.g., `epoll`).

### Card 2: Virtual Memory & MMU Page Table Translation
*   **Theory**: Virtual memory decouples process addressing from physical hardware. The Memory Management Unit (MMU) translates virtual addresses (VA) to physical frame numbers (PFN) via hierarchical page tables.
*   **Details**: VA is split into indexes for multi-level page tables and a physical page offset. CPU-level Translation Lookaside Buffers (TLB) cache translations to bypass page-table walks.
*   **Trade-off**: Hierarchical page tables reduce memory footprint but add access latency (e.g., a 4-level page table requires 5 memory lookups on TLB miss). To minimize this overhead, configuring Huge Pages (2MB or 1GB) is critical for memory-bound workloads.

### Card 3: Inter-Process Communication (IPC) & Shared Memory Mutual Exclusion
*   **Theory**: Processes cannot access each other's memory directly due to isolation, necessitating kernel-mediated channels for data exchange or synchronization.
*   **Details**: IPC options include Pipes (half-duplex, kernel ring buffer, parent-child only), FIFOs (named pipes, any processes), and Shared Memory (fastest, direct physical memory mapping, bypasses copy buffers, synchronized via semaphores).
*   **Trade-off**: Shared memory lacks kernel-enforced write safety, leading to race conditions. Developers must integrate synchronization locks (e.g., POSIX semaphores or mutexes) to avoid data corruption and deadlocks.

### Card 4: file System Structures: inode & Virtual file System (VFS)
*   **Theory**: File systems organize persistent storage blocks. VFS abstracts concrete file systems (ext4, NTFS) to expose uniform interfaces (`open`, `write`) to user processes.
*   **Details**: Storage consists of superblocks, inode arrays, and data blocks. An inode records metadata (permissions, block pointers) and is mapped from a filename inside directory listings.
*   **Trade-off**: Inode capacity is fixed during formatting. Writing millions of tiny files can exhaust all inode slots while leaving ample physical space, throwing `No space left on device` errors. Object storage must be used to mitigate this.

### Card 5: Socket & Five I/O Models
*   **Theory**: Socket descriptors serve as endpoints for network data exchange. OS interfaces classify I/O transitions into five models based on event notifications.
*   **Details**: The five models are: 1. Blocking (waits for data); 2. Non-blocking (polling); 3. Multiplexing (`epoll` handles thousands of sockets); 4. Signal-driven; 5. Asynchronous (AIO, kernel copies data to user space in background).
*   **Trade-off**: POSIX AIO is complex and buggy on Linux. High-performance servers prefer the Edge-Triggered (ET) `epoll` multiplexing model coupled with non-blocking sockets in a Reactor pattern for maximum throughput.

### Card 6: Database Transactions (ACID) & MVCC
*   **Theory**: Transactions preserve ACID states. To enhance concurrency, modern database engines implement Multi-Version Concurrency Control (MVCC) to decouple reads from writes.
*   **Details**: Isolation levels scale from Read Uncommitted to Serializable. MVCC tracks records via transaction IDs and rollback pointers, utilizing Undo Logs and ReadViews to reconstruct historical rows without locking.
*   **Trade-off**: Under high write load, row-locking and gap-locking (e.g., InnoDB's Next-Key Lock) in Repeatable Read mode can lead to frequent Deadlocks. Minimizing transaction scope is required to mitigate this.

### Card 7: Redis Single-Threaded Architecture & Persistence (RDB/AOF)
*   **Theory**: High-speed, in-memory key-value database. Employs a single-threaded event loop to eliminate context switching and lock overheads, utilizing multiplexed I/O.
*   **Details**: Persistence options: RDB (point-in-time snapshots created via background process `fork` and Copy-on-Write) and AOF (append-only log of write commands, compacted via background rewriting).
*   **Trade-off**: Highly performant but vulnerable to blocking. Operations like AOF disk syncs (`fsync always`) or executing $O(N)$ operations (such as `KEYS *` or dealing with hot big keys) will stall the single thread, causing cascading timeouts.

### Card 8: Raft Consensus Protocol
*   **Theory**: Replicated State Machines use consensus algorithms to guarantee strong consistency across distributed nodes despite network failures.
*   **Details**: Raft splits consensus into: 1. Leader Election (using heartbeats and monotonic Terms; requires majority votes from nodes with up-to-date logs); 2. Log Replication (Leader replicates writes and commits once a majority acknowledges); 3. Safety.
*   **Trade-off**: Network partitions can trigger Split-Brain states where two leaders exist. To prevent stale reads, Leaders must verify heartbeats with a majority before serving reads, or utilize Lease times.

### Card 9: Kafka Storage: Sequential Writes & Sparse Indexes
*   **Theory**: High-throughput message broker designed around disk I/O optimizations, utilizing sequential file writes and kernel Page Cache management.
*   **Details**: Log segments are split into `.log` (payload) and `.index` (sparse offset mappings) files. Consumers scan sparse indexes via binary search to stream records directly to sockets using `sendfile` zero-copy.
*   **Trade-off**: Relying on Page Cache risks silent data loss on hardware power failures. Achieving zero data loss requires setting `acks=all` and syncing offsets synchronously across In-Sync Replicas (ISR).

### Card 10: gRPC & Protocol Buffers Serialization
*   **Theory**: Service registration and load balancing govern microservice architectures, utilizing low-latency RPC protocols for inter-service communication.
*   **Details**: gRPC uses HTTP/2 for transport and Protocol Buffers (Protobuf) for payload serialization. Protobuf encodes data as binary Tag-Length-Value (TLV) packages using Varints, yielding payloads 3-10x smaller than JSON.
*   **Trade-off**: Binary payloads reduce CPU cycles and bandwidth but are not human-readable, complicating network sniffing, debugging, and traffic routing at API gateways.

### Card 11: Browser Rendering Pipeline: DOM to Composite
*   **Theory**: The process by which browsers transform HTML, CSS, and JS strings into physical pixels.
*   **Details**: Stages: 1. Parse HTML to DOM, CSS to CSSOM; 2. Combine to compute Layout trees (coordinates and sizes); 3. Paint layers with drawing instructions; 4. Rasterize instructions into bitmaps; 5. Composite layers using GPU acceleration.
*   **Trade-off**: Causing Layout (Reflow) or Paint (Repaint) cycles is CPU-expensive. To maintain 60 FPS, developers must use CSS `transform` or `opacity` animations to bypass reflow/repaint, letting the GPU execute Compositing directly.

### Card 12: Virtual DOM & React Reconciliation Diff
*   **Theory**: Bypasses expensive browser layout recalculations by managing lightweight JS representations of the DOM tree in memory, diffing trees to execute minimal batch updates.
*   **Details**: Reconciliation employs a heuristic $O(N)$ Diff algorithm. It assumes different node types produce different trees, and lists of child nodes require stable, unique `key` attributes to track additions, moves, or deletions.
*   **Trade-off**: Using array index `index` as a `key` is a known anti-pattern. Inserting items at the start of a list forces the Diff engine to fail node reuse, triggering full component re-renders.

### Card 13: Build Tools: Webpack Dependency Graphs vs Vite ESM
*   **Theory**: Modern front-end building compiles, tree-shakes, and bundles modular assets for optimal client loading.
*   **Details**: Webpack builds static Module Graphs from entry points, bundling all assets into chunks. Vite leverages native browser ESM support, serving files dynamically during development by compiling modules on-the-fly via ESBuild.
*   **Trade-off**: Webpack is slow during dev but offers rich bundling plugins. Vite is instant in dev but relies on Rollup for production builds, and deep module trees can cause network waterfalls of HTTP requests.

### Card 14: SPA Frontend Routing & State Management
*   **Theory**: Single Page Applications load one HTML shell. Routing intercepts URL changes without reloading; State Management coordinates shared state across isolated component trees.
*   **Details**: Routing utilizes Hash (`window.onhashchange`) or History API (`pushState`). State managers (e.g., Redux) implement unidirectional data flow (Action ➜ Reducer ➜ Store) to ensure predictable updates.
*   **Trade-off**: Placing every local state in a global store pollutes memory and triggers unnecessary component re-renders. Local UI states must remain within local component states.

### Card 15: Web Security: XSS, CSRF & CORS
*   **Theory**: Browser Same-Origin policies restrict cross-site scripts. Attackers exploit missing input sanitization or browser credential forwarding to breach safety.
*   **Details**: XSS (Cross-Site Scripting) is countered via HTML escaping and CSP (Content Security Policy); CSRF (Cross-Site Request Forgery) is countered using anti-CSRF tokens and SameSite Cookie flags; CORS (Cross-Origin Resource Sharing) headers regulate safe domain access.
*   **Trade-off**: Configuring wildcard origins (`Access-Control-Allow-Origin: *`) in CORS bypasses同源 protections entirely, allowing malicious sites to read sensitive REST endpoints.

### Card 16: Linux Namespaces & Cgroups Containerization
*   **Theory**: Foundations of modern containers (e.g., Docker). Containers are regular host processes isolated through kernel-level namespaces and resource limits.
*   **Details**: Namespaces isolate system resources: PID, NET, IPC, MNT, UTS. Cgroups (Control Groups) restrict hardware allocations: CPU quotas, memory limits, and I/O bandwidth.
*   **Trade-off**: Containers share the host kernel. A kernel exploit from within a container can compromise the host. Additionally, violating Cgroup memory limits triggers the OS OOM Killer, terminating the container instantly.

### Card 17: Kubernetes Controllers & Declarative APIs
*   **Theory**: Kubernetes manages container clusters via a declarative API. Instead of imperative scripts, users define a "desired state" reconciled continuously by controllers.
*   **Details**: Controllers (e.g., Deployment) watch cluster state via control loops. If running Pod count drops below desired replicas, the controller schedules new pods; if a node crashes, it schedules replacements.
*   **Trade-off**: Declarative controllers provide self-healing but add control plane latency. Configuration errors (e.g., misconfigured rolling update maxUnavailable thresholds) can quickly roll out broken versions.

### Card 18: CI/CD Automation: Docker Layer Caching & Deployments
*   **Theory**: Continuous Integration and Deployment pipelines automate tests, builds, and releases to production.
*   **Details**: CI speed is optimized by ordering Dockerfile instructions to leverage layer caching (e.g., copying package lockfiles and installing dependencies before copying source code). Deployment strategies include Blue-Green and Canary.
*   **Trade-off**: Blue-Green deployments require double the hardware capacity. Canary deployments avoid this but force the network to handle concurrent API versions, requiring strict API backward compatibility.

### Card 19: Kubernetes Liveness vs Readiness Probes
*   **Theory**: Health checks governing container lifecycle. Liveness determines if the container needs a reboot; Readiness determines if it can receive traffic.
*   **Details**: Probes execute HTTP requests, TCP checks, or CLI commands. If Readiness fails, Service endpoints remove the Pod IP, halting traffic routing to that instance.
*   **Trade-off**: Confusing the two is catastrophic. Linking a Liveness probe to a downstream database check will force all pods to reboot continuously if the database is overloaded, worsening the outage.

### Card 20: Consistent Hashing Ring & Virtual Nodes
*   **Theory**: Traditional hashing (`hash(key) % N`) triggers massive cache invalidations when servers are added or removed. Consistent Hashing limits cache misses.
*   **Details**: Maps keys and servers onto a logical ring of size $2^{32}-1$. Keys route to the first server encountered clockwise. Virtual Nodes map single physical servers to multiple positions on the ring to balance load.
*   **Trade-off**: A node failure causes its clockwise neighbor to absorb all its load, risking cascading crashes. Virtual node allocation and server capacity overhead margins are required.

### Card 21: Microservice Circuit Breaker & Fallbacks
*   **Theory**: Isolates failing downstream dependencies (e.g., slow payment gateways) to prevent thread exhaustion and cascading failures in upstream callers.
*   **Details**: State machine: Closed (traffic flows), Open (errors exceed thresholds; calls fail-fast and execute Fallbacks), Half-Open (sends test requests after a cooldown period to determine recovery).
*   **Trade-off**: Circuit breaking degrades user experience (e.g., displaying static fallbacks instead of personalized recommendations). Thresholds must be tuned via load tests to avoid premature activations.

### Card 22: Rate Limiting: Leaky Bucket vs Token Bucket
*   **Theory**: Bounds incoming traffic to protect backend resources from denial-of-service overloads.
*   **Details**: Leaky Bucket buffers requests in a queue, releasing them at a constant rate; Token Bucket accumulates tokens up to a capacity, letting requests pass if tokens are available.
*   **Trade-off**: Leaky Bucket forces a strict constant rate, smoothing traffic but dropping bursty, valid requests. Token Bucket handles bursty traffic smoothly but requires careful capacity tuning to prevent downstream overloads.

### Card 23: Database Sharding & Snowflake IDs
*   **Theory**: When table sizes exceed millions of rows, indexing latency increases. Sharding splits tables across physical database instances.
*   **Details**: Rows are routed using a Shard Key. In distributed topologies lacking auto-incrementing databases, Snowflake generators create 64-bit monotonically increasing IDs based on timestamps, worker IDs, and sequence numbers.
*   **Trade-off**: Queries lacking the Shard Key force full-cluster scans, reducing throughput. Cross-shard joins and distributed transactions are highly expensive and should be avoided via data denormalization.

### Card 24: System Observability: Metrics, Logs & Traces
*   **Theory**: The capacity to infer internal states of distributed systems by analyzing telemetry outputs.
*   **Details**: Metrics track numeric aggregations over time (CPU, latency); Logs record discrete, detailed events; Traces map requests across microservices via TraceIDs and SpanIDs.
*   **Trade-off**: Storing full traces and logs introduces massive storage and network overhead. Production environments must implement sampling strategies (e.g., recording 1% of successful traces) to keep costs manageable.

### Card 25: Domain-Driven Design (DDD): Contexts & Aggregates
*   **Theory**: Methodology to handle software complexity by structuring systems around business domain models.
*   **Details**: Strategic: Bounded Contexts define explicit linguistic boundaries, linked via Anti-Corruption Layers (ACL). Tactical: Aggregates manage consistency boundaries, exposing an Aggregate Root to validate child entity updates.
*   **Trade-off**: Highly beneficial for complex domains but excessive for simple CRUD services, where DDD layers introduce boilerplate and overhead.

### Card 26: Test-Driven Development (TDD) Cycle
*   **Theory**: Process where tests are written before implementation code, clarifying design boundaries and enabling safe refactoring.
*   **Details**: Three steps: Red (write a failing test), Green (implement minimal code to make test pass), Refactor (improve code structure while keeping test green).
*   **Trade-off**: Yields high test coverage but slows down early prototyping. If business requirements change rapidly, maintaining test suites can become a bottleneck.

### Card 27: Serverless FaaS Cold Starts
*   **Theory**: Developers write code as isolated functions that scale down to zero when idle, eliminating idle compute costs.
*   **Details**: "Cold Starts" refer to the latency of provisioning containers, setting up runtimes, and initializing dependencies when a request hits a scaled-to-zero function.
*   **Trade-off**: Highly cost-efficient but unsuitable for latency-critical user-facing applications. Mitigations like using Node.js instead of Java, minimizing bundle size, and keeping instances warm add complexity.

### Card 28: LLMOps & RAG Vector Database Topology
*   **Theory**: Large Language Models suffer from hallucinations and knowledge gaps. Retrieval-Augmented Generation (RAG) injects factual document chunks into prompts.
*   **Details**: Documents are chunked and vectorized via embedding models, then indexed in vector databases (e.g., HNSW). Queries retrieve the top-K semantically similar chunks to serve as context for the LLM.
*   **Trade-off**: Highly dependent on chunking strategies. Bad chunking or retrieving irrelevant context can confuse the LLM, causing worse hallucinations.
