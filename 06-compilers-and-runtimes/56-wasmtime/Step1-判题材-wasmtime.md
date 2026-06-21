# Step1-判题材-wasmtime.md: WebAssembly 高效 JIT 虚拟机与轻量沙箱运行时审计大纲

本审计项目聚焦于字节码联盟主导的 WebAssembly Standalone 运行时 `wasmtime`。我们将通过 28 张高密度核心知识卡片，深度解构 Wasmtime 引擎初始化与实例生命周期、Cranelift JIT 编译器与机器码生成、虚拟内存线性地址映射与硬件 Guard Pages 隔离防线、WASI Preview 2/3 组件模型与 WIT 类型桥接、控制流异常 Signal Trap 拦截与栈回溯、多租户内存复用及 Fuel/Epoch 协作式抢占限额。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: 引擎初始化与实例生命周期** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - Engine/Config 运行时配置硬化、Store 隔离状态空间管理、Module 字节码校验与编译链、Linker 导入项绑定与路由、Instance 实例内存物理分配、Host-to-Guest Trampoline 蹦床进入流。
*   **M2: Cranelift JIT 编译器与机器码生成** (Cards 7–12) - `#6B8272` (Muted Sage)
    - Cranelift SSA IR (CLIF) 翻译逻辑、Regalloc2 寄存器分配器机制、Wasm 字节码单遍扫描翻译、Cranelift 循环与全局变量优化、蹦床函数机器码动态生成、JIT 编译物序列化与缓存设计。
*   **M3: 沙箱内存隔离与虚拟内存布局** (Cards 13–19) - `#9C6666` (Tea Red)
    - Wasm 线性内存虚拟地址映射、Guard Pages 虚拟地址硬件越界保护、基于信号处理的 Page Fault 硬件陷阱拦截、memory.grow 动态扩容与分配协作、沙箱 OOM 硬件控制策略、多租户虚拟内存池池化复用、多线性内存 (Multi-Memory) 支持。
*   **M4: WASI (WebAssembly System Interface) 规范** (Cards 20–24) - `#7A7A7A` (Iron Grey)
    - WASI Preview 1 经典文件描述符系统调用、WASI Preview 2/3 组件模型演进、WIT 接口定义语言与类型双向转换、WASI 异步流通道机制、虚拟文件系统与网络沙箱网络阻断。
*   **M5: 虚拟机控制流与陷阱 (Traps)** (Cards 25–28) - `#9A825A` (Dusty Gold)
    - 基于 Signal 异常机制的运行时 Trap 路由、Backtrace 原生至 Wasm 栈回溯恢复、Fuel 与 Epoch 协作式执行强占限额、call_indirect 间接调用签名校验。
*   **M6: 嵌入式绑定与安全验证** (Cards 25–28, 部分交叉) - `#755B77` (Muted Grape)
    - Rust/C API 安全边界暴露、多租户隔离与内存共享、侧信道防御与形式化验证、嵌入式资源紧凑优化。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    Wasmtime 的本质是通过 JIT 编译器（Cranelift）将 WebAssembly 虚拟字节码即时编译为本地平台的高效物理机器码，并通过操作系统虚拟地址空间及 Guard Pages 保护页硬件机制，在用户态构建零额外检查开销的强隔离轻量级安全沙箱运行时。
*   **L1 四句话逻辑**：
    1. **静态验证与 SSA 转换**：在加载时对 Wasm 字节码进行严苛的类型与控制流静态验证，转换为 Cranelift 特有的 SSA（单静态赋值）中间表示。
    2. **零开销硬件边界防护**：通过预留 4GB+2GB 的巨大虚拟内存虚拟地址空间（不占用物理内存）配合边界 Guard Pages，利用物理 CPU 页保护中断替代运行时的越界指令检查。
    3. **基于组件的高阶语义互通**：利用 WIT 接口定义语言与组件模型（Component Model），实现复杂对象和异步语义（WASI 0.3）在 Host 与 Guest 之间的强类型安全转换。
    4. **精确的中断与陷阱逃逸**：依托信号处理器（Signal Handler）捕获物理硬件异常（如除零、页错误），结合栈回溯（Stack Unwinding）将其安全映射回运行时的 Wasm Trap 机制，防止宿主进程崩溃。
