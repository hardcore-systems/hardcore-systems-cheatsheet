# Step 1: 判题材 - Kurose & Ross / top-down-networking

## 1. 题材深度分析与判定

`kurose-ross/top-down-networking`（《计算机网络：自顶向下方法》）是全球最经典的计算机网络工程巨著与开源项目。它打破了传统自底向上（从物理层到应用层）的枯燥讲解，从直接面向开发者的应用层和套接字 API 出发，层层剥离协议栈，深入传输层（TCP 流量与拥塞控制）、网络层（IP 转发与 BGP 自治路由）以及链路层（多路访问协议与网络介质）。
本项目包含了网络系统设计、复杂的可靠传输状态机、拥塞控制算法（Cubic 与 BBR 的博弈对决）以及分布式路由共识算法，其信息密度和技术深度完全符合“硬核技术系统工程”的标准。

## 2. 核心架构与 6 大模块划分

为契合 A4 双页 Landscape 布局，我们将本项目划分为 6 个核心技术模块，每个模块对应一个莫兰迪色块（M1 ~ M6）：

*   **M1: 应用层协议与套接字编程** (Slate Blue - `#7A8B99`)
    *   HTTP/1.x vs HTTP/2 多路复用、DNS 递归与迭代解析查询、套接字 API 抽象（`bind`, `listen`, `accept`, `connect`）、多路复用 I/O 模型（`select`/`epoll`）。
*   **M2: 传输层原理与 TCP 可靠传输** (Moss Green - `#7D8F7B`)
    *   传输层多路复用与解复用、滑动窗口机制（回退 N 步 GBN vs 选择重传 SR）、RTT 动态估算与超时期计算（Jacobson/Karels 算法）、TCP 三次握手与四次挥手状态机。
*   **M3: TCP 拥塞控制与算法分流** (Plum Rose - `#9E828A`)
    *   经典 AIMD（加性增、乘性减）模型、慢启动与拥塞避免、快速重传与快速恢复、Cubic 算法（基于时间的三次方曲线）与 BBR 算法（基于瓶颈带宽和最小 RTT 估算）的底层演进与博弈。
*   **M4: 网络层数据面与 IP 转发引擎** (Terracotta - `#B58A7D`)
    *   IP 分组格式（IPv4 首部与 IPv6 首部转换）、最长前缀匹配算法（基于 Trie 树的硬件级查表）、路由器内部交换结构（内存、总线、横跨条交换）、ICMP 诊断协议。
*   **M5: 网络层控制面与自治系统路由** (Indigo - `#5F7582`)
    *   链路状态路由算法（Dijkstra 在 OSPF 中的实现）、距离向量路由算法（Bellman-Ford 在 RIP 中的实现）、边界网关协议 BGP（自治系统 AS 之间的路径属性与多准则路由选择）。
*   **M6: 链路层协议与多路访问控制** (Antique Gold - `#BFA88F`)
    *   差错检测编码（循环冗余校验 CRC 与奇偶校验）、多路访问控制 MAC 协议（以太网 CSMA/CD vs Wi-Fi 无线 CSMA/CA 碰撞避免）、ARP 地址解析协议、VLAN 虚拟局域网隔离与交换机自学习。

---

## 3. L0 ~ L2 知识阶梯

### L0 一句话本质
计算机网络自顶向下系统是一个以套接字为用户态边界，依靠传输层滑动窗口与动态拥塞控制提供高可靠/高吞吐传输，通过网络层最长前缀匹配查表与自治系统 BGP 协议路由转发，并在链路层利用 CSMA 多路访问解决共享介质碰撞冲突的复杂分布式通信体系。

### L1 四句话逻辑
1.  **应用抽象与套接字边界**：应用层通过统一的 Socket 套接字描述符（基于文件抽象）向下递交数据流；DNS 协议通过分布式级联查询将主机名转换为 IP 地址，为传输建立物理导航。
2.  **双端滑动窗口与超时重传**：传输层采用滑动窗口机制平衡发送速率与接收缓冲区限额；通过不断采样的物理 RTT 动态计算安全超时期（RTO），当报文丢失时触发 GBN 或 SR 重传，保证物理信道不可靠时的逻辑可靠。
3.  **拥塞控制从 Cubic 到 BBR 的演进**：传统 Cubic 以丢包作为拥塞触发信号，通过三次方曲线迅速填满网络链路缓冲区，导致排队延迟剧增；现代 BBR 算法放弃丢包驱动，改用主动测量瓶颈带宽与最小 RTT 的交点（Kleinrock 限界），实现最高吞吐与零队列积压。
4.  **分级自治路由与介质争用**：网络层在自治系统内部使用 OSPF/Dijkstra 路由，在自治系统之间使用 BGP 路径属性控制流量走向；链路层在共享总线上采用 CSMA/CD 监听碰撞，而在无线介质中则通过 RTS/CTS 与 CSMA/CA 预约信道，消除多路冲突。

