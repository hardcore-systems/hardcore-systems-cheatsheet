# Step1-判题材-tikv.md: CNCF 毕业级分布式事务 Key-Value 数据库审计大纲

本审计项目聚焦于云原生分布式事务数据库的存储底座 `tikv`。我们将通过 28 张高密度核心知识卡片，深度解构 Multi-Raft 共识分片调度、Percolator 分布式事务模型、RocksDB 双存储引擎集成与调优、Coprocessor 协处理器算子下推与向量化计算、异步 Pipeline 消息通信以及故障自愈混沌测试。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: Multi-Raft 共识与调度机理** (Cards 1–5) - `#4B5F7A` (Slate Blue)
    - Multi-Raft Region 概念与分片范围划定、Raft 日志复制与状态机、Region 物理分裂（Split）与重平衡、Region 物理合并（Merge）时序逻辑、Placement Driver (PD) 全局路由调度。
*   **M2: 分布式事务与 Percolator 模型** (Cards 6–10) - `#6B8272` (Muted Sage)
    - Percolator 两阶段提交（2PC）流程、Lock CF 锁管理与并发冲突控制、MVCC（多版本并发控制）物理读写路径、乐观事务与悲观事务锁差异、分布式死锁检测算法。
*   **M3: RocksDB 存储引擎集成内幕** (Cards 11–15) - `#9C6666` (Tea Red)
    - RocksDB 双存储引擎结构（KV 引擎与 Raft 引擎）、WAL 刷盘与 MemTable 无锁写入、SSTable 分层与 Compaction 整理机制、Block Cache 与 Bloom Filter 检索加速、写入放大（Write Amplification）调优折中。
*   **M4: Coprocessor 协处理器算子下推** (Cards 16–20) - `#7A7A7A` (Iron Grey)
    - 协处理器物理算子下推原理、DAG 有向无环图请求解析器、Vectorized 向量化算子执行、Chunk 内存紧凑布局与传输、宿主 CPU SIMD 指令集编译加速。
*   **M5: 节点通信与网络驱动** (Cards 21–24) - `#9A825A` (Dusty Gold)
    - gRPC 多路复用与心跳检测、Raft 消息流异步 Pipeline 处理、节点间零拷贝 Snapshot 传输、Backpressure 背压限流自愈。
*   **M6: 故障自愈与系统诊断** (Cards 25–28) - `#755B77` (Muted Grape)
    - TiKV 网络分区故障自愈、Failpoint 注入与混沌工程测试、RocksDB 运行期监控指标、分布式 Tracing 链路诊断。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    TiKV 的本质是通过在单进程内多路复用海量独立 Raft 共识组（Multi-Raft）实现水平弹性伸缩，并在无锁 RocksDB 引擎之上构建 Percolator 分布式两阶段提交事务流与协处理器下推算子，实现高并发、高可靠的分布式强一致性列式/KV混合存储系统。
*   **L1 四句话逻辑**：
    1. **海量 Region 多路共识**：通过将全局 Key 空间切分为数万个独立 Region，每个 Region 由一个 Raft 组独立负责复制与读写，彻底打破单 Raft 组的吞吐瓶颈。
    2. **Percolator 分布式锁控制**：利用全局授时服务（TSO）与 RocksDB 的 Lock/Write 列族，以乐观/悲观两阶段提交实现完全去中心化的行级分布式强一致性事务。
    3. **数据就地算子下推**：将 SQL/过滤算子编译为 DAG 下推至 Coprocessor 协处理器，直接在存储节点本地对 RocksDB 数据进行向量化过滤，规避巨量网络数据传输损耗。
    4. **双 LSM-Tree 引擎压实**：底层分离存储 Raft 日志与物理业务数据，通过定制 LSM-Tree Compaction 机制控制写放大，配合布隆过滤器实现单机 $O(1)$ 级物理检索。
*   **L2 核心数据流转拓扑**：
    `Client 开启 Percolator 2PC` ➜ `向 Primary Key 写入 Lock CF (未提交)` ➜ `向 Secondary Keys 写入 Lock CF 引用` ➜ `PD 协调 Region Split/Merge 保持水位` ➜ `提交 Primary Key ➜ 状态变更为 Write CF` ➜ `并发异步清扫 (Resolve Lock) Secondary Keys` ➜ `Coprocessor 接收查询下推` ➜ `直接读取 RocksDB 内存 MemTable / 磁盘 SST` ➜ `向量化过滤 ➜ 返回 Chunk 数据`

---

## 🗂️ 28 张核心知识卡片大纲
1. Multi-Raft Region 概念与分片：将全局 Key 划分为独立 Region 及副本分布。
2. Raft 日志复制与状态机：Leader 接收日志、Quorum 多数派确认提交与 State Machine 应用。
3. Region 物理分裂 (Split) 与重平衡：Key 数量过载自动切分并重选 Leader 保证水位。
4. Region 物理合并 (Merge) 时序：冷 Region 合并以及日志同步与对齐步骤。
5. PD 全局调度与路由：Placement Driver 全局拓扑监控、元数据路由及动态平衡。
6. Percolator 分布式两阶段提交 (2PC)：基于全局 TSO 授时与无全局锁的 2PC 实现。
7. Lock CF 锁管理与并发冲突控制：RocksDB 锁列族结构及 Secondary 锁引用。
8. MVCC (多版本并发控制) 读写路径：通过 Write CF 提取版本号重定向数据层。
9. 乐观事务与悲观事务锁差异：Prewrite 冲突回滚与悲观锁先占坑避免大回滚折中。
10. 分布式死锁检测算法：基于等待关系图的本地与全局死锁快速环路检测。
11. RocksDB 双存储引擎集成：业务数据（kvdb）与 Raft 日志（raftdb）物理引擎分离。
12. WAL 刷盘与 MemTable 无锁写入：追加物理日志与并发 SkipList 内存写入开销。
13. SSTable 分层与 Compaction 机制：LSM-Tree 物理分层压实与空间重平衡优化。
14. Block Cache 与 Bloom Filter 加速：布隆过滤器过滤 $O(1)$ 查找与块缓存缓存命中。
15. RocksDB 写入放大调优折中：Compaction 带来的写/读/空间放大权衡限制。
16. Coprocessor 物理算子下推原理：直接在存储侧执行过滤，避免大量数据网络传输损耗。
17. DAG 有向无环图请求解析器：解析由 SQL 编译生成的执行计划请求。
18. Vectorized 向量化算子执行：成批（Batch）处理数据以发挥 CPU 缓存局部性优势。
19. Chunk 内存紧凑布局与传输：列式存储数据内存对齐与零拷贝网络传输流。
20. CPU SIMD 指令集编译加速：AVX-512 指令加速算子循环。
21. gRPC 多路复用与心跳检测：单连接并发 HTTP/2 数据流与 Raft 节点健康探测。
22. Raft 消息流异步 Pipeline 处理：将网络 IO、Raft 日志写入和状态机应用多线程流水线化。
23. 节点间零拷贝 Snapshot 传输：大 Region 状态同步直接绕过用户态写入 Socket。
24. Backpressure 背压限流自愈：RocksDB 延迟高时自动减慢 Raft 确认响应限流。
25. TiKV 网络分区故障自愈：Leader 离线自动选主与脑裂防御策略。
26. Failpoint 注入与混沌工程测试：运行时注入异常流以验证强一致性完整。
27. RocksDB 运行期监控指标：Flush/Compaction 延迟、Cache 命中与 QPS 遥测。
28. 分布式 Tracing 链路诊断：跨节点 RPC 与 RocksDB 内部执行的跨越生命周期诊断。
