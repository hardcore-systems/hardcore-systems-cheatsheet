# Step1-判题材-tech_selection.md: 高频架构设计折中矩阵审计大纲

本审计项目聚焦于系统架构师做选型决策时的关键技术折中 `Opinionated Tech Selection`。我们将通过 28 张核心知识卡片，解构关系与非关系数据库存储、消息队列事件流、API通信协议、分布式共识协同与微服务网格架构等核心组件。建立 L0-L2 架构阶梯，并设计双流仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: 数据库与分布式存储选型** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - MySQL vs PostgreSQL (关系型数据模型与并发机制)、MongoDB vs Cassandra (NoSQL文档与宽列分片)、Redis vs Memcached (内存缓存结构)、InfluxDB vs TimescaleDB (时序引擎)、RocksDB vs LevelDB (嵌入式LSM引擎)、CockroachDB vs TiDB (NewSQL共识调度)。
*   **M2: 消息队列与事件流选型** (Cards 7–12) - `#6B8272` (Muted Sage)
    - Kafka vs Pulsar (多租户事件流存储分离)、RocketMQ vs RabbitMQ (企业级协议与推拉控制)、Redis Streams vs MQTT (轻量/物联网协议)、Event Sourcing 状态溯源、Push vs Pull 消费者驱动、分区 vs 哈希环分片路由。
*   **M3: 通信协议与 API 架构选型** (Cards 13–18) - `#9C6666` (Tea Red)
    - RESTful HTTP/1.1 vs gRPC HTTP/2 (传输控制)、GraphQL 声明式查询、WebSocket vs SSE (实时推送通道)、Protobuf vs JSON vs Avro (序列化压缩)、Dubbo vs gRPC (微服务通信桩)、Kong vs Envoy (七层网关与网格拦截)。
*   **M4: 分布式共识与协同选型** (Cards 19–24) - `#7A7A7A` (Iron Grey)
    - Raft vs Paxos (强Leader与对等提案)、etcd vs ZooKeeper (协调树与Watch)、强一致 vs 最终一致 (线性化折中)、Gossip 协议对等收敛、Redlock vs ZK临时节点 (分布式锁防越界)、Vector Clocks vs HLC (因果时钟数据结构)。
*   **M5: 微服务与治理架构选型** (Cards 25–28) - `#9A825A` (Dusty Gold)
    - 同步 IPC vs 异步 EDA (微服务协作)、Sidecar 网格 vs Proxyless gRPC (治理侵入性)、Serverless (FaaS) vs K8s常驻计算、蓝绿 vs 灰度金丝雀 vs 滚动部署 (发布控制)。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    架构设计选型的本质是在业务吞吐量、数据一致性要求与运维复杂度（CAP/PACELC）之间，基于硬件资源极限做出的有偏向（Opinionated）的技术折中与工程解耦。
*   **L1 四句话逻辑**：
    1. **存储读写契约**：根据数据模型选择堆表与索引组织表，利用LSM-Tree与B+树折中读写吞吐，划定ACID边界。
    2. **消息流控拓扑**：根据吞吐与可靠性选择推拉模型，运用分区顺序或一致性哈希环漂移降低扩缩容震荡。
    3. **协议流式交互**：在文本JSON与紧凑二进制间平衡调试效率与带宽，使用多路复用长连接根治HOLB阻塞。
    4. **一致性与治理**：采用强共识或物理近似时钟判定事件偏序，衡量网络Sidecar开销，实现微服务零信任拦截。
*   **L2 核心数据流转拓扑**：
    `API请求接入` ➜ `网关Envoy进行xDS动态路由` ➜ `同步gRPC调用 vs 异步Kafka消息分发` ➜ `Kafka分区落盘 (PageCache零拷贝)` ➜ `写请求下发OLTP (MySQL/PostgreSQL MVCC版本控制)` ➜ `热点读取Redis缓存 (单线程多路复用)` ➜ `集群扩容触发一致性哈希环虚拟节点Key漂移` ➜ `Prometheus拉取指标` ➜ `TSDB TimescaleDB 数据归档`

