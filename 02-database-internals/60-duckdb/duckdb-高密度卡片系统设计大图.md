# duckdb-高密度卡片系统设计大图

## 1. 卡片依赖拓扑图 (Mermaid)
```mermaid
graph TD
    classDef default fill:#1A202C,stroke:#24324F,color:#E2E8F0;
    classDef M1 fill:#4B5F7A,stroke:#2D3748,color:white;
    classDef M2 fill:#6B8272,stroke:#2D3748,color:white;
    classDef M3 fill:#9C6666,stroke:#2D3748,color:white;
    classDef M4 fill:#7A7A7A,stroke:#2D3748,color:white;
    classDef M5 fill:#9A825A,stroke:#2D3748,color:white;
    classDef M6 fill:#755B77,stroke:#2D3748,color:white;

    C1[Card 1: Columnar Layout & Row Groups]:::M1 --> C2[Card 2: Block & Physical Storage]:::M1
    C2 --> C3[Card 3: Buffer Manager & Limit]:::M1
    C1 --> C4[Card 4: Zone Map Interval Filter]:::M1
    C2 --> C5[Card 5: In-Memory WAL & Persist]:::M1

    C3 --> C6[Card 6: Vectorized Exec Core]:::M2
    C6 --> C7[Card 7: Volcano vs Vectorized]:::M2
    C6 --> C8[Card 8: SIMD Register Speed]:::M2
    C8 --> C9[Card 9: Dynamic Cast & Projection]:::M2
    C6 --> C10[Card 10: Window Vector Buffer]:::M2

    C5 --> C11[Card 11: Transaction Epochs]:::M3
    C11 --> C12[Card 12: MVCC Visibility Rules]:::M3
    C12 --> C13[Card 13: Logical Deletes & Chain]:::M3
    C12 --> C14[Card 14: Column Compaction Compress]:::M3
    C11 --> C15[Card 15: ACID Compliance & Checkpoint]:::M3

    C6 --> C16[Card 16: Push-Based Exec Model]:::M4
    C16 --> C17[Card 17: Parallel Scan & Stealing]:::M4
    C16 --> C18[Card 18: In-Memory Hash Join]:::M4
    C18 --> C19[Card 19: External Merge Sort Spill]:::M4
    C19 --> C20[Card 20: Out-of-Core Memory Spill]:::M4

    C17 --> C21[Card 21: Parquet Zero-Copy Parse]:::M5
    C21 --> C22[Card 22: Arrow Memory Map]:::M5
    C21 --> C23[Card 23: CSV & JSON Fast Parsers]:::M5
    C22 --> C24[Card 24: Embedded Thread Safety]:::M5

    C24 --> C25[Card 25: Catalog Schema Metadata]:::M6
    C25 --> C26[Card 26: UDF & Extension SDK]:::M6
    C26 --> C27[Card 27: Profiling Trace Metrics]:::M6
    C27 --> C28[Card 28: Resource Limits Throttling]:::M6
```

## 2. 源码符号映射
- `duckdb/common/vector.hpp` (Card 6, 9) - 内存向量的物理封装与批数据布局。
- `duckdb/storage/block_manager.cpp` (Card 2, 3) - 数据块的读写、物理分配和缓存回收。
- `duckdb/transaction/transaction_manager.cpp` (Card 11, 12) - Epoch 状态转换与 MVCC 读写过滤。
- `duckdb/execution/physical_operator.cpp` (Card 16, 17) - Push 架构物理执行算子与工作线程。
- `duckdb/main/catalog/catalog.cpp` (Card 25) - 数据库目录元数据管理器。
