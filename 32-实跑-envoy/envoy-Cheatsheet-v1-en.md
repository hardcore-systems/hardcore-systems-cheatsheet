# Envoy Proxy High-Density Core Card System (English Version)

## L0 ~ L2 Knowledge Ladder

*   **L0 One-Line Essence**: Envoy is a high-performance L4/L7 proxy and service gateway that eliminates lock contention using a thread-per-core event loop, reloads configurations dynamically via xDS APIs, processes protocol pipelines using three-tiered filter chains, and enforces circuit breaking at pool limits.
*   **L1 Four-Line Logic**:
    1.  **Non-blocking Event Loop per Thread**: Pins an independent dispatcher thread to each core with zero cross-thread locking, utilizing `SO_REUSEPORT` socket sharing to distribute downstream connections natively via kernel hashing.
    2.  **API-driven 0-Downtime Hot Loading**: Updates listener, route, cluster, and endpoint states subscriptions over dual-channel gRPC xDS streams, swapping config objects in memory atomically via pointer swops with no latency penalty.
    3.  **Hierarchical Filter Chains & Translation**: Sniffs TLS SNI metadata via listener filters, terminates TLS in network filters, processes auth/routing inside HTTP filters, and bridges protocols (H1, H2, H3, gRPC) bi-directionally.
    4.  **Resiliency Limits & Active Probing**: Halts cascading overloads using upstream circuit-breaking counters, ejects failing endpoints via passive outlier detection, and validates server health status with active background probing.
*   **L2 Core Data Flow Topology**:
    *   `Downstream Request` ➜ `Listener Socket (Worker Event Loop)` ➜ `Listener Filters (TLS SNI)` ➜ `Network Filters (TLS Decrypt)` ➜ `HTTP Filters (Auth / Router)` ➜ `xDS RDS Match` ➜ `Upstream Conn Pool` ➜ `EDS Endpoint` ➜ `Upstream Server`.

---

## ⚔️ Envoy Threading Model & Concurrency Control Trade-offs Matrix (Page 1 Bottom)

*   **Developer Intuition (⚠)**: Share a single listening socket descriptor across multiple threads to maximize connection processing. ➜ **Underlying Physics (★)**: Multiple worker threads competing to run `accept()` on a single socket triggers kernel mutex contention and thundering herd CPU spikes. You must use `SO_REUSEPORT` to distribute connections directly via kernel symmetric hash flow.
*   **Developer Intuition (⚠)**: Reload static configuration files synchronously on the main thread to update routing tables. ➜ **Underlying Physics (★)**: Parsing large configurations blocks the main event loop, delaying network I/O events. You must use asynchronous gRPC subscription (xDS) compiled in background threads, swapping pointer values atomically in a lockless RCU-like transition.
*   **Developer Intuition (⚠)**: Trigger retries immediately on downstream clients when upstream nodes return timeouts or errors. ➜ **Underlying Physics (★)**: If the upstream cluster is overloaded, aggressive client retries multiply request volume, driving the system into a retry storm collapse. You must restrict retry allocations under 20% using retry budgets.
*   **Developer Intuition (⚠)**: Multiplex all upstream streams onto a single TCP connection to eliminate proxy network overhead. ➜ **Underlying Physics (★)**: On lossy links, a single packet drop halts the entire TCP connection window, blocking all multiplexed streams (Head-of-Line Blocking). On unstable links, you must utilize UDP-based HTTP/3 (QUIC) to isolate streams.

---

## 📂 M1: Non-blocking Threading Model & Event Loop (Cards 1-4)

#### Card 1. Thread-per-core Event Loop Architecture & Lockless Data Path
*   **Technical Mechanism**: Envoy runs a Thread-per-core model. A main thread orchestrates system states while worker threads process request pipelines. Each worker core runs a dedicated event dispatcher (`DispatcherImpl`) to run non-blocking network loops with zero cross-thread locking or thread contention.
*   **Application Strategy**: Set thread concurrency (`--concurrency`) to match the exact number of physical CPU cores to optimize processing efficiency.

#### Card 2. Libevent Event Loop Integration & Min-Heap Timer Architecture
*   **Technical Mechanism**: Wraps Libevent's `event_base` loops. File descriptors are polled asynchronously using epoll/kqueue. Timer structures (e.g. check intervals, request timeouts) are organized inside a Min-Heap tree, ensuring O(1) time complexity to find expired timers.
*   **Application Strategy**: Avoid blocking actions or intensive computations inside HTTP filters or Wasm plugins to prevent stalling the worker core's dispatcher.

