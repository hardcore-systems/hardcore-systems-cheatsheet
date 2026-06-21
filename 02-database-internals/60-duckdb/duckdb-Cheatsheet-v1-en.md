# DuckDB Embedded Analytical SQL Engine Cheatsheet

## M1: Columnar Storage & Memory Layout

### Card 1: Columnar Physical Layout & Row Groups
- **Mechanism**: DuckDB groups table records into horizontal chunks called Row Groups of $122,880$ rows each. Inside a Row Group, column values are partitioned vertically and stored as contiguous Column Segments.
- **Benefits**: Columnar layout enables high-efficiency data fetching, isolating scans to requested columns while ignoring irrelevant attributes to maximize hardware bandwidth.
- **Trade-off**: Row Groups that are too large limit delete visibility performance, whereas small Row Groups reduce memory scanning bandwidth advantages.

### Card 2: Block Allocation & Segment Storage Management
- **Structure**: Persistence is consolidated into a single database file divided into fixed $256KB$ storage blocks. Segments map columns to specific physical data blocks.
- **Metadata**: Segment headers record physical block offsets and compression encoding types.
- **Trade-off**: Fixed-size blocks simplify space allocation and vacuuming but cause internal fragmentation in small tables.

### Card 3: Buffer Manager & Memory Thresholds
- **Caching**: The Buffer Manager loads and unloads blocks into RAM. Under memory pressure, it ejects unpinned blocks and dumps intermediate operator states to temporary disk files.
- **Mechanics**: Manages memory structures via PINNED/UNPINNED flags in an eviction queue.
- **Warning**: Prevent long-lived PINs in concurrent execution loops to avoid deadlock scenarios or fake OOM aborts.

### Card 4: Zone Map Filters & Min-Max Indexes
- **Filtering**: Segment metadata embeds Zone Maps capturing minimum and maximum values for local columns.
- **Optimization**: The query parser compares filtering conditions against Zone Maps before execution, bypassing non-overlapping Segments to eliminate unnecessary read operations.
- **Limitation**: Unsorted datasets widen the range of Min-Max fields, reducing skip ratios and requiring physical sorting.

### Card 5: In-Memory WAL & Persisted Checkpoint
- **Durability**: Write transactions log records to an append-only Write-Ahead Log (WAL) on disk.
- **Checkpointing**: During a checkpoint, WAL records are merged into permanent column blocks on disk, and the WAL is truncated.
- **Trade-off**: Synchronous disk flushes slow down transaction throughput. Read-heavy OLAP systems can run in-memory without logs.

---

## M2: Vectorized Engine & Core Execution

### Card 6: Vectorized Execution Core
- **Execution**: DuckDB processes data in contiguous vectors of $1024$ or $2048$ values rather than using row-by-row Volcano loops.
- **Locality**: Operators loop contiguously through elements in memory, keeping datasets in the CPU L1 cache to avoid dynamic virtual dispatch overhead.
- **Limitation**: Excessively large vectors overflow L1 caches and degrade execution performance.

### Card 7: Volcano vs Vectorized Execution Trade-offs
- **Models**:
  - **Volcano**: Employs row-by-row dispatches, causing high branches and poor cache utilization.
  - **Vectorized**: Processes vectors in batches, running execution loops locally inside cache boundaries.
- **Trade-off**: Vectorization increases analytical speeds by orders of magnitude but adds initialization overhead for simple point lookups.

### Card 8: CPU SIMD Register Instruction Acceleration
- **SIMD**: Clang auto-vectorizes loop arrays when operators process contiguous vector datasets.
- **Acceleration**: Compilers generate SIMD code (AVX2, AVX-512) to compute multiple arithmetic or logical filters in a single CPU cycle.
- **Hardware**: Binaries require specific instruction extensions. Compile targets must detect CPU flags to prevent illegal instruction crashes.

### Card 9: Dynamic Casting & Selection Vector Projection
- **Casting**: Vector calculations check data types, instantiating specialized conversion operators (Casting Vectors) to align types.
- **Selection Vector**: Filters generate index arrays (Selection Vectors) referencing qualified rows instead of copying datasets.
- **Trade-off**: Reduces allocation and copy overheads but introduces non-contiguous memory access paths.

### Card 10: Window Function Vector Buffering
- **State**: Analytical window operations (e.g. LAG, LEAD) require tracking relative row boundaries.
- **Buffering**: DuckDB stores active window datasets in vectorized chunk buffers, resolving offsets using pointer additions.
- **Limitation**: Extensive windows consume massive RAM, triggering eviction to disk when limits are hit.

---

## M3: Transaction Manager & Epoch MVCC

### Card 11: Transaction Manager & Epoch Lifecycle
- **Concurrency**: DuckDB implements Epoch-based Multi-Version Concurrency Control (MVCC) with a monotonic system Epoch counter.
- **Binding**: Transactions bind to the active Read Epoch at startup and obtain write Epochs at commit.
- **Trade-off**: Simple and fast for embedded setups, but does not support scale-out distributed consistency out of the box.

### Card 12: MVCC Visibility Rules & Read Path
- **Visibility**: Records carry insertion transaction IDs (insert_id) and deletion IDs (delete_id) in their metadata headers.
- **Lock-Free Reads**: Read transactions assess visibility dynamically by checking if `insert_epoch <= read_epoch` and `delete_epoch > read_epoch` without acquiring locks.
- **Performance**: Eliminates lock contention between readers and writers.

