# tokio-高密度卡片系统设计大图.md

本文件定义了 **tokio (Rust 工业级异步运行时与并发调度器)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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

    Card1["Card 1: Work-Stealing Queue"]:::M1
    Card2["Card 2: LIFO Slot"]:::M1
    Card3["Card 3: Coop Budget"]:::M1
    Card4["Card 4: Spawn & Sleep"]:::M1
    Card5["Card 5: Scheduler Types"]:::M1
    
    Card6["Card 6: Mio Event Loop"]:::M2
    Card7["Card 7: Token Registration"]:::M2
    Card8["Card 8: Park & Unpark Driver"]:::M2
    Card9["Card 9: OS Multiplexing"]:::M2
    Card10["Card 10: AsyncRead AsyncWrite"]:::M2

    Card11["Card 11: Hashed Wheel Timer"]:::M3
    Card12["Card 12: Sleep & Timeout Futures"]:::M3
    Card13["Card 13: Monotonic Clock"]:::M3
    Card14["Card 14: DelayQueue"]:::M3
    Card15["Card 15: Time Driver Thread"]:::M3

    Card16["Card 16: Async Mutex"]:::M4
    Card17["Card 17: Async Semaphore"]:::M4
    Card18["Card 18: Oneshot Cell"]:::M4
    Card19["Card 19: MPSC Ring Buffer"]:::M4

    Card20["Card 20: Broadcast Lag"]:::M5
    Card21["Card 21: Watch Value"]:::M5
    Card22["Card 22: Join & Select Macro"]:::M5
    Card23["Card 23: Task Local Storage"]:::M5
    Card24["Card 24: Graceful Shutdown"]:::M5

    Card25["Card 25: Tokio Console gRPC"]:::M6
    Card26["Card 26: Loom Concurrency Check"]:::M6
    Card27["Card 27: Future Size Stack"]:::M6
    Card28["Card 28: Spawn Boxing Cost"]:::M6

    Card1 --> Card2
    Card1 --> Card4
    Card2 --> Card3
    Card4 --> Card5
    Card6 --> Card7
    Card7 --> Card8
    Card8 --> Card15
    Card9 --> Card10
    Card11 --> Card12
    Card12 --> Card14
    Card13 --> Card12
    Card15 --> Card12
    Card16 --> Card17
    Card18 --> Card19
    Card19 --> Card20
    Card21 --> Card22
    Card22 --> Card24
    Card23 --> Card24
    Card25 --> Card28
    Card26 --> Card19
    Card27 --> Card28
```

---

## 📍 Tokio 物理源码位置映射

本设计大图的知识节点与 Tokio 核心类库及 Crate 物理源码强关联：
1. **Work-Stealing Scheduler**: `tokio/src/runtime/sched/multi_thread/` 及其核心数据结构 `queue.rs`。
2. **Coop Budget**: `tokio/src/coop.rs` 内部的上下文计数控制。
3. **Mio Reactor Driver**: `tokio/src/io/driver/` 关联 `mio::Poll`。
4. **Hashed Wheel Timer**: `tokio/src/util/time/` 内部基于槽位的时间轮实现。
5. **Loom Concurrency Verify**: `loom` 独立 Crate（`https://github.com/tokio-rs/loom`）。
