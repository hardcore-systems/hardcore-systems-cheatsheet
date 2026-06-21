# Step1-判题材-cilium.md: 基于 eBPF 的高性能云原生网络与安全网关审计大纲

本审计项目聚焦于云原生高性能容器网络与安全组件 `cilium`。我们将通过 28 张高密度核心知识卡片，深度解构 eBPF 内核加载与 JIT 编译、XDP 网卡层包处理与 tc 钩子、Conntrack 连接跟踪、NAT 转换与隧道封装、sockops 同机 Socket 缓冲区短路（SOCKMAP）、Maglev 一致性哈希与负载均衡、身份安全网络隔离策略以及 Hubble 流量观测元数据追踪。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: eBPF 网络内核加载与编译** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - eBPF 内核程序类型与生命周期、LLVM/Clang 字节码生成、libbpf 与 bpftool 内核验证安全防线、CO-RE 与 BTF 重定位机制、BPF Map 类型与哈希表布局、Cilium Agent 动态编译编译链。
*   **M2: XDP & tc-bpf 物理网络数据路径** (Cards 7–12) - `#6B8272` (Muted Sage)
    - XDP 驱动层极速数据包拦截、tc Ingress/Egress 流量控制钩子、无锁 IP/TCP/UDP 数据包头解析、Conntrack 连线跟踪 Map 机制、内核 NAT 与三层路由重写、VxLAN/Geneve 隧道封装解包。
*   **M3: sockops 套接字层旁路重定向** (Cards 13–18) - `#9C6666` (Tea Red)
    - sockops 套接字层事件挂载、SOCKMAP 描述符绑定哈希表、bpf_msg_redirect_hash 缓冲区短路、同机 Pod 本地加速折中、Socket 锁竞争与系统调用规避、Unix Domain Socket 对比。
*   **M4: Maglev 一致性哈希与负载均衡** (Cards 19–22) - `#7A7A7A` (Iron Grey)
    - Maglev 查找表构建与哈希分配、负载均衡 VIP 映射表、BPF LRU Map 会话保持、DSR (Direct Server Return) 极速回包。
*   **M5: 身份标识安全与策略决策** (Cards 23–26) - `#9A825A` (Dusty Gold)
    - 身份标识安全标签模型、IP-to-Identity 哈希表映射、L3/L4/L7 混合安全策略强力执行、IPsec 与 WireGuard 内核级加密。
*   **M6: 服务网格与系统诊断** (Cards 27–28) - `#755B77` (Muted Grape)
    - Envoy Sidecarless 网格互联架构、Hubble 流量观测与 BPF 元数据追踪。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    Cilium 的本质是在 Linux 内核的关键网络路径（XDP/tc/Socket 层）动态加载与即时编译（JIT）eBPF 字节码，用可编程内核路由与无锁哈希表（BPF Maps）全面替代传统的 iptables，实现云原生环境下容器间的高性能、安全隔离与负载均衡网络。
*   **L1 四句话逻辑**：
    1. **数据包源头拦截与分流**：当网络包刚到达网卡驱动层，未分配 sk_buff 结构体前，通过 XDP（eXpress Data Path）极速判断是抛弃、本地转发还是送入协议栈。
    2. **套接字层直接短路**：利用 sockops 捕获 TCP 握手事件，将同宿主机容器的 Socket 描述符存入 SOCKMAP，通过 BPF helper 直接打通发送端与接收端的缓冲区，免除内核协议栈全部打包解析开销。
    3. **基于身份的无锁访问控制**：摒弃 IP 级动态防火墙规则，将容器安全标签（Identity）编码进数据包元数据，利用 BPF Map 在内核层以 $O(1)$ 的哈希查找实现极其高效的安全策略检查。
    4. **零开销一致性负载均衡**：通过 BPF Map 存储服务虚拟 IP 与 Maglev 哈希环，结合会话保持 LRU Map，在内核态直接重写包头（DSR/NAT），实现完全去中心化的极速负载分发。
