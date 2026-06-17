
# GitHub 硬核技术/系统工程“值得出手”项目审计报告

为了匹配《System Design Primer》、《Awesome Kubernetes》与《Database Internals》的硬核级别，我们从 **分布式系统、存储引擎、操作系统内核、高性能网络、编译器虚拟机、硬件体系结构** 等方向，在 GitHub 上检索并筛选出了 **37 个最值得抽取高密 cheatsheet 的标杆项目/经典巨著**。

我们将它们划分为 8 大门类，这些项目普遍具有“学术与工业结合紧密、架构极其复杂、信息密度极高”的特点，极具商业知识变现的壁垒：

---

## 1. 分布式系统与共识算法 (Distributed Systems & Consensus)

- **01. [MIT 6.824 / distributed-systems](https://github.com/mit-pdos/6.824-golabs-2020)**
    - _描述_：分布式系统工程的圣经级实践课程（MapReduce, Raft, Sharded KV, ZooKeeper）。
    - _核心信号_：Raft 状态机分裂、心跳对齐、网络分区脑裂收敛、成员变更。
- **02. [spanner / cockroachdb](https://github.com/cockroachdb/cockroach)**
    - _描述_：全球分布式 SQL 引擎。
    - _核心信号_：TrueTime API (原子钟+GPS)、MVCC 混合逻辑时钟 (HLC)、Raft Group 分片、两阶段提交 (2PC)。
- **03. [etcd-io / etcd](https://github.com/etcd-io/etcd)**
    - _描述_：K8s 底座，强一致性键值协调器。
    - _核心信号_：Raft Go 语言实现、WAL 物理持久化、MVCC 历史版本树、Watch 监听机制。
- **04. [apache / kafka-internals](https://github.com/apache/kafka)**
    - _描述_：高性能分布式事件流平台。
    - _核心信号_：页缓存读写、Zero-Copy 零拷贝发送、分区副本同步机制 (ISR)、Group Coordinator 协调负载再平衡。

---

## 2. 数据库内幕与存储引擎 (Database Internals & Storage)

- **05. [architecture-of-a-database-system (The Red Book)](https://github.com/josephmhellerstein/db-book)**
    - _描述_：经典“红书”——数据库系统架构设计公理。
    - _核心信号_：进程线程模型、查询优化器流控、锁管理器、日志缓冲区、崩溃恢复协议。
- **06. [facebook / rocksdb](https://github.com/facebook/rocksdb) (含 LevelDB)**
    - _描述_：最主流的 LSM-Tree KV 嵌入式存储引擎。
    - _核心信号_：MemTable 并发写入、WAL 原子刷盘、SSTable 多层结构与 Bloom Filter、多级后台 Compaction 算法。
- **07. [sqlite / sqlite-internals](https://github.com/sqlite/sqlite)**
    - _描述_：嵌入式 B-Tree 关系型引擎。
    - _核心信号_：B-Tree 分页调度、缓存页回滚（Rollback Journal）、VDBE（虚拟数据库引擎）字节码执行机制。
- **08. [redis / redis-internals](https://github.com/redis/redis)**
    - _描述_：单线程内存 KV。
    - _核心信号_：IO 多路复用事件轮询、渐进式 Rehash 结构、RDB 快照与 AOF 顺序追加机制、哨兵集群心跳。
- **09. [postgres / postgres](https://github.com/postgres/postgres)**
    - _描述_：最著名的开源学院派 RDBMS 架构。
    - _核心信号_：多进程共享内存架构、MVCC 空间回收 (Vacuum)、基于代价的查询 planner 与各种 Scan 算子。
- **10. [ClickHouse / ClickHouse](https://github.com/ClickHouse/ClickHouse)**
    - _描述_：极速列存 OLAP 引擎。
    - _核心信号_：MergeTree 存储引擎、向量化执行引擎 (SIMD 加速)、稀疏索引设计。

---

## 3. 操作系统与 Linux 内核 (Operating Systems & Kernels)

- **11. [OSTEP / Operating Systems: Three Easy Pieces](https://github.com/remzi-arpacidusseau/ostep-projects)**
    - _描述_：现代操作系统三部曲（虚拟化、并发、持久化）。
    - _核心信号_：地址转换 (TLB & Multi-level Page Tables)、调度调度器 (MLFQ & CFS)、文件系统一致性 (FSCK & Journaling)。
- **12. [torvalds / linux-insides](https://github.com/0xAX/linux-insides)**
    - _描述_：Linux 内核初始化与底层实现细节。
    - _核心信号_：中断陷阱处理、完全公平调度器 (CFS)、伙伴系统与 Slab 分配器、内核页面置换算法。
- **13. [mit-pdos / xv6-riscv](https://github.com/mit-pdos/xv6-riscv)**
    - _描述_：MIT 教学级 Unix 操作系统，代码精简但五脏俱全。
    - _核心信号_：RISC-V 特权模式转换、陷阱向量分配、页面级物理内存隔离、管道缓冲竞态。
- **14. [iovisor / bcc (eBPF)](https://github.com/iovisor/bcc)**
    - _描述_：Linux 内核可编程性与可观测性底座。
    - _核心信号_：kprobes/uprobes 动态插桩、Tracepoints 静态跟踪、eBPF 字节码安全沙盒与 Maps 内存共享机制。

---

## 4. 计算机网络与高性能 IO (Computer Networking & High-Perf IO)

- **15. [top-down-networking (Kurose & Ross)](https://github.com/kurose-ross-net-app-code/7th-edition-code)**
    - _描述_：计算机网络自顶向下方法。
    - _核心信号_：TCP 可靠数据传输逻辑、拥塞控制状态机（BBR vs Cubic）、BGP 路径矢量寻址、IP 路由转发引擎。
- **16. [DPDK / dpdk](https://github.com/DPDK/dpdk)**
    - _描述_：数据平面开发套件。
    - _核心信号_：UIO 内核旁路、无锁循环队列、内存页大页映射、Poll Mode Driver（PMD）避免中断开销。
- **17. [envoyproxy / envoy](https://github.com/envoyproxy/envoy)**
    - _描述_：云原生服务网格高性能边缘代理。
    - _核心信号_：非阻塞单线程事件模型（Event-Loop）、基于 xDS 协议的动态服务发现、网络过滤器链。
- **18. [h2o / quic-http3](https://github.com/h2o/h2o)**
    - _描述_：下一代 HTTP/3 及 QUIC 协议实现。
    - _核心信号_：UDP 承载的可靠传输、多路复用避免头部阻塞 (HOLB)、连接迁移 (Connection ID)、QPACK 头部压缩。

---

## 5. 系统设计与高扩展架构 (System Design & Scalability)

- **19. [donnemartin / system-design-primer](https://github.com/donnemartin/system-design-primer)**
    - _描述_：全球最著名的大型系统设计大图。
    - _核心信号_：一致性哈希 (Consistent Hashing)、读写分离与分库分表、CDN与多级缓存、熔断降级。
- **20. [checkcheckzz / awesome-scalability](https://github.com/checkcheckzz/awesome-scalability)**
    - _描述_：大厂（Netflix, Twitter, Airbnb）高并发扩展模式合集。
    - _核心信号_：微服务重构、无状态服务设计、CQRS 读写职责分离、事件驱动架构 (EDA)。
- **21. [google / sre-book](https://github.com/google/sre-book)**
    - _描述_：谷歌 SRE 运维与架构稳定性圣经。
    - _核心信号_：SLI/SLO/SLA 稳定性指标定义、错误预算管理、过载保护与重试退避机制、容量预测模型。

---

## 6. 编译器、虚拟机与语言运行时 (Compilers, VMs & Runtimes)

- **22. [munificent / craftinginterpreters](https://github.com/munificent/craftinginterpreters)**
    - _描述_：手写编译器与虚拟机的最高指南。
    - _核心信号_：词法扫描、AST 树形遍历、字节码指令集设计、基于栈的 VM、三色标记垃圾回收 (GC)。
- **23. [v8 / v8](https://github.com/v8/v8)**
    - _描述_：谷歌高性能 JavaScript 引擎。
    - _核心信号_：隐藏类与内联缓存 (IC)、JIT 编译（Ignition & TurboFan）、并发垃圾回收（Scavenger & Major MC）。
- **24. [openjdk / jdk](https://github.com/openjdk/jdk)**
    - _描述_：Java 虚拟机（HotSpot VM）核心。
    - _核心信号_：Java 内存模型 (JMM)、JIT 编译屏障、无暂停垃圾回收器（ZGC/Shenandoah）、轻量级虚拟线程（Project Loom）。
- **25. [golang / go-runtime](https://github.com/golang/go)**
    - _描述_：Go 语言运行时。
    - _核心信号_：GMP 协程调度模型、非分代并发三色标记清除 GC、基于 mcache 的内存分配算法。
- **26. [llvm / llvm-project](https://github.com/llvm/llvm-project)**
    - _描述_：现代编译器基础设施。
    - _核心信号_：LLVM IR 静态单赋值形式 (SSA)、中间编译优化 Passes、特定目标代码生成架构。

---

## 7. 性能调优与系统诊断 (Performance Tuning & Diagnostics)

- **27. [brendangregg / system-performance](https://github.com/brendangregg)**
    - _描述_：Brendan Gregg 的系统性能调优系列（包含 FlameGraph）。
    - _核心信号_：USE 性能分析法、火焰图 (Flame Graphs) 绘制与分析、Linux Profiling (perf) 性能监控点排查。
- **28. [google / sanitizers](https://github.com/google/sanitizers)**
    - _描述_：Google 开发的底层 C/C++ 内存与并发检测套件。
    - _核心信号_：AddressSanitizer 影子内存映射、ThreadSanitizer 读写依赖图死锁检测。

---

## 8. 计算机体系结构与芯片设计 (Computer Architecture & ISA)

- **29. [riscv / riscv-isa-manual](https://github.com/riscv/riscv-isa-manual)**
    - _描述_：RISC-V 指令集体系结构规范。
    - _核心信号_：流水线冒险、乱序执行 (OoO) 缓冲、指令分支预测器、L1/L2 多级缓存一致性协议 (MESI)。
- **30. [nvidia / cuda-samples](https://github.com/NVIDIA/cuda-samples)**
    - _描述_：GPU 异构计算与大规模并行设计。
    - _核心信号_：SIMT 单指令多线程模型、Warp 线程束调度、共享内存（Shared Memory）冲突、Memory Coalescing。

---

## 9. 现代密码学与硬件安全隔离 (Cryptography & Hardware Isolation)

- **31. [openssl / openssl](https://github.com/openssl/openssl)**
    - _描述_：工业级密码学底层。
    - _核心信号_：对称/非对称加密数学模型（ECC/RSA）、TLS 握手协议密码套件协商、定点数侧信道攻击防御。
- **32. [intel / sgx-software-enable](https://github.com/intel/sgx-software-enable)**
    - _描述_：可信执行环境（TEE）硬件级隔离。
    - _核心信号_：安全飞地（Enclave）内存页面管理、远程证明协议。

---

## 10. 复杂物理系统、控制理论与电网 (Physical Systems & Control)

- **33. [awesome-robotics (含控制理论与 ROS)](https://github.com/ahundt/awesome-robotics)**
    - _描述_：复杂物理机器人的状态控制。
    - _核心信号_：PID 控制反馈闭环、卡尔曼滤波 (Kalman Filter) 状态估计、SLO 运动规划轨迹生成。
- **34. [PowerSystemsProject / PowerSystems.jl](https://github.com/NREL-Sienna/PowerSystems.jl)**
    - _描述_：工业级电网调度与多变量物理流计算系统。
    - _核心信号_：交流/直流功率流潮流方程（Power Flow）、发电机机组组合（Unit Commitment）对偶分析。

---

## 11. 现代软件开发方法论与高级模式 (Software Engineering & Patterns)

- **35. [kamranahmedse / developer-roadmap](https://github.com/kamranahmedse/developer-roadmap)**
    - _描述_：全球最高 star 的软件工程师技能树拓扑。
    - _核心信号_：后端/前端/DevOps 完备技术分层、标准接口栈、工业级演进路由。
- **36. [iluwatar / java-design-patterns](https://github.com/iluwatar/java-design-patterns)**
    - _描述_：经典设计模式的硬核实现（不仅仅是常规的23种）。
    - _核心信号_：命令查询职责分离 (CQRS)、两阶段提交模式、并发反应器 (Reactor)、三层隔离沙盒模式。
- **37. [jwasham / coding-interview-university](https://github.com/jwasham/coding-interview-university)**
    - _描述_：从零开始学习底层计算机科学的硬核学习路线。
    - _核心信号_：时间复杂度分析公理（Big O）、物理数据结构限界（堆/红黑树/Trie）、图论寻路算法决策。