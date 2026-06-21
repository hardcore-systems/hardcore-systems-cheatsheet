# Seastar Shard-per-Core Zero-Lock Network Framework Cheatsheet

## M1: Shard-per-core Zero-Lock Scheduler

### Card 1: Shard-per-Core Shared-Nothing Architecture
- **Mechanism**: Seastar abandons dynamic multi-thread scheduling in favor of Shard-per-Core (one OS thread pinned per physical CPU core). Each core runs its own Reactor event loop and owns all associated memory resources.
- **Lock-Free**: Core resources are isolated, establishing a Shared-Nothing model that eliminates Mutex lock contention and cache bouncing, allowing linear performance scaling.
- **Trade-off**: Cross-core data exchanges require message passing via explicit queues, introducing minor memory and scheduling overhead.

### Card 2: CPU Affinity Pinning & Core Isolation
- **Affinity**: Working threads are pinned to physical cores using system calls like `sched_setaffinity` at startup.
- **No Context Swapping**: Bypasses context switches and L1/L2 cache pollution by preventing thread migrations.
- **Warning**: Configure the host OS boot parameters with `isolcpus` to isolate target cores from system task allocation.

### Card 3: Lock-Free Physical Memory Allocator
- **Isolation**: Standard allocators suffer from global heap lock contention. Seastar divides the host physical memory among cores at initialization, assigning each core its own exclusive Page arena.
- **Speed**: Memory allocations run in local arenas with $O(1)$ lock-free complexity.
- **Deallocation**: Cross-shard deallocations route pointers back to their allocating core via lock-free ring buffers for asynchronous cleanup.

### Card 4: Cooperative Fibers Scheduling
- **Cooperation**: Seastar threads schedule operations cooperatively. Each core executes thousands of lightweight fibers.
- **Non-Preemptive**: Fibers cannot be preempted. They yield control to the Reactor voluntarily upon encountering async limits (e.g. I/O wait).
- **Warning**: Synchronous or blocking loops block the Reactor loop, stopping network heartbeat ticks and crashing connections.

### Card 5: SPSC Cross-Core Ring Buffers
- **Messaging**: Core A communicates with Core B by sending tasks wrapped as messages to B's single-producer single-consumer (SPSC) ring buffer.
- **Atomic Operations**: Read and write pointers are managed atomically on separate cache lines to avoid CPU cache line conflicts.
- **Limitation**: Ring buffers have fixed sizes. Full buffers trigger backpressure on sender cores, throttling incoming traffic.

---

## M2: Asynchronous Non-Blocking Model

### Card 6: Future & Promise Mechanics
- **State**:
  - **Future**: A container representing a yet-to-be-completed asynchronous operation.
  - **Promise**: The producer of the async computation, responsible for fulfilling or breaking the Future.
- **Lock-Free**: Future structures contain zero mutexes and are optimized for single-threaded execution inside Reactor threads.
- **Warning**: Futures must only be accessed by the shard that owns them; cross-shard calls trigger data races.

### Card 7: Chained Continuation Chain
- **Chaining**: Continuations are chained using the `.then()` operator (e.g. `ReadPacket().then(Parse).then(WriteBack)`).
- **Optimization**: Employs move semantics to forward results through the future chain without heap copies.
- **Debugging**: Async chains break execution call stacks. Standard stack traces (Backtraces) show only Reactor loops.

### Card 8: C++20 Coroutines & co_await Integration
- **Coroutines**: Native support for C++20 coroutines, enabling developers to write async code using `co_await seastar_future`.
- **Compilation**: The compiler translates code past `co_await` points into continuation closures and registers them on futures.
- **Overhead**: Coroutines allocate state frames on the heap, which can add microsecond overheads in latency-critical loops.

### Card 9: Microsecond Task Scheduler
- **Queues**: Reactor loops maintain multiple FIFO Task Queues with distinct scheduling priorities.
- **Execution**: The Reactor executes ready tasks in a round-robin loop, allocating CPU weights to isolate control and data plane workloads.
- **Benefit**: Fiber switches run user-space pointer updates in nanoseconds, bypassing OS thread scheduling overheads.

### Card 10: Exception Propagation & Chained Cleanup
- **Exceptions**: Future chains capture producer errors via `.handle_exception()` or native `try/catch` blocks inside coroutines.
- **Cleanups**: Chained `.finally()` operators ensure that allocated resources (e.g. descriptors, buffers) are freed regardless of success.
- **Limitation**: Exceptions must be propagated down future chains; a broken chain leaks resources and leaves errors unhandled.

---

## M3: Network & I/O Engine

### Card 11: Linux io_uring & Epoll Engines
- **Kernel I/O**: Seastar utilizes `io_uring` on modern Linux kernels, communicating via shared Submission and Completion ring buffers.
- **No Syscalls**: Submitting tasks to io_uring ring buffers bypasses system call context switches.
- **Fallback**: Automatically falls back to Epoll-based network loops on older kernels.

### Card 12: Zero-Copy Fragmented Packets
- **Packets**: Sled's `packet` structures store network payloads using fragmented memory layouts.
- **Reference Count**: Sharing or framing packets allocates reference-counted views without copying raw bytes.
- **Trade-off**: Fragmented memory structures require scatter-gather chains during network writes, adding minor driver execution overhead.

### Card 13: Direct I/O (O_DIRECT) DMA Storage
- **O_DIRECT**: Bypasses the OS page cache entirely by enforcing Direct I/O (`O_DIRECT`).
- **DMA Access**: Storage controllers read and write directly to Seastar user-space buffers via DMA, preventing memory pressure.
- **Trade-off**: Requires applications to build and maintain custom block caches in user space.

