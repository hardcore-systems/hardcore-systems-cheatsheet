# Step 1: 判题材 - h2o / quic-http3 (H2O HTTP Server)

## 1. 题材深度分析与判定

`H2O` 是现代高性能 HTTP 服务器与代理的巅峰之作，由 Kazuho Oku 设计并主导开发。相较于传统反向代理（如 Nginx）与重量级服务网格（如 Envoy），H2O 的核心设计原则是追求极致的协议效率、更低的 CPU 开销与更少的内存复制。

H2O 内部的技术栈不仅包含了一个高度优化的自研事件循环 `h2o_evloop_t`，还集成了从零实现的模块化 QUIC 传输层实现 `quicly` 与极其紧凑的 TLS 1.3 库 `picotls`。H2O 对 HTTP/2 的底层实现包含了一个基于 O(1) 开销的依赖优先级调度树（WDRR 调度算法），能够以 16KB 分片粒度进行公平混流，彻底消除了应用层头部阻塞（HOLB）。同时，其对 HTTP/3 的实现通过 QPACK 的流式解析与双向单向控制流的精确流控，在复杂丢包环境下实现了极高的网络吞吐与抗延迟抖动表现。

上述极其硬核的技术组合（`h2o_evloop_t` 事件循环、`quicly` 传输层、`picotls` 握手、HTTP/2 调度树、QPACK 编解码与 103 Early Hints 抢跑推送），具有极高的技术密度与深度，非常符合“硬核技术系统工程”的卡片拆解与仿真标准。

## 2. 核心架构与 6 大模块划分

为契合 A4 双页 Landscape 布局，我们将本项目划分为 6 个核心技术模块，每个模块对应一个莫兰迪色块（M1 ~ M6）：

*   **M1: 自研轻量级事件循环 (h2o_evloop)** (Slate Blue - `#7A8B99`)
    *   自研事件循环结构体 `h2o_evloop_t`、基于 `epoll`/`kqueue`/`select` 的多后端动态绑定、套接字状态变更链表 (`_statechanged`)、定时器轮 (`h2o_timerwheel_t`) 与常数级 O(1) 定时器添加释放、单进程多线程工作模式下的套接字读写事件派发机制。
*   **M2: QUIC 传输层与 quicly 库集成** (Moss Green - `#7D8F7B`)
    *   QUIC 连接上下文 `quicly_conn_t` 与流状态机 `quicly_stream_t`、使用 `picotls` 完成 TLS 1.3 握手与密钥衍生、首包嗅探（First Byte Routing）与 Connection ID (CID) 路由分发、0-RTT 安全恢复与会话令牌管理、连接迁移（Connection Migration）与双向路径主动验证机制（PATH_CHALLENGE / PATH_RESPONSE）。
*   **M3: HTTP/3 协议栈与 QPACK 头部压缩机理** (Plum Rose - `#9E828A`)
    *   HTTP/3 连接承载结构 `h2o_http3_conn_t`、QPACK 编解码器设计（静态表、动态表管理）、单向控制流（Control Stream）与单向动态表同步流（Encoder/Decoder Stream）、读阻断排空控制与多流无序传输下的抗头部阻塞（HOLB）逻辑。
*   **M4: HTTP/2 依赖优先级调度树** (Terracotta - `#B58A7D`)
    *   调度树节点结构体 `h2o_http2_scheduler_node_t` 依赖树扑拓、`h2o_http2_scheduler_run` 调度器运行逻辑、基于加权赤字轮询（WDRR）的 16KB 分片混流控制、服务端启发式权重修正（优化未按标准实现的浏览器行为）、流并发度上限与流状态跃迁控制。
*   **M5: 零拷贝内存管理与 I/O 缓冲区优化** (Indigo - `#5F7582`)
    *   轻量级 I/O 向量 `h2o_iovec_t`、高效动态可扩展缓冲区 `h2o_buffer_t` 与零拷贝链表拼接、TCP 拥塞控制支持与套接字可写状态感知（`sendfile` 零拷贝文件发送）、高级 UDP 批量发送与分段卸载 GSO (Generic Segmentation Offload) 优化。
*   **M6: 高级特性与容灾优化** (Antique Gold - `#BFA88F`)
    *   103 Early Hints 的规范实现与抢跑推送、QUIC 连接迁移下 Session Ticket 热更与分发、拥塞控制模块解耦与插入（Cubic/Reno 状态演进）、优雅停机（Graceful Shutdown）时活跃流排空（Connection Draining）、突发大流量限制与恶意包过滤安全防范。

---

## 3. L0 ~ L2 知识阶梯

### L0 一句话本质
H2O 是一个极致优化的 HTTP/1/2/3 高性能服务器，它通过自研无锁 `evloop` 事件循环、基于 O(1) 的 HTTP/2 优先依赖树、以及紧密耦合的 `quicly` 传输层与 `picotls` 密码栈，在极低内存拷贝与 CPU 周期开销下实现高吞吐的网络流处理。

### L1 四句话逻辑
1.  **自研事件循环与定制定时器轮**：不依赖庞大的通用库，基于 `h2o_evloop_t` 封装轻量级多路复用，使用专门优化的 `h2o_timerwheel_t`，确保海量连接下超时的增删操作达到 O(1) 物理复杂度。
2.  **紧密耦合的模块化 QUIC/TLS 栈**：采用原生 `quicly` 作为传输引擎，并结合 `picotls` 实现 TLS 1.3 的零拷贝加解密；依靠自研的 AES-GCM 融合引擎加速数据包生成，达成极高的 HTTP/3 吞吐。
3.  **零 HOLB 的 HTTP/2 优先级调度树**：在发送端构建严格的依赖优先级树，所有连接并发流的数据均被切分为 16KB 的分片，通过加权赤字轮询进行复用发送，从根本上防止单个大文件阻塞其它敏感资源。
4.  **零拷贝 I/O 与 GSO 批量卸载**：全程使用 `h2o_iovec_t` 传递分段指针，配合 UDP 的分段卸载 (GSO) 机制，把数十个 UDP 数据包合并为一次系统调用发送，极大消除了用户态与内核态上下文切换的开销。

