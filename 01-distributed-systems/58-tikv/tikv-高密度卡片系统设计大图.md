# tikv-高密度卡片系统设计大图.md

本文件定义了 **tikv (CNCF 毕业级分布式事务 Key-Value 数据库)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

---

## 🗺️ 28 张卡片依赖拓扑图 (Mermaid)

```mermaid
graph TD
    classDef default fill:#151d30,stroke:#24324f,color:#e2e8f0;
    classDef M1 fill:#4B5F7A,stroke:#2f3d52,color:white;
    classDef M2 fill:#6B8272,stroke:#4b5c50,color:white;
    classDef M3 fill:#9C6666,stroke:#704949,color:white;
    classDef M4 fill:#7A7A7A,stroke:#595959,color:white;
    classDef M5 fill:#9A825A,stroke:#6e5c40,color:white;
    classDef M6 fill:#755B77,stroke:#534054,color:white;

    Card1["Card 1: Region Range Partition"]:::M1
    Card2["Card 2: Raft Log Commit"]:::M1
    Card3["Card 3: Region Split"]:::M1
    Card4["Card 4: Region Merge"]:::M1
    Card5["Card 5: PD Scheduler"]:::M1
    
    Card6["Card 6: Percolator 2PC"]:::M2
    Card7["Card 7: Lock CF Concurrency"]:::M2
    Card8["Card 8: MVCC CF Paths"]:::M2
    Card9["Card 9: Opt vs Pessimistic Locks"]:::M2
    Card10["Card 10: Distributed Deadlock"]:::M2

    Card11["Card 11: RocksDB Dual Engine"]:::M3
    Card12["Card 12: WAL & MemTable Write"]:::M3
    Card13["Card 13: SST Compaction"]:::M3
    Card14["Card 14: Cache & Bloom Filter"]:::M3
    Card15["Card 15: Write Amplification Tuning"]:::M3

    Card16["Card 16: Coprocessor Pushdown"]:::M4
    Card17["Card 17: DAG Plan Parser"]:::M4
    Card18["Card 18: Vectorized Exec"]:::M4
    Card19["Card 19: Chunk Memory Format"]:::M4
    Card20["Card 20: SIMD Intel Acceleration"]:::M4

    Card21["Card 21: gRPC Heartbeat"]:::M5
    Card22["Card 22: Raft Async Pipeline"]:::M5
    Card23["Card 23: Zero-Copy Snapshot"]:::M5
    Card24["Card 24: Backpressure Throttle"]:::M5

    Card25["Card 25: Network Partition Resilient"]:::M6
    Card26["Card 26: Failpoint Chaos Test"]:::M6
    Card27["Card 27: RocksDB Monitor Metrics"]:::M6
    Card28["Card 28: Distributed Tracing Trace"]:::M6

    Card1 --> Card3
    Card1 --> Card5
    Card2 --> Card3
    Card3 --> Card4
    Card5 --> Card6
    Card6 --> Card7
    Card7 --> Card8
    Card8 --> Card9
    Card9 --> Card10
    
    Card11 --> Card12
    Card12 --> Card13
    Card13 --> Card15
    Card14 --> Card13
    Card11 --> Card8

    Card8 --> Card16
    Card16 --> Card17
    Card17 --> Card18
    Card18 --> Card19
    Card19 --> Card20

    Card21 --> Card22
    Card22 --> Card23
    Card23 --> Card24

    Card25 --> Card26
    Card27 --> Card28
```

---

## 📍 TiKV 物理源码位置映射

本设计大图的知识节点与 TiKV 核心类库及 Crate 物理源码强关联：
1. **Multi-Raft Core**: `components/raftstore/` 目录下的 `src/store/` 包含 Raft 状态机核心。
2. **Percolator & MVCC**: `src/storage/` 目录下的 `txn/` 和 `mvcc/` 分层。
3. **RocksDB Integration**: `components/engine_rocks/` 内部封装了 RocksDB 双引擎访问逻辑。
4. **Coprocessor Pushdown**: `components/coprocessor/` 包含向量化算子、DAG 结构解析。
