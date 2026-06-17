# 《redis-internals》高密知识图谱与速查手册 (Cheatsheet)

*   **L0 一句话本质**：在单线程内存模型与异步 I/O 多路复用的物理限制下，通过紧凑对齐的变长 Packed 内存结构和渐进式 Hash 降低检索复杂度，并利用 COW 子进程持久化及无中心化哈希槽 Gossip 协议实现超高吞吐的分布式内存 KV 系统。
*   **L1 四句话逻辑**：
    1.  **单线程事件网络多路复用** (M1) 采用极其轻量的事件循环 `ae.c`，完全绕过线程上下文切换与锁保护竞态，利用 `epoll` 驱动连接。
    2.  **紧凑与渐进数据结构** (M2) 设计 SDS 二进制安全串，并在 Dict 哈希争用时通过双表指针进行增量 Rehash 削平 CPU 峰值。
    3.  **写时复制与顺序日志追加** (M3) 利用 `fork()` 的 Copy-On-Write 物理机制后台隔离 RDB 写入，配合 AOF 增量追加降低丢失风险。
    4.  **近似驱逐淘汰与无中心分片** (M4-M6) 采用内存池对象引用计数与随机采样 LRU 降内存开销，通过 16384 槽 Gossip 组网实现水平拓展。
*   **L2 存储引擎数据流演进拓扑**：
    *   [RESP 客户端连接] $\rightarrow$ [ae.c 事件循环监听] $\rightarrow$ [SDS Query Buffer 读入] $\rightarrow$ [Dict 命令哈希匹配] $\rightarrow$ [内存结构定位与渐进 Rehash] $\rightarrow$ [AOF 重写/RDB 异步 COW] $\rightarrow$ [Master Backlog 物理分发] $\rightarrow$ [Cluster 哈希槽重定向转发]

---

## 🌐 经典内存数据库存储内核物理滤镜 (Redis Physical Epistemic Filter)

- **认识论 (Epistemology) - 以内存速度与 CPU 缓存物理对齐为设计起点**：
  Redis 认识论的核心假设在于“内存读写比网络和磁盘快 5 个数量级”。在这个假设下，传统数据库基于随机 I/O 设计的 B-Tree 索引和 Buffer Pool 均被抛弃。Redis 直接在内存虚拟地址上构建紧凑排列的 SDS 串、跳表与 Ziplist。每一层设计都致力于提升 CPU 高速缓存（L1/L2 Cache）命中率，减少内存对齐浪费（Padding），使用 Packed 结构紧凑压实字节流，消除繁琐的内存碎块开销。
- **一致性论 (Consistency Theory) - 弱持久化妥协与主从无锁复制的物理现实**：
  在高速内存场景中，高频强一致性持久化（如每指令 fsync）会产生磁盘物理写瓶颈。Redis 妥协性地采用 AOF 每秒刷盘（everysec）和后台 RDB 快照，构建弱持久性防线。它的主从同步依赖 Master 发送命令流与本地 Backlog 环形缓冲区，从节点基于非阻塞复制。这种设计在面对网络分区和脑裂时，依靠 Sentinel 或 Cluster 的 Gossip 纪元投票（Epoch Vote）达成多数派仲裁，提供了最终一致性的高并发边界。
- **方法论 (Methodology) - 单线程事件轮询与 CPU 分流的极客美学**：
  不同于多线程数据库复杂的加锁控制，Redis 的方法论是“单线程主宰核心逻辑”。通过 ae.c 事件轮询，将 Socket 可读可写抽象为文件事件，顺序执行，免去了线程调度和原子锁的开销。而在耗时操作上，如 RDB 写入利用 OS 的 Copy-on-Write 机制，Lazy Freeing 利用后台 bio.c 辅助线程，以及 Redis 6.0 中多线程 offload socket 读写，展现了主从逻辑分离、CPU 核心精细分流的方法论美学。

---

## ⚔️ 数据库并发与崩溃恢复折衷矩阵 (Database Concurrency & Recovery Trade-offs)

