# 《iovisor / bcc (eBPF)》高密知识图谱与速查手册 (Cheatsheet)

*   **L0 一句话本质**：在无需修改内核源码或重启系统的物理限界下，通过在内核态安全沙盒中动态运行即时编译的字节码，实现零延迟的内核行为拦截、性能可观测性诊断与极速网络数据包旁路。
*   **L1 四句话逻辑**：
    1.  **安全沙盒虚拟机** (M1 VM/Verifier) 构筑静态分析防线，通过 DAG 环检测与指针类型追踪，确保载入内核的代码绝不挂起或崩溃。
    2.  **多源插桩探测点** (M2 内核/M3 用户态) 将内核 kprobes/tracepoints 和用户态 uprobes/USDT 统一转化为事件，提供全栈透明无感追踪。
    3.  **极速网络数据面** (M4 Network) 依托网卡驱动层 XDP 与 TC，直接在物理包分配前进行就地过滤、修改与 Sockmap 零拷贝重定向。
    4.  **无锁通信神经网** (M5 Maps/Ringbuf) 通过共享的 BPF Maps 与全局 Ring Buffer，安全低损耗地支撑内核态与用户态双向事件流及生命周期持久化。
*   **L2 八大内核子系统拓扑**：
    *   [C/eBPF 源码编译] $\rightarrow$ [Verifier 安全静态验证] $\rightarrow$ [JIT 即时机器码生成] $\rightarrow$ [内核态 Hooks (Kprobes/XDP/LSM)] $\rightarrow$ [无锁 Maps/Ring Buffer 数据存储] $\rightarrow$ [bpffs Pinning 持久化映射] $\rightarrow$ [用户态 bpftool/BCC 交互消费] $\rightarrow$ [BTF 元数据跨内核移植 (CO-RE)]

---

## 🌐 系统第一性思维滤镜 (eBPF Kernel Epistemic Filter)

- **认识论 (Epistemology) - 无感知监控与动态热补丁**：
  传统的系统诊断依赖于应用层打点（侵入性强）或内核重新编译（代价极高）。eBPF 的核心认识论在于“物理不可知，行为可控制”。系统无需为了获取内部监控数据而做任何前置修改，可在运行中动态注入行为捕获钩子，将整个内核视为一个可热插拔、可安全编程的运行时环境。
- **人性观/虚拟机观 (VM Sandbox Philosophy) - 绝对不信任与物理隔离**：
  eBPF 视开发者编写的 C 字节码为“潜在的恶意指令集”。虚拟机并不假定编写者有高超的内核安全意识，而是利用静态验证器 (Verifier) 拦截任何可能导致无限循环、空指针解引用或越界读写的代码。安全性是最高优先级（宁可拒绝加载，也不冒 0.001% 的内核崩溃风险）。
- **方法论 (Methodology) - 内核旁路 (Bypass) 与零拷贝 (Zero-Copy)**：
  传统网络与 IO 处理的物理极限瓶颈在于“内核态与用户态的频繁上下文切换”以及“大内存块的内存拷贝”。eBPF 通过在驱动层（XDP）直接拦截包和在内核中直接重定向 socket（Sockmap），将协议栈处理开销直接降为零，践行“就地计算、尽早过滤、无锁共享”的极限吞吐方法论。
- **道德直觉/伦理 (Ethics) - 安全隔离与可观测性的度**：
  可观测性是一把双刃剑。eBPF 能捕获系统调用入参，意味着能静默拦截用户输入的敏感密码、SSL 明文包及私密通信。因此，eBPF 的安全伦理要求加载权限必须严格锁定在特权用户（`CAP_SYS_ADMIN` 或 `CAP_BPF`），且提供 BPF LSM 等防线防止追踪器本身沦为 rootkit 级隐蔽攻击工具。

---

## ⚔️ 经典架构折衷与异常矩阵 (eBPF Architectural Trade-offs)

