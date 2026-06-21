# 高频架构设计折中矩阵速查表
## J-Ladder 架构分层体系

### L0 一句话本质
架构设计选型的本质是在业务吞吐量、数据一致性要求与运维复杂度（CAP/PACELC）之间，基于硬件资源极限做出的有偏向（Opinionated）的技术折中与工程解耦。

### L1 四句话逻辑
1. **存储读写契约**：根据数据模型选择堆表与索引组织表，利用LSM-Tree与B+树折中读写吞吐，划定ACID边界。
2. **消息流控拓扑**：根据吞吐与可靠性选择推拉模型，运用分区顺序或一致性哈希环漂移降低扩缩容震荡。
3. **协议流式交互**：在文本JSON与紧凑二进制间平衡调试效率与带宽，使用多路复用长连接根治HOLB阻塞。
4. **一致性与治理**：采用强共识或物理近似时钟判定事件偏序，衡量网络Sidecar开销，实现微服务零信任拦截。

### L2 核心数据流转拓扑
`API请求接入` ➜ `网关Envoy进行xDS动态路由` ➜ `同步gRPC调用 vs 异步Kafka消息分发` ➜ `Kafka分区落盘 (PageCache零拷贝)` ➜ `写请求下发OLTP (MySQL/PostgreSQL MVCC版本控制)` ➜ `热点读取Redis缓存 (单线程多路复用)` ➜ `集群扩容触发一致性哈希环虚拟节点Key漂移` ➜ `Prometheus拉取指标` ➜ `TSDB TimescaleDB 数据归档`

---

## 📂 核心知识卡片 (Cards 1-28)

### Card 1: MySQL vs PostgreSQL (关系型数据抉择)
*   **核心原理**: MySQL采用索引组织表且多用B+树二级索引回表，MVCC基于Undo Log链；PostgreSQL使用堆表，MVCC通过在行头存储xmin/xmax并在共享缓冲区中生成新版本行（页内写多）。
*   **技术细节**: MySQL InnoDB通过隔离级别（如可重复读下的Gap Lock）防止幻读，支持快照读和当前读；PostgreSQL使用多版本并发控制与SSI（基于SIREAD锁追踪的冲突依赖环检测），依靠VACUUM清理死元组。
*   **折中与防范**: MySQL对频繁写入与二级索引优化较好但复杂查询慢；PostgreSQL支持丰富类型与复杂查询但有WAL与Table Bloat膨胀风险，需定期Auto-vacuum。

### Card 2: MongoDB vs Cassandra (NoSQL 抉择)
*   **核心原理**: MongoDB是强一致文档数据库，使用B-Tree索引；Cassandra是高可用宽列存储，基于Dynamo架构的无中心哈希环与SSTable，采用最终一致性与Gossip协议。
*   **技术细节**: MongoDB依靠WiredTiger引擎的B-Tree与锁机制，采用Replica Set多数派协议；Cassandra依靠客户端读写一致性级别（Quorum = R + W > N）控制一致性，写入时追加CommitLog后入MemTable。
*   **折中与防范**: MongoDB适合Schema多变且需要强一致读写的业务；Cassandra写入吞吐极高且无单点故障，但不支持ACID复杂事务，且存在读放大（Read Repair）开销。

### Card 3: Redis vs Memcached (分布式缓存抉择)
*   **核心原理**: Redis是单线程事件循环的多数据结构存储，支持持久化和复制；Memcached是多线程Slab Allocation内存级KV存储，不支持持久化。
*   **技术细节**: Redis使用epoll I/O多路复用，通过RDB快照/AOF日志和哨兵/集群实现高可用；Memcached使用LRU链表和Slab隔离物理内存，避免内存碎片但限制了Value大小为1MB。
*   **折中与防范**: Redis功能强、支持丰富类型和持久化，但单线程大Key阻塞会导致整体响应抖动；Memcached适合简单且纯内存的极速KV读取，但需在客户端实现一致性哈希分片。

### Card 4: InfluxDB vs TimescaleDB (时序数据库抉择)
*   **核心原理**: InfluxDB是专为指标优化的自研TSM/TSI引擎时序库；TimescaleDB是基于PostgreSQL的Hypertable关系型扩展时序库。
*   **技术细节**: InfluxDB通过WAL和TSM文件将时间序列数据列式存储，使用TSI哈希表维护Tag索引；TimescaleDB将大表自动按时间和分区列拆分为Chunk，继承Postgres的B-Tree索引和完整SQL生态。
*   **折中与防范**: InfluxDB写吞吐高、存储压缩率极佳，但高基数Tag爆炸会导致内存溢出；TimescaleDB支持复杂JOIN和SQL，但数据压缩开销较大且吞吐略低于InfluxDB。

