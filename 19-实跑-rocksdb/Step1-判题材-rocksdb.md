# Scoping & Categorization Document: 《facebook / rocksdb》 (嵌入式LSM存储引擎内幕)

本文件定义了《facebook / rocksdb》的底层存储分析与高密拆解框架，规划 28 张核心卡片在 6 大模块及 Zone T 辅助版块中的物理映射，并确立莫兰迪色彩体系与页面布局参数。

---

## 一、 核心元数据与物理页面预算

*   **目标书籍/项目**：[facebook / rocksdb](https://github.com/facebook/rocksdb) 嵌入式键值存储引擎项目与 LSM-Tree 存储结构
*   **拆书定位**：硬核存储引擎物理结构与性能调优版（专注于 WAL 写入流、SSTable 磁盘布局、Leveled Compaction 策略、事务并发控制与 write stall 调优）
*   **物理页面预算**：**精确 2 页 A4 横向** (Landscape A4, 297mm x 210mm)
*   **LaTeX 布局引擎**：`extarticle` + `multicol` + `tcolorbox` 三列流式自适应排布，0 溢出。
*   **字体配置**：
    *   主要中文字体：`Microsoft YaHei` (微软雅黑) 或 `SimSun` (宋体)
    *   等宽字体：`Courier New` 或 `Consolas`
    *   西文字体：`Arial` 或 `Times New Roman`

---

## 二、 莫兰迪存储引擎色彩系统 (Morandi Palette)

为每个模块定义一个低饱和度、高质感的莫兰迪色彩 Token，以在 LaTeX 海报和 HTML 互动沙盒中保持一致的视觉层级：

| 模块编码 | 模块名称 | 莫兰迪色系 | RGB / Hex 代码 | 视觉语义 |
| :--- | :--- | :--- | :--- | :--- |
| **M1** | **写入流与 MemTable (Write Flow)** | 苔绿 (Moss Green) | `#7D8F7B` / `RGB(125,143,123)` | WAL 追加写、MemTable 并发写入与刷盘 |
| **M2** | **SSTable 磁盘布局 (SSTable Layout)** | 石板蓝 (Slate Blue) | `#7A8B99` / `RGB(122,139,153)` | Data Block、布隆过滤器、索引块结构 |
| **M3** | **Compaction 压实机制 (Compaction)** | 陶土红 (Terracotta) | `#B58A7D` / `RGB(181,138,125)` | 空间与写放大、Leveled 与 FIFO 算法 |
| **M4** | **缓冲与缓存治理 (Block Cache)** | 靛青 (Indigo) | `#5F7582` / `RGB(95,117,130)` | Block Cache、行缓存、索引与过滤器常驻 |
| **M5** | **事务与并发控制 (Transactions)** | 梅玫瑰 (Plum Rose) | `#9E828A` / `RGB(158,130,138)` | MVCC 快照、WriteBatch 原子写、乐观/悲观锁 |
| **M6** | **运维调优与诊断 (Operations)** | 古董金 (Antique Gold) | `#BFA88F` / `RGB(191,168,143)` | 列族隔离、限速、SST 离线导入与调优 |

---

## 三、 L0 ~ L2 压缩阶梯设计

在海报页眉处放置“RocksDB 存储引擎第一性原理公理阶梯”：

*   **L0 一句话本质**：为了在非易失性随机读写受限的存储介质上实现极致写入吞吐，通过内存缓冲顺序追加写（MemTable/WAL）与后台分层归并合并（Compaction），提供基于快照隔离（Snapshot）的高性能并发读写状态机。
*   **L1 四句话逻辑**：
    1.  **WAL 追加写与并发 MemTable (M1)** 确保写入零磁盘寻道时延，多线程通过无锁跳表最大化写入并发。
    2.  **SSTable 磁盘分块与 Bloom Filter (M2)** 解决随机读瓶颈，利用二分前缀压缩与快速位图过滤排除无效检索。
    3.  **Leveled Compaction 压实机制 (M3)** 充当空间治理者，通过层级有序规避空间放大，在读写放大与空间放大间做极限折中。
    4.  **快照隔离与并发原子批处理 (M5)** 规避昂贵的锁开销，依靠 Sequence Number 构造零锁 MVCC 视图。
*   **L2 知识大图**：
    *   [M1: 用户写 $\rightarrow$ WAL $\rightarrow$ MemTable] $\rightarrow$ [Immutable Memtable $\rightarrow$ Background Flush] $\rightarrow$ [M2: L0 SSTable $\rightarrow$ M3: Leveled Compaction] $\rightarrow$ [M4: Block Cache] $\rightarrow$ [M5: MVCC 快照读] $\rightarrow$ [M6: 列族隔离与运维限速]

---

## 四、 28 张核心卡片物理映射表

每一张卡片代表一个高密度的系统设计考点或机制，包含中文表述与专业西文术语。

### M1：写入流与 MemTable (M1_1 至 M1_5) - 【苔绿】
*   **Card 1 (M1_1)**: 写入链路与 WAL 日志 (User Write, WAL 顺序追加, WriteOptions 同步刷盘参数)
*   **Card 2 (M1_2)**: MemTable 无锁跳表机制 (SkipList 物理拓扑, CAS 无锁并发写, Arena 内存分配器防碎片)
*   **Card 3 (M1_3)**: MemTable 物理分类选型 (VectorMemTab 批量载入, HashSkipList 配合前缀索引快速检索)
*   **Card 4 (M1_4)**: Immutable MemTable 与 Flush 刷盘 (活跃块写满冻结, 触发后台单线程顺序写入 L0 SSTable)
*   **Card 5 (M1_5)**: Write Stall 写入限速挂起机制 (L0 文件堆积/压实积压触发客户端写入限速, 避免 I/O 崩溃)

### M2：SSTable 磁盘布局 (M2_1 至 M2_4) - 【石板蓝】
*   **Card 6 (M2_1)**: SSTable 整体物理架构 (Block-Based Table 格式, Data Block 数据区, Index/Filter 元数据区, Footer 尾锚定)
*   **Card 7 (M2_2)**: Block 内部结构与前缀压缩 (Sorted Keys 前缀共享, Restart Points 重启点二分查找, Block Trailer 校验)
*   **Card 8 (M2_3)**: 布隆过滤器 (Bloom Filter) 布局 (Full Filter 全局过滤器 vs Block-based Filter 分块过滤器, 假阳性率公式)
*   **Card 9 (M2_4)**: Index Block 与二分检索 (块索引键范围定位, 块前缀哈希查找优化)

### M3：Compaction 算法 (M3_1 至 M3_5) - 【陶土红】
*   **Card 10 (M3_1)**: Size-Tiered Compaction 机制 (相似大小 SST 合并, 写入放大低, 空间放大极高, 临时空间翻倍隐患)
*   **Card 11 (M3_2)**: Leveled Compaction 机制 (分层有序, 层级间 10x 大小比率, 写入放大高, 空间放大极低, 单点覆盖)
*   **Card 12 (M3_3)**: Compaction Pick 压实选择策略 (Score 计分公式, L0 文件数触发 vs 其它层 size 溢出触发, 动态层级大小限制)
*   **Card 13 (M3_4)**: Write/Space/Read Amplification 放大定理 (LSM 引擎的核心折中, Compaction 回垦开销)
*   **Card 14 (M3_5)**: FIFO Compaction 与时序存储 (删除最老 SST, 零写放大, 适合定长缓存与监控指标存储)

### M4：缓冲与缓存治理 (M4_1 至 M4_4) - 【靛青】
*   **Card 15 (M4_1)**: Block Cache 块缓存架构 (LRUCache 与 ClockCache, ShardedCache 分片设计避开全局互斥锁竞争)
*   **Card 16 (M4_2)**: Compressed Block Cache 选型 (解压后 C++ 对象缓存 vs 原始压缩数据块缓存, CPU 与内存的二次权衡)
*   **Card 17 (M4_3)**: Row Cache 行缓存机制 (直接缓存 Key-Value 对, 绕过 Block 索引检索直接命中热点行)
*   **Card 18 (M4_4)**: Pinning 锁定机制 (Index/Filter 内存锁定常驻, 规避数据页置换带来的随机磁盘 I/O)

### M5：事务与并发控制 (M5_1 至 M5_5) - 【梅玫瑰】
*   **Card 19 (M5_1)**: ReadOptions Snapshot 快照机制 (Sequence Number 版本标记, 读绕过锁实现最终 MVCC)
*   **Card 20 (M5_2)**: WriteBatch 原子写机制 (单锁批量写入, Leader 线程代表写入 WAL, 多线程并发写 MemTable 状态收敛)
*   **Card 21 (M5_3)**: Optimistic Transaction 乐观事务 (记录起始 SeqNum, 提交时进行因果检测, 冲突回滚冲突链)
*   **Card 22 (M5_4)**: Pessimistic Transaction 悲观事务 (LockManager 行级锁, 共享/排他锁矩阵, 死锁等待检测图)
*   **Card 23 (M5_5)**: Two-Phase Commit (2PC) 事务 (两阶段提交, Prepare 标记写入 WAL, 跨引擎协调)

### M6：运维调优与诊断 (M6_1 至 M6_5) - 【古董金】
*   **Card 24 (M6_1)**: Column Families 列族隔离 (逻辑隔离数据库, 共享 WAL 保障恢复原子性, 独立 MemTable/Compaction)
*   **Card 25 (M6_2)**: SST File Writer 离线批量导入 (Bulk Loading 机制, 绕过写路径直接生成 SST 并 Ingest 导入)
*   **Card 26 (M6_3)**: Rate Limiter 磁盘吞吐限速 (配置 Flush 与 Compaction 最大写带宽, 稳定前端读写延迟波动)
*   **Card 27 (M6_4)**: Stats Dump 性能指标审计 (读取 RocksDB 打印日志, 诊断 Write Stall 耗时与 Cache 命中率)
*   **Card 28 (M6_5)**: 表空间损坏与 MANIFEST 修复 (MANIFEST 版本控制原理, 崩溃恢复流程)

---

## 五、 辅助版块 Zone T 设计

作为 cheatsheet 底部或侧边的实用工具箱，为存储引擎调优提供即时数值与参数参考：

1.  **T1 RocksDB 核心参数调优速查**：
    *   `write_buffer_size`: 单个 MemTable 大小，推荐 64MB。
    *   `max_write_buffer_number`: 最大 Immutable MemTable 数，推荐 4-6。
    *   `level0_file_num_compaction_trigger`: L0 触发压实文件数，推荐 4。
    *   `max_bytes_for_level_base`: L1 最大容量，推荐 256MB。
2.  **T2 性能放大调优金标准**：
    *   *写放大 (WA) 指标* -> 内存追加写为 1 | Leveled Compaction 生产环境下通常为 10-30。
    *   *内存配比* -> Block Cache 推荐分配物理内存的 30%~50%，列族多时需设置 `WriteBufferManager` 全局限制。
3.  **T3 Write Stall 故障排查三步法**：
    *   *检测* -> 查看 Stats Dump 输出中 `Stall Time` 是否大于零。
    *   *根因* -> L0 文件数超限（默认 20 减速，30 彻底挂起）| 待压实字节数超限。
    *   *防线* -> 调大 `max_background_jobs`（Compaction 线程池） | 优化磁盘吞吐。
