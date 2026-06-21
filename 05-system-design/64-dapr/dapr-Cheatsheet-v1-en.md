# Dapr Distributed Application Runtime Cheatsheet

## M1: Sidecar Architecture & API Interception

### Card 1: App-to-Sidecar Bridge
- **Communication Path**: Applications communicate with the Dapr Sidecar via localhost. Default ports: HTTP 3500, gRPC 50001.
- **Benefit**: Decouples application logic from backend infrastructure; no heavy databases or broker SDKs needed.
- **Trade-off**: Introduces a loopback network hop, adding microsecond-level latency.

### Card 2: Unified API Abstraction
- **Abstraction**: Simplifies microservice operations into uniform APIs (e.g. `/v1.0/state/store/key`, `/v1.0/publish/pubsub/topic`).
- **Seamless Swap**: Swap Redis with MongoDB, or Kafka with RabbitMQ, without editing application code.
- **Limitation**: Restricts access to advanced engine-specific features (e.g. SQL joins) due to the lowest-common-denominator approach.

### Card 3: Port Lifecycle Hook
- **NetNS Sharing**: Dapr sidecars run in the same Kubernetes Pod, sharing the network namespace with the main application.
- **Ready Probe**: Placement and Sentry block requests until application ports are fully initialized, preventing startup packet drops.
- **Complexity**: Incorrect configurations can cause Pod startup lockups and failing readiness probes.

### Card 4: Sidecar Process Proxy
- **Bilateral Proxy**: Proxies outbound client calls and delivers subscribed events/input triggers back to application routers.
- **Fault Recovery**: If the application crashes, the Sidecar remains active and buffers events, ensuring zero-loss reconnection.
- **Memory Cost**: Each Sidecar consumes 30-50MB RAM, which aggregates to a significant footprint in large microservice groups.

### Card 5: Seamless App Integration
- **Auto-Injection**: Automatically injects Sidecars in Kubernetes deployment manifests using annotations (e.g. `dapr.io/enabled: "true"`).
- **Polyglot**: Seamlessly integrates legacy systems (COBOL/C++) or modern languages (Go/Python) using localhost HTTP/gRPC.
- **Debugging**: Transparent traffic routing requires network inspection tools to trace localhost redirections.

---

## M2: Pluggable State Management & Distributed Locks

### Card 6: Pluggable State Stores
- **Dynamic Loading**: Loads Redis, PostgreSQL, CosmosDB, or DynamoDB dynamically via component YAML metadata.
- **Portability**: Uses localhost Redis in development and switches to CosmosDB in production without codebase changes.
- **Limitation**: Custom functions are restricted by Dapr's `StateStore` interface contract.

### Card 7: ETag Optimistic Locking
- **ETags**: State reads return a unique ETag version identifier to trace current database mutations.
- **Conflict Avoidance**: Concurrent state updates must include the read ETag; mismatched writes trigger a `409 Conflict` error.
- **Trade-off**: High-frequency writes cause execution retries, requiring queues or Last-Write-Wins configurations.

### Card 8: State API Transactions
- **Transactions**: Supports bulk transactional reads/writes (Upsert, Delete) via `/v1.0/state/store/transaction`.
- **Atomicity**: Translates transactional batches into native database transactions if supported by the backend driver.
- **Limitation**: Two-phase commits (2PC) across different state store backends are not supported.

### Card 9: Distributed Lock API
- **Distributed Locks**: Exposes `/v1.0/lock/store` to lease key rents with configurable time-to-live (TTL) limits.
- **Component Agnostic**: Standardizes locks across Redlock (Redis) or Zookeeper, shielding apps from custom locking logic.
- **Risk**: Brain-split errors in the underlying lock store can lead to concurrent lease approvals.

### Card 10: State TTL Expiry Control
- **TTL Control**: Employs `ttlInSeconds` metadata to assign temporary expiry limits to cached values.
- **Memory Save**: Automatically purges stale session data to reclaim storage space.
- **Limitation**: SQL databases lack native TTLs and require Dapr-simulated scans, increasing IOPS overhead.

---

## M3: Publish & Subscribe & Event-Driven

### Card 11: CloudEvents Protocol
- **Specification**: Wraps pub/sub payloads inside W3C CloudEvents schemas containing `id`, `source`, `type`, and `specversion`.
- **Interoperability**: Standardized metadata headers preserve routing information and tracing contexts across cloud environments.
- **CPU Cost**: JSON wrapper serialization/deserialization adds minor CPU overhead to high-throughput message flows.

### Card 12: Pluggable Message Brokers
- **Abstract Pub/Sub**: Routes messages to Kafka or RabbitMQ when applications POST to `/v1.0/publish/pubsub/topic`.
- **Consumer Isolation**: Handles Kafka consumer group setups in the Sidecar, decoupling apps from message broker APIs.
- **Implication**: At-least-once delivery differences require application-level message deduplication.

### Card 13: Subscription Registry
- **Registry**: Supports declarative XML components or dynamic subscriptions via `/dapr/subscribe` scanners.
- **Push Delivery**: Sidecars push incoming topic events to applications via HTTP POST calls (e.g. `/orders`).
- **Limitation**: Pusher execution timeouts (60s default) trigger redeliveries of unacknowledged events.