### Card 5: RocksDB vs LevelDB (单机嵌入式存储抉择)
*   **核心原理**: RocksDB是基于LSM-Tree并针对多核CPU、SSD进行深度优化的嵌入式KV引擎；LevelDB是经典轻量级单线程LSM存储引擎。
*   **技术细节**: RocksDB引入Column Families隔离数据空间，支持多线程并发MemTable写入和并发Compaction；LevelDB使用单一SSTable结构和单线程后台合并。
*   **折中与防范**: RocksDB吞吐极高、配置参数极丰富，但调优难度极大且写放大/空间放大显著；LevelDB极其轻巧稳定，适合嵌入式或资源受限环境。

### Card 6: CockroachDB vs TiDB (分布式NewSQL抉择)
*   **核心原理**: CockroachDB采用多Raft组（Range）对等架构，底座为基于Pebble的KV存储；TiDB采用计算与存储分离架构，计算层TiDB无状态，存储层TiKV/TiFlash使用Multi-Raft。
*   **技术细节**: CockroachDB使用无共享对等网络，通过HLC协调事务可见性，依靠2PC与写意图锁执行事务；TiDB通过PD（Placement Driver）集中调度物理Region，使用TSO全局授时和Percolator分布式事务模型。
*   **折中与防范**: CockroachDB在多地域多中心部署时容灾与本地读写支持较好，但全局写延迟高；TiDB生态成熟且支持混合HTAP（OLTP+OLAP），但架构较重、组件多，维护成本高。

### Card 7: Kafka vs Apache Pulsar (高性能事件流抉择)
*   **核心原理**: Kafka采用计算与存储合一的单文件追加写与PageCache零拷贝架构；Pulsar采用计算与存储分离（Broker与BookKeeper）的三层物理部署架构。
*   **技术细节**: Kafka分区物理绑定到指定Broker的Segment日志文件上；Pulsar将分区抽象为分段（Ledgers），由BookKeeper分布式节点（Bookies）分散存储，Broker仅作无状态分发。
*   **折中与防范**: Kafka架构极简、吞吐率极高，但分区数量上限受限于Broker文件句柄，且扩容需重平衡移动数据；Pulsar支持海量分区和即时水平扩容，但系统链路深、运维成本极高。

### Card 8: RocketMQ vs RabbitMQ (企业级队列抉择)
*   **核心原理**: RocketMQ是基于CommitLog单文件混写与高并发拉取模式的Java企业级消息队列；RabbitMQ是基于AMQP协议和Erlang并发Actor模型的推模式消息队列。
*   **技术细节**: RocketMQ所有Topic写入同一个CommitLog，通过ConsumerQueue二级索引拉取消息，支持分布式事务消息；RabbitMQ通过Exchange路由键转发消息至Queue，利用Erlang信道和内存队列向消费者Push数据。
*   **折中与防范**: RocketMQ适合金融级事务、海量堆积与延迟队列，但消费端基于拉模式有空轮询开销；RabbitMQ实时延迟极低、路由灵活，但堆积能力较弱且Erlang语言门槛较高。

### Card 9: Redis Streams vs MQTT (轻量/物联网队列抉择)
*   **核心原理**: Redis Streams是内存级自增只追加日志消息流；MQTT是专为物联网无线传感器设计的轻量级发布订阅传输协议。
*   **技术细节**: Redis Streams使用Radix Tree存储消息ID，支持Consumer Groups and ACK确认机制；MQTT基于TCP长连接，通过Broker分发主题，支持QoS 0/1/2 三种消息递交保证等级。
*   **折中与防范**: Redis Streams适合微服务内部轻量级事件分发，但受限于内存容量且不支持复杂协议扩展；MQTT是海量物联网设备连接的首选，但后台Broker的集群扩容与状态一致性维护成本高。

### Card 10: Event Sourcing vs Event-Driven (事件流转抉择)
*   **核心原理**: Event Sourcing将应用的所有状态变更记录为一系列不可变的事件流，状态通过重播事件计算得出；EDA则是子系统通过发布/订阅离散事件进行解耦协作。
*   **技术细节**: Event Sourcing依靠Event Store存储全量历史变更，使用快照进行数据状态加速重建；EDA使用常规中间件路由消息，消费完即物理删除，无需留存全部事件轨迹。
*   **折中与防范**: Event Sourcing能实现完美的审计回溯与时间旅行，但数据量随时间膨胀严重且Schema演进复杂；EDA设计灵活直观，但由于事件无序性或丢失可能引发数据不一致风险。

