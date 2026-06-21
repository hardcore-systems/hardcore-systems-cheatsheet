# Sled Modern Embedded B-Tree Storage Engine Cheatsheet

## M1: Lock-Free B-Link Tree Layout

### Card 1: Lock-Free B-Link Tree Core
- **Mechanism**: Sled is built on a B-Link Tree. Traditional B-Trees lock the path from root to leaf during splits. B-Link Trees add right-sibling pointers, letting readers traverse to split nodes without holding locks.
- **Benefits**: Eliminates read-write blocking, minimizing concurrent latency bottlenecks on multi-core systems.
- **Limitation**: Splits occur locally and incrementally, requiring an asynchronous parent update engine.

### Card 2: Atomic Page Table CAS & Epoch EBR
- **Mapping**: A global Page Table maps logical Page IDs to memory pointers referencing the head of the page's Delta Chain.
- **Atomics**: All modifications of page pointers are performed atomically via Compare-And-Swap (CAS) operations.
- **Trade-off**: Removes page locks but can trigger cache bouncing under high write concurrency, mitigated by Epoch-Based Reclamation (EBR).

### Card 3: Leaf Page Logical Layout
- **Structure**: Leaf nodes contain sorted Key-Value slots stored in a contiguous, binary-searchable memory array.
- **Writes**: Instead of editing pages in place, writes are appended as Put delta nodes to the page's Delta Chain.
- **Reads**: Queries run binary search on the Base Page, merging updates from the Delta Chain.

### Card 4: Inner Node Routing & Traversal
- **Routing**: Inner nodes store child Page IDs and boundary keys.
- **Right-Link Route**: If a child page splits before parent routes update, queries detect boundary violations and traverse Right-Link pointers.
- **Trade-off**: Adds boundary check steps to read paths but keeps traversals lock-free.

### Card 5: Concurrency Trade-offs
- **Models**:
  - **Locked B-Tree**: Read-write locking limits throughput under write contention.
  - **Sled B-Link**: Combines atomics and Right-Links to enable lock-free reads and localize write CAS competition.
- **Write Amplification**: Delta Chains avoid rewriting entire pages for minor updates, reducing write amplification.

---

## M2: Log-Structured Page Store & Delta Chains

### Card 6: Log-Structured Page Allocation
- **Writes**: Sled uses log-structured storage. All page writes, updates, splits, and deletes are appended sequentially to active file segments.
- **Hardware**: Sequential writes map to flash memory garbage collection layouts, reducing wear and extending SSD lifespan.
- **Trade-off**: Non-in-place updates accumulate stale versions, requiring background GC.

### Card 7: Page Delta Chains
- **Representation**: Logical pages are stored as linked Delta Chains. The chain ends at a flat Base Page, preceded by Delta nodes (e.g. Put, Delete).
- **Updates**: New Deltas point to the old chain head and are linked atomically by swapping the Page Table pointer via CAS.
- **Limitation**: Long chains degrade read speed by increasing merge overhead, requiring consolidation.

### Card 8: Segment Mapping & Disk Redirects
- **Access**: Reads look up memory pointers in the Page Table. Evicted blocks are loaded from absolute disk Log Sequence Numbers (LSN).
- **Segments**: Log files are divided into fixed $8MB$ segments.
- **Trade-off**: Evicted pages require disk lookups, causing latency spikes that require prefetching.

### Card 9: Page Consolidation
- **Consolidation**: Background workers consolidate Delta Chains when chain lengths exceed thresholds (default 8).
- **Process**: Workers merge deltas into a flat Base Page, update the Page Table pointer via CAS, and free the old chain.
- **Trade-off**: Consolidation consumes CPU and disk IO but keeps read latency low.

### Card 10: WAL Group Commits
- **ACID**: Write transactions are buffered in memory before being flushed to the log.
- **Group Commit**: Multi-threaded writes are grouped into single I/O flushes to reduce physical disk seek times.
- **Control**: Users can configure flush frequencies to balance durability against execution speed.

---

## M3: Concurrent Page Splits

### Card 11: Leaf Page Split Sequence
- **Splits**: Leaf nodes exceeding physical boundaries (e.g. $8KB$) trigger split operations.
- **Stages**:
  - Stage 1: Allocate a new Page ID, copy the right half of the data to Leaf B, point Leaf B's right link to Leaf A's old neighbor, and update Leaf A's right link to point to Leaf B.
  - Stage 2: Asynchronously update the parent node route.
- **Visibility**: Readers can navigate from Leaf A to Leaf B during splits via right-sibling links.

### Card 12: Split-Phase Traversal
- **Recovery**: If a read request targets keys in Leaf B before parent routes update.
- **Path**: The query lands on Leaf A, detects key ranges exceed Leaf A's maximum boundaries, and traverses the Right-Sibling link to Leaf B.
- **Trade-off**: Adds boundary check steps to reads but keeps traversals uninterrupted and lock-free.

### Card 13: Asynchronous Parent Updates
- **Design**: Thread pools run Parent Node Updates asynchronously to keep splits from blocking write loops.
- **Linking**: Workers append a Split record to the parent node's Delta Chain.
- **Warning**: Slow parent updates force queries to traverse long right-link paths, requiring write throttling.

