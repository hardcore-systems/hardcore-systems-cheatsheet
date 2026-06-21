# ROS 2 DDS Real-Time Middleware & Shared Memory Cheatsheet
## J-Ladder Hierarchical Model

### L0 One-Line Essence
ROS 2 packages DDS industrial bus standards into a robotic compute-graph model, leveraging RMW abstraction and zero-copy shared memory to distribute high-throughput sensor telemetry with bounded latencies.

### L1 Four-Sentence Logic
1. **Pluggable Middleware Drivers**: Employs the ROS Middleware Interface (RMW) to bridge rclcpp/rclpy client engines with diverse DDS vendors (FastDDS, CycloneDDS) dynamically.
2. **Deterministic Control Loops**: Pin-locks virtual addresses in RAM via POSIX `mlockall` and assigns `SCHED_FIFO` priorities to preempt thread execution jitter.
3. **Zero-Copy Memory Bypass**: Bypasses network sockets and serialization for heavy image frames by loaning shared memory pages directly via iceoryx allocators.
4. **Coordinated Lifecycle Transitions**: Governs nodes via life-cycle state machines to enforce deterministic configurations, initialization, and safe failure fallbacks.

### L2 Core Data Flow
`rclcpp/rclpy publish` ➜ `Intra-Process Check` ➜ `If same process: pass unique_ptr direct to subscriber` ➜ `If cross-process: DDS SPDP/SEDP discovery match` ➜ `RMW serializes structure to CDR` ➜ `write to iceoryx shm / or RTPS multicast UDP` ➜ `DDS receive queue` ➜ `RMW deserialize` ➜ `Executor triggers Callback Group` ➜ `Subscriber callback`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: Node Lifecycle & Architecture
*   **Theory**: Atomic computation blocks inside the robot graph, managed by executors for event loop schedules.
*   **Details**: Nodes correspond to `rclcpp::Node` handlers, sharing a common container for intra-process efficiency or running on dedicated OS threads.
*   **Trade-off**: Container-sharing exposes all concurrent nodes to single-point failures; a segfault in one component crashes the host process.

### Card 2: DDS Adapter & RMW Abstraction Layer
*   **Theory**: Decouples API client libraries from vendor-specific DDS bindings using standard C APIs.
*   **Details**: Provides the ROS Middleware Interface (RMW) specification. Vendors implement drivers like `rmw_fastrtps_cpp` or `rmw_cyclonedds_cpp` loaded at boot.
*   **Trade-off**: Abstracting DDS hides advanced vendor-specific features, requiring custom XML injections for edge-case configuration overrides.

### Card 3: DDS Dynamic Peer Discovery
*   **Theory**: Discards central master nodes (ROS 1 Master) by using peer-to-peer multicast network discovery.
*   **Details**: Operates SPDP (Simple Participant Discovery Protocol) and SEDP (Simple Endpoint Discovery Protocol) to exchange participant and endpoint schemas.
*   **Trade-off**: Large subnets generate discovery multicast storms, requiring strict uncast peer lists to minimize discovery traffic.

### Card 4: DDS Quality of Service (QoS) Policies
*   **Theory**: Configures network reliability and buffering behaviors for robust data transmission.
*   **Details**: Relies on three base configurations: Reliability (Best Effort / Reliable), Durability (Transient Local / Volatile), and History (Keep Last / Keep All).
*   **Trade-off**: QoS configurations must be compatible to establish connections. Under Offered vs Requested constraints, mismatched parameters silently drop packets.

### Card 5: Intra-Process Communication (IPC)
*   **Theory**: Passes heap pointers directly inside a single process container, bypassing serialization and socket copies.
*   **Details**: Activates `use_intra_process_comm` configuration, passing a `std::unique_ptr` across queues without converting structures to CDR format.
*   **Trade-off**: Transferring unique pointers transfers ownership. For multiple subscribers, deep copies are forced unless read-only shared pointers are used.

### Card 6: Zero-Copy Shared Memory (iceoryx)
*   **Theory**: Offloads inter-process socket serialization via a lock-free shared memory ring buffer.
*   **Details**: Interlinks RMW with iceoryx, loaning shared memory regions via `borrow_loaned_message()` to write payloads directly without data copying.
*   **Trade-off**: Requires fixed-size structures. Variable-length arrays or strings force fallback to socket copying.

### Card 7: Topic Publishing & CDR Serialization
*   **Theory**: Asymmetric, unidirectional data streams serialized using Common Data Representation (CDR) standards.
*   **Details**: Compiles `.msg` schemas into C++ structures, utilizing CDR's binary alignment rules to stream values over sockets.
*   **Trade-off**: CDR alignment rules generate padding bytes, which consume extra network bandwidth for micro-sized high-frequency payloads.

