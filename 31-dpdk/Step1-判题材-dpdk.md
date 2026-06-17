# Step 1: 判题材 - DPDK / dpdk (Data Plane Development Kit)

## 1. 题材深度分析与判定

`DPDK` (Data Plane Development Kit) 是现代高性能网络处理与数据平面的行业基石与核心底座。它打破了传统操作系统内核协议栈的层层包处理限制，通过内核旁路 (Kernel Bypass)、用户态 PCI 驱动 (UIO/VFIO)、轮询模式驱动 (PMD) 以及精妙的大页内存管理与无锁环形队列，将每秒数千万个数据包 (Mpps) 的处理延迟降至微秒级，吞吐推向物理链路极限（如 100GbE+）。
DPDK 的架构设计将计算机体系结构（CACHE 伪共享、NUMA 拓扑、TLB 抖动）与数据结构（无锁 CAS 队列、内存对齐）压榨到了极致，是系统工程领域无可争议的硬核题材。

## 2. 核心架构与 6 大模块划分

为契合 A4 双页 Landscape 布局，我们将本项目划分为 6 个核心技术模块，每个模块对应一个莫兰迪色块（M1 ~ M6）：

*   **M1: 内核旁路与用户态驱动** (Slate Blue - `#7A8B99`)
    *   UIO (Userspace I/O) 与 VFIO (Virtual Function I/O) 旁路机制、IOMMU 硬件安全页映射、用户态 PCI 总线扫描与网卡设备绑定、用户态内存空间映射设备 BAR 空间 (mmap)。
*   **M2: 无锁环形队列与内存池** (Moss Green - `#7D8F7B`)
    *   `rte_ring` 无锁环形队列实现、单/多生产者与单/多消费者 CAS (Compare-And-Swap) 并发冲突处理、`rte_mempool` 数据池管理、本地核心缓存 (Lcore Local Cache) 绕过全局队列锁。
*   **M3: 大页内存与 NUMA 亲和性** (Plum Rose - `#9E828A`)
    *   Hugepages 大页减少 TLB Miss 频率与页表项级数、物理连续内存管理、NUMA 架构本地内存分配、Cacheline 对齐与结构体 Padding 消除伪共享 (False Sharing) 影响。
*   **M4: 轮询驱动 (PMD) 与 Burst I/O** (Terracotta - `#B58A7D`)
    *   Poll Mode Driver (PMD) 机制消除中断上下文切换开销、物理 RX/TX 环形描述符队列管理、`rte_eth_rx_burst` 与 `rte_eth_tx_burst` 批量收发数据包、CPU 核心独占绑定。
*   **M5: mbuf 报文结构与数据流转管线** (Indigo - `#5F7582`)
    *   `rte_mbuf` 结构体精密排版（Headroom & Tailroom 及元数据对齐）、mbuf 引用计数与多播克隆、Run-to-completion（运行至完成）与 Pipeline（流水线）线程分工模式、网络访问控制列表 (ACL)。
*   **M6: 硬件网卡卸载与流量分流** (Antique Gold - `#BFA88F`)
    *   接收端接收侧缩放 RSS (Receive Side Scaling) 网卡硬件负载均衡、流向引导 (Flow Director / FDIR) 硬件级精准路由规则、IP/TCP/UDP 校验和 (Checksum) 硬件卸载、TSO (TCP Segmentation Offload) 分段卸载、虚拟化 SR-IOV 硬件直通。

---

## 3. L0 ~ L2 知识阶梯

### L0 一句话本质
DPDK 是一个网络数据平面零拷贝与极低延迟的开发套件，它通过内核旁路与用户态 PCI 驱动剥离中断，利用大页内存及无锁环形队列消解上下文切换与内存锁竞争，依托 CPU 独占核心的轮询模式（PMD）以批量（Burst）读写方式直通网卡，实现硬件级的线速数据包吞吐。

### L1 四句话逻辑
1.  **内核旁路与驱动用户态化**：借由 UIO/VFIO 将网卡 PCI 控制权及物理内存直接映射至用户空间，让数据包直接通过 DMA 写入用户内存，彻底绕过操作系统的内核态协议栈与内存拷贝。
2.  **大页内存与 NUMA 锁消解**：通过 2MB/1GB 大页内存最大化 TLB 命中率，并将数据池分配与处理线程绑定在同一个 NUMA 节点的物理 CPU 与物理内存上，利用无锁 ring 并发环及 Local Cache 避免全局竞争。
3.  **批量轮询与独占核心驱动**：开启轮询驱动（PMD）不间断查询网卡收发环形队列，消除网络包引发的中断风暴与 CPU 调度上下文开销，并通过 `Burst` 批量接口一次性收发 32 个包，均摊处理路径开销。
4.  **精细控制与网卡硬件卸载**：通过优化设计的 `rte_mbuf` 紧凑结构实现报文生命周期管理，并利用网卡硬件 ASIC 执行 RSS 多队列分流、FDIR 过滤和 Checksum/TSO 计算，最大程度减轻 CPU 负载。

