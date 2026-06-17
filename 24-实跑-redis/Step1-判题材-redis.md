# Scoping & Categorization Document: 《redis-internals》 (Redis 存储内核与高并发内幕)

本文件定义了《redis-internals》（Redis 存储内核实现）的底层分析与高密拆解框架，规划 28 张核心卡片在 6 大模块及 Zone T 辅助版块中的物理映射，并确立莫兰迪色彩体系与页面布局参数。

---

## 一、 核心元数据与物理页面预算

*   **目标书籍/项目**：[redis / redis-internals](https://github.com/redis/redis) 经典内存键值数据库存储引擎
*   **拆书定位**：Redis 单线程 ae.c 事件循环、SDS/Dict/Skiplist/Listpack 动态数据结构实现、RDB 与 AOF 持久化写时复制、内存分配与近似 LRU 淘汰、主从复制与 Cluster 分片共识、以及 Redis 6.0 多线程 I/O 优化。
*   **物理页面预算**：**精确 2 页 A4 横向** (Landscape A4, 297mm x 210mm)
*   **LaTeX 布局引擎**：`extarticle` + `multicol` + `tcolorbox` 三列流式自适应排布，0 溢出。
*   **字体配置**：
    *   主要中文字体：`Microsoft YaHei` (微软雅黑) 或 `SimSun` (宋体)
    *   等宽字体：`Courier New` 或 `Consolas`
    *   西文字体：`Arial` 或 `Times New Roman`

---

## 二、 莫兰迪技术色彩系统 (Morandi Palette)

为每个模块定义一个低饱和度、高质感的莫兰迪色彩 Token，以在 LaTeX 海报和 HTML 互动沙盒中保持一致的视觉层级：

| 模块编码 | 模块名称 | 莫兰迪色系 | RGB / Hex 代码 | 视觉语义 |
| :--- | :--- | :--- | :--- | :--- |
| **M1** | **核心事件循环与网络 (Event Loop & RESP)** | 石板蓝 (Slate Blue) | `#7A8B99` / `RGB(122,139,153)` | ae.c 单线程事件驱动、多路复用抽象、RESP 协议解析 |
| **M2** | **动态内存数据结构 (Data Structures)** | 苔绿 (Moss Green) | `#7D8F7B` / `RGB(125,143,123)` | SDS 字符串、Dict 渐进式 Rehash、Ziplist/Listpack、Skiplist 跳表 |
| **M3** | **物理持久化机制 (Persistence)** | 梅玫瑰 (Plum Rose) | `#9E828A` / `RGB(158,130,138)` | RDB 快照写时复制、AOF 追加重写、混合持久化恢复 |
| **M4** | **内存管理与过期淘汰 (Memory & Eviction)** | 陶土红 (Terracotta) | `#B58A7D` / `RGB(181,138,125)` | jemalloc 碎片率、robj 对象系统、近似 LRU 淘汰、UNLINK 异步释放 |
| **M5** | **高可用复制与分片 (HA & Sharding)** | 靛青 (Indigo) | `#5F7582` / `RGB(95,117,130)` | 主从 PSYNC 复制、Sentinel 故障转移、Cluster Gossip 与槽位路由 |
| **M6** | **高阶优化与调试监控 (Optimization & Labs)** | 古董金 (Antique Gold) | `#BFA88F` / `RGB(191,168,143)` | Threaded I/O 多线程优化、Defrag 碎片整理、Client-side cache、Slowlog |

---

## 三、 L0 ~ L2 压缩阶梯设计

在海报页眉处放置“Redis 存储引擎第一性原理公理阶梯”，作为阅读的心智锚点：

*   **L0 一句话本质**：在单线程内存模型与异步 I/O 多路复用的物理限制下，通过紧凑对齐的变长 Packed 内存结构和渐进式 Hash 降低检索复杂度，并利用 COW 子进程持久化及无中心化哈希槽 Gossip 协议实现超高吞吐的分布式内存 KV 系统。
*   **L1 四句话逻辑**：
    1.  **单线程事件网络多路复用** (M1) 采用极其轻量的事件循环 `ae.c`，完全绕过线程上下文切换与锁保护竞态，利用 `epoll` 驱动连接。
    2.  **紧凑与渐进数据结构** (M2) 设计 SDS 二进制安全串，并在 Dict 哈希争用时通过双表指针进行增量 Rehash 削平 CPU 峰值。
    3.  **写时复制与顺序日志追加** (M3) 利用 `fork()` 的 Copy-On-Write 物理机制后台隔离 RDB 写入，配合 AOF 增量追加降低丢失风险。
    4.  **近似驱逐淘汰与无中心分片** (M4-M6) 采用内存池对象引用计数与随机采样 LRU 降内存开销，通过 16384 槽 Gossip 组网实现水平拓展。
*   **L2 存储引擎数据流演进拓扑**：
    *   [RESP 客户端连接] $\rightarrow$ [ae.c 事件循环监听] $\rightarrow$ [SDS Query Buffer 读入] $\rightarrow$ [Dict 命令哈希匹配] $\rightarrow$ [内存结构定位与渐进 Rehash] $\rightarrow$ [AOF 重写/RDB 异步 COW] $\rightarrow$ [Master Backlog 物理分发] $\rightarrow$ [Cluster 哈希槽重定向转发]

---

## 四、 28 张核心卡片物理映射表

每一张卡片代表一个高密度的系统设计考点或机制，包含中文表述与专业西文术语。

### M1：核心事件循环与网络 (M1_1 至 M1_4) - 【石板蓝】
*   **Card 1 (M1_1)**: ae.c 单线程事件驱动模型 (Single-Threaded Event Loop, 将网络 I/O 事件与时间事件注册合并，避免锁争用与切换代价)
*   **Card 2 (M1_2)**: I/O 多路复用操作系统抽象 (I/O Multiplexing Abstraction, 封装 `select/epoll/kqueue` 统一接口，实现毫秒级异步触发)
*   **Card 3 (M1_3)**: 客户端查询缓冲区读写队列 (Client Query Buffers, 读缓冲区动态扩容限制、命令解析流，保护主线程运行安全)
*   **Card 4 (M1_4)**: RESP 通信协议解析与内存序列化 (RESP Protocol Parser, 简易人类可读二进制安全协议，Bulk Strings 块串与整数流打包)

### M2：动态内存数据结构 (M2_1 至 M2_5) - 【苔绿】
*   **Card 5 (M2_1)**: SDS 简单动态字符串结构 (Simple Dynamic String SDS, 头部携带 `len/alloc/flags`，二进制安全且支持 $O(1)$ 长度获取)
*   **Card 6 (M2_2)**: 双 HashTable 与渐进式 Rehash (dict.h & Progressive Rehash, 使用双表 `ht[0]/ht[1]`，在单次命令中增量迁移 `rehashidx`)
*   **Card 7 (M2_3)**: Ziplist 压缩列表与 Listpack 物理对齐 (Ziplist & Listpack, 连续内存紧凑字节流结构，消除链表指针消耗，支持双向遍历)
*   **Card 8 (M2_4)**: Dict 与 Skiplist 跳表双向融合 (Dict & Skiplist for ZSET, 跳表维护有序层级方便范围扫描，哈希表维护 $O(1)$ 元素查找)
*   **Card 9 (M2_5)**: Quicklist 双向链表与 Intset 整数集 (Quicklist & Intset, Quicklist 链表链接多个 Ziplist 规避物理大块内存碎片开销)

### M3：物理持久化机制 (M3_1 至 M3_4) - 【梅玫瑰】
*   **Card 10 (M3_1)**: RDB 内存镜像快照与 Copy-on-Write (RDB Snapshotting via COW, 调用 `fork()` 派生子进程，利用系统页表写时复制物理保护主进程写)
*   **Card 11 (M3_2)**: AOF 命令追加日志与同步刷盘策略 (AOF Append & Fsync Policies, 缓冲区追加，提供 always/everysec/no 三档 fsync 平衡安全与写瓶颈)
*   **Card 12 (M3_3)**: AOF 后台重写 BGREWRITEAOF 压缩 (AOF Rewrite BGREWRITEAOF, 创建子进程根据当前数据状态逆向合成最小指令集以瘦身日志)
*   **Card 13 (M3_4)**: 混合持久化恢复加载效率优化 (Hybrid Persistence, 数据库头部加载二进制 RDB 数据，尾部流式消费 AOF 增量指令，极速加载)

### M4：内存管理与过期淘汰 (M4_1 至 M4_4) - 【陶土红】
*   **Card 14 (M4_1)**: jemalloc 物理内存分配器集成与碎片率 (jemalloc & Memory Fragmentation, 规避系统 malloc 碎块，`mem_fragmentation_ratio` 超标触发 defrag)
*   **Card 15 (M4_2)**: robj 对象层设计与共享对象池 (Redis Object System robj, 封装具体结构，定义 LRU/LFU 统计信息及引用计数 refcount 物理共享)
*   **Card 16 (M4_3)**: 随机采样近似 LRU 淘汰算法 (Approximated LRU Eviction, 避免全局 LRU 链表多线程锁竞争，随机抽样 $N$ 个键淘汰最久未使用者)
*   **Card 17 (M4_4)**: UNLINK 异步非阻塞内存释放 (Lazy Freeing via UNLINK, 多线程 bio.c 在后台悄悄释放大内存，规避主线程物理删除阻塞)

### M5：高可用复制与分片 (M5_1 至 M5_6) - 【靛青】
*   **Card 18 (M5_1)**: 主从复制复制积压缓冲区与 PSYNC (Master-Replica Replication, 主节点复制积压缓冲区 `backlog` 循环链，实现断线后 Partial Sync 增量重连)
*   **Card 19 (M5_2)**: Sentinel 故障判定与故障转移共识 (Sentinel Failover Consensus, 主观下线 sdown 与客观下线 odown 判定，Raft 选主转移 VIP)
*   **Card 20 (M5_3)**: Cluster Gossip 无中心元数据传播 (Cluster Gossip Protocol, 集群节点间通过 PING/PONG 每秒广播拓扑状态，规避中心化服务单点故障)
*   **Card 21 (M5_4)**: 16384 哈希槽物理路由与 ASK/MOVED 重定向 (Hash Slots Routing & Redirect, 客户端直连槽，重定向 MOVED (物理转移) 与 ASK (临时重定向))
*   **Card 22 (M5_5)**: 事务机制 ACID 限界与 WATCH 悲观防御 (Redis Transaction Limits, `MULTI/EXEC` 块不支持回滚，`WATCH` 借助 CAS 版本监控判断是否回滚提交)
*   **Card 23 (M5_6)**: Lua 脚本嵌入沙盒与原子执行原理 (Lua Script Atomicity, 虚拟机嵌入，脚本执行期间主事件循环独占，保障原子性)

### M6：高阶优化与调试监控 (M6_1 至 M6_5) - 【古董金】
*   **Card 24 (M6_1)**: Threaded I/O 多线程网络处理优化 (Threaded I/O in Redis 6.0, 仅将 Socket 读写网卡瓶颈分发给 I/O 线程，逻辑执行仍保持单线程安全)
*   **Card 25 (M6_2)**: 内存动态碎片整理机制原理 (Active Defragmentation, 开启后台 defrag，通过将存活对象拷贝到物理新分配区以回收碎片物理页)
*   **Card 26 (M6_3)**: 客户端缓存失效跟踪机制 (Client-Side Caching, 利用 RESP3 协议的 Invalidation 消息主动向客户端推送内存缓存过期指令)
*   **Card 27 (M6_4)**: 慢日志排查与延迟医生诊断工具 (Slowlog & Latency Doctor, 记录事件循环内超过阈值命令，`LATENCY DOCTOR` 深度挖掘时延抖动根源)
*   **Card 28 (M6_5)**: 模块 API 物理拓展接口 (Redis Modules API, 支持以动态库插件形式载入自定义数据类型与高密集度计算算子)

---

## 五、 辅助版块 Zone T 设计

作为 cheatsheet 底部或侧边的实用工具箱，为内存数据库生产调优提供即时参数与命令视图参考：

1.  **T1 Redis 典型参数调优对照表**：
    *   `maxmemory <bytes>` (内存限制)：超过则触发驱逐策略。
    *   `hash-max-ziplist-entries 512` (数据结构转型阈值)：低于则用 ziplist 压缩紧凑内存，超过自动转型为 dict。
    *   `repl-backlog-size 1mb` (复制积压缓冲区大小)：网络抖动大时建议调大至 100mb，规避主从全量同步导致的网卡穿透崩溃。
2.  **T2 经典性能监控与诊断 INFO 指令字典**：
    *   实时内存与碎片监控：`INFO memory` (重点观察 `used_memory_rss` 与 `mem_fragmentation_ratio`)
    *   集群槽位状态查询：`CLUSTER NODES` / `CLUSTER SLOTS`
    *   客户端连入状态与限速控制：`CLIENT LIST` / `CLIENT KILL`