### Card 8: Asynchronous Services
*   **Theory**: Bounded request-response transaction model mapped over dedicated communication paths.
*   **Details**: Implements requests and responses using matched topics, where executors map transaction IDs to client futures on response arrivals.
*   **Trade-off**: Synchronous calls block executor loops. Invoking a nested synchronous service in a single-threaded executor causes permanent deadlocks.

### Card 9: Action Task Execution Framework
*   **Theory**: Manages long-running tasks via feedback loops, goal cancellations, and asynchronous result tracking.
*   **Details**: Incorporates goal/cancel/result services with feedback/status topics to monitor remote algorithms safely.
*   **Trade-off**: State machines are complex and prone to synchronization issues on packet losses, requiring client-side timeouts.

### Card 10: Executor Scheduling Engine
*   **Theory**: Schedules and executes incoming callback events from the DDS graph.
*   **Details**: Supports SingleThreaded and MultiThreaded Executors. Controls callback concurrency via callback groups (MutuallyExclusive, Reentrant).
*   **Trade-off**: Multithreading risks race conditions on shared node data; non-atomic variables require mutex protections in reentrant groups.

### Card 11: Dynamic Parameter System
*   **Theory**: Provides runtime configuration modifications and event callbacks without rebuilding nodes.
*   **Details**: Modifies parameters via setter callbacks, broadcasting parameter changes to the global `/parameter_events` topic.
*   **Trade-off**: Heavy parameter updates across large fleets consume network bandwith; changes require debouncing filters.

### Card 12: Lifecycle Node State Machine
*   **Theory**: Enforces deterministic initialization, configuration, activation, and cleanup sequences for nodes.
*   **Details**: Restricts publishing/subscribing until the node is in the Active state. Defines transition callbacks like `on_configure` and `on_activate`.
*   **Trade-off**: Transition callbacks block the main initialization thread. Long setup operations must run asynchronously to avoid startup lockups.

### Card 13: Launch Engine
*   **Theory**: Orchestrates distributed node deployments and monitors lifecycle status dynamically.
*   **Details**: Uses Python launch scripts to parse parameters and register event handlers for node crashes.
*   **Trade-off**: Python orchestration adds startup latency; real-time hard systems should favor static C++ containers.

### Card 14: MCAP/Bag Recording
*   **Theory**: Captures topic streams to disk for offline analysis and playback.
*   **Details**: Replaces SQLite with MCAP, an append-only file format designed for robot sensors, supporting embedded schemas.
*   **Trade-off**: High-bandwidth camera feeds risk disk I/O bottlenecks; setups require separate I/O queues and compression.

### Card 18: Real-Time Guarantee
*   **Theory**: Bounds execution latency by locking process memory and adjusting scheduler parameters.
*   **Details**: Invokes `mlockall` to pin addresses in RAM, preventing Linux swap delays. Upgrades executor threads to `SCHED_FIFO`.
*   **Trade-off**: Requires root access and real-time RT-PREEMPT kernels to avoid scheduling priority inversions.

### Card 25: QoS Compatibility Rules
*   **Theory**: Establishes connections using compatibility checks (offered vs requested rules).
*   **Details**: Requires the Offered QoS level from the publisher to equal or exceed the Requested QoS from the subscriber.
*   **Trade-off**: Mismatches silently block communication without throwing compile errors; diagnostics must be checked via RMW log alerts.

---

## 🔬 Zone T1: ROS 2 Core APIs
*   `rclcpp::Node`: Core C++ class managing publishers, subscriptions, and parameter events.
*   `rcl::rcl`: C interface wrapping wait sets, execution contexts, and clocks.
*   `rmw::rmw`: Middleware abstraction level managing RTPS node discovery and endpoint mappings.
*   `rcutils::rcutils`: Utility crate containing allocator helpers and logger formatters.
*   `tf2::BufferCore`: Coordinates transform manager resolving relative frames over time.
*   `rclcpp::executors::MultiThreadedExecutor`: Concurrently dispatches events across multiple worker threads.

## 🔬 Zone T2: Protocol & Verification Errors
*   `dds_qos_incompatible_match`: Mismatched publisher and subscriber QoS configurations preventing topics from communicating.
*   `shared_memory_segment_overflow`: Shared memory ring exhaustion in iceoryx causing loan message failures.
*   `executor_deadlock_callback_stall`: Single-threaded executor deadlock due to nested synchronous service calls.
*   `realtime_priority_inversion`: Jitter caused by high-priority real-time threads blocking on locks held by low-priority tasks.
*   `tf2_extrapolation_out_of_range`: Transform timestamp query falls outside the tf2 buffer window.
*   `lifecycle_invalid_state_transition`: State machine transition request rejected due to invalid active state.
