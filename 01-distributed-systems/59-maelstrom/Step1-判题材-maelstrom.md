# Step1-判题材-maelstrom: 分布式系统一致性与容错测试床

## 1. 题材重要性与壁垒分析
Maelstrom 是由 Jepsen 团队（Kyle Kingsbury）开发的开源分布式系统测试工具，基于 Clojure 并利用 JVM 运行时仿真网络拓扑。它是评估分布式一致性协议（如 Raft、Paxos）和存储模型（如 Causal Consistency、Linearizability）在面临网络丢包、延迟、分区及节点宕机等故障时，是否真正满足安全约束的黄金标准。

## 2. 知识模块组织 (M1-M6) & 配色映射
- **M1: Clojure 运行层与内核拓扑 (M1: #4B5F7A)** - JVM 运行时、标准 I/O 协议、节点通信与 RPC 编解码。
- **M2: Jepsen 核心引擎与测试框架 (M2: #6B8272)** - Knossos 线性一致性校验、Elle 事务检测器与事件流。
- **M3: 一致性模型与定理约束 (M3: #9C6666)** - 因果一致性、读未提交/已提交、可重复读与串行化一致性。
- **M4: 分布式共享存储与服务 (M4: #7A7A7A)** - Lin-KV、Seq-KV、Lww-Register 与 PN-Counter。
- **M5: 网络混沌注入与状态转换 (M5: #9A825A)** - 网络分区、丢包、时延及 Nemesis 故障引入器。
- **M6: 诊断调试与测试调优 (M6: #755B77)** - History 日志分析、Gnuplot 时延吞吐图与命令行调优。

## 3. J-Ladder 阶梯设计逻辑
- **L0: 协议基流层** (M1-M2) - 标准 stdin/stdout JSON RPC 协议，Jepsen 的运行管道和测试循环。
- **L1: 共享服务层** (M3-M4) - Maelstrom 提供的 KV/注册表/计数器组件，一致性模型约束校验。
- **L2: 故障容错层** (M5-M6) - 网络混沌注入、一致性违背（Anomalies）诊断与性能时延调优。