#### Card 3. Socket SO_REUSEPORT listener Sharing & Worker Distribution
*   **Technical Mechanism**: By default, Envoy enables `SO_REUSEPORT` on listening ports, binding a separate socket instance per worker thread to the same port. The kernel hashes incoming connection packets to dispatch traffic directly to worker event queues, avoiding accept lock contention.
*   **Application Strategy**: Under short-lived high-frequency connections, hash collisions may cause worker load imbalance. Enable `SO_INCOMING_CPU` socket options to process packets on the CPU core handling NIC interrupts.

#### Card 4. Hot Restart Socket Transfer via Unix Domain Sockets
*   **Technical Mechanism**: During hot restarts, the new child process boots and establishes communication with the parent process over a Unix Domain Socket. The parent process passes its active listening file descriptors (FD) to the child via `SCM_RIGHTS` ancillary control messages, allowing the child to take over ports with 0 packet drop.
*   **Application Strategy**: Use hot restarts to upgrade binaries or reload static configurations gracefully without dropping active connections.

---

## 📂 M2: Dynamic Configuration & xDS API (Cards 5-9)

#### Card 5. xDS Configuration Dependencies & Validation Sequence
*   **Technical Mechanism**: xDS APIs govern Envoy configuration: LDS (Listener), RDS (Route), CDS (Cluster), and EDS (Endpoint). To prevent routing requests to uninitialized endpoints, Envoy updates dependencies in order: CDS and EDS are populated first, followed by LDS and RDS activation.
*   **Application Strategy**: Control planes must synchronize updates in order (CDS ➜ EDS ➜ LDS ➜ RDS) or use Aggregated Discovery Service (ADS) gRPC streams to guarantee configuration consistency.

#### Card 6. gRPC Subscription Stream & Configuration ACK/NACK
*   **Technical Mechanism**: Worker threads subscribe to configuration updates over gRPC streams. The control plane pushes updates inside a `DiscoveryResponse` with a version tag. Envoy parses the structure; if valid, it applies the configuration and sends an ACK; if invalid, it rejects with a NACK and continues running the active version.
*   **Application Strategy**: Set alerts on NACK rates to capture invalid configuration pushes or schema validation failures from the control plane.

#### Card 7. std::shared_ptr Atomic Pointer Swap & Lockless Configuration Reload
*   **Technical Mechanism**: Envoy workers read configurations in a thread-safe read-only format. When the main thread receives a configuration patch, it compiles the structures and performs an atomic pointer swap (`std::atomic_store`) to redirect worker loops to the new configuration object, recycling old structures when references hit zero.
*   **Application Strategy**: Eliminates read-write lock latency, ensuring configuration updates do not degrade gateway forwarding throughput under high loads.

#### Card 8. Static Configuration Bootstrap & Dynamic Overrides
*   **Technical Mechanism**: On startup, Envoy parses a bootstrap file (YAML/JSON). If dynamic subscriptions are configured, it initializes EAL structures and static listeners first, then connects to the xDS server over gRPC to subscribe to dynamic clusters and routes.
*   **Application Strategy**: Static configs are ideal for local testing; production environments should rely on dynamic control planes to avoid physical restart delays.

#### Card 9. Listener Draining & Connection Draining Timeout
*   **Technical Mechanism**: When a listener is updated or removed via LDS, active connections are not closed immediately. The listener stops accepting new sockets, flags connections as "Draining", and injects a `Connection: close` header into outbound packets, allowing clients to reconnect over a configurable timeout.
*   **Application Strategy**: Tune `drain_timeout` (typically 10s-30s) to match client keepalive policies to allow connections to drain gracefully.

---

## 📂 M3: Filter Chain Architecture & Extensions (Cards 10-13)

#### Card 10. SNI Listener Filter & TLS inspector Parsing
*   **Technical Mechanism**: Listener filters execute immediately after TCP handshakes succeed. The `tls_inspector` filter sniffs the initial bytes of incoming streams, parsing the TLS Client Hello payload in plaintext to extract the Server Name Indication (SNI) and ALPN values, storing them in the connection metadata.
*   **Application Strategy**: Map SNI parameters to separate filter chains to host multiple domain certificates on a single IP and port.

#### Card 11. Network Filter Chain & L4 TLS/TCP Lifecycle
*   **Technical Mechanism**: Network filters manage L4 network connections. By stacking read/write callbacks (`onData`/`onWrite`), filters process TLS termination (`SslSocket`), protocol parsing, and access logs, before passing decrypted payload bytes up to L7 handlers.
*   **Application Strategy**: Create network filters to build custom TCP gateways, TLS decryptors, or protocol inspection filters (e.g. Redis/Mongo sniffers).

#### Card 12. HTTP Filter Pipeline & decodeHeaders/decodeData Callbacks
*   **Technical Mechanism**: L7 processing pipeline. Filters implement callbacks like `decodeHeaders()`, `decodeData()`, and `encodeHeaders()`. Requests pass through decode stages sequentially; responses pass through encode stages. A filter can halt execution by returning `StopIteration` (e.g., Auth failures returning 401).
*   **Application Strategy**: Streamline request modifications, request header rewriting, auth validations, and local routing rules using custom HTTP filters.