### Card 14: Concurrent Split Conflicts
- **Races**: Concurrent splits or writes on the same parent node trigger CAS failures.
- **Retries**: Sled uses Spin-CAS loops instead of locks. Threads retry CAS operations upon failure by reloading parent pointers.
- **Limitation**: High write contention increases CPU spinning, requiring EBR to manage allocations.

### Card 15: Crash Recovery & Reconstruction
- **Startup**: Sled rebuilds the global Page Table directory during startup.
- **Scan**: The engine scans log segments from the last checkpoint, applying log records to reconstruct memory structures.
- **Trade-off**: Infrequent checkpoints increase startup scan times.

---

## M4: LSN & Epoch Garbage Collection

### Card 16: Log Sequence Number (LSN) Tracking
- **Scale**: Every physical log entry receives a monotonic 64-bit Log Sequence Number.
- **Ordering**: LSNs establish absolute physical write order, allowing the engine to identify the latest updates during reconstruction.
- **Trade-off**: High-frequency LSN allocation requires optimized atomic generation paths.

### Card 17: Epoch-Based Reclamation (EBR)
- **Problem**: Memory nodes cannot be freed immediately in lock-free structures because readers might be traversing them.
- **EBR**: Threads register to the active Epoch on entry. Replaced nodes go to garbage queues and are freed once all threads exit the associated Epoch.
- **Benefits**: Secures concurrent memory freeing without global locks.

### Card 18: GC Segment Cleanups
- **GC**: Log files accumulate stale delta records and deleted blocks over time.
- **Evacuation**: Workers scan segments, copy active pages to the active log segment, and format empty segments for reuse.
- **Trade-off**: Copying active pages creates write amplification, requiring active threshold tuning.

### Card 19: Write Amplification Mitigation
- **Wear Leveling**: Sled tunes GC and write pathways to optimize solid-state storage lifespans.
- **Balance**: Segments sort and separate hot and cold records to reduce write amplification.
- **Trade-off**: Preserving drive lifespans requires maintaining larger storage footprints.

### Card 20: Segment Compaction Thresholds
- **Monitoring**: GC threads monitor segments using cleanup thresholds, active density, and stale ratios.
- **Trigger**: When the stale ratio of a segment exceeds 70%, compaction is triggered.
- **Tuning**: Adjust these thresholds to manage write amplification in performance-critical deployments.

---

## M5: Transactions & API

### Card 21: Interactive Transaction CAS Loops
- **ACID**: Sled supports fully ACID interactive transactions using optimistic concurrency controls.
- **Mechanism**: Transactions track read-write sets in thread-local buffers. At commit time, they update pages via Page Table CAS, retrying on conflict.
- **Trade-off**: Low overhead in clean environments, but high conflict rates trigger CPU-intensive retries.

### Card 22: Key Range Locks
- **Isolation**: Lock managers enforce key range locks during the final commit validation phase.
- **Scope**: Locks protect key intervals from write modifications during validation to prevent phantom reads.
- **Trade-off**: Simple and fast in-memory locking, but does not support distributed environments.

### Card 23: Read-Your-Own-Writes Consistency
- **Session**: Transactions must read their own uncommitted modifications.
- **Implementation**: Queries search local transaction write buffers before falling back to the global Page Table.
- **Trade-off**: Adds search steps to transaction reads but guarantees session consistency.

### Card 24: Concurrent Handle Safety
- **Handles**: Sled's `Db` handle implements `Send` and `Sync`.
- **Clones**: Threads clone DB handles to access the database concurrently without channel bottlenecks.
- **Atomics**: Page Table CAS and EBR handle synchronization, eliminating global locks.

---

## M6: Tuning & Diagnostics

### Card 25: Block Cache Eviction
- **Caching**: The cache manages flat Base Pages in memory.
- **Eviction**: Runs LRU algorithms to eject pages when memory limits are reached.
- **Warning**: In memory-constrained environments, cap cache sizes to avoid OS OOM kills.

### Card 26: Failpoint Chaos Injection
- **Validation**: Embeds Failpoints in page CAS, segment GC, and WAL flushes.
- **Chaos**: Injects OOM or I/O errors during splits to verify page table reconstruction during recovery.
- **Trade-off**: Target code is removed in Release builds to eliminate runtime overhead.

### Card 27: Telemetry Metrics
- **Diagnostics**: Exposes metrics on active pages, dirty ratios, EBR queue lengths, and WAL throughput.
- **Usage**: Integrate these metrics into monitoring tools to detect drive wear or performance bottlenecks.

### Card 28: Performance Tuning Dictionary
- **Tuning**:
  - `segment_cleanup_threshold`: Larger settings reduce GC overhead but increase disk footprint.
  - `use_compression`: Sacrifices CPU cycles to compress column data and save I/O bandwidth.
- **Optimization**: Adjust segment sizes (e.g. from 8MB to 1MB) to match SSD write blocks and reduce write amplification.
