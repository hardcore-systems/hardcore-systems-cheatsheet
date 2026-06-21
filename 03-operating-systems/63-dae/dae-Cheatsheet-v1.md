# dae 内核级 eBPF 透明代理系统速查卡片

## M1: eBPF 内核钩子与网络架构

### Card 1: tc BPF 钩子与入站拦截
- **核心机制**：dae 挂载在 Linux 流量控制（Traffic Control, TC）的 clsact 队列钩子上。在网络包刚通过网卡驱动进入 Linux 网络栈（Ingress）时，直接解析包头。
- **性能优势**：在极其早期的网络处理链路过滤并重写数据包，比传统的 iptables PREROUTING 节点节省了大量的中断和协议栈栈分流算力。
- **设计折衷**：由于处于极底层，处理分片数据包时需要内核重组（Reassembly），增加了部分复杂性。

### Card 2: sockops 套接字流事件重定向
- **快速通路**：挂载在 `sockops` 内核事件钩子上，实时监听 TCP 连接建立（ESTABLISHED）、握手及状态迁移。
- **内核关联**：在 TCP 连接建立时，自动抓取并记录两端 Socket 的 IP/端口对，并物理写入 SOCKMAP 中。
- **性能红利**：实现了数据的直接路由，避开了操作系统的文件描述符查找开销。

### Card 3: XDP 驱动级过滤与丢包
- **极限防火墙**：dae 集成 XDP（eXpress Data Path）处理链路。在网卡驱动层（e1000/ixgbe等）接收网络包时瞬间执行 BPF 过滤。
- **极速丢包**：可直接返回 `XDP_DROP` 抛弃垃圾包，或者返回 `XDP_TX` 将防御流量原地弹回，不消耗任何 Linux 物理 CPU 软中断。
- **硬件限制**：XDP 强依赖网卡驱动的原生支持（Native Mode），否则会降级为 Generic Mode，此时丧失旁路提速比率。

### Card 4: SOCKMAP 映射表结构
- **寻址映射**：SOCKMAP 是 eBPF 专用的哈希表，表键为 `sock_key`（源/目的端 IP 与 Port 对），表值为对应套接字的内核 `struct sock` 物理指针。
- **物理关联**：通过将本地 TProxy 代理套接字与远端客户端套接字关联写入 map，实现了数据包的瞬间内核重定向。
- **容量折衷**：必须配置足够大的 map 槽位。写冲突会导致连接建立变慢。

### Card 5: bpf_msg_redirect 物理内存短路
- **短路通道**：当 TCP Socket A 向 Socket B 发送数据时，挂载在 `BPF_MAP_TYPE_SOCKMAP` 上的 BPF 程序触发。
- **零栈拷贝**：调用 `bpf_msg_redirect` 助手函数，将数据包从 A 的发送缓冲区（Write buffer）直接拷贝物理对倾到 B 的接收缓冲区（Read buffer），跳过 Linux TCP/IP 协议栈。
- **延迟降低**：将两端都在本地的转发延迟缩短至接近物理内存拷贝（Memory-copy）的极限速度。

---

## M2: TProxy 透明代理与内核路由

### Card 6: Linux TProxy 物理透明代理
- **无感拦截**：使用 `IP_TRANSPARENT` 套接字选项。dae 可以在不修改入站网络包目的 IP/端口的情况下，强行将 TCP/UDP 流量导向本地代理监听端口。
- **物理机理**：TProxy 在内核级别劫持连接。客户端完全感知不到代理的存在，依然以为是在与远端服务器直接通信。
- **设计折衷**：TProxy 依然需要建立内核套接字，会有一定的系统文件描述符（FD）资源开销。

### Card 7: 策略路由与 ip rule 表项
- **路由重定向**：在 Linux 策略路由中创建独立表（如表号 100）。
- **拦截逻辑**：添加 `ip rule add fwmark 1 lookup 100` 表项，将打上了特定防火墙标记（Mark）的数据包强制重定向至 loopback（lo）接口。
- **运行保障**：如果策略路由配置丢失，数据包将直接按普通出口发送，从而导致透明拦截失效。

### Card 8: conntrack BPF 状态表维护
- **连接跟踪**：传统 Linux conntrack 存在严重的并发自旋锁（spinlock）瓶颈。dae 在 BPF 内部实现了一个自定义的 conntrack 哈希映射表。
- **状态流转**：记录和追踪每一个流的会话状态（SYN, ACK, FIN, RST）。
- **性能红利**：完全免除了 Linux 内核 conntrack 模块的锁抢占，大幅提升了多核大并发连接下的新建连接速率。

### Card 9: NetNS 跨命名空间路由
- **容器代理**：在云原生和容器化场景下，各容器位于独立的网络命名空间（NetNS）中。
- **跨界转移**：dae 可以通过在 veth pair 接口上挂载 eBPF，实现包从容器 NetNS 到宿主机物理 NetNS 的内核级跨界转移，零网桥拷贝开销。
- **设计成本**：需要在启动时扫描宿主机上所有的 veth 接口并动态插入 BPF 实例。