#### Card 13. WebAssembly (Wasm) Sandbox Isolation & Async gRPC Client
*   **Technical Mechanism**: WebAssembly (Wasm) filters run in independent sandbox VMs (e.g., V8, Wasmtime). If Wasm code executes invalid instructions or runs out of memory, the sandbox crashes and returns 500, isolating the main Envoy process from crashes. It supports non-blocking async gRPC requests to external auth servers.
*   **Application Strategy**: Wasm is ideal for custom extensions in multi-tenant mesh environments, balancing runtime extensibility with stability.

---

## 📂 M4: Connection Pools & Load Balancing (Cards 14-17)

#### Card 14. Downstream vs Upstream Connection Pool Management
*   **Technical Mechanism**: Downstream client connections use long-lived reuse patterns. For backend servers, Envoy maintains an Upstream connection pool for each cluster and protocol version (H1, H2). New requests map to an active socket, while HTTP/2 requests multiplex over single TCP sockets.
*   **Application Strategy**: Set limits on `max_connections` and `max_requests_per_connection` to prevent socket exhaustion or memory leaks on upstream clusters.

#### Card 15. Round Robin vs Least Request (2-Choice) Load Balancing
*   **Technical Mechanism**: Traditional Round Robin can route requests to congested backends. The Least Request algorithm picks nodes using a `2-Choice` pattern: it selects two healthy upstream nodes at random, compares their active request counters, and routes the request to the node handling fewer active connections.
*   **Application Strategy**: Least Request is recommended for high-concurrency workloads to minimize P99 latency spikes.

#### Card 16. Maglev Consistent Hashing Table & Ring Hash Stability
*   **Technical Mechanism**: Traditional Ring Hashing has O(log N) lookup complexity. Maglev maps nodes to a pre-computed lookup table of size $M$ (typically a prime number like 65537). Lookup is O(1) via hash modulo. When nodes join or leave, table reconstruction overhead is low, minimizing routing shifts.
*   **Application Strategy**: Use Maglev for session affinity based on headers or cookies in large-scale deployments to avoid high lookup costs.

#### Card 17. Locality-Weighted Routing & Local Failover Priority
*   **Technical Mechanism**: Locality routing prioritizes traffic within the same machine room or zone (Locality 0). If the health score of local nodes drops below a threshold (e.g. 70%), Envoy redirects a proportion of requests to neighboring localities (Locality 1), balancing local preference with failover capacity.
*   **Application Strategy**: Deploy locality routing to reduce cross-zone latency and save network egress costs.

---

## 📂 M5: Protocol Translation & Buffers (Cards 18-23)

#### Card 18. HTTP/1.1 to HTTP/2 Multiplex Translation
*   **Technical Mechanism**: Receives downstream HTTP/1.1 connections, parses headers, and instantiates logical HTTP streams. The gateway connection pool translates and multiplexes these streams over active upstream HTTP/2 TCP sockets.
*   **Application Strategy**: Use Envoy as an edge gateway to upgrade legacy client connections to multiplexed HTTP/2 backend services.

#### Card 19. gRPC over HTTP/2 Frame Transcoding
*   **Technical Mechanism**: gRPC runs over HTTP/2 stream channels. The Envoy JSON transcoder parses HTTP/1.1 JSON requests, extracts payload fields, and wraps them in HTTP/2 protobuf frame segments (`HEADER` and `DATA` frames) destined for backend gRPC endpoints.
*   **Application Strategy**: Expose internal gRPC microservices as REST APIs without modifying application code.

#### Card 20. HTTP/3 (QUIC) Transport & Connection Migration
*   **Technical Mechanism**: HTTP/3 uses UDP-based QUIC instead of TCP, merging TLS 1.3 to achieve a 1-RTT handshake. Connections are tracked via a unique `Connection ID` instead of client IP/Port, allowing clients to switch networks (e.g., Wi-Fi to 4G) without dropping sessions.
*   **Application Strategy**: Deploy HTTP/3 for mobile clients and edge nodes to maintain persistent connections during network handoffs.

#### Card 21. Stream Head-of-Line Blocking (HOLB) Elimination in QUIC
*   **Technical Mechanism**: In HTTP/2, a packet drop on the underlying TCP socket stalls all multiplexed streams because TCP treats the payload as a single stream window. QUIC isolates stream windows in UDP; a packet drop on one stream only blocks that stream, leaving other active streams unaffected.
*   **Application Strategy**: Improves latency stability on high-loss channels (e.g., wireless environments).