| 开发者直觉 (⚠) | 系统物理现实 (✗) | 架构折衷与防范金标准 (✓) |
| :--- | :--- | :--- |
| **单实例内存越大越好，免去分片管理之苦** | 内存超大（如 64GB）会导致 RDB fork 时页表拷贝开销高达数秒，造成严重的全局 STALL 停顿。同时全量同步会让主节点网卡被打满崩溃。 | 限制单实例内存大小在 4GB-8GB 内。多实例通过 Redis Cluster Gossip 哈希槽或 Codis 分片分担流量，将 Socket I/O offload 到 Threaded I/O。 |
| **直接设置 expire 过期时间可以完美自动回收冷数据** | 当数百万个键在同一秒内过期时，主事件循环惰性删除（Active Expire）会被强制占用时间，导致写请求延迟骤增数十毫秒，发生时延抖动。 | 在配置 expire 过期时间时，人为引入随机时间偏差（Jitter），将过期点平摊到时间轴，并在主从架构中依靠 Master 异步 DEL 广播确保一致。 |
| **开启 AOF everysec 即可完美保证每秒持久化且无性能副作用** | 如果高频写负载导致磁盘 I/O 饱和，当上一秒 fsync 还未返回时，主线程在执行 write 时会被强制挂起（Fsync Stall），导致单线程停滞。 | 合理开启 `no-appendfsync-on-rewrite`。在 AOF 后台重写期间停止强制 fsync 刷盘，以降低主线程被磁盘 write 阻塞的极端概率。 |

---

## 🗺️ 6大核心模块与 28 张高密速查卡片 (Core Cards Map)

### M1：核心事件循环与网络 (Slate Blue)

#### Card 1. ae.c 单线程事件驱动模型 (ae.c Single-Threaded Event Loop)
*   **网络与时间事件合并**：主循环核心由 `aeProcessEvents` 驱动。它在单次循环中不仅处理网络 Socket 可读可写事件，还处理内部时间事件（如过期扫描、主从心跳）。
*   **消除上下文切换**：完全在单线程内运行，消除了多线程上下文切换、互斥锁竞争和 CPU Cache 被其他线程污染的弊端。

#### Card 2. I/O 多路复用操作系统抽象 (I/O Multiplexing Abstraction)
*   **平台自适应包装**：Redis 将操作系统非阻塞 I/O 原语进行了二次封装（`ae_select.c`、`ae_epoll.c`、`ae_kqueue.c`）。
*   **宏编译编译抉择**：在编译期优先使用效率最高的平台原语（Linux 平台为 epoll，BSD 平台为 kqueue），实现 $O(1)$ 的活跃 Socket 监听响应。

#### Card 3. 客户端查询缓冲区读写队列 (Client Query Buffers)
*   **缓冲区空间控制**：每个客户端连入对应一个 `connection` 结构。读缓冲区（Query Buffer）初始分配 16KB，随命令大小自动扩容，但若达到 1GB 硬性阈值将强制断开连接。
*   **写响应队列**：写回时将响应内容先打包入客户端静态 16KB 缓存区或挂载到 `reply` 双向链表中，在 ae 循环可写事件回调中批量 flush 发送。

#### Card 4. RESP 通信协议解析与内存序列化 (RESP Protocol Parser)
*   **二进制安全格式**：RESP（Redis Serialization Protocol）采用前缀标示数据类型。如 `+` 代表简单字符串，`$` 代表 Bulk String，`:` 代表整型。
*   **简易快速解析**：依靠读缓冲区指针前移快速切割命令，不包含复杂的 XML/JSON 解析开销，且因明确定义字符串长度而支持二进制安全。

---

### M2：动态内存数据结构 (Moss Green)

#### Card 5. SDS 简单动态字符串结构 (Simple Dynamic String SDS)
*   **消除 strlen 开销**：SDS 头部结构体定义了已用长度 `len`、分配容量 `alloc` 以及控制头部种类的 `flags`，支持常数时间 $O(1)$ 获取串长度。
*   **防缓冲区溢出**：SDS 修改时会先判定 `alloc` 剩余空间，不足时自动触发内存预分配逻辑（小于 1MB 翻倍，大于 1MB 增量 1MB），并且以 `\0` 结尾兼容 C 标准串。

#### Card 6. 双 HashTable 与渐进式 Rehash (dict.h & Progressive Rehash)
*   **双表内存渐进迁移**：Redis 的字典 `dict` 结构体中包含两个哈希表 `ht[0]` 和 `ht[1]`。当主表冲突率过高时，初始化 `ht[1]` 为其两倍大小。
*   **rehashidx 指针搬运**：将迁移任务平摊。每次对字典执行读、写或删除时，主动将 `ht[0]` 在 `rehashidx` 指向的哈希链表搬运到 `ht[1]` 并在完成后自增指针，规避了一次性大迁移导致的 CPU 停摆。

