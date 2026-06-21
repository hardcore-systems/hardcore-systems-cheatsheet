# Step1-判题材-production_survival.md: 业务工程学与防雪崩重构审计大纲

本审计项目聚焦于业务架构落地与高可用重构方法论 `Production Survival & Refactoring`。我们将通过 28 张核心知识卡片，深度解构绞杀者重构模式、防腐层模型、数据库零停机双写迁移、分布式事务发件箱模式、Saga 事务编排、幂等性通用设计、网关多态限流算法、熔断器三态演进、线程池舱壁隔离、全链路压测与影子库、重试风暴退避、分布式追踪链路传递等核心模块。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: 架构隔离与重构模式** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - 绞杀者模式 (Strangler Fig)、防腐层隔离 (ACL)、双写数据库迁移、发件箱模式 (Outbox)、Saga 事务协调、幂等键设计。
*   **M2: 熔断降级与流控防御** (Cards 7–12) - `#6B8272` (Muted Sage)
    - 漏桶/令牌桶限流算法、熔断器三态演进 (Circuit Breaker)、舱壁隔离线程池 (Bulkhead)、优雅降级降载、乐观锁与 ETag、API 弹性版本控制。
*   **M3: 数据库演进与一致性保障** (Cards 13–19) - `#9C6666` (Tea Red)
    - 数据库 Schema 无停机扩展、影子库全链路压测、故障注入与混沌工程、最终一致性对账同步、重试风暴与退避抖动、死信队列 (DLQ) 治理、外观隔离降噪。
*   **M4: 部署发布与状态管理** (Cards 20–24) - `#7A7A7A` (Iron Grey)
    - 特性开关与金丝雀发布、背压传递流控、零停机滚动更新、分布式锁租约续期、CQRS 读写分离同步。
*   **M5: 探针追踪与日志聚合** (Cards 25–28) - `#9A825A` (Dusty Gold)
    - 客户端负载均衡与容错、健康检查探针自愈、分布式追踪上下文传递、日志聚合关联 ID。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    业务工程学的本质是利用结构化设计模式、防隔离代理与反馈控制链，将无序衰退的代码和易脆的分布式节点转化为具备自愈力、高容错与可平滑演进的韧性架构。
*   **L1 四句话逻辑**：
    1. **绞杀演进剥离**：通过引入路由外观代理，对单体服务进行渐进式微服务抽离，规避一次性重构失控。
    2. **物理舱壁阻断**：基于资源隔离与超时熔断，防止局部慢请求或雪崩风暴蔓延，实现故障快速失败。
    3. **双写入安全过渡**：采用双写、对账与动态切流三阶段法，保证数据存储在不停机的前提下平滑迁移。
    4. **全链路混沌检验**：以影子库和混沌故障注入为检验支柱，在生产环境下模拟极端崩溃，校正系统鲁棒性。
*   **L2 核心数据流转拓扑**：
    `流量发起` ➜ `网关令牌桶拦截` ➜ `舱壁线程池指派` ➜ `若超时或打满` ➜ `触发熔断器 (Closed ➜ Open)` ➜ `直接退回 Fallback 降级数据` ➜ `未被打满的正常节点继续工作` ➜ `数据库双写 (Legacy + New DB)` ➜ `对账任务纠偏漂移` ➜ `追踪上下文 TraceID 随 RPC 链条传递至各层日志`

---

## 🗂️ 28 张核心知识卡片大纲
1. 绞杀者模式 (Strangler Fig Pattern)：外观路由代理实现渐进式重构解耦。
2. 防腐层隔离 (Anti-Corruption Layer)：隔离新旧领域模型防模型污染。
3. 双写数据库迁移 (Dual-Write Database Migration)：零停机数据库迁移的五步渐进式转换演进。
4. 事务发件箱模式 (Outbox Pattern)：基于本地事务与事件轮询保证分布式最终一致性。
5. Saga 事务协调 (Saga Pattern)：编排器（Orchestration）与协同式（Choreography）补偿事务处理。
6. 幂等键生成与校验 (Idempotency Keys)：基于 Redis TTL 的防重复提交防护锁。
7. 漏桶与令牌桶限流 (Rate Limiting Algorithms)：应对突发流量的平滑与突发限流策略。
8. 熔断器三态演进 (Circuit Breaker Pattern)：Closed、Open、Half-Open 状态自动切换与快速失败。
9. 线程池与信号量舱壁 (Bulkhead Isolation)：核心池硬件资源独占隔离防止雪崩传递。
10. 优雅降级与丢弃保护 (Graceful Degradation)：高载状态下剥离非核心逻辑提供有损服务。
11. 乐观锁与 ETag 校验 (Optimistic Locking & ETag)：使用版本号和 ETag 防止中途冲突碰撞覆写。
12. 弹性版本控制 (API Versioning)：URL、Header 与 Media Type 驱动的不兼容接口演进。
13. 数据库 Schema 滚动更新 (Database Schema Evolution)：Expand-Contract 二阶段滚动表变更法。
14. 全链路压测与影子库 (Shadow Database Testing)：全链路流量染色与写向物理影子库方案。
15. 混沌工程与故障注入 (Chaos Engineering)：主动延迟注入、包丢失与网络分区测试验证。
16. 最终一致性对账同步 (Data Reconciliation)：利用定时扫描或流式数据流驱动库对账与偏差点补偿。
17. 重试风暴与退避抖动 (Retry Storms & Jitter)：指数退避与随机因子解除请求碰撞拥堵。
18. 死信队列治理 (Dead Letter Queue)：毒丸消息隔离机制与消费者重试防死锁。
19. 外观隔离解耦 (Facade Pattern for Monoliths)：利用统一门面屏蔽单体内部高耦合以简化剥离。
20. 特性开关与金丝雀发布 (Feature Toggles & Canaries)：代码部署与特性发布的解耦以及灰度切流。
21. 背压传递与流量拦截 (Backpressure Propagation)：当下游队列满时反向通知上游限流的流控链。
22. 零停机滚动更新 (Zero-Downtime Rolling Updates)：配合 K8s 探针和负载均衡进行的逐步替换发布。
23. 分布式锁租约续期 (Distributed Lock Leases)：以守护进程（看门狗）实现业务锁存活自动续展。
24. CQRS 读写分离同步 (CQRS Read/Write Separation)：高写吞吐与复杂读模型的数据流转同步。
25. 客户端负载均衡与容错 (Client-side Load Balancing)：客户端维护实例列表实现去中心化路由。
26. 健康检查与状态探针 (Health Check Probes)：Liveness、Readiness 和 Startup 探针的编排自愈。
27. 分布式追踪上下文传递 (Distributed Trace Propagation)：基于 W3C 规范的 TraceID/SpanID 跨服务透传。
28. 日志聚合与关联 ID (Log Aggregation & Correlation ID)：全链路透传唯一关联 ID 构建微服务诊断链。