### L2 核心数据流转拓扑
```
  [Physical Fiber (Line Rate)]
              │
              ▼
  [NIC Physical RX Ring Buffer] (DMA Transfer)
              │
              ▼ (Zero-Copy Transfer to Userspace)
  [Hugepage Memory (NUMA Node Local)] <───> [rte_mempool Allocation (Lcore Local Cache)]
              │
              ▼ (Batch Packet Poll)
  [rte_eth_rx_burst(rx_pkts, 32)] <───> [Poll Mode Driver (PMD) pinned on Dedicated Core]
              │
              ▼
    [rte_mbuf Packet parsing]
              │
      ┌───────┴───────┐ (Processing Models)
      ▼               ▼
[Run-to-completion] [Pipeline (worker cores via rte_ring)]
      │               │
      └───────┬───────┘
              ▼
   [Hardware Offload (RSS / Checksum / FDIR)]
              │
              ▼
  [NIC Physical TX Ring Buffer] (DMA Transfer)
              │
              ▼
  [Physical Fiber (Line Rate)]
```

---

## 4. 28张卡片大纲规划

### Page 1 (内核旁路、无锁队列与内存管理)

#### M1: 内核旁路与用户态驱动 (Cards 1-4)
*   **Card 1**: UIO 简单旁路与 VFIO 安全旁路（IOMMU 硬件隔离）对比
*   **Card 2**: BAR (Base Address Register) 空间用户态 mmap 映射与 PCI 物理读写
*   **Card 3**: DPDK 设备初始化、PCI 网卡驱动绑定与 EAL (Environment Abstraction Layer) 启动序列
*   **Card 4**: 用户态 DMA 与内核态 `sk_buff` 双次拷贝代价对比

#### M2: 无锁环形队列与内存池 (Cards 5-9)
*   **Card 5**: `rte_ring` 环形队列单生产者/单消费者 (SPSC) 指针滑动模型
*   **Card 6**: `rte_ring` 多生产者/多消费者 (MPMC) 并发 CAS (Compare-And-Swap) 并行预留算法
*   **Card 7**: `rte_ring` 队列的 Memory Barrier (内存屏障) 与写顺序一致性保证
*   **Card 8**: `rte_mempool` 环形队列后端与无锁物理内存块快速分配逻辑
*   **Card 9**: `rte_mempool` 的 `Lcore Local Cache` 机制与核心锁解除

#### M3: 大页内存与 NUMA 亲和性 (Cards 10-13)
*   **Card 10**: Hugepages 大页内存（2MB / 1GB）工作机制与 TLB Miss 消除公理
*   **Card 11**: 物理连续内存寻址 (IOVA as VA vs IOVA as PA) 映射原理
*   **Card 12**: NUMA (Non-Uniform Memory Access) 亲和性、本地内存段与跨 Socket 惩罚
*   **Card 13**: 结构体对齐（`__rte_cache_aligned`）、Padding 机制与 CPU L1/L2 缓存行伪共享消除

---

### Page 2 (轮询驱动、mbuf 报文管线与硬件分流)

#### M4: 轮询驱动 (PMD) 与 Burst I/O (Cards 14-17)
*   **Card 14**: Linux NAPI 中断合并与 DPDK PMD 独占轮询机制对比
*   **Card 15**: 物理 RX/TX Descriptor Ring (网卡描述符环) 与回写 (Write-Back) 状态转移
*   **Card 16**: `rte_eth_rx_burst` / `rte_eth_tx_burst` 批量收发大小（通常 32）与指令预取 (prefetch)
*   **Card 17**: CPU 绑核、孤立核 (isolcpus)、防调度剥夺与无中断运行环境搭建

#### M5: mbuf 报文结构与数据流转管线 (Cards 18-23)
*   **Card 18**: `rte_mbuf` 两级元数据结构体物理排版与 Cacheline 命中分配
*   **Card 19**: mbuf 内存空间的 Headroom 协议头部空间与 Tailroom 尾部增长机制
*   **Card 20**: mbuf 的引用计数与零拷贝多播克隆 (`rte_pktmbuf_clone`) 算法
*   **Card 21**: Run-to-completion (运行至完成) 模式对单核 Cache 一致性的优化
*   **Card 22**: Pipeline (流水线阶段) 模式与跨核心 `rte_ring` 报文分发瓶颈
*   **Card 23**: DPDK 网络数据平面安全：访问控制列表 (ACL) Trie 匹配树与流控安全

#### M6: 硬件网卡卸载与流量分流 (Cards 24-28)
*   **Card 24**: 接收端 RSS (Receive Side Scaling) 对称 Toeplitz 哈希与多队列硬件分流
*   **Card 25**: FDIR (Flow Director) 流向引导的精准五元组硬件流规则映射
*   **Card 26**: IP/TCP/UDP 校验和 (Checksum) 硬件卸载与 L3/L4 头预计算
*   **Card 27**: TSO (TCP Segmentation Offload) 硬件分段与网卡 DMA 循环发送
*   **Card 28**: 单根 I/O 虚拟化 (SR-IOV)、VF (Virtual Function) 直通与物理网卡数据旁路