### L2 核心数据流转拓扑
```
  [Downstream Client Request (UDP / TCP)]
               │
               ▼
   [Network Socket Interface] (Handled by Worker Thread Evloop)
               │
      ┌────────┴────────┐
      ▼ (TCP / TLS 1.2)  ▼ (UDP / QUIC Packet)
 [h2o_socket_t]     [quicly_conn_t] (Managed by quicly engine)
      │                 │
      │                 ▼ (Decrypted by picotls / AES-GCM Fusion)
      │             [quicly_stream_t] (H3 Frame parser)
      │                 │
      └────────┬────────┘
               ▼
     [h2o_req_t / Core Handler] ─── (Matches Rules / Links Static / Proxy)
               │
      ┌────────┴────────┐
      ▼ (HTTP/2 Output)  ▼ (HTTP/3 Output)
 [h2o_http2_scheduler]  [h2o_http3_conn_t]
      │ (16KB Chunking)  │ (QPACK compression & Dynamic Table)
      ▼ (WDRR Scheduling)▼ (QUIC Stream Frame write queue)
   [h2o_socket_write] ───► [UDP GSO / sendmsg Send Out]
```

---

## 4. 28张卡片大纲规划

### Page 1 (事件循环、QUIC 传输与 HTTP/3 协议)

#### M1: 自研轻量级事件循环 (h2o_evloop) (Cards 1-4)
*   **Card 1**: `h2o_evloop_t` 的拓扑结构与基于 Epoll/Kqueue 的多后端抽象绑定
*   **Card 2**: 延迟状态变更链表 `_statechanged` 机制与读写事件非立即触发设计
*   **Card 3**: `h2o_timerwheel_t` 级联定时器轮的数据结构与常数时间增删机理
*   **Card 4**: 多工作线程模式下事件循环的套接字负载均衡与多核事件分发

#### M2: QUIC 传输层与 quicly 库集成 (Cards 5-9)
*   **Card 5**: `quicly_conn_t` 与 `quicly_stream_t` 状态机跃迁与双向回调设计
*   **Card 6**: `picotls` (TLS 1.3) 的握手集成、密钥衍生与 0-RTT 会话票据处理
*   **Card 7**: QUIC 握手阶段首包嗅探（First Byte Routing）与 Connection ID 调度分发
*   **Card 8**: 连接迁移（Connection Migration）机制与双向 PATH_CHALLENGE 验证规程
*   **Card 9**: 自研 AES-GCM Fusion 融合硬件加速引擎的零拷贝发包流程

#### M3: HTTP/3 协议栈与 QPACK 头部压缩机理 (Cards 10-13)
*   **Card 10**: `h2o_http3_conn_t` 连接层上下文结构与单向控制流状态机处理
*   **Card 11**: QPACK 编解码器的表项管理（Static Table vs Dynamic Table）与索引映射
*   **Card 12**: QPACK 单向同步流（Encoder/Decoder Stream）的互阻断检测与内存上限约束
*   **Card 13**: HTTP/3 多流乱序接收与 QPACK 延迟解析的抗头部阻塞（HOLB）保障

---

### Page 2 (H2 调度、零拷贝与安全容灾)

#### M4: HTTP/2 依赖优先级调度树 (Cards 14-18)
*   **Card 14**: `h2o_http2_scheduler_node_t` 调度树的依赖拓扑与权值继承关系
*   **Card 15**: `h2o_http2_scheduler_run` 权重遍历与 WDRR（加权赤字轮询）公平分配算法
*   **Card 16**: 16KB 数据分片分发限制与 TCP 拥塞状态下的动态混流平衡
*   **Card 17**: 服务端启发式权重调度器策略（规避早期 Chrome/Firefox 的行为缺陷）
*   **Card 18**: HTTP/2 流并发上限控制、WINDOW_UPDATE 流量滑动窗口管理

#### M5: 零拷贝内存管理与 I/O 缓冲区优化 (Cards 19-23)
*   **Card 19**: 轻量级 I/O 向量 `h2o_iovec_t` 的定义、切片截取与内存安全性
*   **Card 20**: `h2o_buffer_t` 双向链表/动态内存池的分配策略与释放重用机制
*   **Card 21**: H2O 在 TCP 层面的零拷贝优化与 `sendfile` 内核旁路发送实现
*   **Card 22**: 高级 UDP 批量发送与分段卸载 GSO (Generic Segmentation Offload) 的内核优化
*   **Card 23**: 套接字写缓冲区回压（Backpressure）机制与异步写入队列管理

#### M6: 高级特性与容灾优化 (Cards 24-28)
*   **Card 24**: 103 Early Hints 规范的流式发送控制与静态资源抢跑推送
*   **Card 25**: TLS Session Ticket 会话状态的外部存储热加载与多实例分布式同步
*   **Card 26**: Quicly 中拥塞控制算法的可插拔接口设计及 Cubic 状态转移
*   **Card 27**: 优雅停机（Graceful Shutdown）机制下的连接 Draining 与流优雅关闭
*   **Card 28**: HTTP/2 flood、恶意头部注入等 HTTP/3 拒绝服务攻击的流控防护
