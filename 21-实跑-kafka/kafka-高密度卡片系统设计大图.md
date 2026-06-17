# 《apache / kafka-internals》高密度卡片系统设计大图

本设计大图为《apache / kafka-internals》的分布式高吞吐事件存储与系统设计高密度拆解卡片设计指南。我们将 28 张核心速查卡片划分为六大核心模块，每个模块采用低饱和度的莫兰迪（Morandi）色彩进行视觉归类，并设计了其拓扑交互图与物理源头锚点。

---

## 🎨 莫兰迪内核诊断视觉配色方案 (Morandi Color System)

为保证排版的高级感与学术硬核感，采用低饱和度、高质感的莫兰迪色彩体系：

| 模块编码 | 模块名称 | 莫兰迪色系 | 浅色底色 (Light Mode) | 深色边框 / 文字 (Dark Mode) | 对应设计领域 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **M1** | 日志存储与索引 | 石板蓝 (Slate Blue) | `#F0F3F5` / `#D2DBE0` | `#4E5D6C` / `#2F3C47` | 日志分段、稀疏索引二分查找、TimeIndex 过期清理、Log Compaction 压实 |
| **M2** | 页缓存与零拷贝 | 苔绿 (Moss Green) | `#F2F4F0` / `#D5DDD1` | `#5F6C5B` / `#3A4438` | OS Page Cache 物理写入、Sendfile 零拷贝 DMA 链路、脏页回写参数调优 |
| **M3** | 复制同步与 ISR | 梅玫瑰 (Plum Rose) | `#F5F0F2` / `#E0D2D7` | `#6F525A` / `#4A353A` | ISR 动态变化管理、HW 与 LEO 物理推进边界、Leader Epoch 崩溃日志对齐 |
| **M4** | 协调器与 KRaft | 陶土红 (Terracotta) | `#F5F1EF` / `#E0D3CD` | `#793C2C` / `#522114` | Controller 控制器选举、KRaft 共识替换 ZK 架构、状态机模型与路由同步 |
| **M5** | 生产者批处理 | 靛青 (Indigo) | `#F0F2F5` / `#D1D8E0` | `#3E4C5B` / `#232F3C` | BufferPool 内存分配、Sender 线程网络合并、粘性分区与幂等事务写入 |
| **M6** | 消费者组再平衡 | 古董金 (Antique Gold) | `#F6F4EE` / `#E3DEC8` | `#8C7344` / `#5C4A28` | Group Coordinator 映射选举、Join/Sync 重平衡流、增量平滑再平衡机制 |

---

## 🗺️ 28张高密速查卡片大图拓扑 (Card Topology)

```mermaid
graph TD
    subgraph M5_Producer ["M5: 生产者批处理内存 (Indigo)"]
        C19["Card 19: RecordAccumulator 与内存池"]
        C20["Card 20: 批量发送 Sender 循环"]
        C21["Card 21: 粘性分区缓存调度"]
        C22["Card 22: PID 与 Sequence 幂等性"]
        C23["Card 23: 事务批两阶段提交控制"]
    end

    subgraph M2_IO ["M2: OS 缓存与零拷贝 (Moss Green)"]
        C06["Card 06: Page Cache 物理缓冲区追加"]
        C07["Card 07: Sendfile 零拷贝 DMA 直发"]
        C08["Card 08: 磁盘顺序写寻道消除"]
        C09["Card 09: 脏页回写参数调优"]
    end

    subgraph M1_Storage ["M1: 日志分段存储 (Slate Blue)"]
        C01["Card 01: 日志分段物理布局"]
        C02["Card 02: 稀疏索引快速检索"]
        C03["Card 03: TimeIndex 时间轴定位"]
        C04["Card 04: Log Compaction 与墓碑"]
        C05["Card 05: Checkpoint 与崩溃重建"]
    end

    subgraph M3_Replication ["M3: 副本同步与 ISR (Plum Rose)"]
        C10["Card 10: ISR 成员生死判死"]
        C11["Card 11: HW 与 LEO 水位线物理流"]
        C12["Card 12: Leader Epoch 日志安全对齐"]
        C13["Card 13: 最少同步副本硬约束限制"]
    end

    subgraph M4_KRaft ["M4: 控制器元数据 (Terracotta)"]
        C14["Card 14: Controller 选举与 Epoch 防脑裂"]
        C15["Card 15: KRaft 内置共识机制控制流"]
        C16["Card 16: 分区生命周期状态机"]
        C17["Card 17: 副本生命周期状态机"]
        C18["Card 18: 元数据异步广播同步"]
    end

    subgraph M6_Consumer ["M6: 消费组与 Rebalance (Antique Gold)"]
        C24["Card 24: Coordinator 偏移哈希映射"]
        C25["Card 25: JoinGroup 与 SyncGroup 重分配"]
        C26["Card 26: 心跳保活与活性判死阈值"]
        C27["Card 27: Offsets 物理存储与自动提交"]
        C28["Card 28: Cooperative Sticky 合作式增量重平衡"]
    end

    M5_Producer -->|利用客户端 batch 聚合数据写入| M2_IO
    M2_IO -->|顺序落盘写入| M1_Storage
    M1_Storage -->|日志副本主动拉取复制| M3_Replication
    M3_Replication -->|高水位对齐与汇报| M4_KRaft
    M4_KRaft -->|下发元数据状态变更| M6_Consumer
    M6_Consumer -->|读取 Offset 并触发数据拉取| M2_IO
```

---

## ⚡ 物理代码与规范源头锚点 (Physical Source Anchors)

本设计大图与 Kafka 开源项目的物理代码路径映射如下：
1. **日志存储与物理索引**：映射 `core/src/main/scala/kafka/log/LocalLog.scala` 以及 `core/src/main/scala/kafka/log/AbstractIndex.scala`。核心关注稀疏索引物理写入控制、`.index` 和 `.timeindex` 文件的物理偏移计算。
2. **零拷贝文件通道发送**：映射 `clients/src/main/java/org/apache/kafka/common/network/FileChannelSend.java`，利用 Java NIO 的 `transferTo` 调用，底层映射操作系统的 `sendfile` 系统调用。
3. **ISR 动态调度与 HW 推进**：映射 `core/src/main/scala/kafka/cluster/Partition.scala`，分析 `maybeShrinkIsr()` 自动缩容判定、Follower 获取最新日志进度后 Leader 推进 High Watermark 的调用流程。
4. **Leader Epoch 日志一致性**：映射 `core/src/main/scala/kafka/server/epoch/LeaderEpochFileCache.scala`，跟踪 Leader Epoch 在重启和切主后如何防止历史未持久化日志产生 HW 覆盖。
5. **生产者 RecordAccumulator 与内存池**：映射 `clients/src/main/java/org/apache/kafka/clients/producer/internals/RecordAccumulator.java` 及 `BufferPool.java`，重点阅读以 `Page (默认32KB)` 分配与回收的内存池机制。
6. **消费者二阶段 Rebalance 协议**：映射 `core/src/main/scala/kafka/coordinator/group/GroupCoordinator.scala`，重点阅读 `handleJoinGroup()` 与 `handleSyncGroup()` 的处理时序，以及消费者偏移量存储管理器 `OffsetManager` 逻辑。
