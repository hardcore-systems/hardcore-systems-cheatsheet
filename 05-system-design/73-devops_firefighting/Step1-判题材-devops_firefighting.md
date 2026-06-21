# Step1-判题材-devops_firefighting.md: 云原生与 SRE 生产救火手册审计大纲

本审计项目聚焦于云原生运维与可观测性生产调优故障排查方法论 `DevOps & SRE Firefighting`。我们将通过 28 张核心知识卡片，解构 Kubernetes 容器故障、网络/DNS 与网关排错、基础设施即代码 (IaC) 与云存储安全、可观测性 TSDB 饱和治理、高可用兜底与灾难恢复等模块。建立 L0-L2 阶梯模型，并 design 双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: Kubernetes 容器运行时与故障排查** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - Container OOMKilled (Exit Code 137)、Pod CrashLoopBackOff 指数退避、Pod Pending 调度受阻根因、ImagePullBackOff 故障、临时存储超限驱逐、内存/磁盘压力驱逐自愈。
*   **M2: Linux 网络、DNS 与网关排错** (Cards 7–12) - `#6B8272` (Muted Sage)
    - CoreDNS 解析延迟与 ndots search 冗余、TCP SYN Flood 攻击与 Backlog 队列打满、Connection Reset by Peer (RST)、Nginx 502 Bad Gateway、Nginx 504 Gateway Timeout、Cilium eBPF 网络重定向失败。
*   **M3: 基础设施即代码 (IaC) 与云存储安全** (Cards 13–18) - `#9C6666` (Tea Red)
    - Terraform 状态冲突锁死 (State Lock)、PV/PVC Mount 卡死故障与 CSI 锁死、云端元数据服务 (IMDSv2) 访问限制、VPC Peering 路由冲突与不对称路由、Vault 动态密钥引擎租约失效、云存储 Bucket 越权公开事件防御。
*   **M4: 监控、APM与可观测性实战** (Cards 19–24) - `#7A7A7A` (Iron Grey)
    - Prometheus TSDB 写入路径高基数瓶颈、ELK/Fluentd 收集端背压卡死与队列饱和、分布式链路上下文传递丢失、Grafana 大时间范围慢查询与 TSDB 扫描优化、Alertmanager 告警合并去重、SRE 错误预算烧钱率告警。
*   **M5: 应急预案与高可用兜底** (Cards 25–28) - `#9A825A` (Dusty Gold)
    - 混沌工程注入网络丢包与 CPU 限制、服务分级降级与特性开关 (Feature Flag) 兜底、数据库秒级 PITR 崩溃恢复、SRE 无指责复盘 (Blameless Post-Mortem) 规范。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    SRE 与云原生救火的本质是在复杂的容器化分布式环境中，通过精确划定资源硬界限（CPU/Memory/IO）、理顺网卡/DNS跳转拓扑、缩短故障扩散半径，实现高并发基础设施的快速止血与稳定性自愈。
*   **L1 四句话逻辑**：
    1. **容器边界隔离**：依据容器退出状态码与 Kubelet 驱逐事件，强制限定临时存储与物理内存配额，守护节点 OS 稳定。
    2. **网络短路诊断**：抓包分析 RST 与 502/504 阻断，合并 `ndots` 解析跳转，收敛 DNS 重复查询瓶颈。
    3. **存储与路由收敛**：通过物理 Detach 强制解脱 CSI 独占锁，排查 VPC Peering 路由重叠，防范 IaC 状态脏写。
    4. **可观测性赋能**：治理 TSDB 时间序列 Label 爆炸，运用 SLI/SLO 错误预算烧钱率进行预测性告警，实现科学救火。
*   **L2 核心数据流转拓扑**：
    `流量涌入网关` ➜ `触发 502/504 超时警告` ➜ `eBPF monitor 拦截丢包` ➜ `DNS ndots 进行 5 次 search 查询` ➜ `CoreDNS 压力报警` ➜ `Prometheus TSDB 序列基数打满` ➜ `SRE 错误预算烧钱率预警` ➜ `触发 HPA 副本扩容` ➜ `新 Pod Pending (Node 资源打满)` ➜ `Kubelet 触发驱逐 (Guaranteed 级别 Pod 留存)` ➜ `服务分级降级/Feature Flag 降级非核心依赖` ➜ `Blameless Timeline 回溯止血`

---