#### Card 7. Ziplist 压缩列表与 Listpack 物理对齐 (Ziplist & Listpack)
*   **无指针内存连块**：传统链表指针开销达 24 字节。Ziplist 将元素紧凑地以 `[prev_entry_len][encoding][entry_data]` 物理连续存放在单块内存中。
*   **级联更新消除**：新版 Redis 采用 Listpack 代替 Ziplist。Listpack 将前驱长度改为当前元素后缀长度，消除了插入大元素时由于 prevlen 位数变化引起后续节点级联重分配（Cascade Update）的物理灾难。

#### Card 8. Dict 与 Skiplist 跳表双向融合 (Dict & Skiplist for ZSET)
*   **双重定位折衷**：有序集合（ZSET）底层采用 `dict` 和 `zskiplist` 组合。
*   **跳表加速范围**：跳表通过多层随机节点索引提供对分数值的 $O(\log N)$ 范围扫描和排序输出；同时 `dict` 保存成员与分数的映射，提供 $O(1)$ 的成员寻值，兼顾两种查找优势。

#### Card 9. Quicklist 双向链表与 Intset 整数集 (Quicklist & Intset)
*   **碎片与指针博弈**：`quicklist` 是一个双向链表，但链表中的每个节点都是一个 `ziplist`。既能快速两端插入，又避免了大量小链表节点的碎片问题。
*   **整数集合压缩**：`intset` 是一个紧凑的整数数组，内部根据数值大小支持自动升级（int16 $\rightarrow$ int32 $\rightarrow$ int64），极度节省内存空间。

---

### M3：物理持久化机制 (Plum Rose)

#### Card 10. RDB 内存镜像快照与 Copy-on-Write (RDB Snapshotting via COW)
*   **父子进程页表共享**：执行 `BGSAVE` 时，主进程 `fork` 出一个只读的子进程，子进程共享主进程的物理页表。
*   **写时复制触发**：当主进程继续写数据时，OS 捕获到写保护异常，物理复制受影响的 4KB 内存页，子进程持有的旧页数据不发生变化并安全落盘 RDB，实现了非阻塞内存转储。

#### Card 11. AOF 命令追加日志与同步刷盘策略 (AOF Append & Fsync Policies)
*   **命令缓冲区同步**：Redis 收到修改命令并执行完内存修改后，将命令以 RESP 格式追加写入 `aof_buf`。
*   **fsync 刷盘策略**：
    1.  `always`：每次事件循环都强制调用 fsync 刷盘，最安全但写性能受限于磁盘 I/O。
    2.  `everysec`：主线程异步每秒刷一次盘，性能极高且最多只丢 1 秒数据（金标准）。
    3.  `no`：由操作系统决定何时刷盘，不可控。

#### Card 12. AOF 后台重写 BGREWRITEAOF 压缩 (AOF Rewrite BGREWRITEAOF)
*   **物理体积瘦身**：当 AOF 文件膨胀过大时，执行 `BGREWRITEAOF`，子进程在后台直接扫描当前内存数据，将其翻译为一条条新写入命令写入临时文件。
*   **增量缓冲区合并**：重写期间，主进程的写命令不仅追加进旧 AOF，还会写入专门的 `aof_rewrite_buf`。重写完成后由主线程进行尾部增量追加合并，实现 AOF 极速瘦身。

#### Card 13. 混合持久化恢复加载效率优化 (Hybrid Persistence)
*   **RDB+AOF 双剑合璧**：新版 Redis 默认开启混合持久化。在重写 AOF 时，子进程直接将内存当前的二进制 RDB 快照写入新 AOF 文件的头部，增量部分以 AOF 命令追加在尾部。
*   **恢复效率跃升**：加载时头部二进制极速解析加载，尾部命令重放重塑最新状态，完美结合了 RDB 恢复快和 AOF 丢数据少的特性。

---

### M4：内存管理与过期淘汰 (Terracotta)

#### Card 14. jemalloc 物理内存分配器集成与碎片率 (jemalloc & Memory Fragmentation)
*   **内存块细粒度分配**：Redis 默认选用 jemalloc。它将内存划分为小、大、巨型等固定大小的 Arena，极力减少系统碎片。
*   **碎片比率警戒线**：`INFO memory` 中输出的 `mem_fragmentation_ratio` 代表分配 RSS 与实际 used 内存的比值，比率大于 1.5 说明物理内存出现碎洞，需启动整理。