| 开发者直觉 (⚠) | 客观技术解构 (✓) | 代码与架构级防线 (★) |
| :--- | :--- | :--- |
| **直觉 1**：在用户态程序使用 `uprobe` 对高频调用函数（如 `malloc` 或短小 JSON 解析器）进行插桩，性能影响微乎其微。 | **解构**：`uprobe` 底层依赖于软中断断点，会强制发生两次用户态与内核态的上下文切换。在高频函数上插桩会导致应用吞吐量暴跌达 50% 以上。 | 避免在 QPS > 10,000 的用户态热点路径上设置 `uprobe`。高频追踪应使用 **USDT** (静态探针)，或在应用内将数据聚合成批，通过 BPF Map 共享。 |
| **直觉 2**：`Perf Ring Buffer` 是多核 CPU 下数据上报用户态的最快通道，只要缓冲区开得足够大就绝对不会丢失数据事件。 | **解构**：`Perf Buffer` 按 CPU 分配独立环形缓冲。若单核 CPU 发生流量暴刷而其对应消费线程延迟，会导致该核独占缓冲溢出丢包，且各核间事件顺序在用户态完全乱序。 | 弃用 `Perf Buffer`。在生产环境升级为全局共享的 **BPF Ring Buffer**，通过多生产者单消费者无锁环，在多核间自动平摊开销，支持 `reserve`/`commit` 零拷贝写入。 |
| **直觉 3**：既然 BPF 提供了循环支持，可以在字节码中编写复杂的多维数组搜索或长文本匹配循环逻辑以进行业务过滤。 | **解构**：内核验证器对程序复杂度有极强检测。即使在 5.3+ 内核支持有界循环后，由于状态路径剪枝限制，深度嵌套的大循环极易触发 `instruction limit` (100万条指令限制) 被拒绝加载。 | 必须将复杂数据结构在用户态预处理。在内核中只进行最基础的定长二分查找或哈希匹配。大循环必须显式使用 `#pragma unroll` 或确保循环体退出条件极其简单。 |
| **直觉 4**：使用 BCC 动态编译模式（把 C 源码以 Python 字符串形式传递并在运行时现场调用 Clang/LLVM 编译）是开发 eBPF 监控程序的唯一标准。 | **解构**：BCC 动态编译要求目标机必须安装完整的 LLVM/Clang 工具链及 Linux Headers（大小 >100MB），且在加载瞬间会导致物理 CPU 瞬间飙升至 100% 产生性能刺突。 | 全面采用 **BPF CO-RE (一次编译到处运行)**。在编译期生成 BTF 重定位信息，输出仅几十 KB 且零依赖的二进制程序，载入瞬间耗时 <10ms，无需现场编译。 |
| **直觉 5**：通过 `bpf_probe_read_user()` 能够读取用户态任意指针的数据，因此能够直接在追踪程序中随时随地读取任意用户态的字符串数据。 | **解构**：用户态内存是延迟分配且可能被交换到磁盘的 (Page Fault)。当 eBPF 访问该内存而该页不在物理内存中时，由于内核追踪上下文禁止发生缺页异常，读取会静默失败。 | 读取前必须确认应用已加载该内存页。可通过在紧接着系统调用返回后再行读取，或使用用户态插桩确保其物理页已被唤醒装载，并随时检查读取辅助函数的返回值。 |

---

## 🧠 28个高密速查卡 (Card 01 - 28)

### 模块一：安全沙盒与VM架构

#### Card 01. eBPF 寄存器模型与虚拟 CPU (Registers & CPU Model)
*   **寄存器约束**：eBPF 虚拟机拥有 11 个 64 位硬寄存器。`R0` 承载辅助函数返回值及程序退出状态；`R1-R5` 传递辅助函数调用参数；`R6-R9` 为被调用者保存寄存器（需由 BPF 自动压栈恢复）；`R10` 为只读帧指针（Frame Pointer），指向 512 字节的栈底。
*   **物理限界**：单程序栈大小硬性限制为 **512 字节**，超限将直接触发 Verifier 拒绝。大对象或复杂上下文状态必须存入 BPF Maps 中而决不能声明为栈上局部变量。