### Card 13: Logical Deletes & Version Chain
- **Deletes**: Deletes are registered logically by appending transaction IDs to the row group's Delete Vector.
- **Updates**: Implemented as a delete followed by an insert, linking old versions to new versions via physical Version Chains.
- **Limitation**: Long-lived transactions accumulate stale versions, requiring periodic reclamation to avoid read performance drops.

### Card 14: Column Compaction & Compression
- **Compaction**: When deleted versions fall past the Safe Epoch, background workers trigger compaction.
- **Compression**: Workers compress raw columns into read-only blocks using Run-Length (RLE), Bit-Packing, or Dictionary encoding.
- **I/O Overhead**: Background sorting and encoding consume CPU and disk bandwidth, which can cause minor latency spikes.

### Card 15: ACID Compliance & WAL Truncation
- **Persistence**: Transaction commits flush modified metadata pages and active WAL sectors to disk.
- **Rollbacks**: Aborted transactions discard memory buffers and invalidate associated WAL sequences.
- **Trade-off**: Maintaining strict ACID durability requires exclusive locks in the Transaction Manager, limiting write concurrency.

---

## M4: Parallelism & Query Pipeline

### Card 16: Push-Based Execution Model
- **Data Flow**: Unlike the Pull model where operators call `next()` on children, DuckDB pushes vectorized chunks up from Scan nodes through active pipelines.
- **Parallelism**: The Push model simplifies scheduling across CPU threads and enables fine-grained backpressure controls.
- **Complexity**: Inverting data and control paths increases executor orchestration complexity.

### Card 17: Parallel Scan & Work Stealing
- **Scheduling**: A global Task Scheduler maps execution pipelines to threads matching CPU core counts.
- **Stealing**: Row group scans are divided into Tasks. Idle threads steal scanning tasks from busy queues to balance load.
- **Trade-off**: Eliminates thread starvation but adds lock contention overhead under extreme load imbalances.

### Card 18: In-Memory Hash Join Algorithm
- **Hash Join**: Build operators generate optimized in-memory hash tables, and Probe operators match data vectors in batches.
- **Cache Optimization**: Hash tables align to cache lines and use Selection Vectors to return key matches.
- **Limitation**: Hash tables exceeding memory limits must partition and spill datasets to disk.

### Card 19: External Merge Sort Spilling
- **External Sort**: When sorting datasets exceeding memory allocations, DuckDB runs external merge sorting.
- **Spilling**: Sort partitions are flushed to temporary files on disk, and a multi-way merge operator consolidates results.
- **Latency**: Disk write and read latency slows sorting speeds by orders of magnitude compared to in-memory paths.

### Card 20: Out-of-Core Memory Manager
- **Management**: The Memory Manager tracks RAM consumption during query execution.
- **Spilling**: Reaches limits trigger evictions, forcing Hash Join builders or Aggregators to flush memory states to disk.
- **Benefit**: Prevents query executions from triggering OS OOM process kills at the expense of disk I/O.

---

## M5: Formats & Connectors

### Card 21: Parquet Zero-Copy Parsing
- **Zero-Copy**: The Parquet Reader maps column blocks directly into DuckDB Vector arrays without conversion.
- **Pushdown**: SQL filters are pushed down to Parquet row group headers to bypass decompression of irrelevant segments.
- **Usage**: Enables instant query execution on remote S3 Parquet tables without staging.

### Card 22: Arrow Memory Map Integration
- **Mapping**: Provides direct integration with Apache Arrow memory schemas.
- **Interface**: DuckDB reads Arrow Record Batches directly by reference, bypassing serialization.
- **Trade-off**: Requires dedicated translation code but enables sub-millisecond exchanges with Python data tools.

### Card 23: CSV & JSON Fast Parsers
- **Scanners**: Multi-threaded scanners search for line breaks to divide files into parallel parsing threads.
- **Type Inference**: Analyzes early batches to automatically resolve column types during runtime.
- **Limitation**: Highly deformed or irregular CSV layouts trigger parse failures and rollbacks.

### Card 24: Embedded Architecture & Thread Safety
- **Libraries**: Compiles as a C++ library embedded directly inside host processes (Python, Node.js).
- **Concurrency**: Fine-grained mutexes protect the Catalog, Buffer Manager, and Transactions, securing concurrent connections.
- **Limitation**: Database files support only a single active writer process at a time.

---

## M6: Extensions & Diagnostics

### Card 25: Catalog & Schema Metadata
- **Catalog**: Thread-safe metadata hierarchy managing schemas, views, tables, and UDFs.
- **Versioning**: Catalog changes are versioned using Epoch MVCC, isolating tables across concurrent transactions.
- **Trade-off**: Ensures isolation safety but adds metadata lock evaluation paths.

### Card 26: UDF & Extension SDK
- **Plugins**: Exposes a C++ and C SDK to register custom functions (UDFs) and database engines.
- **Vectorized UDFs**: Extensions process data in batches via Vector structures, utilizing CPU SIMD loops.
- **Warning**: Extensions run in the host process address space. Segment faults crash the entire application.

### Card 27: Profiling Trace Metrics
- **Profiling**: Enforces operator-level performance tracking via `PRAGMA enable_profiling='json'`.
- **Metrics**: Captures operator runtimes, vector counts, disk spillage, and worker activity splits.
- **Debugging**: Pinpoints bottlenecks, such as slow operators or failed Zone Map skipping.

### Card 28: Resource Limits & Throttling
- **Limits**: Restricts database memory footprints via `SET max_memory='32GB'`.
- **Throttling**: Triggers page evictions on memory limits or aborts execution when limits are hit.
- **Benefit**: Secures host environments in multi-tenant deployments or resource-constrained environments.
