# Step1-判题材-bpftrace.md: bpftrace 性能调优与内核动态追踪语言审计大纲

本审计项目聚焦于 Linux 内核动态追踪框架 `bpftrace`。我们将通过 28 张高密度核心知识卡片，深度解构 bpftrace 语法解析与 AST 构建、语义分析与安全类型校验、LLVM IR 即时代码生成、LLVM JIT 编译至 eBPF 字节码、eBPF Map 分配与管理、各类内核探针（kprobe, uprobe, tracepoint, usdt）的挂载逻辑、Perf/Ring Buffer 环形缓冲区双端通信、堆栈回溯与符号表动态解析、以及用户态高频聚合与多维度打印输出。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: 语法解析与 AST 语义** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - flex/bison 语法解析、符号决议与类型分析、AST LLVM IR 生成、局部与全局 Maps 变量、内置函数与聚合算子、BTF 与 DWARF 类型推演。
*   **M2: eBPF 后端与 LLVM JIT** (Cards 7–12) - `#6B8272` (Muted Sage)
    - LLVM JIT 编译字节码、BPF Maps 声明与创建、kprobe/uprobe/USDT 探针挂载、Perf/Ring Buffer 数据传递、多探针联合调度、Verifier 校验防线与指令配额。
*   **M3: 用户态运行与堆栈回溯** (Cards 13–19) - `#9C6666` (Tea Red)
    - 用户态 Maps 轮询打印、生命周期与退出清理、BPF 辅助函数底层映射、用户态/内核态堆栈回溯与符号解析、Tracepoint 参数格式化提取、USDT 信号量检测、过滤表达式内核过滤。
*   **M4: 安全限制与环境自适应** (Cards 20–24) - `#7A7A7A` (Iron Grey)
    - 静态安全执行模式（Safe Mode）、物理资源约束与超限强退、内核版本兼容性判定、内核态堆栈空间边界、信号捕获与资源回支。
*   **M5: 动态追踪架构底座** (Cards 25–28) - `#9A825A` (Dusty Gold)
    - CLI 驱动与前端解析、vmlinux BTF 缓存加速、用户态局部变量隔离、多用户追踪与权限安全防线。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    bpftrace 的本质是通过领域特定语言（DSL）为开发者提供声明式动态追踪手段，利用 flex/bison 解析并即时（JIT）生成 eBPF 字节码，动态注入到 Linux 内核各种探针，并在内核中利用高效 BPF Map 实现高频数据的聚合与双向传输。
*   **L1 四句话逻辑**：
    1. **即时 JIT 编译与注入**：将 bpftrace 追踪脚本解析为 AST，借助 LLVM 动态编译为 eBPF 字节码并即时加载入内核，规避了繁琐的内核模块编写。
    2. **内核级多维挂接与观测**：无缝对齐 kprobes (函数边界)、tracepoints (硬性锚点) 及 USDT (用户级埋点)，实现从用户态库函数到内核网卡驱动的无死角挂接。
    3. **无锁化 Maps 极速聚合**：利用内核 BPF_MAP_TYPE_PERCPU_HASH/ARRAY 等无锁 map 结构，在内核态直接进行计数、求和及直方图多维度运算，极大释放了双端数据传递带宽。
    4. **双向零拷贝环形传输**：使用 `ring_buffer` 或 `perf_event_array` 将发生的高频跟踪记录零拷贝投递至用户态守护进程，保证在几十万级 QPS 下依然不丢包、无阻断。
*   **L2 核心数据流转拓扑**：
    `DSL 脚本输入` ➜ `flex/bison 解析 (AST)` ➜ `Semantic Analyzer (类型校验)` ➜ `Codegen (LLVM IR)` ➜ `LLVM MCJIT 编译 (eBPF 字节码)` ➜ `调用 bpf() 载入内核并 attach 探针` ➜ `探针触发 (内核执行)` ➜ `数据累加至 eBPF Map / 送入 RingBuffer` ➜ `用户态轮询读取` ➜ `符号表解析 (Backtrace)` ➜ `格式化输出直方图`

---

## 🗂️ 28 张核心知识卡片大纲
1. Lexer 与 Parser 语法解析：flex/bison 对追踪 DSL 词法切分与 AST 生成。
2. 语义分析与安全类型校验：符号表决议、探针参数类型校验与 map 键值一致性。
3. AST 降低至 LLVM IR：AST 节点映射、LLVM Builder 构造与 IR 辅助类生成。
4. bpftrace 变量作用域：局部变量、全局 map、线程局部 map 内存模型。
5. 内置变量与高频聚合算子：pid/comm 变量提取、count/sum/hist 聚合映射。
6. 内核结构体与 BTF 类型推演：BTF/DWARF 符号匹配、结构体字段物理偏移计算。
7. LLVM 机器码 JIT 编译后端：编译 IR 到 eBPF 目标码、重定位段填充。
8. BPF Map 声明与物理创建：BPF 哈希/数组 Map 参数设定与 bpf_create_map 阻断。
9. 探针附着与底层挂载：kprobe 劫持、Tracepoint 注册及内核探测项管理。
10. Ring Buffer 环形缓冲区通信：Perf / Ring Buffer 结构初始化与用户态 poll。
11. 多探针统一协调调度：多探针共享 Map、联合触发与统一 Session 注册。
12. Verifier 静态校验与指令安全：Verifier 静态无环扫描、寄存器边界追踪与安全性审查。
13. 用户态 Maps 轮询格式打印：定期 poll 地图、直方图（Hist）对数缩放渲染输出。
14. 会话生命周期与清理：Session 状态转换、探针卸载（Detach）与 Maps 释放垃圾收集。
15. 辅助函数（Helper Calls）映射：BPF 辅助函数 ID 判定、cgo 调用桥接与内核接口限制。
16. 内核态/用户态堆栈回溯：栈帧（fp/sp）读取、内核 symbol 映射及用户态 DWARF 反解析。
17. Tracepoint 参数格式化提取：/sys/kernel/debug/tracing/events 格式解析。
18. USDT 信号量与动态埋点：USDT ELF Section 解析、共享库符号劫持与 semaphore 置位。
19. 过滤表达式（Filters）生成：`if` 条件转换、BPF 寄存器比对与提前返回退出。
20. 静态安全执行模式限制：Safe Mode 开启、危险 Helper 禁用与脚本静默拦截。
21. 物理资源边界限制：Map 最大容量限制、最大栈深保护与单探针指令硬上限。
22. 内核环境自适应检测：BPF 功能自适应探测、vmlinux BTF 版本匹配。
23. 内核栈空间（Stack）安全控制：BPF 512 字节栈限制防护、溢出拦截。
24. 信号捕获与优雅退出：SIGINT 捕获、未决事件排空与系统清理复位。
25. 前端命令行参数驱动：CLI 驱动器、单行命令即时解析与编译执行环。
26. vmlinux BTF 缓存加速：BTF 路径扫描、内核类型元数据缓存。
27. 用户态关联变量追踪：脚本定义常驻状态变量的更新。
28. 多租户并发追踪隔离：并发 Sessions 相互隔离、权限级别静态判定。
