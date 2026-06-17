# Scoping & Categorization Document: 《iovisor / bcc (eBPF)》 (Linux内核可编程性与系统诊断)

本文件定义了《iovisor / bcc (eBPF)》的底层分析与高密拆解框架，规划 28 张核心卡片在 6 大模块及 Zone T 辅助版块中的物理映射，并确立莫兰迪色彩体系与页面布局参数。

---

## 一、 核心元数据与物理页面预算

*   **目标书籍/项目**：[iovisor / bcc (eBPF)](https://github.com/iovisor/bcc) 开源项目与 Linux 内核可观测性底座
*   **拆书定位**：硬核系统底层诊断与安全审计版（专注于内核/用户态动态插桩、XDP 极速网络路径、内核映射共享内存及 CO-RE 跨版本移植）
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
| **M1** | **安全沙盒与VM架构 (Sandbox & VM)** | 石板蓝 (Slate Blue) | `#7A8B99` / `RGB(122,139,153)` | eBPF 寄存器、验证器与 JIT 编译 |
| **M2** | **内核插桩与追踪 (Kernel Tracing)** | 苔绿 (Moss Green) | `#7D8F7B` / `RGB(125,143,123)` | kprobes, tracepoints 静态与动态捕获 |
| **M3** | **用户态追踪与符号 (User Tracing)** | 梅玫瑰 (Plum Rose) | `#9E828A` / `RGB(158,130,138)` | uprobes, USDT 静态插桩与符号表解析 |
| **M4** | **高性能网络与旁路 (Networking & Bypass)** | 陶土红 (Terracotta) | `#B58A7D` / `RGB(181,138,125)` | XDP 驱动层包处理与 TC 流量分类器 |
| **M5** | **共享存储与数据流 (Maps & Data Flow)** | 靛青 (Indigo) | `#5F7582` / `RGB(95,117,130)` | BPF Maps 共享内存、Ring Buffer 事件流 |
| **M6** | **移植适配与防爆 (CO-RE & Operations)** | 古董金 (Antique Gold) | `#BFA88F` / `RGB(191,168,143)` | BPF CO-RE 跨版本编译、安全限制与调试 |

---

## 三、 L0 ~ L2 压缩阶梯设计

在海报页眉处放置“eBPF 第一性原理公理阶梯”，作为阅读的心智锚点：

*   **L0 一句话本质**：在无需修改内核源码和重启系统的物理限界下，通过安全的内核虚拟机运行沙盒，安全、零延迟地拦截和扩展内核行为。
*   **L1 四句话逻辑**：
    1.  **安全沙盒验证器 (Verifier)** 是 eBPF 准入的终极防线，利用严格的静态分析防止程序引起系统崩溃或死锁。
    2.  **内核与用户态插桩 (Kprobe/Uprobe/Tracepoint)** 决定了追踪边界，实现内核函数、系统调用及用户进程的透明无感监控。
    3.  **极速网络旁路 (XDP/TC)** 在网卡驱动层实现零拷贝过滤与重定向，彻底绕过传统协议栈的高额开销。
    4.  **无锁共享存储 (Maps & Ring Buffer)** 充当了数据流的神经系统，安全低损耗地将事件向用户态收集或跨程序共享。
*   **L2 知识大图**：
    *   [M1: 验证器与寄存器] -> [M2: 内核追踪点] -> [M3: 用户态符号还原] -> [M4: XDP网络旁路] -> [M5: Maps/RingBuffer共享] -> [M6: CO-RE适配与线上防爆]

---

## 四、 28 张核心卡片物理映射表

每一张卡片代表一个高密度的系统设计考点或机制，包含中文表述与专业西文术语。

### M1：安全沙盒与VM架构 (M1_1 至 M1_5) - 【石板蓝】
*   **Card 1 (M1_1)**: eBPF 寄存器模型与虚拟 CPU (11个寄存器 R0-R10 语义约束, 512B 栈物理深度限制)
*   **Card 2 (M1_2)**: 字节码生成与内核 JIT 编译 (LLVM C to eBPF target, 机器码即时编译加速)
*   **Card 3 (M1_3)**: Verifier 验证器静态检测边界 (有向无环图 DAG 循环检测, 指针算术指令过滤, 寄存器状态剪枝)
*   **Card 4 (M1_4)**: Helper Functions 辅助函数沙盒安全机制 (内核暴露的可信操作接口, 内存越界防御)
*   **Card 5 (M1_5)**: BPF LSM 可信安全机制 (内核安全模块 LSM 挂钩, 进程行为细粒度访问控制)

### M2：内核插桩与追踪 (M2_1 至 M2_4) - 【苔绿】
*   **Card 6 (M2_1)**: Kprobe 与 Kretprobe 动态内核插桩 (软中断断点注入, 函数进入/返回参数提取, Ftrace 底层集成)
*   **Card 7 (M2_2)**: Tracepoint 静态内核跟踪点 (预定义内核硬编码探针, 接口稳定性保障, 零启用开销)
*   **Card 8 (M2_3)**: Raw Tracepoint 原生跟踪点 (避免内核参数转换开销, 裸指针参数直通提升性能)
*   **Card 9 (M2_4)**: fentry / fexit 内核蹦床机制 (BPF Trampoline 直接调用, 规避 kprobe 软中断高昂上下文切换)

### M3：用户态追踪与符号 (M3_1 至 M3_4) - 【梅玫瑰】
*   **Card 10 (M3_1)**: Uprobe 与 Uretprobe 用户态动态追踪 (用户空间代码段断点拦截, 高上下文切换开销防御)
*   **Card 11 (M3_2)**: USDT 用户态静态探针 (DTrace/SystemTap 兼容, NOP 占位指令运行时动态替换)
*   **Card 12 (M3_3)**: 动态符号表解析与 DWARF 重定位 (ELF 文件 `.symtab`/`.dynsym` 提取, 地址转符号映射)
*   **Card 13 (M3_4)**: 内存泄漏与分配追踪模型 (内存分配 `malloc`/`free` 生命周期跟踪, 调用栈聚合)

### M4：高性能网络与旁路 (M4_1 至 M4_5) - 【陶土红】
*   **Card 14 (M4_1)**: XDP 极速数据路径机制 (网络网卡驱动层极速包处理, XDP_DROP/XDP_TX/XDP_REDIRECT)
*   **Card 15 (M4_2)**: TC (Traffic Control) 流量控制过滤 (clsact 队列分类器, 入站/出站包双向处理, sk_buff 结构读写)
*   **Card 16 (M4_3)**: Socket Filter 套接字过滤机制 (挂钩 BSD socket 过滤包, 传统 tcpdump 升级版)
*   **Card 17 (M4_4)**: Sockmap 与 Sockhash 套接字重定向 (跳过 TCP 本地环回 Loopback 协议栈, 快速套接字直连)
*   **Card 18 (M4_5)**: 容器网络旁路加速设计 (Cilium 容器网络绕过网卡虚拟设备, 缩短包转发路径)

### M5：共享存储与数据流 (M5_1 至 M5_5) - 【靛青】
*   **Card 19 (M5_1)**: BPF Maps 共享存储物理分类 (Hash, Array, LRU 选型, Lockless Per-CPU Maps 规避并发竞争)
*   **Card 20 (M5_2)**: Perf Ring Buffer 事件通道 (Per-CPU 环形缓冲器, 写入自适应事件流, 内存碎片瓶颈)
*   **Card 21 (M5_3)**: BPF Ring Buffer 新一代共享缓冲 (全局多生产者单消费者环, 预分配内存 Reserve/Commit API, 零拷贝)
*   **Card 22 (M5_4)**: 用户态 Map 异步事件驱动与同步 (`bpf()`系统调用, map 更新/查找开销, `epoll` 异步监听)
*   **Card 23 (M5_5)**: Map Pinning 状态持久化机制 (`bpffs` 虚拟文件系统, 进程重启后的 BPF Map 状态继承)

### M6：移植适配与防爆 (M6_1 至 M6_5) - 【古董金】
*   **Card 24 (M6_1)**: BPF CO-RE 一次编译到处运行 (Compile Once – Run Everywhere 物理本质, 规避机器内核依赖)
*   **Card 25 (M6_2)**: BTF (BPF Type Format) 类型元数据 (结构体字段物理偏移重定位, 强类型检查)
*   **Card 26 (M6_3)**: BCC 运行时动态编译 vs libbpf-bootstrap (目标机 LLVM 编译开销 vs 轻量级二进制预加载)
*   **Card 27 (M6_4)**: bpftool 内核工具链与调试诊断 (Map 查看, 字节码装载, JIT 汇编代码反编译指令)
*   **Card 28 (M6_5)**: eBPF 生产防爆与系统自慰防御 (指令数超限, 内核 STW 级垃圾收集规避, 复杂指令导致 verifier 拒绝)

---

## 五、 辅助版块 Zone T 设计

作为 cheatsheet 底部或侧边的实用工具箱，为系统工程诊断提供即时数值与参数参考：

1.  **T1 eBPF 性能开销对照表**：
    *   *探针对开销影响* -> Tracepoint (极轻微) | Raw Tracepoint (极轻微) | Kprobe (中等) | Uprobe (严重，通常为 kprobe 的 10 倍开销，需慎重选择插桩函数)。
    *   *数据流吞吐* -> BPF Ring Buffer (极速，单生产者吞吐最高可达 ~10M eps) | Perf Buffer (次之)。
    *   *网络包延时* -> XDP_DROP (物理时延 ~10ns) | 传统 iptables 过滤 (~100-300ns)。
2.  **T2 bpftool 调优与诊断金标准**：
    *   列出当前所有 BPF 程序：`bpftool prog show`
    *   dump 运行中程序的 JITed 汇编指令：`bpftool prog dump jited id <prog_id>`
    *   实时追踪 map 写入数据：`bpftool map dump id <map_id>`
3.  **T3 eBPF 运行故障三步诊断法**：
    *   *Verifier Rejected* -> 检查循环是否有上界且能被静态展开 | 验证指针是否进行 `NULL` 检查 | 精简指令复杂度。
    *   *Lost Events* -> 检查 Perf Ring Buffer 容量是否饱和 | 升级为共享内存分配的 BPF Ring Buffer | 降低用户态消费延迟。
    *   *Symbol Resolve Failed* -> 确认二进制文件未被 strip 去除符号表 | 显式声明静态插桩地址。