#### Card 02. 字节码生成与内核 JIT 编译 (Bytecode & JIT Compilation)
*   **编译链路**：开发级 C 代码经 LLVM 以 `-target bpf` 选项编译，生成符合 ELF 格式的 BPF 字节码二进制。加载时，内核的 **JIT (Just-In-Time) 编译器**（如 x86_64 JIT）将 64 位 BPF 指令逐条直接对齐翻译为 CPU 原生机器码（如 `x86 MOV/JMP`）。
*   **内核开关**：生产环境必须确保 `/proc/sys/net/core/bpf_jit_enable` 设置为 `1`（开启 JIT）且 `bpf_jit_harden` 设置为 `2`（硬化防止 JIT 喷射攻击），从而使运行效率近乎原生汇编指令。

#### Card 03. Verifier 验证器静态检测边界 (Verifier Static Analysis)
*   **验证流程**：验证器分两阶段执行。第一阶段执行有向无环图（DAG）深度优先遍历，严禁出现死循环、不可达代码以及未初始化指针访问；第二阶段模拟指令执行，追踪每个寄存器的“类型状态”（如 `SCALAR_VALUE`、`PTR_TO_STACK`）和值范围。
*   **防线策略**：一旦寄存器进行未经 NULL 校验的指针操作，或指针跨越栈边界读写，验证器将报错终止载入。它通过“状态剪枝”降低评估分支数，避免长时间占用内核。

#### Card 04. Helper Functions 辅助函数沙盒安全机制 (Helper Functions)
*   **接口限制**：eBPF 程序严禁直接调用内核定义的任意函数（如 `printk` 或内核内部符号），以防破坏内核调用栈安全。所有外部交互必须通过系统预分配的静态 Helper Functions 数组（如 `bpf_map_lookup_elem`、`bpf_ktime_get_ns`）。
*   **特权控制**：每个 BPF 程序类型（如 `BPF_PROG_TYPE_KPROBE` 与 `BPF_PROG_TYPE_SOCKET_FILTER`）有其专属的可调用 Helper 许可子集。在网卡驱动收包上下文中，绝不允许调用可能引发进程挂起或睡眠的辅助函数。

#### Card 05. BPF LSM 可信安全防御机制 (BPF LSM Security Hooks)
*   **安全框架**：BPF LSM 允许将 eBPF 程序作为回调函数直接挂载到 Linux 安全模块（LSM）框架的硬挂钩上（如 `path_mkdir`、`file_permission`、`task_alloc`）。
*   **细粒度拦截**：当特定系统操作触发时，BPF 程序检查操作的主客体元数据，并通过返回 `0`（放行）或非零值（如 `-EACCES`，拒绝）实现内核级的动态安全合规审计，彻底替代笨重且不可热升级的传统 SELinux 策略。

---

### 模块二：内核插桩与追踪

#### Card 06. Kprobe 与 Kretprobe 动态内核插桩 (Kprobes & Kretprobes)
*   **注入机制**：`kprobe` 在运行时动态改写内核目标函数的第一条指令为陷阱指令（如 x86 的 `int3`），触发 CPU 中断陷入。执行先驱 BPF 程序后，执行原指令并返回；`kretprobe` 劫持函数返回地址，实现退出时的参数与耗时审计。
*   **性能隐患**：由于引发软中断，每次 `kprobe` 调用会造成数千个 CPU 周期的上下文切换损耗。避免在调度程序（如 `schedule`）或高频网卡中断函数上使用动态 `kprobe`。

