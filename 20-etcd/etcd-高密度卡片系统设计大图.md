# 《etcd-io / etcd》高密度卡片系统设计大图

本设计大图为《etcd-io / etcd》的分布式强一致性协调与系统设计高密度拆解卡片设计指南。我们将 28 张核心速查卡片划分为六大核心模块，每个模块采用低饱和度的莫兰迪（Morandi）色彩进行视觉归类，并设计了其拓扑交互图与物理源头锚点。

---

## 🎨 莫兰迪内核诊断视觉配色方案 (Morandi Color System)

为保证排版的高级感与学术硬核感，采用低饱和度、高质感的莫兰迪色彩体系：

| 模块编码 | 模块名称 | 莫兰迪色系 | 浅色底色 (Light Mode) | 深色边框 / 文字 (Dark Mode) | 对应设计领域 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **M1** | Raft 共识与日志复制 | 石板蓝 (Slate Blue) | `#F0F3F5` / `#D2DBE0` | `#4E5D6C` / `#2F3C47` | 选主防撞、日志状态追加、分区防御、成员共识变更 |
| **M2** | WAL 预写日志与存储 | 苔绿 (Moss Green) | `#F2F4F0` / `#D5DDD1` | `#5F6C5B` / `#3A4438` | WAL 预分配与顺序写、BoltDB 树结构、持久化提交屏障 |
| **M3** | MVCC 多版本控制 | 梅玫瑰 (Plum Rose) | `#F5F0F2` / `#E0D2D7` | `#6F525A` / `#4A353A` | 全局 Revision 事务定位、内存 KeyIndex 索引映射、历史整理 |
| **M4** | Lease 租约与生存期 | 陶土红 (Terracotta) | `#F5F1EF` / `#E0D3CD` | `#793C2C` / `#522114` | Lease ID 堆排序、KeepAlive 双向流、级联过期、共享租约 |
| **M5** | Watch 异步事件流 | 靛青 (Indigo) | `#F0F2F5` / `#D1D8E0` | `#3E4C5B` / `#232F3C` | synced/unsynced 双队列扫描、背压流控、事件去重与断点 |
| **M6** | 集群变更与强一致读 | 古董金 (Antique Gold) | `#F6F4EE` / `#E3DEC8` | `#8C7344` / `#5C4A28` | gRPC 多路复用、ReadIndex/LeaseRead 强一致读、负载容灾 |

---

## 🗺️ 28张高密速查卡片大图拓扑 (Card Topology)

```mermaid
graph TD
    subgraph M1_Raft ["M1: Raft 共识复制 (Slate Blue)"]
        C01["Card 01: Raft 状态机与任期机制"]
        C02["Card 02: 领导者选举与随机超时"]
        C03["Card 03: 日志复制与两阶段提交"]
        C04["Card 04: 网络分区与 Pre-Vote"]
        C05["Card 05: 动态变更与联合共识"]
    end

    subgraph M2_Storage ["M2: WAL & BoltDB (Moss Green)"]
        C06["Card 06: WAL 追加与预分配"]
        C07["Card 07: BoltDB B+树物理文件"]
        C08["Card 08: WAL 与 BoltDB 提交顺序"]
        C09["Card 09: 数据快照与物理日志截断"]
    end

    subgraph M3_MVCC ["M3: MVCC 多版本 (Plum Rose)"]
        C10["Card 10: Revision 双版本定位号"]
        C11["Card 11: KeyIndex 内存 B-Tree"]
        C12["Card 12: MVCC 快照隔离只读事务"]
        C13["Card 13: 历史压缩与 BoltDB 清洗"]
    end

    subgraph M4_Lease ["M4: Lease 租约 (Terracotta)"]
        C14["Card 14: Lease 动态堆排序结构"]
        C15["Card 15: KeepAlive 双向流续约"]
        C16["Card 16: 级联过期键回收清理"]
        C17["Card 17: 多键共享租约优化"]
        C18["Card 18: 租约分布式锁与防死锁"]
    end

    subgraph M5_Watch ["M5: Watch 监听器 (Indigo)"]
        C19["Card 19: gRPC 多路复用订阅流"]
        C20["Card 20: 同步与未同步双队列扫描"]
        C21["Card 21: 事件缓冲区与背压流控"]
        C22["Card 22: 事件合并与物理去重"]
        C23["Card 23: 传入 Revision 断点续传"]
    end

    subgraph M6_Ops ["M6: 强读与集群运维 (Antique Gold)"]
        C24["Card 24: Protobuf 与 gRPC v3 架构"]
        C25["Card 25: ReadIndex 线性一致性读"]
        C26["Card 26: LeaseRead 物理有效期加速"]
        C27["Card 27: 客户端负载均衡与容灾漂移"]
        C28["Card 28: Etcd Gateway 连接数保护"]
    end

    M1_Raft -->|确保日志日志流顺序追加| M2_Storage
    M2_Storage -->|BoltDB 提交后通知写入| M3_MVCC
    M3_MVCC -->|更新包含 Revision 的 KV 记录| M5_Watch
    M4_Lease -->|过期触发清理| M3_MVCC
    M5_Watch -->|基于全局 Revision 响应监听器| M6_Ops
    M6_Ops -->|线性一致读询问 Raft 节点状态| M1_Raft
```

---

## ⚡ 物理代码与规范源头锚点 (Physical Source Anchors)

本设计大图与 etcd 开源项目的物理代码路径映射如下：
1. **Raft 状态机与日志流**：映射 `server/etcdserver/raft.go` 以及底层依赖的 `go.etcd.io/raft` 包。核心关注 `raft.Step` 状态转移入口、`raftLog` 内存日志缓冲。
2. **WAL 物理预写日志追加**：映射 `server/storage/wal/wal.go`，重点关注 `wal.Save` 物理刷盘操作，以及文件预分配（File Preallocation）如何调用 `fallocate` 规避文件系统磁盘分配抖动。
3. **BoltDB B+树文件调度**：映射 `server/storage/backend/backend.go` 以及 `go.etcd.io/bbolt` 的核心事务提交逻辑，分析只读事务（Read Tx）与读写事务（Batch Tx）的全局缓冲隔离。
4. **KeyIndex 内存索引查找**：映射 `server/mvcc/key_index.go`，重点关注 `keyIndex` 结构体的物理实现，`generations` 数组存储生命周期，以及利用 `go-adaptive-radix-tree` 或内存 B-Tree 进行键名匹配的算法。
5. **LeaseManager 租约堆排调度**：映射 `server/lease/leaseid.go` 与 `server/lease/lessor.go`，分析租约到期轮询线程（Lessor Expired Channel）、最小堆到期判断时间复杂度，以及 gRPC 流式多路复用保活机制。
6. **WatchableStore 双队列流控**：映射 `server/mvcc/watchable_store.go`，关注同步队列（synced）与未同步队列（unsynced）如何通过 `syncWatchersLoop` 进行滑动窗口补发，以及背压控制机制。
7. **ReadIndex 一致性读广播**：映射 `server/etcdserver/linearizer.go` 及 `server/etcdserver/v3_actions.go`，追踪 ReadIndex 如何获取 Leader 确认并等待本地应用指针（ApplyIndex）追上 ReadIndex 的调用图。
