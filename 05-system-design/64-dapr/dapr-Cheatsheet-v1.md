# Dapr 云原生 Sidecar 运行时系统速查卡片

## M1: Sidecar 架构与 API 拦截

### Card 1: App-to-Sidecar 物理桥接
- **通信路径**：应用程序通过本地环回网卡（localhost）与 Dapr Sidecar 进程通信。默认 HTTP 端口为 3500，gRPC 端口为 50001。
- **架构优势**：应用程序无需集成复杂的数据库、消息队列 SDK，只需使用标准 HTTP/gRPC 协议向本地 Dapr 端口发送请求即可。
- **物理折衷**：引入了一层环回网络跳转（Loopback Hop），增加了微秒级网络延迟。

### Card 2: 统一 API 抽象设计
- **抽象解耦**：Dapr 将复杂的网络模式抽象为极简 API。例如，读取状态调用 `/v1.0/state/<store-name>/<key>`，发布消息调用 `/v1.0/publish/<pubsub-name>/<topic>`。
- **透明平替**：底层数据库由 Redis 切换为 MongoDB，或者消息队列由 Kafka 切换为 RabbitMQ 时，应用端代码完全不需要修改。
- **限制约束**：由于采用最小公分母抽象，某些底层存储引擎的特有高级功能（如特定 SQL 的复杂 Join 语法）将无法直接使用。

### Card 3: 端口生命周期拦截
- **容器对齐**：在 Kubernetes 中，daprd 作为 Sidecar 容器运行在同一个 Pod 内，与主业务容器物理共享相同的网络命名空间（NetNS）。
- **同步就绪**：Dapr 的 Placement 和 Sentry 服务通过监控 Pod 生命周期，在应用端口就绪之前拦截流量，防止应用在未完全加载时接收请求。
- **配置复杂度**：配置不当会导致 Pod 启动顺序冲突，致使流量就绪探测失败。

### Card 4: Sidecar 双向进程代理
- **解耦代理**：Dapr daprd 进程与应用进程属于一对一的本地绑定关系。它不仅代理应用的出站请求，还能将订阅的事件、绑定的输入主动投递回应用监听的特定路由。
- **异常恢复**：当应用进程崩溃重启时，Sidecar 会保持运行并缓存未处理请求，应用上线后立即无缝恢复业务，极大提升了弹性容错能力。
- **物理折衷**：每个 Sidecar 独立占用约 30-50MB 内存，在大规模微服务集群下，Sidecar 进程的总内存开销相当可观。

### Card 5: 应用无感集成机制
- **零改动集成**：在 Kubernetes 部署清单中，只需为 Pod 添加 annotations（如 `dapr.io/enabled: "true"` ），Dapr Injector 就会自动注入 Sidecar 容器并重写网络拦截规则。
- **多语言适配**：无论是旧系统（COBOL/C++）还是现代化框架（Python/Go），只要支持向本地端口发送 HTTP/gRPC，就可以立即融入分布式集群生态。
- **调试复杂度**：流量被隐式重定向至本地 gRPC 端口，出现底层连接故障时，追踪工具需要透视这层隐式跳转。

---

## M2: 可插拔状态管理与分布式锁

### Card 6: 可插拔状态存储引擎
- **动态组件**：状态存储定义在外部 YAML 文件中。通过元数据（Metadata）字段指定连接参数，由 Dapr Runtime 动态加载 Redis、PostgreSQL、CosmosDB 等组件驱动。
- **生命周期解耦**：开发者在开发环境中配置本地单机 Redis 组件，在生产环境中直接替换为云端托管 DynamoDB 组件，实现了“一套代码，环境自适应”。
- **折衷约束**：状态存储必须实现 Dapr 的 `StateStore` 接口契约，放弃了特定存储的高级事务特性。

### Card 7: ETag 与并发冲突乐观锁
- **版本校验**：Dapr 状态 API 支持 ETag。在读取状态时，Dapr 会返回一个唯一的 ETag 版本号。
- **乐观写入**：应用在写入（Update/Delete）时，在请求头携带该 ETag。如果底层存储检测到其他进程在此期间修改了数据，则 ETag 会发生错位，Dapr 会返回 `409 Conflict` 错误。
- **设计折衷**：对于高并发写入的 Key，乐观锁会导致大量请求失败重试，设计上需要配合分布式队列或采用“最后写入者胜（Last-Write-Wins）”策略。

