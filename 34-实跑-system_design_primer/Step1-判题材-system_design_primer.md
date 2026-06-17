# Step 1: 判题材 - donnemartin / system-design-primer (System Design Primer)

## 1. 题材深度分析与判定

`System Design Primer` 是全球软件开发领域公认的、系统架构与大规模高并发系统设计的经典知识体系大图。相较于具体的某个底层网络库或数据库，它侧重于对大规模分布式系统拼图块（Building Blocks）进行横向编织与纵向演化分析。

系统设计大图涉及的架构元素不仅在技术层面上相互制约，而且在底层原理上形成了高度内聚的技术深潭：包括 DNS 递归寻址与 CDN 调度策略、四层 (IP/TCP) 负载均衡的 LVS 中断分流与七层 (Nginx/Envoy) 代理、一致性哈希在哈希环上的虚拟节点分布及 Ketama 查表算法、Cache-aside/Write-through/Write-behind 缓存与数据库写一致性折衷、分库分表下基于 Shard Key 的二阶段路由与全局主键冲突、基于 Gossip 的分布式无中心状态同步、Paxos 与 Raft 共识选举时日志项的强一致复制，以及服务治理中的 token 令牌桶限流算法与熔断三态状态机。

上述每一层级都涉及高度硬核的架构设计考量（物理拓扑限制、CAP/PACELC 理论约束、并发写回压、强一致与最终一致折衷），信息密度极高，能够通过卡片拆解与两个高画质交互沙盒（一致性哈希环动态缩容、多级缓存读写策略）进行深度交付。

## 2. 核心架构与 6 大模块划分

为契合 A4 双页 Landscape 布局，我们将本项目划分为 6 个核心技术模块，每个模块对应一个莫兰迪色块（M1 ~ M6）：

*   **M1: 负载均衡与一致性哈希** (Slate Blue - `#7A8B99`)
    *   DNS 递归与 Anycast 广播分流、L4 负载均衡（IP 隧道 / NAT / DR 模式）与 L7 (Nginx / Envoy) 动态路由、基于哈希环的一致性哈希算法原理、引入虚拟节点消解数据倾斜与热点倾斜、Ketama 查找算法优化。
*   **M2: 多级缓存与一致性策略** (Moss Green - `#7D8F7B`)
    *   CDN 静态内容分发与动态路径优化、内存缓存（Redis / Memcached）缓存失效策略、Cache-aside (旁路缓存) 读写次序、Write-through (直写) 与 Write-behind/Write-back (后写) 数据持久化、LRU/LFU/FIFO 缓存逐出算法。
*   **M3: 数据库读写分离与分库分表** (Plum Rose - `#9E828A`)
    *   主从同步复制拓扑（同步、半同步、异步）与主备延迟路由规避、垂直拆分（按业务分库）与水平拆分（Sharding 分片）、分片键（Shard Key）评估与全局 ID 唯一性生成（Snowflake 算法）、数据库联合（Federation）与非规范化反范式优化。
*   **M4: 分布式协同与共识机制** (Terracotta - `#B58A7D`)
    *   CAP 定理极限约束与 PACELC 延时/一致性折衷、Gossip 协议的最终一致性与去中心化状态同步、两阶段提交 (2PC) 的阻塞故障与三阶段提交 (3PC) 引入超时分析、Paxos 与 Raft 状态机日志强一致复制机理。
*   **M5: 异步通信与消息队列流控** (Indigo - `#5F7582`)
    *   消息中间件（Kafka / RabbitMQ）的解耦、异步化与削峰降噪、发布订阅 (Pub/Sub) 模型、重复消息的物理去重与业务级幂等性（Idempotency）保障、死信队列 (DLQ) 与写回压（Backpressure）自适应流控。
*   **M6: 系统弹性设计：熔断、限流与防雪崩** (Antique Gold - `#BFA88F`)
    *   熔断器（Circuit Breaker）三态（Close, Open, Half-Open）转换条件、限流算法（令牌桶、漏桶、滑动窗口）、舱壁模式（Bulkheads）的资源线程隔离、服务降级（Degradation）降级开关设计、缓存击穿、穿透与雪崩防御结构。

---

## 3. L0 ~ L2 知识阶梯

### L0 一句话本质
System Design Primer 概括了从单体到千亿级高并发架构的演进规律，其本质是在网络物理边界和 CAP 理论硬约束下，通过在系统中巧妙引入负载均衡、多级缓存、分布式共识以及异步解耦，实现数据最终一致性与系统高可用的架构折衷。

### L1 四句话逻辑
1.  **负载分流与哈希负载均衡**：在网络入口处通过 DNS 配合 L4/L7 负载均衡将流量逐级卸载，核心依赖一致性哈希环，通过虚拟节点使得节点增减时只有 $1/N$ 的数据发生迁移，避免全网雪崩。
2.  **多级缓存与写一致策略**：将热点数据移入 CDN 与分布式内存缓存，读逻辑实施 Cache-aside 提高读取吞吐，写操作则在直写 (Write-through) 的一致性与后写 (Write-behind) 的极致写入速度间进行系统权衡。
3.  **数据库垂直水平切分拓扑**：单库物理瓶颈通过主从复制读写分离消解，当写入成为瓶颈时，通过选定合理的分片键（Shard Key）对数据水平切割分片，采用 Snowflake 算法消解分布式全局主键冲突。
4.  **共识约束与服务熔断降级**：数据状态依赖 Raft 等共识算法在多数派节点间强一致复制；在遇到流量突发时，系统通过令牌桶防洪限流，并触发熔断三态机制切断故障链路，实施降级以保护全局系统不崩溃。