---

## 🗂️ 28 张核心知识卡片大纲
1. MySQL vs PostgreSQL (关系型数据抉择)：B+树二级索引回表与堆表行头版本管理。
2. MongoDB vs Cassandra (NoSQL文档与宽列分片)：自研Replica Set强一致与Dynamo高可用最终一致。
3. Redis vs Memcached (内存缓存抉择)：单线程事件多数据类型与多线程Slab隔离KV。
4. InfluxDB vs TimescaleDB (时序数据库抉择)：TSM/TSI高基数Tag爆炸与Postgres Hypertable分表。
5. RocksDB vs LevelDB (嵌入式LSM引擎抉择)：多线程并发MemTable/Compaction与单线程SST合并。
6. CockroachDB vs TiDB (分布式NewSQL抉择)：对等Range 2PC意图锁与集中PD TSO混合HTAP架构。
7. Kafka vs Apache Pulsar (高性能事件流抉择)：追加Segment物理分区与BookKeeper计算存储分离分段。
8. RocketMQ vs RabbitMQ (企业级队列抉择)：CommitLog混写拉取与Erlang推模式Exchange路由。
9. Redis Streams vs MQTT (轻量/物联网队列抉择)：内存Radix Tree消费者组与TCP长连QoS主题分发。
10. Event Sourcing vs Event-Driven (事件流转抉择)：历史事件重播还原与轻量发布订阅生命周期。
11. Push vs Pull Consumer (消费者驱动模式)：推送网关流量淹没控制与拉取长轮询延迟平滑。
12. Partition-based vs Hash Ring (分片路由抉择)：静态分区顺序保证与哈希环漂移虚拟节点。
13. RESTful HTTP/1.1 vs gRPC HTTP/2 (传输协议抉择)：文本JSON动态调试与契约Protobuf多路复用。
14. GraphQL vs RESTful API (网关API抉择)：声明式单端点按需获取与多端点资源解耦网络级缓存。
15. WebSockets vs Server-Sent Events (实时推送抉择)：全双工自定义通道与标准HTTP流式通知。
16. Protocol Buffers vs JSON vs Avro (数据编码抉择)：契约Tag变长整型与自解释文本及动态Schema载荷。
17. Dubbo vs gRPC (微服务RPC框架抉择)：面向Java开箱即用治理与多语言低层契约。
18. Kong vs Envoy (API网关选型)：OpenResty Lua插件成熟控制与C++ xDS动态服务网格代理。
19. Raft vs Paxos (强一致共识抉择)：强Leader日志强更与对等二阶段提案碰撞冲突。
20. etcd vs ZooKeeper (分布式协调引擎)：Go与Raft现代 Watch API与Java ZAB临时树会话。
21. Strong Consistency vs Eventual (一致性折中)：线性化分布式锁开销与Quorum读写延迟折中。
22. Gossip Protocol vs Centralized (服务发现通信)：去中心随机UDP收敛与心跳注册表探活单点。
23. Redlock vs ZK Ephemeral Node (分布式锁抉择)：时钟漂移双重锁风险与临时节点排队Watcher机制。
24. Vector Clocks vs Hybrid Logical Clocks (因果时钟)：数组空间线性膨胀与毫秒结合逻辑计数器64位结构。
25. Sync IPC vs Async Event-Driven (服务协作选型)：调用链级联雪崩与MQ缓冲层背压消峰。
26. Sidecar Service Mesh vs Proxyless (网格治理选型)：iptables/eBPF无侵入劫持与gRPC SDK原生控制。
27. Serverless (FaaS) vs Containerized (K8s) (计算部署选型)：冷启动按需事件触发与常驻Pod弹性水平伸缩。
28. Blue-Green vs Canary vs Rolling (发布策略抉择)：两倍资源一键切流与细粒度金丝雀灰度控制。