### Card 10: 本地环形网卡短路加速
- **回环短路**：经过策略路由打标送入 lo（loopback）接口的 TProxy 流量。
- **短路提速**：eBPF 监控 lo 网卡上的流量，一旦发现是发往代理端口的数据，立即在 Ingress 处短路，跳过回环网卡的虚拟 IP 层解析。
- **性能优势**：将 TProxy 代理转发的背景延迟平抑 15% 以上。

---

## M3: DNS 劫持与分流路由

### Card 11: DNS 本地伪装与劫持
- **拦截抓取**：dae 监控 Ingress 流量中的 53 端口（DNS 查询）。一旦捕获到 A/AAAA 记录查询，BPF 程序直接进行就地拦截。
- **快速响应**：对于代理域名，dae 可以在内核态直接构造合法的 DNS 响应包并返回 Fake IP，耗时控制在数十微秒。
- **折中限制**：由于是在内核态原位修改数据包，协议头的 IP/UDP 校验和（Checksum）必须重新计算，对网卡的校验和卸载（CSUM Offload）有一定依赖。

### Card 12: Prefix Tree 域名分流规则
- **高效分流**：dae 支持百万级的域名分流黑白名单。在内存中，这些域名被编译为前缀树（Prefix Tree / Trie）结构。
- **前缀匹配**：在收到 DNS 查询时，利用域名 Trie 树快速检索判定该域名属于“直连（Direct）”、“代理（Proxy）”还是“拦截（Block）”。
- **性能损耗**：大规则下的 Trie 树检索有微量 CPU 消耗。Sled 建议在匹配成功后，将结果写入 IP Set 以免二次检索。

### Card 13: IP Set 动态缓存
- **IP 映射表**：当某个域名被判定为代理路由并解析出 IP 时，该目标 IP 被动态插入到 BPF 的 `ip_set` map 中。
- **查表直达**：后续该 IP 发起的 TCP 握手数据包，直接查 `ip_set` map 即可得知路由策略，免去了复杂的域名解析和比对判定。
- **自愈清理**：map 表项配置了 TTL，过期自动清除以保证 IP 集合的时效性。

### Card 14: Fake IP 物理分流引擎
- **Fake IP 分配**：对于代理域名，dae 从特定的保留网段（如 `198.18.0.0/15`）中动态分配一个虚拟 Fake IP 返回给客户端。
- **映射管理**：在内存中维护一个双向映射表：`Fake IP <-> Real Domain`。
- **并发优势**：客户端向 Fake IP 发起 TCP SYN，dae 在内核感知该 SYN 后，瞬间还原出真实域名，从而做出精准的路由代理分流决策。

### Card 15: IPv4/IPv6 双栈解析与 AAAA 路由
- **双栈管理**：在 DNS 拦截中，同时解析 IPv4 A 记录和 IPv6 AAAA 记录。
- **解析对齐**：为 IPv6 的 AAAA 查询返回对应的 IPv6 Fake IP。
- **系统限制**：需要宿主机及网络物理链路完全支持 IPv6 转发，否则 IPv6 路由可能会由于路径 MTU 或网关不支持而遭遇黑洞丢包。

---

## M4: 协议处理与隧道封装

### Card 16: TCP/UDP 数据流内核拼接
- **零拷贝拼接**：在代理客户端发送数据到远端代理服务器时，dae 通过 BPF stream parser 将客户端 TCP 段与代理隧道（如 VLESS）的协议头在内核态原位拼接。
- **开销削减**：消除了传统的“接收数据包到用户态 -> 用户态拼接隧道协议头 -> 再发送到内核”的二次拷贝损耗。
- **物理折衷**：增加了 BPF 字节码的指令数。如果超出内核指令限制，需要拆分为 tail call 尾调用。

### Card 17: 多网卡出接口物理选择
- **多接口分流**：当宿主机存在多个物理网卡（如 eth0 宽带、wlan0 无线）时，dae 可以根据路由表和分流策略。
- **直接出站**：使用 `bpf_redirect` 助手函数，将打标的出站数据包强行发送到指定的物理网卡接口，绕过系统内核的路由决策层。
- **性能红利**：实现了完全由策略驱动的极速物理接口分流，没有网卡切换延迟。

### Card 18: 隧道协议物理封装 (VLESS/VMess/Trojan)
- **协议栈整合**：dae 原生支持主流代理隧道的物理打包。
- **加密封装**：使用内置的轻量级加密库，在内核旁路计算出 VLESS/Trojan 头部元数据后，封装进发往代理服务器的物理 TCP 报文中。
- **吞吐优势**：通过批量封装，提升了网络吞吐，降低了单数据包的加密打包开销。

