# Step 1: 判题材 - envoyproxy / envoy (Envoy Proxy)

## 1. 题材深度分析与判定

`Envoy` 是现代云原生架构、服务网格 (Service Mesh) 以及微服务网络通信的灵魂核心。作为 CNCF 的毕业项目，它打破了传统反向代理（如 Nginx、HAProxy）以静态配置为主、重启时间长、扩展困难的局限，提出了完全由 API 驱动的动态控制平面（xDS 协议）与高度模块化的三级过滤器链 (Filter Chain) 架构。
Envoy 的底层设计是高性能 C++ 多线程工程与现代异步 I/O 的巅峰之作：其事件驱动的非阻塞事件循环 (Event Loop per Thread)、零拷贝内存管理 (Buffer/Slice)、复杂的连接池复用机制以及热重启 (Hot Restart) 时通过 Unix 域套接字无缝迁移监听描述符的物理细节，具有极高的信息密度与技术深度，完全符合“硬核技术系统工程”的标准。

## 2. 核心架构与 6 大模块划分

为契合 A4 双页 Landscape 布局，我们将本项目划分为 6 个核心技术模块，每个模块对应一个莫兰迪色块（M1 ~ M6）：

*   **M1: 非阻塞事件驱动线程模型** (Slate Blue - `#7A8B99`)
    *   单线程事件循环 (Event-loop per thread) 避免锁竞争、基于 Libevent 封装的非阻塞套接字异步 I/O、主线程 (Main Thread) 协调控制与工作线程 (Worker Thread) 处理数据分工、通过 `SO_REUSEPORT` 物理监听描述符共享、热重启进程级描述符无缝移交。
*   **M2: xDS 动态配置与控制平面 API** (Moss Green - `#7D8F7B`)
    *   动态服务发现 LDS（监听器）、RDS（路由）、CDS（集群）、EDS（端点）xDS 协议设计、配置热加载与无锁数据结构原子指针更新、控制平面 gRPC 双向流式同步状态机。
*   **M3: 过滤器链架构与插件化扩展** (Plum Rose - `#9E828A`)
    *   监听器过滤器 (Listener Filters) 解析握手元数据、网络过滤器 (Network Filters) 驱动 L4 传输（如 TCP/TLS）、HTTP 过滤器链 (HTTP Filter Chain) 双向流水线逻辑、基于 WebAssembly (Wasm) 与 Lua 的沙箱运行时可扩展性。
*   **M4: 连接池、负载均衡与集群管理** (Terracotta - `#B58A7D`)
    *   Downstream 客户端连接与 Upstream 后端连接池 (Connection Pool) 映射与复用、多元化负载均衡算法（Round Robin, Least Request, 基于 Maglev 算法的高性能一致性哈希, Ring Hash）、端点检测与集群成员权重计算。
*   **M5: 多协议转换与头部阻塞消除** (Indigo - `#5F7582`)
    *   HTTP/1.1、HTTP/2、gRPC 以及 HTTP/3 (QUIC) 双向桥接与流量翻译、连接迁移 (Connection ID)、解决基于 UDP 承载的 QUIC 协议在物理丢包时消除 HTTP/2 级头部阻塞 (HOLB) 的控制模型。
*   **M6: 弹性设计：熔断、重试与主动异常检测** (Antique Gold - `#BFA88F`)
    *   基于计数器的细粒度并发熔断器 (Circuit Breaking Limits) 保护上游、被动异常检测 (Outlier Detection) 连续错误剔除、主动健康检查 (Active Health Checking) 探测、重试预算 (Retry Budgets) 避免重试风暴、局部与全局限流机制。

---

## 3. L0 ~ L2 知识阶梯

### L0 一句话本质
Envoy 是一个高性能、低延迟的云原生反向代理与服务网关，它通过单线程事件循环消解锁开销，依托动态 xDS APIs 实现零停机控制配置热加载，采用三级过滤器流水线对多协议请求进行流式处理，并在连接池与网关边界实施精细的熔断重试与主动容灾保护。

### L1 四句话逻辑
1.  **非阻塞事件驱动与无锁多线程**：采用单核心绑定单事件循环（Event Loop）的工作线程模型，工作核间没有任何数据共享和锁竞争，网卡包通过系统 `SO_REUSEPORT` 均衡分发到各个核心上独立处理。
2.  **控制面 API 驱动与零停机动态配置**：使用以 gRPC 为传输的 xDS 协议簇动态订阅监听器、路由、集群及端点配置，内存中利用原子指针切换瞬时更新配置拓扑，实现无需重启的 0 毫秒级配置热更。
3.  **三级过滤器链与协议翻译**：请求由监听过滤器解析二层信息，通过网络过滤器处理 L4 TLS/TCP 状态，再送入 HTTP 过滤器链完成重写与安全鉴权；双向支持 HTTP/1.x, HTTP/2, HTTP/3 及 gRPC 间的任意桥接。
4.  **弹性弹性边界与主动容灾**：在集群出口处维护细粒度连接池限制以实现熔断阻断，结合连续故障时的被动剔除与主动发送空闲探测包的健康检查，拦截异常网关端点，避免服务雪崩。