#### Card 7. Tracepoint 静态内核跟踪点 (Tracepoints)
*   **静态锚点**：由内核开发者预先在核心子系统（如 `sched_switch`、`sys_enter_read`）的关键物理位置硬编码的 Tracepoints。其 ABI 接口在内核版本更迭中极度稳定。
*   **零启用损耗**：未被加载挂钩时，Tracepoint 在内核中仅表现为 NOP 指令，完全无性能损耗；一旦启用，会跳转执行绑定的 eBPF 程序，调用开销远低于 `kprobe` 的软中断。

#### Card 08. Raw Tracepoint 原生跟踪点 (Raw Tracepoints)
*   **性能提纯**：传统的 `tracepoint` 挂载点在内核中有一个参数序列化适配器（Format adapter），将内核结构转化为易读的参数包。`raw_tracepoint` 绕过了此层包装，使 eBPF 能够直接访问原始的内核上下文（如直接获取 `struct pt_regs`）。
*   **适用场景**：在大流量网络或磁盘 IO 监控中，使用 `raw_tracepoint` 能减少内核级参数转换开销，使追踪吞吐极限提升达 15% 以上。

#### Card 09. fentry / fexit 内核蹦床机制 (fentry / fexit & Trampoline)
*   **无缝跳板**：由 5.5 内核引入的新一代动态插桩技术。通过 BPF Trampoline（内核蹦床）技术，直接修改内核函数开头的 `fentry` 钩子，用直接的汇编 `call` 跳转替换原指令。
*   **性能碾压**：由于彻底规避了 `kprobe` 带来的 `int3` 中断陷入与寄存器状态大保存开销，`fentry/fexit` 执行速度与内核静态 tracepoint 一样快，是替代传统 `kprobe` 的生产首选。

---

### 模块三：用户态追踪与符号

#### Card 10. Uprobe 与 Uretprobe 用户态动态追踪 (Uprobes & Uretprobes)
*   **机制原理**：对用户态空间进程的虚拟内存地址（如共享库 `libc.so` 内的 `malloc`）进行断点指令劫持。当程序运行到插桩地址时，陷于内核态，触发绑定的 eBPF 程序执行。
*   **致命瓶颈**：由于涉及“用户态 $\rightarrow$ 内核态 $\rightarrow$ 用户态”的双向上下文切换，每次 `uprobe` 调用耗时可达 1-2 微秒。严禁在 C++/Rust 高频循环函数内设置 `uprobe` 插桩。

#### Card 11. USDT 用户态静态探针 (USDT Probes)
*   **预置监控**：User-level Statically Defined Tracing。由应用开发者（如 MySQL、JVM、Node.js）使用 `DTRACE_PROBE` 宏在用户态源码中埋设的静态探测点。
*   **运行时插桩**：编译后为 NOP 指令，并记录在 ELF 文件的 `.note.stapsdt` 段中。启用追踪时，内核将该 NOP 指令动态 patch 为断点指令，实现应用级事件捕捉，避免过度侵入。

#### Card 12. 动态符号表解析与 DWARF 重定位 (Symbol Resolution)
*   **符号映射**：eBPF 内核程序捕捉到的调用栈仅为虚拟内存地址（如 `0x7fff81a2f10b`）。必须在用户态（如 BCC / Golang 监控端）解析进程的 ELF 二进制符号段（`.symtab`、`.dynsym`）将其映射为类名与方法名。
*   **调试还原**：如果二进制文件已被 `strip`，必须提供外部 DWARF 调试符号文件；针对运行时即时编译的语言（如 Java JVM、V8 JIT），需通过 `/tmp/perf-<pid>.map` 动态加载 JIT 编译器的内存符号映射。

#### Card 13. 内存泄漏与分配追踪模型 (Memory Leak Tracing)
*   **内存模型**：在 BCC `memleak` 中，通过挂载 `malloc` 与 `calloc` 的 `uprobe`（或 `fentry` 如果内核直接分配），记录每次内存申请的地址指针及调用栈，以指针为 Key 存入 BPF Hash Map。
*   **生存差值**：同步挂载 `free` 的 `uprobe`，释放时将该 Key 从 Hash Map 中剔除。运行一段时间后，Map 中残留的分配项即为泄漏物理地址与调用栈源头。

