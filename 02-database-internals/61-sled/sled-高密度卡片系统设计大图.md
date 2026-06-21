# sled-高密度卡片系统设计大图

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

    C1[Card 1: Lock-Free B-Link Tree]:::M1 --> C2[Card 2: Atomic Page Table CAS]:::M1
    C2 --> C3[Card 3: Leaf Page Layout]:::M1
    C2 --> C4[Card 4: Inner Node Routing]:::M1
    C1 --> C5[Card 5: Concurrency Trade-off]:::M1

    C3 --> C6[Card 6: Log-Structured Layout]:::M2
    C6 --> C7[Card 7: Page Delta Chains]:::M2
    C7 --> C8[Card 8: Segment Mapping]:::M2
    C7 --> C9[Card 9: Page Consolidation]:::M2
    C6 --> C10[Card 10: WAL Flush Sync]:::M2

    C9 --> C11[Card 11: Leaf Overflow Split]:::M3
    C11 --> C12[Card 12: Right-Link Traversal]:::M3
    C11 --> C13[Card 13: Async Parent Update]:::M3
    C12 --> C14[Card 14: Split Conflicts Retry]:::M3
    C10 --> C15[Card 15: Crash Recovery Build]:::M3

    C10 --> C16[Card 16: LSN Generation]:::M4
    C2 --> C17[Card 17: Epoch Reclamation EBR]:::M4
    C17 --> C18[Card 18: GC Segment Evacuate]:::M4
    C18 --> C19[Card 19: Write Amplification]:::M4
    C18 --> C20[Card 20: Compaction Thresholds]:::M4

    C17 --> C21[Card 21: Transaction CAS Loops]:::M5
    C21 --> C22[Card 22: Key Range Conflict]:::M5
    C21 --> C23[Card 23: Thread-Local Buffer]:::M5
    C22 --> C24[Card 24: Concurrent Handle Safety]:::M5

    C24 --> C25[Card 25: Cache Eviction LRU]:::M6
    C25 --> C26[Card 26: Failpoint Test Chaos]:::M6
    C26 --> C27[Card 27: Monitor Metrics Trace]:::M6
    C27 --> C28[Card 28: Performance Tunings]:::M6
```

## 2. 源码符号映射
- `sled/src/pagecache/mod.rs` (Card 2, 7) - Page Cache 和内存 Delta 链的逻辑抽象。
- `sled/src/tree.rs` (Card 1, 11) - 无锁 B-Link 树的分裂、遍历和路由更新。
- `sled/src/pagecache/segment.rs` (Card 18) - 闪存段（Segment）GC 及页面移动整理。
- `sled/src/pagecache/log.rs` (Card 6, 10) - Append-Only 日志的物理追加与持久化。
