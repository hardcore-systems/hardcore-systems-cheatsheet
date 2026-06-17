# Scoping & Categorization Document: 《etcd-io / etcd》 (分布式协调与强一致性存储)

本文件定义了《etcd-io / etcd》的底层分析与高密拆解框架，规划 28 张核心卡片在 6 大模块及 Zone T 辅助版块中的物理映射，并确立莫兰迪色彩体系与页面布局参数。

---

## 一、 核心元数据与物理页面预算

*   **目标书籍/项目**：[etcd-io / etcd](https://github.com/etcd-io/etcd) 开源项目与分布式共识系统
*   **拆书定位**：强一致性协同底座与分布式存储版（专注于 Raft 状态机日志流、WAL 持久化保障、BoltDB 磁盘存储、MVCC 版本索引、Lease 租约生命周期以及 Watch 异步时间驱动机制）
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
| **M1** | **Raft 共识与日志复制 (Raft Consensus)** | 石板蓝 (Slate Blue) | `#7A8B99` / `RGB(122,139,153)` | Raft 选主、日志复制、网络分区与 Pre-Vote |
| **M2** | **WAL 预写日志与存储 (WAL & BoltDB)** | 苔绿 (Moss Green) | `#7D8F7B` / `RGB(125,143,123)` | WAL 持久化、BoltDB B+树文件结构、Commit 屏障 |
| **M3** | **MVCC 多版本并发控制 (MVCC & Index)** | 梅玫瑰 (Plum Rose) | `#9E828A` / `RGB(158,130,138)` | Revision 全局递增、内存 KeyIndex 索引与历史压缩 |
| **M4** | **Lease 租约与生存期管理 (Lease & TTL)** | 陶土红 (Terracotta) | `#B58A7D` / `RGB(181,138,125)` | 租约动态绑定、KeepAlive 心跳续约、分布式锁实现 |
| **M5** | **Watch 异步事件流 (Watch & Streaming)** | 靛青 (Indigo) | `#5F7582` / `RGB(95,117,130)` | WatchableStore 同步与未同步队列、事件缓存与背压 |
| **M6** | **集群变更与强一致读 (Ops & Concurrency)** | 古董金 (Antique Gold) | `#BFA88F` / `RGB(191,168,143)` | 线性一致性读 (ReadIndex/LeaseRead)、动态配置、gRPC v3 架构 |

---

## 三、 L0 ~ L2 压缩阶梯设计

在海报页眉处放置“etcd 第一性原理公理阶梯”，作为阅读的心智锚点：

*   **L0 一句话本质**：通过 Raft 协议在分布式节点间维护强一致性的复制状态机，配合底层的 MVCC 多版本存储与 BoltDB，为云原生架构提供高可用、强一致的元数据协调与服务发现底座。
*   **L1 四句话逻辑**：
    1.  **Raft 复制状态机** (M1) 保证了多节点间强一致性的日志追加与状态提交，通过 ReadIndex/LeaseRead 规避脑裂期间的脏读。
    2.  **WAL 与嵌入式 BoltDB** (M2) 是底层的物理存储支柱，WAL 顺序刷盘保证日志不丢，BoltDB 的 B+ 树则维护了 KV 状态。
    3.  **MVCC 与 KeyIndex** (M3) 实现了事务隔离与历史版本追溯，通过 Revision 递增版本控制解决写写冲突并支持历史压缩。
    4.  **Lease 与 Watch 机制** (M4 & M5) 提供了动态租约与异步事件流订阅，是分布式协调、分布式锁与配置下发的神经元枢纽。
*   **L2 知识大图**：
    *   [M1: Raft 共识复制] -> [M2: WAL持久化/BoltDB] -> [M3: MVCC/KeyIndex多版本] -> [M4: Lease 租约租期] -> [M5: Watch 异步监听] -> [M6: 读写吞吐/ReadIndex强读/集群管理]

---

## 四、 28 张核心卡片物理映射表

每一张卡片代表一个高密度的系统设计考点或机制，包含中文表述与专业西文术语。

### M1：Raft 共识与日志复制 (M1_1 至 M1_5) - 【石板蓝】
*   **Card 1 (M1_1)**: Raft 状态机与任期机制 (Raft State Machine & Term, Leader/Follower/Candidate 转换逻辑)
*   **Card 2 (M1_2)**: 领导者选举与心跳对齐 (Leader Election & Heartbeats, 随机选举超时 Random Election Timeout 避撞)
*   **Card 3 (M1_3)**: 日志复制与两阶段提交 (Log Replication & Commit, MatchIndex/NextIndex 物理维护)
*   **Card 4 (M1_4)**: 网络分区与 Pre-Vote 机制 (Pre-Vote Phase, 规避网络分区引起无意义的 Term 自增)
*   **Card 5 (M1_5)**: 集群成员动态变更 (Membership Changes, 单节点变更与 Joint Consensus 联合共识防脑裂)

### M2：WAL 预写日志与存储 (M2_1 至 M2_4) - 【苔绿】
*   **Card 6 (M2_1)**: WAL 预写日志顺序追加结构 (WAL Append-Only Structure, 64MB 文件空间预分配与写对齐)
*   **Card 7 (M2_2)**: BoltDB/bbolt 嵌入式存储引擎 (bbolt B+ Tree structure, 叶子节点数据排布与只读事务)
*   **Card 8 (M2_3)**: WAL 与 BoltDB 提交顺序屏障 (WAL Pre-write & BoltDB Commit Sequence, fsync 保证持久化屏障)
*   **Card 9 (M2_4)**: 快照机制与日志历史截断 (Snapshotting & WAL Truncation, 内存状态快闪与物理日志回收)

### M3：MVCC 多版本并发控制 (M3_1 至 M3_4) - 【梅玫瑰】
*   **Card 10 (M3_1)**: Revision 全局递增序列号 (Global Revision, Main/Sub 双版本号定位事务内修改)
*   **Card 11 (M3_2)**: KeyIndex 内存 B-Tree 索引 (KeyIndex struct, 键名到多版本 Revision 的内存映射关系)
*   **Card 12 (M3_3)**: MVCC 快照读隔离与 Range 查询 (MVCC Snapshot Isolation, 基于特定 Revision 的只读事务)
*   **Card 13 (M3_4)**: 历史版本压缩与空间整理 (Compaction & DB Defragmentation, 物理清除历史 Revision 释放 BoltDB 页)

### M4：Lease 租约与生存期管理 (M4_1 至 M4_5) - 【陶土红】
*   **Card 14 (M4_1)**: Lease 租约管理器物理结构 (LeaseManager, 最小堆优先队列维护 TTL 与 Lease ID 映射)
*   **Card 15 (M4_2)**: KeepAlive 续约与 gRPC 双向流 (Lease KeepAlive, 客户端双向流式续约降低网络连接开销)
*   **Card 16 (M4_3)**: 租约撤销与键联级清理 (Lease Revocation, 关联键物理过期与内存清理事件通知)
*   **Card 17 (M4_4)**: 租约共享与可扩展性设计 (Lease Sharing, 多个键绑定同一租约规避独立心跳风暴)
*   **Card 18 (M4_5)**: 基于租约的分布式锁 (Distributed Locks via Lease, 锁自动续期与超时自动释放防死锁)

### M5：Watch 异步事件流 (M5_1 至 M5_5) - 【靛青】
*   **Card 19 (M5_1)**: Watcher 与 WatchStream 订阅管道 (Watcher & WatchStream, gRPC 多路复用事件推送机制)
*   **Card 20 (M5_2)**: WatchableStore 同步与未同步队列 (WatchableStore, synced/unsynced watcher 双向队列滑动扫描)
*   **Card 21 (M5_3)**: 事件缓冲区与背压流控 (Event Buffer & Flow Control, 缓冲区溢出保护与客户端慢消费降级)
*   **Card 22 (M5_4)**: Watch 事件合并与去重机制 (Event Consolidation, 极短时间内多次修改的事件物理聚合)
*   **Card 23 (M5_5)**: 基于历史 Revision 的断点续传 (Watch Resume, 支持传入 Revision 追溯历史事件通道)

### M6：集群变更与强一致读 (M6_1 至 M6_5) - 【古董金】
*   **Card 24 (M6_1)**: gRPC v3 强协议架构设计 (gRPC Protocol v3, 基于 HTTP/2 的底层多路复用与 Protobuf 序列化)
*   **Card 25 (M6_2)**: Linearizable 线性一致读与 ReadIndex (Linearizable Read, 规避 Raft 写盘的 ReadIndex 广播确认)
*   **Card 26 (M6_3)**: LeaseRead 租约强一致读优化 (LeaseRead, 依靠 Leader 逻辑租期有效期内直接本地读)
*   **Card 27 (M6_4)**: 客户端智能负载均衡与容灾 (Client Load Balancing, Endpoint 动态感知与透明故障重试)
*   **Card 28 (M6_5)**: Etcd 网关与代理转发模式 (Etcd Gateway & L4 Proxy, 边缘流量聚合与连接数保护)

---

## 五、 辅助版块 Zone T 设计

作为 cheatsheet 底部或侧边的实用工具箱，为系统工程诊断提供即时数值与参数参考：

1.  **T1 etcd 关键配置调优阈值表**：
    *   `--max-request-bytes` (默认 1.5MB)：客户端单个请求的最大字节限制。
    *   `--quota-backend-bytes` (默认 2GB，最高建议 8GB)：存储底座配额大小，超限即触发 DB Alter 报警并写挂起。
    *   `--heartbeat-interval` (建议 100ms - 250ms)：Raft 心跳间隔周期。
    *   `--election-timeout` (建议 1000ms - 3000ms)：选主超时周期（必须为心跳间隔的 5 - 10 倍以防抖动）。
2.  **T2 etcdctl 生产调试高频指令字典**：
    *   读写检测：`etcdctl put /key "val"`, `etcdctl get /key --rev=10`
    *   范围监听：`etcdctl watch /key --prefix`
    *   租约维护：`etcdctl lease grant 60`, `etcdctl lease keepalive <lease_id>`
    *   状态监控：`etcdctl endpoint status --write-out=table`
    *   灾难备份：`etcdctl snapshot save backup.db`