---

### 模块四：高性能网络与旁路

#### Card 14. XDP 极速数据路径机制 (eXpress Data Path)
*   **就地收包**：XDP 程序挂载在网络驱动收包队列的最早期（`RX` 环），在网络包尚未被包装为内核 `sk_buff` 结构体、且未触发内核网络协议栈分配前直接执行。
*   **终极裁决**：支持 `XDP_DROP`（直接丢包，抗 DDoD 降本神器）、`XDP_TX`（原网卡送回）、`XDP_REDIRECT`（绕过 CPU 路由，直达另一块网卡或转发到用户态 `AF_XDP` 套接字），处理单包时延仅需纳秒级。

#### Card 15. TC (Traffic Control) 流量控制与过滤 (TC Clsact)
*   **双向钩子**：挂载在 Linux 流量控制框架的 `clsact` 队列上。相比 XDP，TC 能够同时接管入站（Ingress）和出站（Egress）的双向流量。
*   **结构解析**：TC 程序已能完整读写内核 `struct __sk_buff`，直接获取网络分流元数据、VLAN 标签、协议端口等，是构建高性能容器防火墙、负载均衡与微服务流量限速的底座。

#### Card 16. Socket Filter 套接字过滤机制 (Socket Filters)
*   **套接字附着**：通过 `setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, ...)` 直接附着在特定 Socket 描述符上。
*   **应用沙箱**：作为最古老的 BPF 应用形式，它对套接字收发包进行被动复制与监控，无法修改数据包内容，仅用于流量镜像、包结构过滤和轻量级审计（如安全边界流量监控）。

#### Card 17. Sockmap 与 Sockhash 套接字重定向 (Sockmap & Sockhash)
*   **环回直连**：`BPF_MAP_TYPE_SOCKMAP` 存储了活跃网络连接的 Socket 文件描述符。通过将 eBPF 程序挂载到 `sk_msg` 上，数据包在发送端 Socket 写入缓冲区时被直接复制到接收端 Socket 的读取队列中。
*   **绕过本环**：完全跨越了传统网络层、传输层，对于本地主机（`localhost` / 容器间通信）的 TCP 通信，直接缩短网络传输耗时达 50% 以上。

#### Card 18. 容器网络旁路加速设计 (Container Network Bypass)
*   **经典痛点**：传统 Kubernetes 容器间通信依赖 veth-pair 设备，数据包需遍历两次网络协议栈（Pod 协议栈 $\rightarrow$ 宿主机协议栈），引发高开销。
*   **Cilium 实践**：Cilium 使用 eBPF 动态感知容器网络接口（CNI）拓扑，直接利用 eBPF 将源 Pod 暴露的 Socket 直接与目的 Pod 绑定，越过桥接网卡与 iptables 过滤，实现容器通信极速直通。

---

### 模块五：共享存储与数据流

#### Card 19. BPF Maps 共享存储物理分类 (BPF Maps)
*   **类型选型**：
  * `BPF_MAP_TYPE_HASH`：动态哈希表，适合稀疏大键值查找；
  * `BPF_MAP_TYPE_ARRAY`：静态数组表，预分配内存，读取时延最低且稳定；
  * `BPF_MAP_TYPE_LRU_HASH`：内置冷热逐出机制，适合处理流量峰值防内存暴涨。
*   **并发防护**：高频写入必须选用 `Per-CPU` 变体（如 `BPF_MAP_TYPE_PERCPU_HASH`），每个 CPU 核心独占一份内存分区，实现无锁并发写入，彻底避开内核自旋锁竞争引起的 Cache 抖动。

