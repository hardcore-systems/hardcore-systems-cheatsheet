# h2o / quic-http3 High-Density Core Card System (English Edition)

## L0 ~ L2 Knowledge Ladder

*   **L0 One-Sentence Essence**: H2O is a highly optimized HTTP/1/2/3 web server that eliminates locking overhead through a custom lock-free `evloop` event loop, achieves zero-HOLB multiplexing via an O(1) HTTP/2 scheduler dependency tree, and leverages a tightly integrated `quicly` transport layer and `picotls` crypto stack to deliver high-throughput, low-latency stream processing with minimal memory copies.
*   **L1 Four-Sentence Logic**:
    1.  **Custom Event Loop & Timer Wheel**: Eschewing bloated generic libraries, it builds a lightweight `h2o_evloop_t` engine with a tailored `h2o_timerwheel_t`, ensuring O(1) timer additions and deletions under massive concurrency.
    2.  **Tightly-Coupled QUIC/TLS Stack**: It implements HTTP/3 natively using `quicly` as the QUIC engine and `picotls` for TLS 1.3 handshakes, utilizing a custom AES-GCM fusion engine to achieve high encrypted throughput.
    3.  **Zero-HOLB HTTP/2 Prioritization**: It structures strict stream dependency trees where concurrent data streams are sliced into 16KB chunks and multiplexed via Weighted Deficit Round Robin (WDRR) scheduling to prevent head-of-line blocking.
    4.  **Zero-Copy I/O & UDP GSO Offloading**: By utilizing non-owning `h2o_iovec_t` slices throughout the proxy path and UDP Generic Segmentation Offload (GSO), it aggregates multiple UDP packets into single system calls, minimizing context switches.
*   **L2 Core Data Flow Topology**:
    *   `Downstream Packet (UDP / TCP)` ➜ `Network Socket (Worker Thread)` ➜ `h2o_evloop_t` ➜ `quicly_conn_t (Decrypted by picotls)` ➜ `quicly_stream_t` ➜ `h2o_req_t / Core Handler` ➜ `h2o_http2_scheduler / h2o_http3_conn_t` ➜ `16KB Chunking / QPACK` ➜ `h2o_socket_write` ➜ `UDP GSO / sendmmsg` ➜ `Physical Network`.

---

## ⚔️ H2O & QUIC Multi-streaming Trade-off Matrix (Page 1 Bottom)

| Design Dimension | Naive Intuition (⚠) | Physical Reality (★) | Architectural Solution & Evolution |
| :--- | :--- | :--- | :--- |
| **High-Concurrency Timers** | Use generic Red-Black Trees or Min-Heaps to manage timeouts for each TCP/UDP connection. | Tree/heap updates under millions of concurrent connections consume massive CPU cycles, yielding $O(\log N)$ latency. | **Custom Timer Wheel (Timer Wheel)**: Uses a cascaded bucket structure to achieve constant $O(1)$ additions and deletions of timeouts sharing common values. |
| **Event Loop State Updates** | Immediately trigger `epoll_ctl` syscalls when a socket state changes to maintain real-time accuracy. | Updating the kernel state for every packet send/receive results in high context switch overhead and degrades CPU performance. | **Deferred Statechanged List (`_statechanged`)**: Buffers state changes in user-space lists and flushes them in a single batch at the start of each event loop tick. |
| **HTTP/2 Prioritization** | Sequentially read stream buffers and write them directly into the socket send buffer based on client request time. | Large file streams easily saturate the socket buffer, completely stalling small, critical assets (causing application-layer HOLB). | **16KB Chunking + WDRR Tree**: Limits data writes to 16KB chunks and recalculates Weighted Deficit Round Robin (WDRR) tree allocation per chunk. |
| **HTTP/3 UDP Packet I/O** | Iteratively call `sendto()` system calls for each generated HTTP/3 packet to transmit down the wire. | Emitting individual UDP packets (dozens to thousands of bytes) triggers excessive syscall overhead and causes CPU exhaustion. | **UDP GSO (Segmentation Offload) + sendmmsg**: Groups up to dozens of packets into a single `sendmmsg` payload and lets the NIC handle MTU fragmentation. |
| **Weak-Network Multiplexing** | Multiplex hundreds of logical HTTP streams over a single physical TCP connection to reduce handshakes. | A single packet drop triggers TCP sliding window retransmission, freezing all unrelated streams on that link (Head-of-Line Blocking). | **QUIC Stream Flow Windows**: Operates over UDP where each stream has its own flow control and packet recovery, isolating packet loss effects. |

---

## 📂 M1: 自研轻量级事件循环 (h2o_evloop) (Cards 1-4)

