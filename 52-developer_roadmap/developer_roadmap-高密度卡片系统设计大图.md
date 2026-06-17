# developer_roadmap-高密度卡片系统设计大图.md

本文件定义了 **developer-roadmap (开发者技术栈)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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

    Card1["Card 1: Scheduling"]:::M1
    Card2["Card 2: Page Tables"]:::M1
    Card3["Card 3: IPC Memory"]:::M1
    Card4["Card 4: inode & VFS"]:::M1
    Card5["Card 5: Socket & IO"]:::M1
    
    Card6["Card 6: ACID & MVCC"]:::M2
    Card7["Card 7: Redis Single"]:::M2
    Card8["Card 8: Raft & Paxos"]:::M2
    Card9["Card 9: Kafka Storage"]:::M2
    Card10["Card 10: gRPC Protobuf"]:::M2

    Card11["Card 11: Browser Pipeline"]:::M3
    Card12["Card 12: V-DOM & Diff"]:::M3
    Card13["Card 13: Webpack & Vite"]:::M3
    Card14["Card 14: SPA State Machine"]:::M3
    Card15["Card 15: XSS/CSRF/CORS"]:::M3

    Card16["Card 16: Namespace & Cgroup"]:::M4
    Card17["Card 17: Kubernetes API"]:::M4
    Card18["Card 18: CI/CD Pipeline"]:::M4
    Card19["Card 19: Liveness/Readiness"]:::M4

    Card20["Card 20: Hashing Ring"]:::M5
    Card21["Card 21: Circuit Breaker"]:::M5
    Card22["Card 22: Token Bucket"]:::M5
    Card23["Card 23: Shard Key & Snowflake"]:::M5
    Card24["Card 24: Observability"]:::M5

    Card25["Card 25: DDD Boundaries"]:::M6
    Card26["Card 26: TDD Cycles"]:::M6
    Card27["Card 27: Serverless Cold"]:::M6
    Card28["Card 28: RAG Vectors"]:::M6

    Card1 --> Card2
    Card2 --> Card3
    Card3 --> Card5
    Card4 --> Card5
    Card5 --> Card10
    Card6 --> Card8
    Card7 --> Card8
    Card8 --> Card23
    Card9 --> Card20
    Card10 --> Card17
    Card11 --> Card12
    Card12 --> Card14
    Card13 --> Card14
    Card14 --> Card15
    Card16 --> Card17
    Card17 --> Card19
    Card18 --> Card19
    Card20 --> Card21
    Card21 --> Card22
    Card22 --> Card21
    Card24 --> Card23
    Card25 --> Card26
    Card27 --> Card28
```

---

## 📍 Developer Roadmap 物理源码与系统拓扑映射

本设计大图的知识节点与计算机科学及主流全栈中间件源码模块强关联：
1. **Linux Kernel Systems**: Linux Kernel 调度器 (`kernel/sched/`) 与虚拟内存 (`mm/`) 源码。
2. **Database Core**: MySQL MVCC 事务引擎 (`storage/innobase/`) 与 Redis 事件库 (`src/ae.c`)。
3. **Frontend Renders**: Chromium Blink 渲染引擎 (`third_party/blink/renderer/`) 与 React 协调器 (`packages/react-reconciler/`)。
4. **Cloud Infrastructure**: Kubernetes 部署控制器 (`pkg/controller/deployment/`)。