### L2 核心数据流转拓扑
```
  [Downstream Client Request]
              │
              ▼ (SO_REUSEPORT distribution)
      [Listener Socket] (Handled by Worker Thread Event Loop)
              │
              ▼ (Read Packet Buffer)
   [Listener Filter Chain] (TLS / Proxy Protocol detection)
              │
              ▼
    [Network Filter Chain] (e.g. HTTP Connection Manager)
              │
              ▼
     [HTTP Filter Chain] (e.g. Auth ➜ Rate Limit ➜ Router)
              │
              ├──────────────────────────────┐ (Dynamic Config Lookup)
              ▼ (Route Match: Path / Header)  ▼
     [RDS / Routing Table Lookup] <─── [xDS Dynamic APIs (LDS/RDS/CDS/EDS)]
              │
              ▼
   [CDS / Cluster selection]
              │
              ▼ (Upstream Connection Pool)
   [EDS / Endpoint select (Maglev / Round Robin)]
              │
              ▼ (Write Outbound Buffer)
  [Upstream Server Response]
```

---

## 4. 28张卡片大纲规划

### Page 1 (线程模型、xDS 配置与过滤器链)

#### M1: 非阻塞事件驱动线程模型 (Cards 1-4)
*   **Card 1**: Thread-per-core (单核单事件循环) 架构与无锁数据处理原理
*   **Card 2**: 基于 Libevent 的非阻塞事件循环封装与定时器堆结构
*   **Card 3**: 监听套接字 `SO_REUSEPORT` 物理套接字复用与负载不均问题
*   **Card 4**: Envoy 经典热重启进程间 Socket 描述符 Unix Domain Socket 移交机制

#### M2: xDS 动态配置与控制平面 API (Cards 5-9)
*   **Card 5**: 动态发现服务 xDS (LDS/RDS/CDS/EDS) 依赖更新与最终一致性对齐
*   **Card 6**: 控制面 gRPC 双向流式订阅同步状态机与版本控制 (ACK/NACK)
*   **Card 7**: 配置热更新中原子指针 (`std::shared_ptr`) 快速切换与无锁读取
*   **Card 8**: 文件系统静态配置热加载机制与配置校验策略
*   **Card 9**: LDS/CDS 更新时的连接优雅重置与连接维持控制

#### M3: 过滤器链架构与插件化扩展 (Cards 10-13)
*   **Card 10**: Downstream 监听过滤器 (Listener Filters) TLS 握手元数据嗅探
*   **Card 11**: 网络过滤器 (Network Filters) 读写数据回调链与 TCP/TLS 链路控制
*   **Card 12**: HTTP 过滤器链 (HTTP Filter Chain) 双向流式 `decodeHeaders/decodeData` 流水线
*   **Card 13**: 插件化可扩展性：WebAssembly (Wasm) 沙箱运行时隔离与外部进程 gRPC 拦截

---

### Page 2 (集群路由、协议转换与弹性熔断)

#### M4: 连接池、负载均衡与集群管理 (Cards 14-17)
*   **Card 14**: Downstream 与 Upstream 连接池 (Connection Pool) 状态机与复用上限
*   **Card 15**: 负载均衡算法：Round Robin 与 最小活跃请求 (Least Request) 算法分配规则
*   **Card 16**: 高性能一致性哈希 Maglev 算法查表实现与节点抖动消解
*   **Card 17**: 负载报告服务 (LRS) 与多区域 (Locality) 路由优先级分配

#### M5: 多协议转换与头部阻塞消除 (Cards 18-23)
*   **Card 18**: HTTP/1.1 与 HTTP/2 物理连接复用与翻译路由机制
*   **Card 19**: gRPC 协议底层基于 HTTP/2 的 Frames 报头包装与流控制转换
*   **Card 20**: HTTP/3 基于 UDP (QUIC) 架构的快速握手与连接迁移原理
*   **Card 21**: 解决 HTTP/2 级 TCP 头部阻塞 (HOLB) 在 QUIC 多路独立流中的消除机制
*   **Card 22**: 内存管理：Envoy Buffer 链表切片 (Slice) 零拷贝结构
*   **Card 23**: 压缩卸载：Gzip/Brotli 过滤器异步压缩与网卡卸载

#### M6: 弹性设计：熔断、重试与主动异常检测 (Cards 24-28)
*   **Card 24**: 熔断器资源上限控制（最大连接数、挂起请求数、最大重试数）
*   **Card 25**: 被动异常检测 (Outlier Detection) 连续 5xx 错误与异常节点瞬时驱逐
*   **Card 26**: 主动健康检查 (Active Health Checking) 探测报文心跳控制逻辑
*   **Card 27**: 重试预算 (Retry Budgets) 算法限制与重试雪崩防御机制
*   **Card 28**: 局部速率限制 (Local Rate Limiting) 令牌桶与分布式 Redis 全局限流对比
