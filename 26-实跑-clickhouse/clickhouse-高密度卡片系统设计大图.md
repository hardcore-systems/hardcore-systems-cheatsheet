# ClickHouse Internals Design Map

This design map links the 28 planned cards to the ClickHouse codebase architecture, files, and core algorithms.

---

## 🗺️ Codebase Architecture & File Mappings

### M1: Distributed Queries & Connections
*   **Card 1 (Connections)**: Maps to `src/Server/TCPHandler.cpp` and `src/Server/HTTPHandler.cpp`. Resolves client query contexts.
*   **Card 2 (Distributed Execution)**: Maps to `src/Interpreters/InterpreterSelectQuery.cpp` and `src/Interpreters/ClusterProxy/`. Explains the two-stage execution pipeline.
*   **Card 3 (Topology Routing)**: Maps to `src/Client/ConnectionPool.cpp` and `src/Client/ConnectionPoolWithFailover.cpp`. Handles weights and failovers.
*   **Card 4 (External FDWs)**: Maps to `src/Storages/StorageMySQL.cpp` and `src/Storages/StorageODBC.cpp`. Downward filter pushdowns.

### M2: Vectorized Execution & SIMD
*   **Card 5 (Column Vector)**: Maps to `src/Columns/ColumnVector.h` and `src/Columns/IColumn.h`. Contiguous memory alignment layout.
*   **Card 6 (Vector Loop)**: Maps to `src/Processors/ISimpleTransform.cpp` and `src/Processors/QueryPipeline.cpp`. Streaming block data chunks.
*   **Card 7 (SIMD Instructions)**: Maps to `src/Common/SIMDHelpers.h` and `src/Functions/FunctionsLogical.cpp`. SSE4.2/AVX2/AVX512 optimizations.
*   **Card 8 (Projections & MV)**: Maps to `src/Storages/StorageMaterializedView.cpp` and `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp`. Dynamic sync writes.
*   **Card 9 (Memory Tracker)**: Maps to `src/Common/MemoryTracker.cpp`. Monitors thread-level and query-level memory limits.

### M3: MergeTree Storage Engine
*   **Card 10 (Part Structure)**: Maps to `src/Storages/MergeTree/IMergeTreeDataPart.h` and `src/Storages/MergeTree/MergeTreeDataPartWriterOnDisk.cpp`. Part directories.
*   **Card 11 (Merge Algorithm)**: Maps to `src/Storages/MergeTree/MergeTreeDataMergerMutator.cpp`. Background merge worker loop.
*   **Card 12 (TTL Cleaning)**: Maps to `src/Storages/MergeTree/MergeTreeDataPartTTLInfo.h`. Purging expired rows during merges.
*   **Card 13 (Mutations)**: Maps to `src/Storages/MergeTree/MergeTreeMutationEntry.h` and `src/Storages/MergeTree/MergeTreeMutationStatus.h`. Asynchronous UPDATE/DELETE.

### M4: Sparse Index & Partitions
*   **Card 14 (Primary Index)**: Maps to `src/Storages/MergeTree/MergeTreeIndexPrimary.cpp`. Index granularity sampling.
*   **Card 15 (Mark Files)**: Maps to `src/Storages/MergeTree/MergeTreeReaderStream.cpp`. `.mrk` files resolving primary marks to `.bin` offsets.
*   **Card 16 (MinMax Key)**: Maps to `src/Storages/MergeTree/MergeTreeDataSelectExecutor.cpp`. Segment pruning via partition bounds.
*   **Card 17 (Skip Indexes)**: Maps to `src/Storages/MergeTree/IMergeTreeDataPart.cpp` (indices like Set, BloomFilter).

### M5: Replication & ZooKeeper
*   **Card 18 (Replication Core)**: Maps to `src/Storages/StorageReplicatedMergeTree.cpp` and `src/Storages/MergeTree/ReplicatedMergeTreeQueue.cpp`.
*   **Card 19 (Log Sync)**: Maps to `/clickhouse/tables/` path updates in `src/Common/ZooKeeper/ZooKeeper.cpp`.
*   **Card 20 (Queue Process)**: Maps to `src/Storages/MergeTree/ReplicatedMergeTreeQueue.cpp` parsing queue tasks and running HTTP downloads.
*   **Card 21 (ON CLUSTER DDL)**: Maps to `src/Interpreters/DDLWorker.cpp`. Distributed cluster execution workers.
*   **Card 22 (Keeper Core)**: Maps to `src/Coordination/` and `src/Coordination/KeeperStateMachine.cpp`. Raft consensus implementation.
*   **Card 23 (Split Brain)**: Maps to `src/Storages/MergeTree/ReplicatedMergeTreeAddress.h`. Read-only lock downgrades.

### M6: Block Layout & Compression
*   **Card 24 (Chunk Layout)**: Maps to `src/Processors/Chunk.cpp`.
*   **Card 25 (Compression Block)**: Maps to `src/IO/CompressedWriteBuffer.cpp` and `src/IO/CompressedReadBuffer.cpp`.
*   **Card 26 (LZ4/ZSTD)**: Maps to `src/IO/CompressionCodecLZ4.cpp` and `src/IO/CompressionCodecZSTD.cpp`.
*   **Card 27 (Special Codecs)**: Maps to `src/IO/CompressionCodecDoubleDelta.cpp` and `src/IO/CompressionCodecT64.cpp`.
*   **Card 28 (Too Many Parts)**: Maps to `src/Storages/MergeTree/MergeTreeData.cpp`. Parts write throtlling.
