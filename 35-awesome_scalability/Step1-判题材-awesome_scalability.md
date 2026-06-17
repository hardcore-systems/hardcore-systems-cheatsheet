# Step1-判题材-awesome_scalability.md (大厂高并发与高扩展模式系统架构)

## 1. 题材背景与技术脉络

`checkcheckzz / awesome-scalability` 是全球最著名的大厂高并发、高可用与高可扩展模式的合集。它提炼了各大互联网巨头（如 Netflix, Twitter, Amazon, Google, LinkedIn 等）在海量请求和 PB 级数据下的架构演进最佳实践。其技术脉络涵盖了从基本的无状态水平扩展，到数据库分库分表、微服务解耦、CQRS/事件驱动架构（EDA），以及高可用稳定性保障（熔断限流、隔离模式）等全栈系统设计拼图。

---

## 2. 6 大莫兰迪分类模块设计

为了确保卡片系统的高清晰度与严谨结构，我们规划了以下 6 大模块，采用莫兰迪色系（M1-M6）：

*   **M1: 扩展原则与模式 (Scalability Principles & Patterns) - 莫兰迪 Slate Blue (`#7A8B99`)**
    *   卡片 1: 垂直扩展 (Scale-up) 与水平扩展 (Scale-out) 的物理边界与演进规律
    *   卡片 2: 无状态服务 (Stateless Services) 的共享无状态设计与会话共享机制
    *   卡片 3: 共享无脑 (Shared Nothing) 架构在分布式计算与存储中的分流隔离性
    *   卡片 4: 边缘计算 (Edge Computing) 与 CDN 动态/静态资源按区域就近分流
*   **M2: 数据库分片与路由 (Database Partitioning & Routing) - 莫兰迪 Moss Green (`#7D8F7B`)**
    *   卡片 5: 数据库主从复制读写分离下的同步延迟与“写后立即读主库”策略
    *   卡片 6: 水平分片 (Sharding) 的分片键 (Shard Key) 选择偏差与散列不均
    *   卡片 7: 分布式跨分片关联 (JOIN) 消除：反范式设计与二次查询归并路由
    *   卡片 8: 分布式全局主键：Snowflake 雪花算法时钟回拨异常与自增序列性能
    *   卡片 9: 多数据中心多活 (Active-Active) 架构下的双向复制与数据冲突判定
*   **M3: 事件驱动与 CQRS 架构 (Event-Driven & CQRS) - 莫兰迪 Plum Rose (`#9E828A`)**
    *   卡片 10: CQRS (命令查询职责分离) 读写模型物理隔离与最终一致性窗口
    *   卡片 11: 事件溯源 (Event Sourcing) 的事件日志追加写与查询重建状态回放
    *   卡片 12: 事件驱动架构 (EDA) 下的发布订阅模式与松耦合微服务拓扑
    *   卡片 13: 领域驱动设计 (DDD) 中的限界上下文与事件风暴限界划分
*   **M4: 异步消息传递与队列 (Asynchronous Messaging & Queues) - 莫兰迪 Terracotta (`#B58A7D`)**
    *   卡片 14: 消息队列在海量流量突发下的削峰填谷与流量整形缓冲作用
    *   卡片 15: 最多一次、最少一次与精确一次的交付语义保障与事务消息机制
    *   卡片 16: 消息幂等处理：唯一消息 ID、Redis 去重与数据库唯一约束防重入
    *   卡片 17: 队列积压下的主动回压 (Backpressure) 与生产者发送速率动态反馈
    *   卡片 18: 死信队列 (DLQ) 的消息重新投递阈值、异常隔离与告警人工处置
*   **M5: 系统容灾与网格舱壁 (Resilience & Service Mesh) - 莫兰迪 Indigo (`#5F7582`)**
    *   卡片 19: 熔断器模式的三态状态机转换、探测窗口与故障自愈冷却机制
    *   卡片 20: 令牌桶与漏桶限流算法差异：突发流量平滑度与排队排阻决策
    *   卡片 21: 舱壁隔离 (Bulkhead) 模式的线程池隔离、信号量限额与故障级联控制
    *   卡片 22: 微服务网格 (Service Mesh) 的 Sidecar 模式、控制面与数据面解耦
    *   卡片 23: 超时与指数退避重试 (Exponential Backoff with Jitter) 防重试风暴
*   **M6: 大厂架构实践案例 (Big Tech Case Studies) - 莫兰迪 Antique Gold (`#BFA88F`)**
    *   卡片 24: Netflix 混沌工程 (Chaos Engineering) 的 Chaos Monkey 随机注入与系统免疫
    *   卡片 25: Twitter 极速推文时间线 (Timeline) 拉取 (Pull) 与推入 (Push) 的写扩散折衷
    *   卡片 26: Amazon DynamoDB 无主多活复制、向量时钟 (Vector Clocks) 与 Read/Write Quorum
    *   卡片 27: Google Spanner 原子钟 + GPS (TrueTime API) 下的全球分布式事务强一致性
    *   卡片 28: Uber 基于 Sagas 模式的分布式多段柔性事务补偿机制与最终一致

---

## 3. L0 ~ L2 知识阶梯设计

*   **L0 一句话本质**: awesome-scalability 汇总了大厂在高并发与高扩展演进中的经典架构模式，其本质是通过服务的无状态化与水平扩展，将复杂业务拆分为微服务与事件驱动架构，在读写职责分离（CQRS）、消息队列异步削峰与系统级弹性（舱壁/熔断）的折衷下，构建具备高容错性与近乎无限扩展能力的分布式系统。
*   **L1 四句话逻辑**:
    1.  **服务无状态化与水平扩展**：在应用层剥离有状态的会话，通过 L4/L7 负载均衡实现无缝水平扩容，使用多级缓存与共享数据库隔离物理资源，扫清扩展单点瓶颈。
    2.  **写读职责分离与事件溯源**：在复杂场景下推行 CQRS，使写端（强业务约束）与读端（极致反范式检索结构）彻底解耦，依靠只追加写的 Event Sourcing 事件流日志进行状态持久化，通过异步投影实现最终一致性。
    3.  **异步通信削峰与可靠交付**：通过引入高吞吐的消息队列将生产者与消费者解耦，提供 At-least-once 交付语义，辅以消费端基于分布式锁的幂等性逻辑去重，实现在海量突发下的流量整形。
    4.  **舱壁隔离限流与混沌容错**：在微服务群落中通过 Sidecar 构建服务网格，强制实施线程池隔离的舱壁模式避免雪崩；利用混沌工程主动注入故障，逼迫系统建立重试退避与降级容灾自我免疫力。
*   **L2 核心数据流转拓扑**:
    *   `Client Command` ➜ `Gateway Router` ➜ `JWT & Rate Limiter` ➜ `Command Controller` ➜ `Command Model` ➜ `Event Store (Append Only WAL)` ➜ `Publish Event to Kafka` ➜ `Event Consumer (Projector)` ➜ `Update Query DB (Read Model)` ➜ `Client Query` ➜ `Gateway Router` ➜ `Read Service` ➜ `Redis Cache (Hit)` ➜ `Cache Miss` ➜ `Query DB (Fast Return)` ➜ `Return Client`.