*   **L2 核心数据流转拓扑**：
    `物理网络包进入物理网卡` ➜ `驱动层 XDP 拦截判断` ➜ `tc 钩子进行包解析与 Conntrack 哈希查找` ➜ `检查 IP-to-Identity 哈希表获取安全标签` ➜ `符合安全策略 ➜ 重写包头 (SNAT/DNAT)` ➜ `如果是本地同机容器通信 ➜ 触发 sockops 强短路直接写入接收 Socket 缓冲区` ➜ `如果是跨机通信 ➜ 封装 VxLAN/Geneve 隧道送出网卡`

---

## 🗂️ 28 张核心知识卡片大纲
1. eBPF 内核程序类型与加载流程：各钩子函数（XDP/tc/Socket）内核入口控制。
2. LLVM/Clang eBPF 机器码生成：Clang 编译器生成静态 ELF 目标机器码限制与汇编规范。
3. libbpf 与 bpftool 内核验证机制：内核验证器静态分析、安全防线与 BPF 系统调用。
4. CO-RE 与 BTF 重定位机制：一次编译处处运行、内核结构偏移重定位与运行时类型描述。
5. BPF Map 类型与内核哈希表：BPF_MAP_TYPE_HASH 结构与高并发原子修改。
6. Cilium Agent 动态编译编译链：运行时根据容器网络配置实时编译加载 eBPF 机器码。
7. XDP 网卡驱动层极速拦截：网卡 DMA 阶段包过滤与 XDP 快速返回值（DROP/REDIRECT）。
8. tc (Traffic Control) 接口钩子：tc ingress/egress 挂载点在内核 sk_buff 阶段的控制。
9. eBPF 数据包头无锁解析：以指针算术进行 L2/L3/L4 协议头无锁偏移读取与合法性校验。
10. Conntrack 连线跟踪 Map 机制：利用双向元组哈希 BPF Map 进行 TCP 状态机追踪与超时清理。
11. eBPF 内核 NAT 与三层路由重写：校验和（Checksum）重算与内核网卡直接转发。
12. VxLAN/Geneve 隧道封装与解包：网络覆盖（Overlay）模式下的网卡层封装字节序列与内核解包。
13. sockops 套接字层事件拦截：捕获 sock_ops 连接建立，跳过 TCP 拥塞控制与打包逻辑。
14. SOCKMAP 描述符绑定哈希表：关联 TCP 连接元组与 Socket 描述符内核映射。
15. bpf_msg_redirect_hash 缓冲区短路：将发送端 sk_msg 直接写入目标 Socket 的 sk_receive_queue。
16. 同机 Pod 本地加速折中：零拷贝带来的网络安全观测死角与性能爆发。
17. Socket 锁竞争与系统调用规避：跳过内核网络栈时锁粒度减小与用户/内核态上下文切换规避。
18. Unix Domain Socket 对比：Cilium TCP 短路相比传统 IPC UDS 在复杂微服务下的开发普适性。
19. Maglev 一致性哈希算法：查找表构建算法、后端节点置换序列生成与请求哈希计算。
20. 负载均衡 VIP 映射表：SVC 虚拟 IP 在 BPF Map 中对 Real Pod IP 的映射与会话映射。
21. BPF LRU Map 会话保持：在高频网络吞吐下利用 BPF 内置 LRU Map 自动淘汰冷链接。
22. DSR (Direct Server Return) 路由：重写数据包源地址，使后端容器绕过网关直接发回客户端。
23. 身份标识安全标签模型：通过 K8s Label 解析为全局唯一 32 位 Identity，解耦 IP 变动。
24. IP-to-Identity 地址映射：基于 BPF LPM Trie Map 的 IP 段安全身份匹配。
25. L3/L4/L7 混合安全策略：网卡层 Identity 阻断与 Envoy 代理七层协议解析。
26. IPsec 与 WireGuard 加密：利用内核原生 XFRM/WireGuard 模块在 eBPF 决定路由后自动加密。
27. Envoy Sidecarless 网格架构：通过 tc-bpf 自动将 L7 流量重定向到单宿主机全局共享的 Envoy。
28. Hubble 流量观测与追踪：BPF ring buffer 抛出流量元数据、 Hubble 探针对网络延迟和抖动的遥测。