### Card 11: Push vs Pull Consumer (消费者驱动模式)
*   **核心原理**: Push模式由服务端主动向消费者推送消息，消费者处于被动接收状态；Pull模式由消费者根据自身处理能力主动向服务端发起拉取请求。
*   **技术细节**: Push模式通过网关维持长连接信道或HTTP/2 Server Push推送，实时性强但易打爆消费端缓冲；Pull模式通过维持拉取长轮询来平衡低延迟与消费速率。
*   **折中与防范**: Push模式延迟极低，但当消费者慢时需要服务端做复杂的流量控制与挂起；Pull模式让消费者拥有主动权，易于批量合并拉取，但需要妥善处理无消息时的空轮询资源开销。

### Card 12: Partition-based vs Hash Ring (分片路由抉择)
*   **核心原理**: Partition Sharding将消息根据Key哈希路由到物理分区的固定段中；Hash Ring Sharding（一致性哈希环）将消息与节点都映射到统一的虚拟环形哈希空间。
*   **技术细节**: Partition Sharding分区数固定，动态增加分区需要对数据重新哈希和物理迁移；一致性哈希环通过在环上顺时针查找节点路由，支持节点扩缩容时仅漂移部分Key。
*   **折中与防范**: Partition Sharding在静态拓扑下路由速度快、顺序消费保证极强，但不利于频繁弹性伸缩；一致性哈希环弹性高但存在负载倾斜，必须引入虚拟节点。

### Card 13: RESTful HTTP/1.1 vs gRPC HTTP/2 (传输协议抉择)
*   **核心原理**: RESTful默认使用HTTP/1.1的文本JSON传输，通过短连接或Keep-Alive复用；gRPC基于HTTP/2，使用二进制Protobuf序列化，实现强类型契约长连接。
*   **技术细节**: RESTful通过URI和HTTP动词表达语义，无内建Schema约束；gRPC支持双向流控制，通过`.proto`生成多语言桩代码，依靠HTTP/2多路复用彻底消除头部阻塞。
*   **折中与防范**: RESTful对浏览器极其友好，生态开放易调试，但冗余标头大、序列化开销高；gRPC在内部微服务网络性能极高，但对外部Web浏览器不直观且调试相对复杂。

### Card 14: GraphQL vs RESTful API (网关API抉择)
*   **核心原理**: GraphQL是单端点的声明式查询语言，允许客户端指定所需字段；RESTful是多端点面向资源的传统API设计模式。
*   **技术细节**: GraphQL通过Schema定义数据图谱，客户端使用Query DSL单次请求获取关联对象；RESTful对每个资源暴露独立端点，获取关联数据往往需要发起多次物理请求。
*   **折中与防范**: GraphQL有效防止Over-fetching并减少网络开销，但后端解析复杂，易引发N+1查询问题；RESTful简单稳定，容易在网关或CDN级别执行HTTP缓存优化。

### Card 15: WebSockets vs Server-Sent Events (实时推送抉择)
*   **核心原理**: WebSockets提供双向、全双工的长连接通信通道；Server-Sent Events是基于常规HTTP的单向、从服务器到客户端的事件流推送。
*   **技术细节**: WebSockets在TCP握手后升级协议，不再遵循常规HTTP报文；SSE使用标准HTTP，标头为`text/event-stream`，支持自动断线重连。
*   **折中与防范**: WebSockets适合实时协同、双向高频交互游戏，但穿透防火墙难且对网关层有负载要求；SSE开销低、更符合常规HTTP生态，适合实时通知、大模型流式响应输出。

### Card 16: Protocol Buffers vs JSON vs Avro (数据编码抉择)
*   **核心原理**: JSON是人类可读的无模式自解释文本格式；Protobuf是带有显式Tag编号的二进制紧凑编码；Avro是结合动态Schema与纯二进制载荷的免代码生成序列化格式。
*   **技术细节**: JSON在运行时需要昂贵的字符串扫描与解析；Protobuf将字段名映射为整数Tag并使用Varint变长整型编码；Avro在反序列化时必须读取或传递Schema JSON定义进行解析。
*   **折中与防范**: JSON极易调试与跨语言交互，但体积大、解析慢；Protobuf性能好、类型安全，但需维护契约文件；Avro适合大数据海量存储，但较依赖生态上下文传输。

