# Step1-判题材-dae: 基于 eBPF 的高性能内核层透明代理

## 1. 题材重要性与壁垒分析
dae 是一款基于 eBPF 技术的高性能 Linux 内核层透明网关/透明代理程序。它颠覆了传统的 iptables/nftables 用户态流量重定向方案。通过在 Linux 内核流量控制（TC）和套接字（Socket）层直接挂载 BPF 字节码，实现了协议栈旁路物理短路，将透明代理的吞吐性能拉升至内核线速，彻底打穿了高并发下软中断（softirq）高 CPU 消耗及多余内存拷贝的物理壁垒。

## 2. 知识模块组织 (M1-M6) & 配色映射
- **M1: eBPF 内核钩子与网络架构 (M1: #4B5F7A)** - tc BPF 钩子入站拦截、sockops 套接字重定向、XDP 驱动级过滤、SOCKMAP 映射表、bpf_msg_redirect 内存短路。
- **M2: TProxy 透明代理与内核路由 (M2: #6B8272)** - IP_TRANSPARENT 选项、ip rule 策略路由表、conntrack 状态表、NetNS 跨空间路由、本地环形短路加速。
- **M3: DNS 劫持与分流路由 (M3: #9C6666)** - DNS 本地伪装劫持、Prefix Tree 域名分流规则、IP Set 动态缓存、Fake IP 映射分流引擎、双栈 AAAA 路由。
- **M4: 协议处理与隧道封装 (M4: #7A7A7A)** - TCP/UDP 内核流拼接、多网卡出接口选择、隧道协议物理封装（VLESS/VMess）、UDP Over TCP 转接、MTU 片段 GSO 卸载。
- **M5: Go/C 混合编程与管理 (M5: #9A825A)** - cgo 物理绑定、配置无缝热更新、命令行控制 RPC 接口、Cgroup 层流量拦截绑定。
- **M6: 故障诊断与内核监控 (M6: #755B77)** - Ring Buffer 日志遥测、bpftool 编译图谱诊断、trace_pipe 过滤拦截、内核性能压降优化字典。

## 3. J-Ladder 阶梯设计逻辑
- **L0: 内核流拦截** (M1-M2) - Linux TC / sockops / XDP 内核钩子拦截，SOCKMAP 寻址，ip rule 环回重定向。
- **L1: 协议分流层** (M3-M4) - Fake IP 映射引擎，TProxy 拦截，DNS 本地劫持，VLESS/Trojan 隧道封装。
- **L2: 控制与维护** (M5-M6) - cgo 双语言装载，配置热重载，ring buffer 遥测，bpftool 字节码反编译诊断。