#### Card 22. Zero-Copy Buffer Management via Slice Chain Linked Lists
*   **Technical Mechanism**: Instead of allocating large contiguous buffers, Envoy's `WatermarkBuffer` stores packet data in linked lists of small memory segments called Slices. Modifying headers or splitting buffers changes only node pointers and offsets, avoiding data copies.
*   **Application Strategy**: Cuts CPU and memory usage when routing large payloads or handling file uploads.

#### Card 23. Non-Blocking Compression Filters (Gzip / Brotli)
*   **Technical Mechanism**: Compression is CPU-intensive. Compression filters execute gzip/brotli algorithms in non-blocking slices, preventing large payloads from stalling the worker core event loop, and can offload processing to dedicated encryption cards.
*   **Application Strategy**: Enable compression filters for text payloads on edge gateways to reduce public egress bandwidth without blocking worker threads.

---

## 📂 M6: Resiliency (Circuit Breaking & Outlier Detection) (Cards 24-28)

#### Card 24. Resource-Counter-based Circuit Breaking
*   **Technical Mechanism**: Envoy circuit breaking triggers immediately based on real-time resource usage counters rather than rolling error rates. Counters track `max_connections`, `max_pending_requests`, `max_requests`, and `max_retries`. If a counter overflows, new requests are rejected locally with a 503.
*   **Application Strategy**: Provides fast O(1) protection against cascading failures, isolating upstream clusters from overload.

#### Card 25. Passive Outlier Detection & Endpoint Ejection
*   **Technical Mechanism**: Outlier detection monitors inline response codes. If an endpoint returns 5 consecutive 5xx errors or connection timeouts, Envoy marks it as unhealthy and ejects it from the load balancer list for a configurable duration.
*   **Application Strategy**: Combines with load balancing to route traffic around failing nodes in seconds without control plane intervention.

#### Card 26. Active Health Checking Probe Loops
*   **Technical Mechanism**: Active health checkers run periodic background timers to send probe requests (e.g., `GET /healthz`) to all endpoints. Unhealthy nodes are removed, and only returned to rotation after passing consecutive successful probes.
*   **Application Strategy**: Use active and passive health checks in tandem to prevent routing requests to starting or failing instances.

#### Card 27. Retry Budgets & Retry Storm Prevention
*   **Technical Mechanism**: To prevent retries from overloading downstream systems, the retry budget limits retry attempts to a percentage of total active traffic (default 20%). If retries exceed this ratio, Envoy drops further retry requests.
*   **Application Strategy**: Essential safeguard in mesh architectures to prevent retry storms from collapsing failing backends.

#### Card 28. Local Token Bucket vs Global Distributed Rate Limiting
*   **Technical Mechanism**: Local rate limiting runs in-memory using token bucket filters with 0 network overhead. Global rate limiting makes asynchronous gRPC requests to an external Redis cluster to validate quota across the grid.
*   **Application Strategy**: Combine local rate limiting on edge gateways (for DDoS defense) with global rate limiting for business-logic API quotas.

---

## 🛠️ Zone T: Envoy Diagnostics & Tuning Utilities (Page 2 Bottom)

### T1: Configuration Tuning Parameters

*   `concurrency`: CLI argument. Defines worker thread count, recommended to bind 1:1 to physical CPU cores.
*   `drain_time_s`: Admin configuration. Graceful draining period for connections during hot restarts (default 15s).
*   `listener.xxxx.filter_chains`: LDS structure. Configures SNI rules and matching filters to prevent processing mismatches.
*   `stats_flush_interval_ms`: Admin setting. Prometheus metric flush interval (default 5000ms).
*   `cluster.xxxx.max_requests_per_connection`: CDS configuration. Max requests allowed on an upstream connection before graceful retirement.
*   `overload_manager`: Global resource monitor. Triggers rate limits or rejects connections when memory or FD limits are breached.

### T2: Admin Server Diagnostic Commands

Envoy's administration interface listens on `http://127.0.0.1:9901` by default:

*   `curl http://127.0.0.1:9901/stats/prometheus`: Fetches metrics including connection counts, active streams, and xDS version states.
*   `curl http://127.0.0.1:9901/config_dump`: Dumps the active LDS, RDS, CDS, and EDS configurations currently applied in memory.
*   `curl http://127.0.0.1:9901/clusters`: Shows active upstream endpoints, health status, and active connection counts.
*   `curl -X POST http://127.0.0.1:9901/logging?level=debug`: Toggles logging levels dynamically to debug network packet traces without restarting.
*   `curl -X POST http://127.0.0.1:9901/drain_listeners`: Triggers graceful listener draining manually prior to node decommissioning.
*   `envoy --mode validate -c /etc/envoy/envoy.yaml`: Validates configuration syntax against static schema specs.