#### Card 1. h2o_evloop_t Structure & Multi-Backend Abstraction
*   **Mechanism**: The custom `h2o_evloop_t` structure avoids third-party networking library dependencies. It features a lightweight multi-backend abstraction layer that dynamically binds to native kernel APIs (`epoll` on Linux, `kqueue` on BSD/macOS, `select` on Windows) at compile time.
*   **Strategy**: Automatically leverage native platform multiplexers to maximize I/O throughput and avoid translation wrappers.

#### Card 2. Deferred Statechanged List `_statechanged` & Merged Updates
*   **Mechanism**: Socket read/write state changes do not trigger immediate `epoll_ctl` syscalls. Instead, affected `h2o_socket_t` instances are linked to the `_statechanged` queue. At the beginning of the next loop iteration, these changes are batch-processed, minimizing user-to-kernel context switching.
*   **Strategy**: Highly effective for proxy environments with high packet rates, significantly reducing the host CPU overhead.

#### Card 3. h2o_timerwheel_t Cascaded Timer Wheel Operations
*   **Mechanism**: The `h2o_timerwheel_t` manages timers using cascaded buckets aligned with common keep-alive durations. Timeout additions and deletions occur in constant $O(1)$ time by placing timers in slots relative to expiration offsets, avoiding red-black tree search degradations.
*   **Strategy**: Tune keep-alive configurations knowing that the timer wheel efficiently reclaims idle connections with negligible CPU impact.

#### Card 4. Multi-Worker Threading & SO_REUSEPORT Socket Load Balancing
*   **Mechanism**: H2O runs a multi-worker model where each worker thread runs an independent `h2o_evloop_t`. Rather than sharing a single socket or using listener locks, workers bind to the same port using `SO_REUSEPORT`, letting the kernel distribute incoming connections lock-free via hashing.
*   **Strategy**: Spawn worker threads matching the physical CPU core count (`num-threads`) to achieve symmetric, independent core scaling.

---

## 📂 M2: QUIC 传输层与 quicly 库集成 (Cards 5-9)

#### Card 5. quicly_conn_t & quicly_stream_t Dual State Machine Control
*   **Mechanism**: H2O delegates QUIC handling to the `quicly` library. The `quicly_conn_t` structure maintains connection-level transport contexts, congestion control state, and encryption handshakes, while individual `quicly_stream_t` structures coordinate data frames and stream-level retransmissions.
*   **Strategy**: Treat connection-level and stream-level states as distinct metrics to properly diagnose packet loss vs congestion behaviors.

#### Card 6. picotls (TLS 1.3) Integration & 0-RTT Session Resumption
*   **Mechanism**: `quicly` relies on the lightweight `picotls` library. TLS 1.3 handshakes are parsed directly in memory to generate key materials. The 0-RTT session token is decoded, allowing clients to send encrypted application data in their first packet, eliminating handshake round-trips.
*   **Strategy**: Enable session ticket caching on clients to trigger 0-RTT resumption, drastically reducing initial page load delays.

#### Card 7. First Byte Routing & Connection ID Packet Dispatch
*   **Mechanism**: In `h2o_http3_handle_packet`, H2O inspects the first byte of incoming UDP packets to categorize them. Regular data packets yield their Destination Connection ID (DCID), which is searched in a lock-free hash table to locate the `quicly_conn_t` context, avoiding payload parsing.
*   **Strategy**: Keep routing operations minimal on the packet receive path to sustain high UDP packet-per-second thresholds.

#### Card 8. Connection Migration & Dual-Path Validation
*   **Mechanism**: If a client moves from Wi-Fi to cellular, modifying its IP/Port, the server keeps the connection alive using the unique Connection ID. To prevent spoofing, the server issues a random 64-bit `PATH_CHALLENGE` frame to the new destination and validates it via the client's `PATH_RESPONSE` before migrating.
*   **Strategy**: Implement path validation to secure connection migration against amplification and reflection attacks.

#### Card 9. AES-GCM Fusion Speedup Engine & Zero-Copy Packet Encryption
*   **Mechanism**: QUIC encrypts both the header and payload of every UDP packet. The integrated `fusion` engine in `picotls` utilizes AVX2/AVX512 registers to merge AES-NI and PCLMULQDQ instructions, encrypting memory slices and performing data copies in a single pass.
*   **Strategy**: Build H2O with appropriate SIMD compiler flags to halve the encryption overhead on modern server CPUs.

---

## 📂 M3: HTTP/3 协议栈与 QPACK 头部压缩机理 (Cards 10-13)

#### Card 10. h2o_http3_conn_t Context & Control Stream Orchestration
*   **Mechanism**: The `h2o_http3_conn_t` context acts as the L7 translation layer above `quicly`. It establishes unidirectional Control Streams to negotiate settings (e.g., QPACK table capacity, concurrent stream limits) and manages connection-level HTTP/3 frame parsing.
*   **Strategy**: Align connection settings to ensure smooth control stream frame exchanges and prevent sudden connection closes.