### Card 8: 状态 API 分布式事务与批量处理
- **事务批处理**：提供 `/v1.0/state/<store-name>/transaction` 端点，支持在单次网络调用中批量执行 Upsert、Delete 混合操作。
- **一致性保证**：如果底层组件支持事务（如 Redis Cluster、Postgres），Dapr 会将批量操作转化为底层原生事务执行，保证原子性；对于不支持事务的组件，则进行软事务模拟。
- **折衷限制**：跨多个不同状态存储引擎（如同时往 Redis 和 MongoDB 提交）的分布式两阶段提交（2PC）并不被支持。

### Card 9: 分布式锁 API
- **排他资源锁**：提供了统一的分布式锁 API `/v1.0/lock/<lock-store-name>`，应用可以请求抢占特定 Key 的租约，并设置生存时间（TTL）。
- **锁状态维护**：Dapr 屏蔽了底层利用 Redis Redlock 算法或 Zookeeper 临时节点的具体实现差异，为无状态应用提供极简的强排他性控制。
- **物理折衷**：底层锁存储的高可用性直接决定了业务锁的安全，锁存储如果发生脑裂，可能会引起双锁获准的问题。

### Card 10: 状态值过期与 TTL 控制
- **自动消亡**：在写入状态时，允许指定 metadata 元数据项 `ttlInSeconds`。底层驱动将该时效性注入具体的数据库记录中。
- **缓存管理**：适合存放临时性分布式 Session 数据或热点缓存，时间到期后底层组件会自动物理擦除，节约存储空间。
- **限制提示**：部分状态存储组件（如某些传统的关系型 SQL 驱动）对 TTL 的支持有限，需要 Dapr 在上游以周期扫描模拟，增加了性能开销。

---

## M3: 发布订阅与事件驱动

### Card 11: CloudEvents 标准协议封装
- **格式规范**：Dapr 发布订阅 API 强制遵循 CNCF CloudEvents v1.0 协议规范。传输的 payload 会自动被包装在含有 `id`、`source`、`type`、`specversion` 等元数据的 JSON 结构中。
- **跨平台追踪**：标准化的头部信息保证了消息在跨系统、跨网关、甚至跨云平台传输时，路由判定、溯源和分布式追踪 ID 能够完好无损地传递。
- **设计开销**：消息体包装会引入微量的 JSON 序列化和反序列化 CPU 开销，在大吞吐场景下，需要优化 JSON 解析引擎。

### Card 12: 可插拔消息代理 (Redis/Kafka/RabbitMQ)
- **统一路由**：向 `/v1.0/publish/<pubsub-name>/<topic>` 发布消息，Dapr 自动将消息路由至配置中指定的物理消息中心。
- **拓扑隔离**：应用层完全不知道 Kafka 消费者组（Consumer Group）的概念，Dapr Sidecar 会在后台自动为每个订阅的应用实例配置独立的订阅者通道。
- **一致性保障**：不同消息代理提供的送达保证（如 Kafka 的 At-least-once  vs Redis 的 Fire-and-forget）存在物理差异，应用需要针对特定后端的特性做好幂等逻辑。

### Card 13: 声明式与动态订阅注册
- **订阅配置**：支持两种订阅注册方式：声明式（配置 YAML 动态监听主题）与动态（应用启动时提供端点 `/dapr/subscribe` 供 Sidecar 扫描）。
- **极速投递**：当有新消息到达订阅主题时，Dapr Sidecar 会自动将消息以 POST 请求的形式，推送到应用对应的处理器路由（如 `/orders`）。
- **局限约束**：如果应用在处理推送请求时超时（默认 60s），Dapr 会判定投递失败，触发重试流程。

### Card 14: 弹性重试与指数退避投递
- **容错恢复**：当应用在接收事件推送返回 `500` 或网络超时，Dapr 能够根据配置的弹性策略（Resiliency）执行自动重试。
- **退避策略**：支持配置指数退避算法（Exponential Backoff），逐步拉长重试间隔，防范在应用崩溃时发起高频请求导致“雪崩效应”。
- **性能开销**：过多的重试积压会占用 Sidecar 内部的消息队列内存缓冲区，必须配置合理的重试上限。