## 🗂️ 28 张核心知识卡片大纲
1. K8s 容器 OOMKilled 根因与定位：Exit Code 137 的内核 OOM Killer 机制与 limits 内存配额精调。
2. K8s Pod 启动 CrashLoopBackOff 排错流：Logs --previous 查看、入口点执行权限校验与退出码审计。
3. Pod Pending 调度受阻根因排查：Scheduler 资源约束判定、污点 (Taints) 与容忍度 (Tolerations) 冲突修复。
4. K8s 运行期 ImagePullBackOff 故障排错：私有库 imagePullSecrets 配置、网络超时与 Docker Hub 限频规避。
5. 临时存储超限导致的容器被驱逐：ephemeral-storage 资源限制、logrotate 配置与 emptyDir 容器逃逸。
6. Pod 内存/磁盘压力驱逐 (Evicted) 自愈：Kubelet 驱逐阀值设定与 QoS 级别 (Guaranteed/Burstable/BestEffort)。
7. CoreDNS 解析延迟与 ndots search 冗余：K8s dnsConfig 微调、ndots:1 对比 ndots:5 的 CoreDNS 吞吐减压。
8. TCP SYN Flood 攻击与 Backlog 队列打满：半连接与全连接队列排查，调优 tcp_syncookies 与 somaxconn。
9. Connection Reset by Peer (RST) 根因排查：TCP 状态异常分析、conntrack 映射表打满与超时时间调优。
10. Nginx 502 Bad Gateway 经典故障排错：应用连接被拒绝、后端套接字监听失效与 FPM 进程上限排查。
11. Nginx 504 Gateway Timeout 超时故障排错：后端执行超时、proxy_read_timeout 设定与慢 SQL 卡死治理。
12. Cilium eBPF 容器网络重定向失败排查：SOCKMAP 内存重定向路径、网络安全策略阻止与 Cilium Monitor 分析。
13. Terraform 状态冲突锁死 (State Lock)：S3/Consul 后端锁死错误修复、force-unlock 与 tfstate 差异对账。
14. PV/PVC Mount 卡死故障定位：CSI 独占锁 Multi-Attach 报错、存储卷物理 Detach 挂载与日志排错。
15. 云端元数据服务 (IMDSv2) 访问限制：SSRF 安全隔离、HTTP PUT Token 认证与容器 Hop Limit 跳数打满。
16. VPC Peering 路由冲突与不对称路由：CIDR 重叠网段冲突、路由表优先级冲突与 Reachability Analyzer 可达性分析。
17. Vault 动态密钥引擎租约失效灾难修复：Lease TTL 超时数据库账号销毁、自动 Renew 守护进程与灾备账号替换。
18. 云存储 Bucket 越权公开事件防御：Block Public Access 限制、Bucket Policy 越权审计与预签名 URL 限时保护。
19. Prometheus TSDB 写入路径高基数瓶颈：高维 Label 时间序列爆炸、卡性查询分析与 Metric Relabel 降维。
20. ELK/Fluentd 收集端背压卡死与队列饱和：ES 磁盘慢写入 429 报错限制、收集端 Disk Queue 缓存与限流保护。
21. 分布式链路上下文传递丢失与 Traceparent 剥离：Trace 链条断裂排查、W3C Trace Context 头透传与异步线程传播。
22. Grafana 大时间范围慢查询与 TSDB 扫描优化：$__interval 动态步长计算、Recording Rules 预先聚合调优。
23. Alertmanager 告警抖动、合并与去重调优：group_wait/group_interval 配置、告警收敛合并与 Flapping 抑制。
24. SRE 错误预算烧钱率告警设计：SLI/SLO 燃尽速度预测、双窗口多比例烧钱率告警规则编写。
25. 混沌工程注入网络丢包与 CPU 限制：Chaos Mesh 挂载网络延迟注入、stress-ng 压测与 HPA 弹性响应验证。
26. 服务分级降级与特性开关 (Feature Flag) 兜底：核心依赖梳理、轻量化 fallback 返回与 Feature Flag 动态熔断。
27. 数据库秒级 PITR 崩溃恢复校验：Postgres 增量 WAL 归档回放、时间戳恢复点指定与备用实例校验。
28. SRE 无指责复盘 (Blameless Post-Mortem) 规范：故障 Timeline 精准回溯、5 Whys 深度分析与 Action Items 闭环。