### Card 17: Dubbo vs gRPC (微服务RPC框架抉择)
*   **核心原理**: Dubbo是面向Java生态、提供完整服务治理与多协议支持的分布式RPC框架；gRPC是Google开源的基于Protobuf契约的通用低层多语言RPC框架。
*   **技术细节**: Dubbo支持Zookeeper/Nacos注册中心，内建动态代理、路由负载均衡与降级策略；gRPC原生只提供底层的传输通道，服务治理需要配合Istio网格等外部系统。
*   **折中与防范**: Dubbo对Java应用开箱即用，治理能力强，但多语言互通性弱；gRPC跨语言能力极强、底层性能高，但微服务治理的附加组件配置较重。

### Card 18: Kong vs Envoy (API网关选型)
*   **核心原理**: Kong是基于OpenResty (Nginx + Lua) 构建的经典七层反向代理API网关；Envoy是基于C++开发的高性能、面向云原生的服务网格代理。
*   **技术细节**: Kong使用Lua插件生态，通过关系型数据库或声明式YAML管理路由状态；Envoy基于事件驱动和多线程设计，通过控制面动态下发xDS协议热更新配置。
*   **折中与防范**: Kong在传统网关限流、鉴权插件拓展上成熟简易；Envoy更适合作为Sidecar或Service Mesh边界网关，支持高度动态的配置热更，但运维和二开门槛较高。

### Card 19: Raft vs Paxos (强一致共识抉择)
*   **核心原理**: Raft是通过强领导者（Leader）模型进行多副本日志强行同步的共识算法；Paxos是去中心化的、基于两阶段提案（Prepare/Accept）提案凝聚的一致性共识协议。
*   **技术细节**: Raft将共识拆分为Leader选举、日志复制与安全保证，Leader拥有最新且唯一的提交日志项；Multi-Paxos支持多提案者并行提交，允许乱序确定Log位置，再通过Commit补偿一致。
*   **折中与防范**: Raft直观易懂、工程实现难度较低，但强Leader会导致局部脑裂切换时的短时间写中断；Paxos极其鲁棒且并发写入好，但工程实现难度极大，极难保证无死角正确性。

### Card 20: etcd vs ZooKeeper (分布式协调引擎)
*   **核心原理**: etcd是基于Go与Raft的现代KV存储协调引擎，面向容器云；ZooKeeper是基于Java与ZAB协议的经典树形节点分布式协调组件。
*   **技术细节**: etcd提供HTTP/gRPC API，使用MVCC存储历史版本，通过Watch机制监听版本变化；ZooKeeper使用内存树结构，提供原生的TCP长连接会话与Watcher通知机制。
*   **折中与防范**: etcd易于容器部署、性能好且天然适配K8s，但大KV值存储开销大; ZooKeeper连接管理极其强壮、生态库丰富，但JVM开销大，且ZAB协议选举切换时间较长。

### Card 21: Strong Consistency vs Eventual (一致性折中)
*   **核心原理**: 强一致性要求任何读取都能立刻返回最新的写入，系统表现得像单机一样；最终一致性允许系统状态暂时不一致，但承诺在没有新写入后数据最终收敛。
*   **技术细节**: 强一致性使用分布式锁、2PC、线性化Read或Raft ReadIndex校验；最终一致性使用Quorum读写、向量时钟或CRDT冲突解决机制。
*   **折中与防范**: 强一致性完全防止脏读和乱序，但导致写可用性降低（CAP理论中放弃A）并拉长网络延迟；最终一致性写可用性极高、延时低，但客户端需容忍读取过时数据。

### Card 22: Gossip Protocol vs Centralized (服务发现通信)
*   **核心原理**: Gossip Protocol（闲话协议）通过节点间随机对等传播来实现去中心化的状态收敛；Centralized Registry（中心化注册表）依靠所有客户端定时与中心节点进行健康探活。
*   **技术细节**: Gossip基于UDP广播或TCP随机对等交换，收敛时间与节点数对数相关；注册表客户端使用Heartbeat向服务端定时发送心跳并拉取全量实例表。
*   **折中与防范**: Gossip无单点故障、极具弹性，但网络通信包冗余度高且状态收敛存在延迟；中心注册表一致性强、路由准确，但若中心端网络切分会导致全局调度瘫痪。