### Card 15: 死信队列 (DLQ) 流转
- **有毒消息隔离**：当某条异常消息在达到最大重试次数后仍然无法被应用正确消费，Dapr 会自动将该消息路由至配置的死信队列（Dead Letter Queue）。
- **运维防范**：死信队列保证了“有毒消息”不会无限循环阻塞正常的消费链路，便于运维人员拉取死信进行离线诊断。
- **配置要点**：必须在 Pub/Sub 组件的 YAML 配置中显式声明死信主题（Dead Letter Topic），否则异常包将被自动丢弃。

---

## M4: Actor 模型与分布式状态机

### Card 16: 虚拟 Actor 激活与生命周期
- **状态实体**：Dapr 实现了“虚拟 Actor”模型（基于 Orleans 思想）。Actor 具有唯一的 `ActorType` 和 `ActorID`，在被调用时才会被自动加载入内存（激活，Activation）。
- **按需休眠**：当 Actor 长时间（默认 1小时）无请求交互，系统会自动回收其内存物理占用（休眠，Deactivation），并将状态持久化至状态数据库中。
- **架构设计**：完全免去了传统 Actor 框架中需要手动销毁和创建实体的复杂度。

### Card 17: Actor Placement 服务与位置路由
- **位置路由**：Dapr placement 是一个常驻的集群服务。它利用一致性哈希环（Consistent Hashing Ring）算法计算并跟踪各个 Actor 实例当前位于集群中哪一个 Dapr 实例的内存中。
- **自动漂移**：当某个持有 Actor 实例的 Dapr 节点崩溃时，Placement 服务检测到心跳丢失，会瞬间在哈希环上将该 Actor 重新定位并漂移激活到其他健康的实例节点。
- **网络开销**：Placement 成员变更需要全量广播哈希表给所有 Sidecar，高频的扩缩容可能导致短暂的路由抖动。

### Card 18: Actor 一致性状态与并发锁
- **单线程执行**：为了保证状态的一致性，同一个 Actor 实例在同一时刻只能执行一个请求。Dapr 会在内存中对所有发往该 Actor 的请求进行严格的“排队化隔离执行”。
- **状态同步**：在执行方法前后，Dapr 自动为 Actor 执行底层数据库的读取与写回，免去开发者手动加锁和提交事务的繁琐步骤。
- **并发约束**：如果某个 Actor 实例的方法执行时间过长，会导致排队队列积压，引起上游调用链严重超时。

### Card 19: Actor 定时器与提醒器 (Timer & Reminder)
- **Timer (定时器)**：生命周期随 Actor 的休眠而消亡。只在 Actor 激活驻留内存期间按指定时间间隔循环触发。
- **Reminder (提醒器)**：即使 Actor 被休眠，Reminder 信息也会安全持久化在状态存储中。时间一到，Placement 强制将对应的 Actor 重新唤醒激活并调用执行。
- **折衷约束**：Reminder 的高频轮询依赖高可用且读写性能优秀的底层状态数据库，这可能会增加底层存储的 IOPS 负载。

### Card 20: 协同调用与防死锁机制
- **重入保护**：Dapr 支持 Actor 的重入调用（Reentrancy）。在同一个调用链条中（通过 `Dapr-Reentrancy-Id` 跟踪），允许 Actor A 调用 Actor B，然后 B 再反向调用 A。
- **死锁避免**：排队执行机制默认会阻塞这种回环，开启重入后，Dapr 可以识别相同的调用链路，允许其重入执行，防止产生系统级级联死锁。
- **安全边界**：需要限制重入的最大深度，避免因无限循环递归调用导致 Sidecar 线程栈溢出。

---

## M5: 声明式绑定与外部集成

### Card 21: 输入绑定监听 (Input Bindings)
- **事件捕获**：输入绑定能够将 Dapr 挂载到外部系统（如 RabbitMQ、Twitter、MySQL binlog）的事件源上。
- **主动推送**：当外部系统产生数据变化时，Dapr Sidecar 捕获该事件，并主动将其以 HTTP POST 数据包投递给应用，实现事件驱动。
- **解耦优势**：应用端完全不需要知道外部源的驱动 API，只需提供一个接收 HTTP JSON 的接口。