#### Card 20. Perf Ring Buffer 事件通道 (Perf Event Buffer)
*   **环形结构**：传统的 eBPF 与用户态事件通信管道。内核为每个 CPU 核心独立分配一个专用的 Perf 环形内存区。
*   **丢包设计**：当某一 CPU 突发海量事件，而用户态对应的特定线程消费不及时，该单核缓冲区将直接溢出丢包。此外，由于各核独立存储，多核并发上报的数据在用户态接收端会产生明显的事件时间重叠乱序。

#### Card 21. BPF Ring Buffer 新一代共享缓冲 (BPF Ring Buffer)
*   **全局无锁**：5.8 内核引入的新一代环形缓冲区。由所有 CPU 核心共享同一个全局无锁多生产者、单消费者（MPSC）队列。
*   **零拷贝API**：提供 `bpf_ringbuf_reserve` API，直接在共享的环形物理内存上预留空间给程序写入，写入完成调用 `bpf_ringbuf_commit` 瞬间提交，彻底消除了“内核栈写临时结构 $\rightarrow$ 拷贝至 Perf 环”的二次内存拷贝消耗。

#### Card 22. 用户态 Map 异步事件驱动与同步 (Map Operations & User Sync)
*   **操作接口**：用户态通过 `bpf(BPF_MAP_LOOKUP_ELEM, ...)` 系统调用与 Map 交互。若频繁在死循环中轮询 Map，会导致高昂系统调用开销。
*   **事件驱动**：Ring Buffer 内置 epoll 文件描述符支持。用户态程序通过 `epoll_wait` 在 Ring Buffer 上进行阻塞等待，当内核通过 `bpf_ringbuf_submit` 写入数据跨越水位线时自动唤醒用户态消费，维持极低待机 CPU 开销。

#### Card 23. Map Pinning 状态持久化机制 (Map Pinning)
*   **生存局限**：默认情况下，BPF Maps 的生命周期与加载它的 BPF 程序文件描述符（FD）强绑定。一旦用户态守护进程退出且 eBPF 程序卸载，内核中的 Maps 及历史统计数据将随之销毁。
*   **状态承袭**：通过挂载 eBPF 专用虚拟文件系统 `bpffs`（通常在 `/sys/fs/bpf/` 下），调用 `bpf_obj_pin()` 将 Map 锚定到该目录下。这确保了即便程序重启，新的 eBPF 实例依然能重新加载并继承先前的 Map 状态数据。

---

### 模块六：移植适配与防爆

#### Card 24. BPF CO-RE 一次编译到处运行 (BPF CO-RE)
*   **移植困局**：不同发行版 Linux 内核的数据结构（如 `struct task_struct`）中字段偏移量随时在变，导致在 A 机器编译的 BPF 字节码在 B 机器上会因读取到错位数据而触发崩溃。
*   **重定位原理**：Compile Once - Run Everywhere。使用带有调试信息的 Clang 编译 BPF 源码，将对结构体成员访问改写为带重定位标记（`__builtin_preserve_access_index()`）的虚拟访问。加载时，由 `libbpf` 根据本地内核 BTF 动态计算并重写字节码中的偏移量，完成无感移植。

#### Card 25. BTF (BPF Type Format) 类型元数据 (BTF Metadata)
*   **轻量调试**：BTF 是对传统庞大 DWARF 调试元数据格式的高度压缩替代版（体积可缩小 100 倍以上）。它以结构化的紧凑二进制形式编码了内核所有的类型定义、结构体字段、函数原型。
*   **内核自省**：现代内核通过 `/sys/kernel/btf/vmlinux` 暴露自省元数据，使编译期不需要本地包含繁重的 `linux/sched.h` 等复杂头文件，只需包含一个 `vmlinux.h` 即可获得当前内核的完整类型视图。

#### Card 26. BCC 动态编译 vs libbpf-bootstrap (BCC vs libbpf)
*   **BCC 痛点**：BCC 在运行期调用目标机的 Python + Clang 现场生成字节码，导致部署容器需携带数以百 MB 的编译环境，启动慢且有 CPU 刺突隐患。
*   **libbpf 极简**：使用 `libbpf-bootstrap` 架构，将 BPF C 字节码在编译阶段即固化并压缩进最终的用户态 C 二进制程序内。部署文件仅几十 KB，秒级瞬间无依赖加载，适合极致轻量化生产推送。