#### Card 11. QPACK Static & Dynamic Table Compression Engine
*   **Mechanism**: QPACK compresses headers using a 99-entry read-only static table and a dynamic table. Frequently used header key-value pairs (e.g., User-Agent, Cookie) are indexed in the dynamic table, translating strings into compact indices.
*   **Strategy**: Adjust dynamic table capacities (`hpack-capacity`) to balance memory footprints against header compression ratios.

#### Card 12. QPACK Encoder & Decoder Streams Synchronization
*   **Mechanism**: To maintain dynamic table consistency, H2O opens dedicated Encoder and Decoder Streams. When new table entries are created, the encoder notifies the receiver. Unfinished streams referencing these new entries are blocked until the decoder confirms synchronization.
*   **Strategy**: Limit concurrent unresolved requests to prevent stream stalling during periods of high latency.

#### Card 13. HTTP/3 Out-of-Order Delivery & HOLB Elimination
*   **Mechanism**: Unlike TCP, HTTP/3 allows UDP packets of different streams to arrive and process out of order. If a stream awaits QPACK dynamic table updates, its parsing is suspended while other completed streams are processed immediately, eliminating head-of-line blocking.
*   **Strategy**: Leverage out-of-order execution to maintain page response speeds over lossy wireless links.

---

# Page 2 (H2 Scheduler, Zero-Copy & Safety)

## 📂 M4: HTTP/2 依赖优先级调度树 (Cards 14-18)

#### Card 14. h2o_http2_scheduler_node_t Dependency Tree Topologies
*   **Mechanism**: The HTTP/2 scheduler in `lib/http2/scheduler.c` structures active streams into a dependency tree using `h2o_http2_scheduler_node_t` structs. Child nodes divide their parent's bandwidth allocations based on assigned weights.
*   **Strategy**: Support runtime tree updates via `PRIORITY` frames to dynamically align resource delivery with browser rendering states.

#### Card 15. h2o_http2_scheduler_run & Weighted Deficit Round Robin (WDRR)
*   **Mechanism**: Instead of sorting or traversing deep trees (which degrades under high stream counts), H2O implements WDRR. It tracks byte deficits for active scheduler branches, scanning nodes in $O(1)$ complexity to allocate write credits fairly.
*   **Strategy**: Use WDRR to handle thousands of concurrent streams per connection without causing CPU scheduling bottlenecks.

#### Card 16. 16KB Chunking Limits & Buffer Multiplexing Balance
*   **Mechanism**: To prevent large response streams from monopolizing socket write buffers, H2O limits single write calls to 16KB. Once a 16KB chunk is written, the scheduler is re-run, interleaving bytes from different streams.
*   **Strategy**: Interleave data in 16KB slices to optimize time-to-first-byte (TTFB) metrics for critical page assets.

#### Card 17. Heuristic Client Scheduler Adaptations
*   **Mechanism**: Certain web clients build suboptimal flat priority trees. H2O detects flat dependencies and applies heuristic corrections, boosting weights for critical HTML/CSS files above image assets to speed up page rendering.
*   **Strategy**: Retain default heuristic tunings to improve user-perceived page loads without changing client code.

#### Card 18. Concurrent Stream Constraints & Sliding Window Flow Control
*   **Mechanism**: H2O enforces maximum concurrent stream boundaries (default: 100). Both streams and connections monitor flow control windows (`WINDOW_UPDATE` frames). Writes are suspended when windows are exhausted, avoiding buffer bloat.
*   **Strategy**: Align window sizes with high Bandwidth-Delay Product (BDP) paths to prevent flow control bottlenecks.

---

## 📂 M5: 零拷贝内存管理与 I/O 缓冲区优化 (Cards 19-23)

#### Card 19. h2o_iovec_t Memory Slices & Non-Owning Pointer Offsets
*   **Mechanism**: H2O uses the `h2o_iovec_t` struct (a `base` pointer and a `len`) to represent memory slices. It allows slicing and parsing headers or body payloads without allocating new blocks or copying data, ensuring memory safety.
*   **Strategy**: Avoid accessing data behind `h2o_iovec_t` slices outside their lifetime scopes during asynchronous callbacks.

#### Card 20. h2o_buffer_t Mempool Allocation & Dynamic Slicing
*   **Mechanism**: The `h2o_buffer_t` structure dynamically grows buffer chains. Payloads below threshold limits (e.g., 64KB) utilize worker thread mempools, whereas larger payloads spill to temporary files, preventing excessive memory usage.
*   **Strategy**: Tune payload thresholds (`max-request-entity-size`) to protect memory limits during bulk upload surges.