### Card 22: 输出绑定触发 (Output Bindings)
- **单向触发**：向本地 Dapr `/v1.0/bindings/<binding-name>` 发送请求，Dapr 会将数据包路由并投递到指定的外部服务（如发送邮件 SendGrid、写入 AWS S3、触发 AWS Lambda）。
- **组件平替**：输出绑定隐藏了各个目标服务的特定 API。例如，将数据写入 S3 和写入 Azure Blob 的代码完全一致。
- **物理折衷**：由于移除了特定 SDK 专有的交互参数，对于部分外部服务的高级功能和细粒度配置支持不足。

### Card 23: 声明式元数据配置 (Declarative YAML)
- **声明配置**：所有的绑定和中间件组件都通过声明式的 YAML 配置文件进行统一管理。
- **安全保障**：支持在组件配置文件中通过 Secret Store（如 Kubernetes Secrets、HashiCorp Vault）以引用的形式注入敏感连接字符串，防止明文泄露。
- **配置维护**：随着微服务数量增多，组件 YAML 配置文件数量将呈线性增长，需要使用 GitOps 机制进行统一审计与自动同步。

### Card 24: 双向数据流协议管道
- **双向流通信**：绑定 API 不仅支持单次的“发布 - 接收”，对于部分长连接数据源（如 WebSocket、gRPC Stream），还提供了流式管道支持。
- **低延迟流**：允许应用与外部数据源保持双向数据交互通道，通过 gRPC 协议传输，将 Sidecar 的代理开销降至最低。
- **复杂度限制**：流式管道要求应用程序处理复杂的连接中断重连和流式状态对齐逻辑。

---

## M6: 系统安全与弹性可观测

### Card 25: 双向 TLS 身份认证 (mTLS)
- **通信加密**：Dapr daprd 边车进程之间的跨节点网络流量默认强制开启 mTLS 加密，防止局域网内的流量窃听与篡改。
- **自动轮换**：Dapr 内部的 Sentry 安全服务充当证书颁发机构（CA），在后台自动为每个 Sidecar 实例签发、轮换和更新 X.509 证书。
- **性能损耗**：TLS 握手和加解密会消耗微量的 CPU 算力，对高频、大吞吐量的内部微服务调用会有微小的性能压降。

### Card 26: SPIFFE 统一身份管理
- **规范对齐**：Dapr 使用 SPIFFE（Secure Production Identity Framework for Everyone）标准来定义微服务的身份标识。
- **格式结构**：证书中的 Subject Alternative Name (SAN) 被赋予 `spiffe://<trust-domain>/ns/<namespace>/sa/<service-account>/app/<app-id>` 的统一格式。
- **安全控制**：支持基于 SPIFFE 身份定制严苛的细粒度访问控制策略（ACL），只允许特定 Namespace 下的 app-id 访问当前资源。

### Card 27: 弹性熔断与断路器配置
- **熔断降级**：Dapr 提供了独立的弹性配置文件（Resiliency）。当检测到下游依赖服务错误率超出阈值时，自动切断调用链路（断路器状态切换为 Open）。
- **自我修复**：设定冷却期后，断路器会转为 Half-Open 状态，允许微量探测流量通过，若成功则恢复链路，若失败则继续切断。
- **架构优势**：不需要在代码中手写熔断器（如 Resilience4j 或 Hystrix），在 Sidecar 外部即可声明式调优。

### Card 28: OpenTelemetry 链路传播与链路遥测
- **链路传播**：Dapr 默认兼容 W3C Trace Context 规范。在 HTTP/gRPC 请求头中自动解析和传播 `traceparent` 及 `tracestate` 头部。
- **遥测输出**：Sidecar 能够自动生成跨越状态读写、Pub/Sub 事件分发、Actor 调用的全链路 Span 日志，并以 OpenTelemetry 协议（OTLP）推送至 Zipkin、Jaeger 或 Prometheus。
- **开发红利**：实现了零代码入侵的微服务全网“调用链透视”，大幅降低了分布式系统故障定位的难度。