#### Card 15. robj 对象层设计与共享对象池 (Redis Object System robj)
*   **底层结构抽象**：每个 Redis 键值在底层都是一个 `redisObject` 结构体。`type` 标示对象类型，`encoding` 指示底层具体编码。
*   **引用计数与 LRU 字段**：`refcount` 控制引用计数（0~9999 共享整型无需重复分配），`lru` 占用 24 字节记录最后访问时间或 LFU 访问频次。

#### Card 16. 随机采样近似 LRU 淘汰算法 (Approximated LRU Eviction)
*   **淘汰链全局锁屏障**：传统的 LRU 淘汰需要维护一个全局的双向链表，每次读取数据都要移动链表节点，这在高并发下会产生巨大的锁冲突。
*   **常数时间常数采样**：Redis 开启 `maxmemory` 后，随机抽取 $N$（默认 5）个键，根据 `lru` 字段计算它们的闲置时间，淘汰闲置时间最长的键，逼近全局 LRU 且免除了全局链表锁。

#### Card 17. UNLINK 异步非阻塞内存释放 (Lazy Freeing via UNLINK)
*   **主线程时延释放**：传统的 `DEL` 指令在删除含有数百万元素的集合时，会在主线程中释放大量物理内存，导致整个数据库瞬时挂起。
*   **后台线程 bio 搬运**：使用 `UNLINK` 仅在 dict 中移除键，将实际的物理内存指针交给后台 `bio.c` 线程池去执行物理 Free，保障了主循环的微秒级平滑。

---

### M5：高可用复制与分片 (Indigo)

#### Card 18. 主从复制复制积压缓冲区与 PSYNC (Master-Replica Replication)
*   **复制偏移量对齐**：主库和从库分别维护复制偏移量 `offset`。
*   **积压环形缓冲区**：主库维护一个固定大小的 `repl_backlog_buffer`（复制积压缓冲区）环形 FIFO 队列。断线重连时，从库发送其 `offset`。如果该偏移量依然在主库环形缓冲区内，执行 `PSYNC` 增量同步，否则退化为全量 RDB 物理传输。

#### Card 19. Sentinel 故障判定与故障转移共识 (Sentinel Failover Consensus)
*   **三向判定转移**：
    1.  **sdown (主观下线)**：单个 Sentinel 节点心跳超时判定下线。
    2.  **odown (客观下线)**：多数 Sentinel 共同判定主观下线后，主库被确认为客观下线。
    3.  **选主 Failover**：Sentinel 集群通过 Raft 共识选举出 Leader 哨兵，Leader 负责将某个 Replica 晋升为 Master 并在集群中广播。

#### Card 20. Cluster Gossip 无中心元数据传播 (Cluster Gossip Protocol)
*   **集群拓扑发现**：Redis Cluster 节点间没有配置中心，而是使用 Gossip 协议。
*   **PING/PONG 握手**：节点之间定期互发 `PING/PONG` 包，携带自身状态以及集群中 1/10 节点的元数据信息，实现拓扑状态的异步自治收敛。

#### Card 21. 16384 哈希槽物理路由与 ASK/MOVED (Hash Slots Routing & Redirect)
*   **槽位空间分配**：集群将数据划分为 16384 个虚拟哈希槽（Slots）。
*   **跳转重定向**：客户端可以向任意节点发起请求。若目标槽位不在该节点上，节点返回 `MOVED <slot> <ip>:<port>` 强制客户端更新本地槽路由图。若处于槽迁移中，返回 `ASK` 临时跳转，保证了数据的迁移一致性。

#### Card 22. 事务机制 ACID 限界与 WATCH (Redis Transaction Limits)
*   **无回滚机制**：Redis 事务（`MULTI/EXEC`）仅提供一组命令的打包顺序执行，执行过程中若有单条命令出错，其他命令依然会继续执行，不支持 ACID 的 Atomicity 原子回滚。
*   **WATCH 乐观锁控制**：使用 `WATCH` 监控特定键，在 `EXEC` 之前如果有其他连接修改了该键，事务执行自动失败。

#### Card 23. Lua 脚本嵌入沙盒与原子执行原理 (Lua Script Atomicity)
*   **执行互斥锁机制**：Redis 嵌入了 Lua 虚拟机。任何 Lua 脚本在执行时都被视为一条单命令。
*   **执行独占防干扰**：脚本执行期间，主事件循环保持独占状态，任何其他客户端的网络读写都被挂起，保证了脚本内多步逻辑对内存操作的绝对原子性。