### Card 14: Disk I/O Fair Scheduler
- **Fairness**: Multiple cores sharing single storage controllers can overload queues and starve disk access.
- **Fair Queue**: Coordinates core disk access using a token bucket scheduler to distribute IOPS and bandwidth fairly across cores.
- **Tuning**: Run the `seastar-io-setup` utility to profile storage hardware limits before starting the engine.

### Card 15: DPDK User-Space Networking
- **Kernel Bypass**: Integrates DPDK to acquire exclusive control of network interfaces directly in user space.
- **Protocol Stack**: Bypasses the Linux TCP/IP network stack entirely, parsing ARP, IP, UDP, and TCP packets in user space for microsecond network response times.
- **Limitation**: DPDK intercepts the network interface, making standard OS network diagnostic tools (e.g. `iptables`, `ifconfig`) unusable.

---

## M4: Concurrency & Hardware Topology

### Card 16: Atomic Lock-Free Shared Structs
- **Atomics**: Global variables (e.g. cluster configs) utilize atomic primitives (`std::atomic`) and memory barriers to ensure cross-core visibility.
- **Cache Storms**: Restrict atomic writes to rare operations; frequent atomic writes trigger cache coherency bus storms that slow down Reactor loops.

### Card 17: Read-Copy-Update (RCU) Concurrency
- **Lock-Free Reads**: High-frequency read structures use RCU-like pointer swaps. Readers access read-only snapshots without locks.
- **Writes**: Writers clone target data, apply updates, and atomically swap pointers, deferring garbage collection using Epoch sweeps.
- **Memory**: Trades minor heap storage footprints for lock-free read execution.

### Card 18: SMP Coordinate Barriers
- **Coordination**: System startup and shutdown phases use asynchronous synchronization barriers.
- **Asynchronous Barriers**: The `smp::invoke_on_all` interface broadcasts functions across shards, returning a future array for synchronization.
- **Warning**: Do not use blocking synchronization barriers (`std::barrier`, mutexes) inside active Reactor loops.

### Card 19: Cache Line Padding & False Sharing
- **False Sharing**: Cores writing to independent variables located on the same 64-byte cache line trigger cache line invalidations.
- **Padding**: Force cache line alignment using `alignas(64)` or padding bytes in cross-core structures.
- **Trade-off**: Sacrifices a few bytes of memory spacing to prevent thread execution contention.

### Card 20: NUMA-Aware Memory Allocation
- **Locality**: Multi-socket systems read local socket memory faster than remote socket arrays.
- **NUMA Allocations**: Seastar's memory allocator tracks host CPU topologies, ensuring that memory blocks belong to the local NUMA node of the executing thread.
- **Deployment**: Bind memory paths using the `--numa-node` startup flag.

---

## M5: Distributed Sidecar & Services

### Card 21: Distributed Sharded Services
- **Sharded Template**: The `sharded<Service>` class template instantiates a local Service copy on each CPU core.
- **Local Routing**: Directs client requests to the executing core's local service copy to prevent lock contention.
- **Aggregation**: Combines states across shards asynchronously using `sharded::map_reduce`.

### Card 22: Connection Rate Limiters
- **Overload Guard**: The Reactor manages connection pools via local Connection Managers.
- **Throttling**: Reaching connection limits or CPU load spikes triggers connection drops or redirects.
- **Trade-off**: Protects active transactions at the cost of rejecting new incoming requests during traffic spikes.

### Card 23: Backpressure Flow Control
- **Flow Control**: Unbalanced processing rates across cross-core queues or network pipelines trigger memory buffering.
- **Backpressure**: Exceeding buffer watermarks throttles upstream socket reads, forcing clients to drop transmission rates.
- **Benefit**: Prevents infinite buffer allocation and secures engines against memory exhaustion.

### Card 24: Zero-Copy JSON & HTTP Parsers
- **Speed**: Built-in HTTP parser and JSON compiler optimize API routing.
- **Zero-Copy**: The parser splits URL strings and payloads directly in packet fragment buffers without copying.
- **Usage**: Handles high-throughput cloud-native microservice routing without memory copy bottlenecks.

---

## M6: Tuning & Diagnostics

### Card 25: Stalled Core Watchdog
- **Watchdog**: Reactor loops run a microsecond watchdog timer.
- **Alerts**: Executions exceeding 500 microseconds log a Stalled Thread warning with debug backtraces.
- **Usage**: Identifies blocking system calls (e.g. `std::this_thread::sleep_for`) or CPU-heavy loops in async code.

### Card 26: Asynchronous Stack Trace Reconstruction
- **Reconstruction**: Reactor loops break traditional GDB stack frames, showing only `reactor::run()`.
- **Debugging**: Seastar tracks active async span links in debug builds to reconstruct logical stack traces across future chains.
- **Overhead**: Context tracking adds scheduling overhead; restrict this utility to diagnostic environments.

### Card 27: Telemetry Profiling Counters
- **Counters**: Reactor loops expose Prometheus metrics on processed tasks, pending futures, cross-shard requests, and io_uring queues.
- **Optimization**: Analyze metrics to locate load skew imbalances or I/O bottleneck points.

### Card 28: Compiler Optimizations Flags
- **Compiler Flags**: C++ compilation flags significantly affect binary throughput.
- **Optimization Flags**:
  - `-O3`: Enable compiler optimizations.
  - `-flto`: Optimize inline code paths across compilation units.
  - `-march=native`: Generate SIMD and cache layout instructions optimized for the host architecture.
- **Warning**: Link-Time Optimization (LTO) increases compile times and compiler RAM consumption.
