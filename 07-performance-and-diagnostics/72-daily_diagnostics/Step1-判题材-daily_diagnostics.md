# Step1-判题材-daily_diagnostics.md: 线上故障定位与性能榨汁机审计大纲

本审计项目聚焦于系统层与应用层的性能调优及故障排查方法论 `Daily Diagnostics & Tuning`。我们将通过 28 张核心知识卡片，解构 JVM/Go GC 调优、内存逃逸与 pprof 探针、异步事件循环耗时诊断、磁盘 IO 与网络缓冲区调优、CPU 亲和性与上下文切换、MySQL 索引跳转深度与并发死锁等模块。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: 语言运行时与内存诊断** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - JVM 垃圾回收参数微调、Go 内存逃逸与 pprof 采样、Python 异步事件循环耗时诊断、堆内存泄漏监测、JVM CPU 100% 物理定位、Go GC 瞬时停顿分析。
*   **M2: CPU 调度与物理缓存** (Cards 7–12) - `#6B8272` (Muted Sage)
    - 线程池最佳线程数计算与工作队列设计、CPU 火焰图栈采样分析、CPU 缓存行对齐 (Cacheline)、CPU 亲和性与核绑定、上下文切换损耗削减、JVM 安全点停顿 (Safepoint)。
*   **M3: 数据库性能与索引优化** (Cards 13–19) - `#9C6666` (Tea Red)
    - MySQL B+ 树索引物理跳转优化、MySQL 行级锁与死锁日志分析、数据库连接池参数精调、Redis 慢查询大 Key 治理、Redis 淘汰与 LFU 算法、CQRS 模型分离时序。
*   **M4: 磁盘 I/O 与网络短路** (Cards 20–24) - `#7A7A7A` (Iron Grey)
    - 操作系统页面缓存与脏页冲刷控制、磁盘 I/O 调度器选型、Linux TCP Socket 读写缓冲区调优、基于 sendfile/mmap 零拷贝、TIME_WAIT 与 HTTP 长连接复用。
*   **M5: 异步驱动与性能聚合** (Cards 25–28) - `#9A825A` (Dusty Gold)
    - 异步事件驱动对比单线程单连接、Linux Epoll 惊群效应、SIMD 向量化计算、APM 慢接口延迟追溯。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    性能调优的本质是在计算资源（CPU、内存、I/O、网络）冲突状态下，通过消除无效开销（如上下文切换、垃圾回收、页扫描、锁自旋），实现数据物理传输与计算路径的极致收缩。
*   **L1 四句话逻辑**：
    1. **内存边界防御**：基于 pprof/heaptrack 追踪动态分配，收缩对象生存期，规避 GC 瞬时停顿与 OOM 假死。
    2. **CPU 损耗榨汁**：通过线程数合理配比与 CPU 绑定，将计算阻断移出核心执行环路，减少系统上下文切换开销。
    3. **I/O 跳数收敛**：优化 B+ 树索引匹配、采用零拷贝传输，将多次磁盘与网络 IO 转化为内存页级快速读写。
    4. **并发锁冲突化解**：以细粒度乐观锁及合理的资源排序访问规避死锁，配合 AQS/MPSC 环形缓冲区消除竞争。
*   **L2 核心数据流转拓扑**：
    `请求到达` ➜ `网关 mmap 零拷贝读取` ➜ `线程池绑定 CPU 执行核心` ➜ `pprof 采样记录热点` ➜ `通过索引 (仅2次 B+树寻址) 直达内存页` ➜ `并发更新 ➜ 乐观锁版本匹配` ➜ `若锁冲突 ➜ 触发重试退避抖动` ➜ `脏页异步冲刷回盘` ➜ `垃圾回收彩色指针读屏障自愈`

---