---

### M6：高阶优化与调试监控 (Antique Gold)

#### Card 24. Threaded I/O 多线程网络处理优化 (Threaded I/O in Redis 6.0)
*   **卸载 Socket 读写瓶颈**：随着万兆网卡普及，单线程 Socket 读写和 RESP 解析成为瓶颈。
*   **逻辑单线程核心**：Redis 6.0 引入多线程 I/O。多个 helper 线程并发从客户端 Socket 读取字节流并解析，写回时也是并发写回 Socket，但核心的数据结构修改执行仍然在主线程中顺序单线程执行，兼顾了安全与并发。

#### Card 25. 内存动态碎片整理机制原理 (Active Defragmentation)
*   **物理内存移动重组**：长期运行下 jemalloc 产生的碎片无法自动归还 OS。
*   **原地拷贝页对齐**：开启 active defragmentation 后，Redis 扫描内存，当发现碎小的分配块时，动态将其物理拷贝到一片连续的新地址，并更新原 dict 的数据指针，以此腾出整页物理空间归还 OS。

#### Card 26. 客户端缓存失效跟踪机制 (Client-Side Caching)
*   **缓存二级重定向**：配合 RESP3 协议实现。Redis 允许客户端在本地内存中缓存数据。
*   **失效推送控制**：主库维护一个 Tracking 列表记录哪些客户端读取了哪些键。一旦这些键发生修改，主库向对应的客户端发送 `Invalidation`（失效通知）消息，实现高效的多进程二级缓存同步。

#### Card 27. 慢日志排查与延迟医生诊断工具 (Slowlog & Latency Doctor)
*   **阻塞时间记录**：`slowlog` 记录的是命令在主事件循环中执行的净时间（不含网络 I/O 等待）。
*   **系统健康度诊断**：使用 `LATENCY DOCTOR` 命令可以分析系统各引擎的延迟峰值（如 fork 耗时、AOF 刷盘耗时、过期删除耗时），输出系统级诊断报告。

#### Card 28. 模块 API 物理拓展接口 (Redis Modules API)
*   **加载共享库注入**：Redis 提供模块系统。开发者可以使用 C/C++ 编写共享库并在启动时加载。
*   **原生性能拓展**：模块可以直接调用 Redis 内存核心，创建新的物理数据类型或实现高性能计算逻辑，绕过了 Lua 虚拟机的转换损耗。

---

## 🔬 辅助版块 Zone T：生产诊断与调试字典 (Zone T)

### T1 Redis 典型参数调优对照表

| 配置参数 / redis.conf 指令 | 经典建议数值 | 物理调优语义 |
| :--- | :--- | :--- |
| `maxmemory <bytes>` | `50% - 70% 物理内存` | 限制当前 Redis 实例能使用的最大物理内存。预留至少 30% 以上内存作为 AOF/RDB fork 时的 COW 缓冲。 |
| `hash-max-ziplist-entries` | `512 (根据内存余量调整)` | 当哈希表元素小于此限制且所有 Value 小于 `64` 字节时，底层自动采用 ziplist/listpack，可以压缩 5 倍以上空间。 |
| `repl-backlog-size` | `10 MB - 100 MB` | 环形复制积压缓冲区大小。大写入场景下建议调大，否则网络微小抖动就会导致从节点重新触发灾难性的全量同步。 |
| `activedefrag` | `yes` | 开启主动内存碎片整理。能够在不需要重启服务的情况下动态减少内存碎片占用。 |

### T2 经典性能监控与诊断 INFO 指令字典

*   **1. 诊断实时内存与碎片状态**
    ```shell
    -- 获取实时内存统计，重点关注 used_memory_rss (分配内存) 和 mem_fragmentation_ratio (碎片率)
    redis-cli INFO memory
    ```
*   **2. 分析主从复制与网络回滚偏移**
    ```shell
    -- 查看复制角色状态、偏移量 offset 增量，以及 backlog 是否溢出退化全量
    redis-cli INFO replication
    ```
*   **3. 实时慢命令采集排查**
    ```shell
    -- 查看最近 10 条执行时延超过 slowlog-log-slower-than 阈值的慢命令详情
    redis-cli SLOWLOG GET 10
    ```
