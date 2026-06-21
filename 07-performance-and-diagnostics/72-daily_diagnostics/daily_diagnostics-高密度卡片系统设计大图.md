# daily_diagnostics-高密度卡片系统设计大图.md

本文件定义了 **daily_diagnostics (线上故障定位与性能榨汁机)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码/组件映射锚点。

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

    Card1["Card 1: JVM GC Tuning"]:::M1
    Card2["Card 2: Go Allocator pprof"]:::M1
    Card3["Card 3: Python Async Loop"]:::M1
    Card4["Card 4: heaptrack Leaks"]:::M1
    Card5["Card 5: JVM CPU 100%"]:::M1
    Card6["Card 6: Go GC write barrier"]:::M1
    Card7["Card 7: Thread Pool Formula"]:::M2
    Card8["Card 8: Flame Graph"]:::M2
    Card9["Card 9: Cacheline align"]:::M2
    Card10["Card 10: CPU Affinity"]:::M2
    Card11["Card 11: Context Switch"]:::M2
    Card12["Card 12: JVM Safepoint"]:::M2
    Card13["Card 13: B+Tree Selection"]:::M3
    Card14["Card 14: MySQL Lock Deadlock"]:::M3
    Card15["Card 15: HikariCP pool"]:::M3
    Card16["Card 16: Redis Slow Big Key"]:::M3
    Card17["Card 17: Redis LFU Sample"]:::M3
    Card18["Card 18: CQRS separating"]:::M3
    Card19["Card 19: Page Cache Flush"]:::M4
    Card20["Card 20: Disk IO Scheduler"]:::M4
    Card21["Card 21: TCP Socket Window"]:::M4
    Card22["Card 22: Zero-copy sendfile"]:::M4
    Card23["Card 23: TIME_WAIT reuse"]:::M4
    Card24["Card 24: Async IO vs Thread"]:::M4
    Card25["Card 25: Epoll Thundering Herd"]:::M5
    Card26["Card 26: SIMD Vector"]:::M5
    Card27["Card 27: APM latency trace"]:::M5
    Card28["Card 28: Lock Reduction AQS"]:::M5

    Card1 --> Card12
    Card2 --> Card6
    Card3 --> Card24
    Card4 --> Card1
    Card5 --> Card8
    Card7 --> Card11
    Card8 --> Card7
    Card9 --> Card28
    Card10 --> Card9
    Card11 --> Card10
    Card13 --> Card14
    Card15 --> Card13
    Card16 --> Card17
    Card19 --> Card20
    Card21 --> Card23
    Card22 --> Card19
    Card24 --> Card25
    Card26 --> Card9
    Card27 --> Card28
```

---

## 📂 核心调优物理/组件映射锚点

在系统与应用调优中，性能榨汁模式映射于以下核心开源组件与代码结构中：

*   `sysctl.conf / limits.conf`: Linux 操作系统性能核弹阀，调优脏页参数、打开文件句柄（`nofile`）、TCP 缓冲区参数。
*   `async-profiler / perf`: Linux CPU 栈采样利器，非在安全点下采样，直接生成 SVG 交互式火焰图的最佳辅助组件。
*   `HikariCP / Druid`: 数据库连接池，在内存中管理多线程连接复用、探活以及并发 CAS 连接获取的性能库。
*   `Intel AVX-512 / ARM Neon`: 物理 CPU SIMD 向量化并行处理寄存器与指令集支持。
*   `LMAX Disruptor RingBuffer`: 无锁环形缓冲区，基于 CAS 和内存屏障，替代传统并发 BlockingQueue 的超高性能无锁队列库。
*   `std::net::TcpStream / sendfile`: POSIX 零拷贝底层网络方法，将文件描述符直接由内核写回网卡，跳过用户空间转换。