#### Card 21. TCP sendfile Zero-Copy Kernel Bypass
*   **Mechanism**: For static files, H2O bypasses standard user-space read/write cycles by passing the file descriptor to the socket layer, invoking the `sendfile()` syscall to transfer data directly from page caches to TCP buffers.
*   **Strategy**: Combine sendfile with OS page cache tuning to achieve near line-rate delivery for static files.

#### Card 22. UDP GSO Offloading & sendmmsg Bulk Syscalls
*   **Mechanism**: UDP packet writes are computationally expensive. H2O supports Generic Segmentation Offload (GSO). It constructs large aggregated buffers and transmits them via a single `sendmmsg` syscall using `UDP_SEGMENT` options, letting the NIC partition them.
*   **Strategy**: Deploy H2O on GSO-supporting Linux kernels (4.18+) to optimize HTTP/3 network stack CPU efficiency.

#### Card 23. Write Backpressure Management & Socket Wait Queueing
*   **Mechanism**: When network bottlenecks trigger socket `EAGAIN` write errors, H2O buffers pending `h2o_iovec_t` slices, unsubscribes write notifications from the event loop, and resumes writes once the socket is ready (`EPOLLOUT`).
*   **Strategy**: Trust backpressure mechanisms to isolate slow clients and prevent unbounded buffer consumption.

---

## 📂 M6: 高级特性与容灾优化 (Cards 24-28)

#### Card 24. 103 Early Hints Flow & Static Asset Preloading
*   **Mechanism**: Before the backend generates a final response, H2O can emit a 103 Early Hints response containing Link headers. This lets clients resolve DNS and download CSS/JS assets while the server processes slow backend database queries.
*   **Strategy**: Implement 103 Early Hints for critical static resources to reduce perceived page rendering times.

#### Card 25. Distributed Session Ticket Key Synchronization
*   **Mechanism**: H2O supports reloading TLS session ticket keys dynamically from external key files (monitored via inotify). This ensures multiple independent instances behind a load balancer share the same ticket decryption keys.
*   **Strategy**: Set up cron processes to rotate ticket key files across H2O clusters, maintaining session resumption and forward secrecy.

#### Card 26. Pluggable Congestion Control Interfaces (cc-cubic)
*   **Mechanism**: `quicly` features a pluggable congestion control interface (`quicly_cc_type_t`). It supports CUBIC and Reno, feeding ACKs and packet loss signals back to state machines to adjust congestion windows.
*   **Strategy**: Choose CUBIC for standard WAN deployments to maximize throughput across varying network paths.

#### Card 27. Connection Draining & Graceful Server Shutdowns
*   **Mechanism**: Upon receiving a exit signal, H2O closes listener sockets, issues `GOAWAY` frames to HTTP/2 clients, and puts HTTP/3 connections into a Draining state, allowing active transactions to complete within timeouts before exiting.
*   **Strategy**: Configure grace periods to match maximum backend transaction times to ensure zero-downtime rolling updates.

#### Card 28. Anti-Flood Safeguards & HTTP/2 Stream Controls
*   **Mechanism**: H2O mitigates DoS threats (e.g., rapid stream resets, header flooding) by restricting maximum HEADERS frame counts and limiting the rate of incoming RST_STREAM frames, tearing down offending connections.
*   **Strategy**: Apply strict stream resets limits to protect servers from malicious resource exhaustion attacks.

---

## ⚔️ H2O Execution Parameters & OS Isolations Dict (Page 2 Bottom)

### T1: H2O Core Configuration Tuning Parameters
*   **`num-threads: <integer>`**: Sets the worker threads count, matching physical core counts to remove cross-core lock contentions.
*   **`hpack-capacity: <bytes>`**: Sets the HPACK/QPACK dynamic table limit (default 4096). Raising it improves compression on headers.
*   **`early-hints: ON | OFF`**: Configures 103 Early Hints. If enabled, pushes link headers before generation of final page outputs.
*   **`limit-request-body: <bytes>`**: Configures maximum request body size allowed to prevent malicious large uploads from exhausting memory.
*   **`http2-idle-timeout: <seconds>`**: Connection idle timeout, gracefully closing inactive sessions to reclaim descriptor resources.

### T2: System-level Isolation, Debugging & Client Diagnostic Commands
*   Raise socket send/receive buffer sizes to prevent UDP drops under high HTTP/3 packet rates:
    `sysctl -w net.core.wmem_max=16777216`
    `sysctl -w net.core.rmem_max=16777216`
*   Raise file descriptor limits to sustain millions of high-concurrency client sessions:
    `ulimit -n 1048576`
*   Use curl client to issue HTTP/3 requests and inspect control stream negotiations:
    `curl --http3 https://127.0.0.1:443/`
*   Utilize OpenSSL to diagnose TLS 1.3 handshakes and verify ALPN negotiations:
    `openssl s_client -connect 127.0.0.1:443 -alpn h3`