### L2 核心数据流转拓扑
```
  [Downstream Client Request]
               │
               ▼
      [Anycast DNS Lookup] ───► [CDN Edge Cache Hit (Return Static)]
               │ (Miss)
               ▼
    [L4 Load Balancer (LVS)] (Distributes by IP hash)
               │
               ▼
  [L7 Reverse Proxy (Envoy/Nginx)] ───► [Rate Limiting (Token Bucket)]
               │ (Pass)
               ├───────────────────────┐
               ▼ (Read Request)        ▼ (Write Request)
       [Cache-aside Check]        [Write-behind Queue]
         ├─── Hit ──► [Return]           │ (Async Drain)
         └─── Miss ──┐                   ▼
                     ▼             [Master DB Write]
             [Slave DB Read]             │ (Binary Log Replication)
                     │                   ▼
                     ▼             [Slave DB Update]
             [Update Cache]
```

---

## 4. 28张卡片大纲规划

### Page 1 (分流、缓存、扩展与共识)

#### M1: 负载均衡与一致性哈希 (Cards 1-4)
*   **Card 1**: DNS 智能解析、Anycast IP 广播与 CDN 调度策略
*   **Card 2**: 四层负载均衡（LVS DR/NAT/TUN 模式）与七层代理的区别及适用场景
*   **Card 3**: 一致性哈希环的拓扑模型、顺时针寻址与节点增减的数据流变化
*   **Card 4**: 虚拟节点（Virtual Nodes）技术在消除数据分配倾斜与热点倾斜中的作用

#### M2: 多级缓存与一致性策略 (Cards 5-9)
*   **Card 5**: 多级缓存体系：客户端缓存、网关缓存、本地内存缓存与分布式缓存
*   **Card 6**: Cache-aside 旁路缓存策略的读写并发序列、双写不一致及其规避
*   **Card 7**: Write-through（直写）与 Write-behind/Write-back（后写）的数据一致性与延迟折衷
*   **Card 8**: 缓存逐出算法：LRU（双向链表+哈希表）与 LFU（频次桶）的物理实现复杂度
*   **Card 9**: 应对大并发下的缓存穿透、击穿、雪崩：布隆过滤器与互斥锁的防护策略

#### M3: 数据库读写分离与分库分表 (Cards 10-13)
*   **Card 10**: 主从复制拓扑结构中的同步、半同步、异步机制与主备延迟路由算法
*   **Card 11**: 水平分片（Sharding）与分片键（Shard Key）的选择偏好与热点倾斜防御
*   **Card 12**: 分布式全局主键：Snowflake（时间戳+工作节点ID+序列号）算法的碰撞消解
*   **Card 13**: 数据库联合（Federation）与非规范化反范式（Denormalization）的读性能置换

---

### Page 2 (分布式、异步、限流与容灾)

#### M4: 分布式协同与共识机制 (Cards 14-18)
*   **Card 14**: CAP 定理的物理局限、BASE 弱一致性模型与 PACELC 延迟折衷矩阵
*   **Card 15**: 去中心化 Gossip 协议在集群状态同步、拓扑变迁中的弱一致收敛
*   **Card 16**: 两阶段提交（2PC）的协调者单点故障与参与者同步阻塞死锁逻辑
*   **Card 17**: 三阶段提交（3PC）通过引入 PreCommit 状态与超时机制的故障缓解
*   **Card 18**: Raft 强一致性共识：Leader 选举状态转移与 Log Replication 日志复制机制

#### M5: 异步通信与消息队列流控 (Cards 19-23)
*   **Card 19**: 消息队列在分布式系统中的应用：解耦、异步流式处理与峰值削峰（Buffer）
*   **Card 20**: 消息传递交付保障：At-least-once、At-most-once 与 Exactly-once (两阶段事务)
*   **Card 21**: 消费者幂等性（Idempotency）保障：利用唯一消息 ID 与乐观锁处理去重
*   **Card 22**: 队列积压下的回压（Backpressure）机制与生产者挂起策略
*   **Card 23**: 死信队列（DLQ）的投递时机、未决异常隔离与手动重试补单通道

#### M6: 系统弹性设计：熔断、限流与防雪崩 (Cards 24-28)
*   **Card 24**: 熔断器（Circuit Breaker）三态（Close, Open, Half-Open）状态机的阈值跃迁
*   **Card 25**: 令牌桶（Token Bucket）与漏桶（Leaky Bucket）算法应对突发流量的限流性能差异
*   **Card 26**: 舱壁隔离（Bulkhead）模式：线程池隔离、信号量隔离与下游微服务容灾
*   **Card 27**: 优雅降级（Graceful Degradation）机制与动态熔断降级开关（Feature Toggles）
*   **Card 28**: 分布式系统架构的高可用性评估：MTBF、MTTR 与 SLA (99.99%) 系统度量