### L2 核心数据流转拓扑
```
[User socket write]
    │
    ▼ (TCP Send Buffer)
[Sliding Window Filter] (GBN / SR Window Control)
    │
    ├──────────────────────────────┐ (Cubic / BBR Rate Pacing)
    ▼ (Segment Generation)         ▼
[TCP Congestion Control (cwnd)] [Pacing Rate Controller]
    │
    ▼ (IP Encapsulation)
[Longest Prefix Matching (Trie)] (Router Forwarding Engine)
    │
    ├──────────────────────────────┐ (OSPF / Dijkstra Intradomain)
    ▼ (Routing Control Table)      ▼
[BGP Path Attributes (AS-Path)] [Autonomous System Boundary]
    │
    ▼ (ARP Resolution: IP ➜ MAC)
[Link-Layer Framing (CRC)]
    │
    ▼ (CSMA/CD / CSMA/CA Media Access)
[Physical Ethernet / Wi-Fi Medium]
```

---

## 4. 28张卡片大纲规划

### Page 1 (应用层、传输层与拥塞控制)

#### M1: 应用层协议与套接字编程 (Cards 1-4)
*   **Card 1**: HTTP/1.x 串行阻塞与 HTTP/2 双向帧复用机制
*   **Card 2**: DNS 递归与迭代树形层次查询流控逻辑
*   **Card 3**: BSD Socket API 执行序列与内核底层连接映射
*   **Card 4**: 多路复用 I/O 模型（`select`, `poll`, `epoll` 边缘/水平触发）

#### M2: 传输层原理与 TCP 可靠传输 (Cards 5-9)
*   **Card 5**: 传输层解复用机制与双端端口绑定规则
*   **Card 6**: 滑动窗口回退 N 步 (GBN) 状态机与累积确认限制
*   **Card 7**: 选择重传 (SR) 乱序缓存与接收窗口锁步对齐
*   **Card 8**: RTT 动态估计与超时期 (RTO) 指数加权移动平均算法 (EWMA)
*   **Card 9**: TCP 三次握手建连与四次挥手半关闭状态转移矩阵

#### M3: TCP 拥塞控制与算法分流 (Cards 10-13)
*   **Card 10**: 经典 AIMD 锯齿波与慢启动/拥塞避免转换阀门
*   **Card 11**: 快速重传与快速恢复机制 (FRR / RFC 5681)
*   **Card 12**: Cubic 拥塞算法三次方控制曲线与公平性限界
*   **Card 13**: Google BBR 算法瓶颈带宽/最小 RTT 估算与 Kleinrock 限界控制

---

### Page 2 (网络层、自治系统与链路层)

#### M4: 网络层数据面与 IP 转发引擎 (Cards 14-17)
*   **Card 14**: IPv4 首部分片机制与 IPv6 简化流标记首部对比
*   **Card 15**: 硬件级最长前缀匹配 (LPM) 二叉 Trie 树查找算法
*   **Card 16**: 路由器内部横跨条 (Crossbar) 交换矩阵与头部阻塞 (HOL)
*   **Card 17**: ICMP 差错报文控制协议与 Traceroute 物理寻径原理

#### M5: 网络层控制面与自治系统路由 (Cards 18-23)
*   **Card 18**: 链路状态路由算法 (Dijkstra) 与 OSPF 洪泛同步
*   **Card 19**: 距离向量路由算法 (Bellman-Ford) 环路与无穷大收敛问题
*   **Card 20**: 边界网关协议 (BGP) 路径属性 (AS-Path, Next-Hop, Local-Pref)
*   **Card 21**: 自治系统 (AS) 之间多准则 BGP 决策决策树过滤规则
*   **Card 22**: 内部 BGP (iBGP) 与 外部 BGP (eBGP) 级联注入流向
*   **Card 23**: 软件定义网络 (SDN) 控制面与数据面 OpenFlow 流表解耦

#### M6: 链路层协议与多路访问控制 (Cards 24-28)
*   **Card 24**: 链路层成帧、奇偶校验与 CRC 循环冗余多项式校验
*   **Card 25**: 共享总线以太网 CSMA/CD 载波监听与指数退避冲突避免
*   **Card 26**: 无线局域网 CSMA/CA 碰撞避免、RTS/CTS 信道预约控制
*   **Card 27**: ARP 地址解析协议广播查询与缓存表动态老化
*   **Card 28**: VLAN 虚拟局域网 802.1Q 标签注入与交换机自学习算法