*   **L2 核心数据流转拓扑**：
    `Wasm 字节码加载` ➜ `Cranelift JIT 编译 (SSA CLIF ➜ 物理机器码)` ➜ `内存虚拟分配 (4GB + Guard Pages)` ➜ `Host 调用 Linker 注入导入项` ➜ `Instance 实例化并运行` ➜ `Wasm 执行越界访问` ➜ `物理 CPU 触发 Page Fault 异常` ➜ `Signal Handler 捕获并解析为 Wasm Trap` ➜ `安全回溯到宿主环境`

---

## 🗂️ 28 张核心知识卡片大纲
1. Wasmtime Engine 与 Config 配置：配置参数不可变硬化与编译策略缓存。
2. Store 隔离状态空间：Wasm 实例运行数据、宿主 State 状态隔离与垃圾回收根集。
3. Module 静态验证与编译：字节码词法验证、并行编译单元与机器码生成。
4. Linker 链接器与路由：多模块依赖链接、Host 导出项与函数表无锁哈希路由。
5. Instance 实例化生命周期：元数据绑定、线性内存分配与 Start 函数冷启动。
6. Host-to-Guest Trampoline 蹦床：宿主 ABI 转换为 Guest JIT 调用约定。
7. Cranelift SSA CLIF 中间表示：控制流图、块（Blocks）与值（Values）的单静态赋值定义。
8. Regalloc2 寄存器分配器：基于冷热路径分配物理寄存器与溢出（Spill）处理。
9. Wasm 字节码即时翻译循环：指令单遍解码、操作数栈模拟与 CLIF 指令生成。
10. Cranelift 编译优化：常量折叠、死代码消除与全局变量内联优化。
11. 蹦床函数动态机器码生成：导出函数参数动态打包与寄存器对齐恢复。
12. JIT 编译产物反序列化缓存：编译模块序列化为 ELF 物理文件与快速缓存查找。
13. Wasm 线性内存虚拟映射：mmap 匿名映射、静态大小配置与物理内存延时提交。
14. Guard Pages 虚拟内存防线：4GB 虚拟内存末端追加 2GB 未映射空虚区。
15. Page Fault 信号拦截机制：SIGSEGV/SEH 信号拦截、内核返回宿主堆栈保护。
16. memory.grow 内存动态扩容：在已映射地址区间上推进物理页面提交边界。
17. 沙箱 OOM 硬件级控制：硬内存配额限制与物理 RSS 占用强退出控制。
18. 虚拟内存池多租户复用：预分配大块物理地址并动态细分给轻量级 Instance。
19. 多线性内存 (Multi-Memory) 支持：多地址空间隔离访问与 `memory.copy` 原子指令。
20. WASI Preview 1 系统调用：将 Wasm 系统调用映射为 OS 本地 IO 操作。
21. WASI Preview 2/3 组件模型：接口类型（WIT）、资源生命周期与异步网络适配。
22. WIT 接口语言与桥接：WIT 原生类型转换为 Canonical ABI 物理布局与释放流。
23. WASI 异步资源通道：异步 Future 任务在 Host 轮询器中的就绪流转。
24. 虚拟文件系统与网络沙箱：路径映射、沙箱防目录逃逸与虚拟 Socket 绑定限制。
25. Signal 陷阱路由控制：Wasm 内部 Trap 与宿主致命 Panic 隔离分流。
26. Backtrace 栈回溯分析：利用 DWARF 符号表解析 Wasm 运行时虚拟指令指针（IP）。
27. Fuel 与 Epoch 强占限额：基于执行指令计数与时钟 Tick 的强占挂起或 Trap。
28. call_indirect 签名校验：函数指针类型哈希表动态验证，防范跳转溢出。