### Card 14: Retry Backoff Delivery
- **Resiliency**: Retries event delivery if applications return HTTP `500` or experience execution timeouts.
- **Backoff**: Employs exponential backoff rates to throttle retries, preventing cascade failures.
- **Memory Cost**: Excessive retries buffer events in Sidecar memory, requiring buffer size thresholds.

### Card 15: Dead Letter Queue (DLQ)
- **DLQ**: Isolates failed messages in a dedicated Dead Letter Topic after maximum retry failures.
- **Diagnostics**: Prevents poison messages from blocking queues and preserves data for offline debugging.
- **Prerequisite**: Requires defining dead letter topics in component YAML files; otherwise, unconsumed events are dropped.

---

## M4: Actor Model & Distributed State

### Card 16: Virtual Actor Lifecycle
- **Virtual Actors**: Activates stateful Actors dynamically in memory when called by `ActorType` and `ActorID`.
- **Deactivation**: Garbages idle Actors (1-hour default) and saves states in databases, freeing up RAM.
- **Architecture**: Bypasses manual thread instantiation and locking complexity.

### Card 17: Actor Placement Service
- **Placement**: Employs consistent hashing algorithms to register active Actor locations across cluster nodes.
- **Auto-Rebalance**: Rebalances Actors to healthy Dapr nodes if host node heartbeats are lost.
- **Cost**: Broadcasting hash tables during scaling can cause brief routing interruptions.

### Card 18: Actor Consistency State
- **Turn-based**: Restricts execution to a single thread, queueing incoming requests to prevent race conditions.
- **Auto-State**: Loads state before execution and commits changes on return, avoiding manual locks.
- **Bottleneck**: Long-running Actor methods delay queued requests, causing timeout errors.

### Card 19: Actor Timers & Reminders
- **Timer**: Runs callbacks during active Actor lifetimes; deleted when the Actor deactivates.
- **Reminder**: Persisted in state stores; wakes up inactive Actors to execute callbacks when triggered.
- **Storage Cost**: High-frequency reminders increase read/write IOPS on state databases.

### Card 20: Reentrancy & Deadlock Prevention
- **Reentrancy**: Permits nested reentrant calls (e.g. A ➜ B ➜ A) using a shared `Dapr-Reentrancy-Id` header.
- **Deadlock Avoidance**: Bypasses turn-based locking for related calls, preventing nested system locks.
- **Limit**: Requires configuring maximum reentrancy limits to prevent thread stack overflows.

---

## M5: Declarative Bindings & Integration

### Card 21: Input Bindings
- **Listeners**: Attaches to databases or message brokers, capturing events without custom code integration.
- **Pushes**: Delivers events to applications via local HTTP POST calls, simplifying event-driven code.
- **Bypass**: Hides connection protocols, requiring only local HTTP controllers to consume data.

### Card 22: Output Bindings
- **Triggers**: Invokes external services (SendGrid, AWS S3, Azure Blob) via local POST calls to `/v1.0/bindings/target`.
- **Abstract Triggers**: Standardizes integrations, allowing swaps (e.g. AWS S3 with Azure Blob) without code changes.
- **Trade-off**: Generic API wrappers omit advanced features of custom service SDKs.

### Card 23: Declarative Components (YAML)
- **YAML configs**: Configures bindings, pub/subs, and state stores inside declarative YAML files.
- **Security**: Resolves database passwords securely using references to external Secret Stores.
- **GitOps**: High numbers of components require GitOps control to automate YAML rollouts.

### Card 24: Bidirectional Data Stream
- **Streams**: Supports persistent bidirectional data channels (WebSockets, gRPC stream) via local sidecars.
- **Efficiency**: Lowers message transit latency by using gRPC protocols between app and Sidecar.
- **Complexity**: Requires application-level handlers to manage connection breaks and packet order.

---

## M6: Security & Observability

### Card 25: mTLS Mutual Auth
- **mTLS**: Encrypts traffic between Sidecar nodes using mutual TLS protocols.
- **Certificate Rotation**: Leverages the Sentry service to issue and rotate X.509 certificates automatically.
- **Overhead**: Encryption handshakes add minor CPU overhead to inter-service calls.

### Card 26: SPIFFE Identity Management
- **Identities**: Formats identities via the SPIFFE specification: `spiffe://domain/ns/name/sa/acc/app/id`.
- **Security Control**: Enforces cluster access control policies (ACLs) based on SPIFFE identity attributes.
- **Setup**: Requires configuring Kubernetes service accounts to map to app identities.

### Card 27: Circuit Breakers & Retry
- **Breakers**: Implements circuit breakers to trip connections if target error thresholds are exceeded.
- **Self-Healing**: Transitions to half-open states to check service health before recovery.
- **Benefit**: Provides out-of-code resiliency configurations, managed in external YAML files.

### Card 28: OpenTelemetry Tracking
- **W3C Contexts**: Implements W3C Trace Context headers (`traceparent`, `tracestate`) to propagate trace IDs.
- **Observability**: Automatically exports span logs across state, pub/sub, and bindings to OTel backends.
- **Benefit**: Visualizes cross-service request traces without code modification.
