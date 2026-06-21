# seastar-高密度卡片系统设计大图

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

    C1[Card 1: Shard-per-Core Core]:::M1 --> C2[Card 2: CPU Affinity Pinning]:::M1
    C2 --> C3[Card 3: Lock-Free Allocator]:::M1
    C2 --> C4[Card 4: Cooperative Fibers]:::M1
    C1 --> C5[Card 5: SPSC Cross-Core Ring]:::M1

    C3 --> C6[Card 6: Future & Promise Core]:::M2
    C6 --> C7[Card 7: Chained Continuation]:::M2
    C6 --> C8[Card 8: C++20 Coroutine Co_Await]:::M2
    C7 --> C9[Card 9: Micro-Scheduler Queue]:::M2
    C7 --> C10[Card 10: Exception Propagation]:::M2

    C5 --> C11[Card 11: Epoll & io_uring Engines]:::M3
    C11 --> C12[Card 12: Zero-Copy Packet Buffer]:::M3
    C11 --> C13[Card 13: Direct I/O O_DIRECT]:::M3
    C13 --> C14[Card 14: Disk I/O Fair Queue]:::M3
    C12 --> C15[Card 15: DPDK User-space Stack]:::M3

    C9 --> C16[Card 16: Lock-Free Shared Atom]:::M4
    C16 --> C17[Card 17: Read-Copy-Update RCU]:::M4
    C16 --> C18[Card 18: Non-Blocking Barriers]:::M4
    C19[Card 19: Cache Line Padding]:::M4 --> C18
    C20[Card 20: NUMA-Aware Allocations]:::M4 --> C3

    C15 --> C21[Card 21: Distributed Shards Service]:::M5
    C21 --> C22[Card 22: Connection Rate Limit]:::M5
    C21 --> C23[Card 23: Backpressure Flow Control]:::M5
    C22 --> C24[Card 24: Zero-Copy HTTP Parser]:::M5

    C24 --> C25[Card 25: Stalled Core Diagnostics]:::M6
    C25 --> C26[Card 26: Async Stack Reconstruction]:::M6
    C26 --> C27[Card 27: Telemetry Counters Profiling]:::M6
    C27 --> C28[Card 28: C++ Compiler Flags]:::M6
```

## 2. 源码符号映射
- `seastar/core/reactor.hh` (Card 1, 9) - 事件循环、CPU 调度及任务执行管道核心。
- `seastar/core/future.hh` (Card 6, 7) - 异步 Promise、Future 及其 Chaining 路由。
- `seastar/core/smp.hh` (Card 5, 18) - 跨核心通信、SPSC 无锁环形队列元信息。
- `seastar/net/dpdk.hh` (Card 15) - 用户态 DPDK 驱动层零拷贝数据包处理。
