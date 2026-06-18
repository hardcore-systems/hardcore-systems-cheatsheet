# awesome_scalability-高密度卡片系统设计大图 (大厂高并发与高扩展模式系统架构)

本大图描绘了 `checkcheckzz / awesome-scalability` 库中 28 张核心卡片的相互依赖和数据/控制流关系，涵盖了系统扩容、数据层拆分、异步流控、容灾弹性和大厂经典架构设计的全景闭环。

## 1. 卡片依赖拓扑 (Mermaid Diagram)

```mermaid
graph TD
    %% M1: 扩展原则与模式
    C1["Card 1: 垂直与水平扩展"] --> C2["Card 2: 无状态服务设计"]
    C1 --> C3["Card 3: 共享无脑 (Shared-Nothing)"]
    C2 --> C4["Card 4: 边缘 CDN 分流"]
    
    %% M2: 数据库分片与路由
    C3 --> C5["Card 5: 数据库主从与延迟"]
    C5 --> C6["Card 6: 水平分片 Shard Key"]
    C6 --> C7["Card 7: 跨片 JOIN 消除"]
    C6 --> C8["Card 8: 全局主键 Snowflake"]
    C5 --> C9["Card 9: 多活双向复制冲突"]
    
    %% M3: 事件驱动与 CQRS
    C2 --> C10["Card 10: CQRS 读写分离模型"]
    C10 --> C11["Card 11: Event Sourcing 事件溯源"]
    C11 --> C12["Card 12: 事件驱动架构 (EDA)"]
    C12 --> C13["Card 13: DDD 限界上下文"]
    
    %% M4: 异步消息传递与队列
    C12 --> C14["Card 14: 消息队列削峰填谷"]
    C14 --> C15["Card 15: 消息交付保障语义"]
    C15 --> C16["Card 16: 消息消费幂等性"]
    C14 --> C17["Card 17: 主动回压 (Backpressure)"]
    C14 --> C18["Card 18: 死信队列 (DLQ) 隔离"]
    
    %% M5: 系统容灾与网格舱壁
    C2 --> C19["Card 19: 熔断器三态状态机"]
    C19 --> C20["Card 20: 令牌桶与漏桶限流"]
    C2 --> C21["Card 21: 舱壁隔离线程池"]
    C21 --> C22["Card 22: Service Mesh Sidecar"]
    C19 --> C23["Card 23: 超时与指数退避重试"]
    
    %% M6: 大厂架构实践案例
    C19 --> C24["Card 24: Netflix 混沌工程"]
    C12 --> C25["Card 25: Twitter 推文推拉扩散"]
    C9 --> C26["Card 26: Amazon Dynamo 多数派复制"]
    C8 --> C27["Card 27: Google Spanner TrueTime"]
    C12 --> C28["Card 28: Uber Sagas 分布式事务"]
    
    %% Style Classes
    classDef m1 fill:#7A8B99,stroke:#333,stroke-width:1px,color:#fff;
    classDef m2 fill:#7D8F7B,stroke:#333,stroke-width:1px,color:#fff;
    classDef m3 fill:#9E828A,stroke:#333,stroke-width:1px,color:#fff;
    classDef m4 fill:#B58A7D,stroke:#333,stroke-width:1px,color:#fff;
    classDef m5 fill:#5F7582,stroke:#333,stroke-width:1px,color:#fff;
    classDef m6 fill:#BFA88F,stroke:#333,stroke-width:1px,color:#fff;
    
    class C1,C2,C3,C4 m1;
    class C5,C6,C7,C8,C9 m2;
    class C10,C11,C12,C13 m3;
    class C14,C15,C16,C17,C18 m4;
    class C19,C20,C21,C22,C23 m5;
    class C24,C25,C26,C27,C28 m6;
```

---

## 2. 核心架构原理与物理概念映射

*   **一致性边界与状态外置 (Statelessness)**：
    *   应用层的无状态化核心在于剥离 HTTP Session、本地临时状态文件，将其存储外置到高速共享介质（Redis 或分布式 DB）。
    *   消除了水平扩展时由于单点机器宕机导致的会话丢失风险，实现了 L7 网关的完全轮询调度。
*   **读写隔离与 CQRS (Command Query Responsibility Segregation)**：
    *   Command Model 处理变更，追求极致的写安全性和领域完整性（常映射到强一致性的数据库或事件存储区）。
    *   Read Model 处理查询，结构非规范化，高频冗余（冗余所需的所有字段，支持极速反范式单表拉取，可投射至 Elasticsearch/Redis/MongoDB 等读优化引擎中）。
*   **回压与过载控制 (Backpressure & Rate Limiting)**：
    *   当队列积压（Queue Lagging）超出警戒阈值时，自动反向缩紧 TCP window size，迫使生产者阻塞或报错，避免内存风暴。
    *   配合熔断器（Circuit Breaker）提供级联灾备隔离，避免微服务调用树上“由于一个叶子节点超时，导致整个树所有中间节点线程池耗尽”的系统性瘫痪。
*   **大厂特有工业算法机制**：
    *   **Vector Clocks**：在无主（Masterless）复制架构中，利用向量时钟追踪数据更新因果依赖，判定并发冲突。
    *   **Sagas Pattern**：在跨微服务事务中，通过定义每个步骤的正向操作（Action）和对应的反向补偿（Compensating Action）来实现最终一致，避免 2PC 阻塞死锁。