## 🗂️ 28 张核心知识卡片大纲
1. JVM 垃圾回收参数微调：新生代老生代配比与 G1/ZGC 停顿限制精调。
2. Go 内存逃逸与 pprof 采样：`-gcflags="-m"` 编译判定与 heap profile 热点追踪。
3. Python 异步事件循环耗时诊断：在 async/await 循环中定位阻塞 IO 并切分至 ThreadPoolExecutor。
4. 堆内存泄漏监测 (heaptrack)：跟踪物理内存 malloc/free 动作及长生存期泄露调用栈。
5. JVM CPU 100% 物理定位：通过 `top -Hp`、十六进制 LWP ID 转换与 `jstack` 线程 nid 匹配锁死死循环。
6. Go GC 瞬时停顿分析：理解三色标记写屏障开销与强三色不变性打破下的重扫抖动。
7. 线程池最佳线程数计算：CPU 密集型（N+1）与 I/O 密集型（N * (1 + W/C)）的公式推导。
8. CPU 火焰图栈采样分析：`perf` 采样与 `async-profiler` 火焰图生成，识读横向宽度与纵向调用栈。
9. CPU 缓存行对齐 (Cacheline)：L1/L2 缓存 64 字节块对齐，防止 False Sharing 并引入 `@Contended` 填充。
10. CPU 亲和性与核绑定：`sched_setaffinity` 系统调用限制线程调度范围，消除 CPU 核心迁移开销。
11. 上下文切换损耗削减：区分自愿与非自愿上下文切换，利用 CAS 无锁队列和自旋锁削减线程争抢。
12. JVM 安全点停顿 (Safepoint)：理解 safepoint 触发机制（GC、偏向锁撤销）及长循环未检测 safepoint 隐患。
13. MySQL B+ 树索引物理跳转优化：索引选择性（Selectivity）计算、回表（Bookmark Lookup）消除及最左前缀索引设计。
14. MySQL 行级锁与死锁日志分析：解析 Shared (S) 与 Exclusive (X) 锁冲突，分析 GAP 锁与 Next-Key 锁竞争边界。
15. 数据库连接池参数精调：HikariCP 大小公式验证，微调 Keep-Alive、超时时间及阻断队列。
16. Redis 慢查询大 Key 治理：定位 O(N) 复杂命令（KEYS、HGETALL），切分大 Hash 桶与 Lazy Free 异步剔除。
17. Redis 淘汰与 LFU 算法：对比 LRU 采样与 LFU 计数器衰减机制，解决短时高频热点驱逐漏洞。
18. CQRS 模型分离时序：CQRS 读写模型同步时延度量与前端乐观更新容错设计。
19. 操作系统页面缓存与脏页冲刷：`vm.dirty_background_ratio` 与 `vm.dirty_ratio` 对系统写入卡顿的影响。
20. 磁盘 I/O 调度器选型：BFQ、Kyber、mq-deadline 与 NVMe 下的 None 调度策略差异。
21. Linux TCP Socket 读写缓冲区调优：动态调节 `rmem`/`wmem` 窗口，应对带宽延迟积（BDP）极限吞吐。
22. 基于 sendfile/mmap 零拷贝：避免数据拷贝至用户空间，将内核 Read Buffer 直接管道输出到 Socket 发送队列。
23. TIME_WAIT 与 HTTP 长连接复用：分析 TIME_WAIT（2MSL）产生原因，配置 `tcp_tw_reuse` 解决端口耗尽。
24. 异步事件驱动对比单线程单连接：epoll 非阻塞 IO 驱动与阻塞多线程连接的内存与连接水位线比对。
25. Linux Epoll 惊群效应：理解惊群成因，通过 `EPOLLEXCLUSIVE` 标志与 `SO_REUSEPORT` 端口复用分流。
26. SIMD 向量化计算：单指令多数据，循环展开内联编译，利用 256 位 AVX 寄存器进行数据并行计算。
27. APM 慢接口延迟追溯：Trace 链路的 span 边界测量、序列化耗时与 RPC 外部网络延时归类分析。
28. 锁竞争削减（AQS/RingBuffer）：Java AQS 双向队列挂起唤醒开销对比 LMAX Disruptor CAS 环形无锁队列。