#### Card 27. bpftool 工具链与 JIT 反编译 (bpftool Diagnostics)
*   **命令行利器**：`bpftool` 是对内核 BPF 子系统进行监控的权威工具。
*   **诊断命令**：
  * 查看运行中程序详情：`bpftool prog show`；
  * 反编译内核已运行的 JIT 机器码：`bpftool prog dump jited id <prog_id>`；
  * 强制将 Map 固定持久化：`bpftool map pin id <map_id> /sys/fs/bpf/<map_name>`，是保障生产排障的核心利器。

#### Card 28. eBPF 生产防爆与 verifier 调优 (eBPF Production Operations)
*   **崩溃防护**：Verifier 一旦检测到死循环、超出栈空间限制、或未检查 NULL 的空指针解引用，将直接拒绝加载，从而彻底保护内核。
*   **调优规避**：对于早期内核不支持循环的情况，必须使用 `#pragma unroll` 将循环展开；调用内核大结构时，必须分段使用 `bpf_probe_read_kernel()` 并随时检测返回值；单条程序指令树深度过高时，可拆分为多程序并通过 `Tail Call`（尾部调用）进行无开销跳转分流。

---

## 叁、 Zone T 性能与诊断实验室 (eBPF Performance & Diagnostic Labs)

### T1 eBPF 性能开销对比与规格限界
*   **插桩性能损耗基准 (Overhead Benchmark)**：
    *   `Tracepoint` -> ~10-20 ns/调用 | `Kprobe` -> ~100-150 ns/调用 | `Uprobe` -> ~1-2 us/调用（上下文切换惩罚严重）。
*   **物理限界指标**：
    *   最大指令数限制: 5.2+ 内核为 1,000,000 条指令（旧内核限制为 4096 条指令）。
    *   BPF 栈物理深度: **512 字节**。
    *   尾部调用最大嵌套级数: **33 层**。

### T2 BCC / bpftool 调优与诊断指令金标准
*   **快速排查 verifier 拒绝原因并输出详细日志**：
    `bpftool prog load ./my_prog.o /sys/fs/bpf/my_prog type tracepoint`（失败可查看系统 `dmesg -w` 内的 verifier dump）
*   **获取系统当前所有 BPF 映射空间及内存占用情况**：
    `bpftool map show`
*   **实时查看 Ring Buffer 消费速率与缓存状态**：
    使用 `bpftool prog profile id <prog_id> runs cycles` 评估指令周期与 CPU cache 命率。

### T3 eBPF 运行故障三步诊断法
1.  **第一步：解决 Verifier 报错**
    *   *现象*：`R2 invalid mem access 'mem_or_null'`
    *   *防线*：在解引用从 Map 查找到的指针 `val` 之前，必须编写 `if (val == NULL) return 0;`。Verifier 会在此处记录状态分支，判定后续访问安全。
2.  **第二步：解决丢包 (Lost Events) 问题**
    *   *现象*：用户态输出 `Lost 1432 events`。
    *   *防线*：将 `Perf Buffer` 环容量设置调大；或迁移代码至 `BPF Ring Buffer`，利用 `bpf_ringbuf_output` 全局共享缓存，并检查用户态消费线程的 epoll 唤醒回调中是否有阻塞逻辑，确保消费速度跟上内核产生速度。
3.  **第三步：解决用户态符号无法解析问题**
    *   *现象*：追踪用户栈只显示 `[unknown]` 或物理十六进制地址。
    *   *防线*：检查目标应用程序编译时是否使用了 `-g` 且未被 `strip` 剔除符号表。如果应用由 Go/JVM 运行时启动，检查 `/tmp/perf-<pid>.map` 是否生成，并在用户态显式重定向符号查找路径。