### Card 19: UDP Over TCP 物理转接
- **防拥塞降噪**：部分宽带运营商会对公网的 UDP 流量进行严重限速。
- **转接转换**：dae 可以将本地截获的 UDP 数据包自动打包进一条持久化的 TCP / HTTP/3 隧道中物理发送，在对端再还原为 UDP。
- **物理折衷**：UDP over TCP 在遇到丢包时，会因为 TCP 的超时重传产生延迟抖动（HOL Blocking），不适合实时竞技游戏。

### Card 20: MTU 大小分片与内核 GSO 卸载
- **分片优化**：隧道封装会占用部分包字节（如 40 字节头），这可能导致网络包超过物理网卡的 MTU（1500）而引发物理分片。
- **GSO 卸载**：dae 配合 Linux 的 Generic Segmentation Offloading (GSO) 机制，允许生成超大包传递，在网卡驱动层才物理切片。
- **设计折衷**：如果网卡不支持 GSO 卸载，会导致严重的 CPU 软中断处理损耗，需要在启动时对出站 MTU 进行动态规整。

---

## M5: Go/C 混合编程与管理

### Card 21: cgo 双语言物理绑定
- **架构设计**：dae 采用 Go 作为控制面与用户态代理管理层，C 语言开发 eBPF 内核层字节码。
- **cgo 加载**：Go 控制端使用 `cilium/ebpf` 库在启动时读取编译好的 ELF 字节码，并通过系统调用加载到 Linux 内核中。
- **运行保障**：Go 垃圾回收（GC）不能波及内核 eBPF map，必须使用固定物理内存的 Go 指针映射（Pinned Maps）。

### Card 22: 配置无缝热更新
- **配置切换**：用户修改代理规则或域名黑名单后，无需重启 dae 进程。
- **Map 热更**：Go 进程读取新配置，直接通过 syscall 增量修改内核中的 Trie / IP Set BPF Maps。
- **零中断红利**：整个更新过程对前台活跃连接完全无感，零包丢弃，无网络连接闪断。

### Card 23: 命令行控制 RPC 接口
- **管控控制**：dae 启动 Unix Domain Socket，暴露 RPC 服务。
- **CLI 管控**：`dae-cli` 可以通过 RPC 读取当前的 eBPF 运行指标、分流状态以及当前 active 连接列表。
- **折中优化**：Unix Socket 相比 TCP 接口更安全，杜绝了网络测对 dae 管理接口的越权读取。

### Card 24: Cgroup 层流量拦截绑定
- **精准拦截**：可以挂载在 Linux Cgroupv2 上。
- **限定容器**：仅对属于特定 Cgroup（如某个 Docker 容器或特定 systemd 服务）的进程流量执行拦截，实现细粒度的应用级透明代理。
- **设计限制**：要求宿主机必须开启 Unified Cgroupv2 层次结构，老旧的 Cgroupv1 无法支持此特性。

---

## M6: 故障诊断与内核监控

### Card 25: eBPF Ring Buffer 日志遥测
- **高性能日志**：BPF 运行期的调试和拦截日志（如“Matched Proxy Rule”）采用 `BPF_MAP_TYPE_RINGBUF` 传输。
- **零锁流式**：Ring Buffer 是内核与用户态 Go 进程共享的单向环形内存缓冲区，支持无锁、高吞吐的流式日志投倾。
- **运行防范**：如果用户态读取太慢，Ring Buffer 满会导致内核态丢弃后续日志，需要及时扩容 buffer size。

### Card 26: bpftool 内核段编译诊断
- **反编译诊断**：`bpftool` 是调试 dae 底层问题的终极诊断工具。
- **命令字典**：
  - `bpftool prog show`：查看 dae 挂载的各个 BPF 字节码程序的 ID 和运行指令数。
  - `bpftool map dump`：直接倾倒当前内核 BPF Trie 树或 conntrack 表的内存数据。
- **诊断用途**：当分流发生错误时，用 bpftool 查看内核 map 的真实状态，直接判定是 Go 端写入错误还是 C 端解析逻辑偏离。

### Card 27: trace_pipe 调试打印过滤
- **内核打印**：在 eBPF 代码中使用 `bpf_trace_printk` 将关键变量（如 `ip_src`、`fwmark`）输出到系统 Trace 通道。
- **过滤指令**：运行 `cat /sys/kernel/debug/tracing/trace_pipe` 查看日志输出。
- **折中限制**：全量 printk 会产生巨大的 CPU 性能损耗，在 Release 版本中必须关闭所有调试打印输出。

### Card 28: 内核性能压降优化字典
- **Map 裁剪**：eBPF Maps（如 conntrack 表）占用的空间在加载时就静态分配。
- **优化折衷**：较大的 Map 大小能防范高并发下的哈希冲突和项溢出，但会占用不可回收的 Linux 内核物理 RAM（Pin 内存）。
- **优化字典**：应根据实际内存容量（如 512MB 的路由器 vs 64GB 的服务器），微调 `max_entries` 限制，确保系统整体不发生内核级内存耗尽。