### Card 23: Redlock vs ZK Ephemeral Node (分布式锁抉择)
*   **核心原理**: Redlock是基于Redis多主节点的红锁算法，依靠时间戳判定锁定；ZooKeeper锁依靠共识协议与临时节点的唯一性来独占控制权。
*   **技术细节**: Redlock需要客户端在多数独立的Redis主节点上成功执行 `SET NX` 才能获取锁；ZooKeeper锁依靠创建同名临时节点，失败者通过Watch监听前一个节点的删除事件实现排队。
*   **折中与防范**: Redlock性能极高、对Redis依赖简单，但在GC暂停或网络时钟漂移时存在双重锁定风险；ZooKeeper锁安全性极高且支持锁超时自动释放，但创建和删除节点会导致频繁的共识开销。

### Card 24: Vector Clocks vs Hybrid Logical Clocks (因果时钟)
*   **核心原理**: Vector Clocks（向量时钟）利用节点ID数组跟踪事件的绝对偏序因果关系；HLC（混合逻辑时钟）将物理时钟与逻辑计数器结合，生成单调递增的物理近似时钟。
*   **技术细节**: Vector Clocks需要每个消息携带所有节点的时钟版本，随着集群规模增大体积呈线性增长；HLC只包含物理毫秒戳与逻辑序号，数据结构极其紧凑。
*   **折中与防范**: Vector Clocks能准确捕获并发冲突，但开销随集群增长严重；HLC能够生成紧凑的因果时间戳并接近物理现实，但无法区分在同一个物理毫秒和逻辑序号内的并发冲突。

### Card 25: Sync IPC vs Async Event-Driven (服务协作选型)
*   **核心原理**: 同步IPC调用由请求方阻塞等待被调用方处理并返回结果；异步EDA调用由请求方投递消息至MQ后立即返回，消息由消费方异步处理。
*   **技术细节**: 同步IPC调用在客户端执行超时与连接池重试，易形成调用链级联雪崩；异步EDA调用利用MQ的队列解耦写入压力，通过ACK保证消息送达。
*   **折中与防范**: 同步IPC调试直观、开发逻辑符合直觉，但强耦合且可用性为各环节乘积；异步EDA极具弹性、天然消峰，但调试与事务保障极难。

### Card 26: Sidecar Service Mesh vs Proxyless (网格治理选型)
*   **核心原理**: Sidecar网格通过拦截应用的网络流量并将其交由旁路代理处理；Proxyless通过集成gRPC SDK直接与服务发现控制面交互，应用自理网络逻辑。
*   **技术细节**: Sidecar网格使用iptables/eBPF劫持套接字流，对应用完全透明；Proxyless gRPC通过gRPC核心内置 of xDS 客户端解析来自控制面的路由规则。
*   **折中与防范**: Sidecar支持任意语言且应用零侵入，但引入了两跳网络延迟与可观的内存开销；Proxyless零网关开销、性能极佳，但限制在gRPC生态且需要应用频繁升级SDK。

### Card 27: Serverless (FaaS) vs Containerized (K8s) (计算部署选型)
*   **核心原理**: Serverless (FaaS) 采用按需触发、函数级的事件驱动执行模型，无状态且零常驻计算；Containerized (K8s) 提供常驻微服务容器实例，管理完整生命周期的微服务集群。
*   **技术细节**: FaaS在请求到达时动态拉起冷启动实例，按执行时间和请求次数精确计费；K8s通过常驻Pod池分发流量，根据HPA控制资源水位。
*   **折中与防范**: FaaS在轻量任务、闲置率高及定时脚本下成本极低，但有冷启动延迟且状态保留难；K8s适合稳定高并发、有状态服务，但需要预先支付常驻资源开销。

### Card 28: Blue-Green vs Canary vs Rolling (发布策略抉择)
*   **核心原理**: 蓝绿部署同时维护两个完整独立的生产环境（老版本和新版本），通过网关一键切换；金丝雀部署逐步将少量生产流量引入新版本，验证无误后再全量替换；滚动部署按批次替换旧实例。
*   **技术细节**: 蓝绿部署在网关级别执行域名解析或负载均衡指向的瞬间切流；金丝雀部署使用网关根据Cookie或特定比率进行流控转发；滚动部署通过 K8s ReplicaSet 的 maxSurge 和 maxUnavailable 控制更新容器数量。
*   **折中与防范**: 蓝绿部署最安全、支持一键回滚，但需要两倍的物理资源成本；金丝雀部署利于在真实生产环境小范围验证以降低事故概率，但需要微服务系统处理好新旧版本API兼容；滚动部署资源开销低但发布过程相对漫长。
