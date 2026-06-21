# Sanitizers 运行时内存与并发检测套件高密卡片系统题材判定

## 1. 核心模块与 28 张卡片大纲

### M1: Sanitizers 体系公理与编译器插桩机理 (Cards 1-5)
*   **Card 1**: 内存与并发安全性痛点：C/C++ 内存不安全行为定义、静态分析与黑盒测试局限性、运行期动态检测的必要性。
*   **Card 2**: 编译器级别插桩机制：LLVM Pass 编译期 AST/IR 转换位置、指令插入逻辑与 compiler-rt 运行期库协同模型。
*   **Card 3**: 系统函数拦截与重写：运行期动态链接器劫持 (LD_PRELOAD)、静态链接弱符号 (Weak symbols) 劫持与库函数包装 (Interceptors) 原理。
*   **Card 4**: 运行时符号化与栈回溯：物理地址到代码位置转换、llvm-symbolizer 解析 DWARF 调试信息与慢速/快速栈回溯 (Stack Unwinding) 逻辑。
*   **Card 5**: 优化与开销折衷机制：Sanitizers 运行期 CPU 慢速/内存溢出开销物理机理、O0/O1/O2 编译器优化选项对检测精度与执行效率的权衡。

### M2: AddressSanitizer (ASan) 影子内存与越界释放检测 (Cards 6-10)
*   **Card 6**: 影子内存 (Shadow Memory) 映射算法：应用内存地址到影子字节的一对八线性映射公式 `(Addr >> 3) + Offset`、无重叠比例与字节压缩编码表示。
*   **Card 7**: 红色区域 (Redzones) 与编译器插桩：栈对象、全局变量与堆分配内存的前后安全边界填充、影子字节毒化 (Poisoning) 状态表示。
*   **Card 8**: 释放后使用 (Use-After-Free) 隔离防御：堆块 `free` 行为劫持、内存隔离区 (Quarantine) 物理队列结构与野指针访问拦截判定。
*   **Card 9**: 堆内存分配劫持：ASan 自定义分配器 (`asan_allocator`) 接管 `malloc/free`、堆元数据 (Header) 记录与物理内存对齐规则。
*   **Card 10**: 栈上释放后使用 (UAR) 与作用域外使用 (UAS)：局部变量生命周期逃逸检测、虚拟调用栈转为物理堆上分配检测机理。

### M3: ThreadSanitizer (TSan) 状态机、向量时钟与数据竞争 (Cards 11-15)
*   **Card 11**: 数据竞争 (Data Race) 与 happens-before 关系：并发多线程非同步读写冲突的数学模型、Happens-Before 偏序关系在 TSan 中的判定条件。
*   **Card 12**: 向量时钟 (Vector Clocks) 状态追踪：线程局部逻辑时钟、向量时钟向量更新公式、多线程并发事件时序捕获。
*   **Card 13**: 影子状态 (Shadow State) 压缩存储：应用内存 8 字节单元对应 4 个影子单元 (Shadow Cell)、Shadow Cell 内部压缩表示 (Thread ID, Epoch, Range, IsWrite)。
*   **Card 14**: 锁操作与状态流转同步：互斥锁 (`pthread_mutex_lock/unlock`)、屏障 (Barrier) 与原子操作触发 of TSan 向量时钟同步机制。
*   **Card 15**: 死锁 (Deadlock) 动态检测：锁获取路径局限、死锁锁依赖图 (Lock Order Graph) 拓扑有向环路实时检测机制。

### M4: MemorySanitizer (MSan) 未初始化内存读取检测 (Cards 16-19)
*   **Card 16**: 未初始化内存读取 (Use of Uninitialized Memory) 安全威胁：堆栈随机残留脏数据、信息泄露与控制流劫持风险。
*   **Card 17**: 影子内存比特级映射 (Shadow Bit-Map)：应用内存字节到影子内存字节的 1:1 精确映射、每一位状态标识（0 标识已初始化，1 标识未初始化）。
*   **Card 18**: 来源追踪 (Origin Tracking) 技术：4 字节 Origin 影子内存记录分配或传递堆栈、多级传递链依赖反查物理根源。
*   **Card 19**: 屏蔽未初始化漏报：条件分支判定 (Branching) 与外部系统调用边界的安全处理、自写汇编代码检测盲区。

### M5: UBSan 与 LSan 运行时检测与分析 (Cards 20-24)
*   **Card 20**: 内存泄漏 (Memory Leak) 物理成因：丢失引用指针、LSan 作为 Conservative GC（保守垃圾回收）的检测逻辑。
*   **Card 21**: LSan 根集合保守扫描：进程退出时暂停线程、扫描活跃寄存器/栈空间/全局变量，通过不可达节点图论判断泄漏。
*   **Card 22**: UBSan 算术与空指针行为检测：有符号整数溢出、无符号溢出环绕、空指针解引用与野指针未对齐访问。
*   **Card 23**: UBSan 对齐与越界校验：C++ 数组边界检查、可变长度数组 (VLA) 指针越界与虚表指针 (`vptr`) 对齐异常编译器插桩。
*   **Card 24**: UBSan 动态类型转换与 CFI 控制流完整性：类型向下转换风险、控制流劫持攻击拦截机理。

### M6: Sanitizers 物理部署、运维排查与故障字典 (Cards 25-28)
*   **Card 25**: 运行时配置与黑名单过滤：`ASAN_OPTIONS/TSan_OPTIONS` 环境变量控制、Suppression 文件局部规避与优化白名单排除。
*   **Card 26**: 典型 ASan 故障栈重建：Heap-Use-After-Free, Stack-Buffer-Overflow, Global-Buffer-Overflow, Memory Leak 报错报告结构与调试指引。
*   **Card 27**: 典型 TSan 与 MSan 诊断故障：第三方未插桩动态链接库带来的数据竞争/未初始化误报、汇编优化逃逸排查。
*   **Card 28**: 模糊测试与持续部署集成：ASan 搭配 libFuzzer/AFL++ 检测、运行期代码覆盖率 (SanitizerCoverage) 与生产环境金丝雀探针折衷。

---

## 2. 莫兰迪色系设计配置 (Morandi Palette)
为了实现视觉设计上的风格化与一致性，cheatsheet 与 HTML 知识大图采用以下精选莫兰迪色系：

*   **M1**: 莫兰迪蓝灰色 (`#4B5F7A` / Slate Blue) - Sanitizers 体系公理与编译器插桩机理
*   **M2**: 莫兰迪茶红色 (`#9C6666` / Tea Red) - AddressSanitizer (ASan) 影子内存与越界释放检测
*   **M3**: 莫兰迪绿灰色 (`#6B8272` / Muted Sage) - ThreadSanitizer (TSan) 状态机、向量时钟与数据竞争
*   **M4**: 莫兰迪铁灰色 (`#7A7A7A` / Iron Grey) - MemorySanitizer (MSan) 未初始化内存读取检测
*   **M5**: 莫兰迪金黄色 (`#9A825A` / Dusty Gold) - UBSan 与 LSan 运行时检测与分析
*   **M6**: 莫兰迪紫灰色 (`#755B77` / Muted Grape) - Sanitizers 物理部署、运维排查与故障字典
*   **L0-L2 梯级底色**: 暗夜黑 (`#0B0F19` / Dark Charcoal) 与 炭灰色 (`#111827` / Carbon)

---

## 3. 梯级架构层级 (J-Ladder)
*   **L0 (一句话本质)**: Sanitizers 是一套集编译器插桩 (Compiler Instrumentation) 与运行时库 (Runtime Library) 于一体的动态程序 analysis 套件，利用影子内存映射 (Shadow Memory) 追踪状态、向量时钟 (Vector Clocks) 维护偏序关系、比特影子比特追踪初始化，以低于传统解释虚拟机的开销，在运行期精确检测内存越界/释放后使用、数据竞争、未初始化读取及未定义行为。
*   **L1 (四句话逻辑)**:
    1.  **插桩拦截与符号解构**: 编译器 LLVM Pass 在 IR 级别插入状态检查代码，运行时拦截器劫持关键 libc 系统调用，配合 llvm-symbolizer 实现高精度符号化和栈帧回溯。
    2.  **影子毒化与隔离防御**: ASan 以 8:1 比例建立影子内存，通过“毒化”插桩对象前后的红色区域 (Redzones) 拦截越界，并通过隔离区 (Quarantine) 延迟物理堆块重分配以捕捉释放后使用 (UAF)。
    3.  **向量时钟与影子竞争**: TSan 通过向量时钟跟踪线程间 Happens-Before 关系，结合每个 8 字节应用内存对应的影子状态（记录写线程、Epoch 和操作范围），在不加锁的情况下实现无竞争检测。
    4.  **比特追踪与保守扫描**: MSan 通过 1:1 比特级影子内存追踪初始化状态，配以来源追踪链定位根源；LSan 利用类似保守垃圾回收的根集合扫描，在程序退出时检索不可达内存实现泄漏报警。
*   **L2 (核心数据流转拓扑)**:
    *   `Compile C/C++ Code` ➜ `LLVM IR` ➜ `ASan/TSan/MSan Pass` ➜ `Insert Instrumentation Checks` ➜ `Link Runtime Library (libclang_rt)` ➜ `Application Startup` ➜ `Interceptors Hijack malloc()/free()` ➜ `Heap Allocation` ➜ `Poison Redzones (Shadow Memory set to 0xf1-0xf8)` ➜ `Application Write/Read` ➜ `Compile-Time Instrumented Check: (Addr >> 3) + Offset` ➜ `Shadow Byte Value != 0?` ➜ `Crash Report (ASan Global/Stack OOB)` ➜ `Free Object` ➜ `Move Page/Block to Quarantine (Poison Shadow Memory to 0xfd)` ➜ `Application Use-After-Free Access` ➜ `Shadow Byte == 0xfd Check` ➜ `Symbolic Stack Unwinding` ➜ `Report ASan UAF` ➜ `Thread A Write` ➜ `Record Vector Clock & Epoch` ➜ `Update TSan Shadow Cell (Thread, Epoch, Lock)` ➜ `Thread B Read (No happens-before link)` ➜ `Compare current Epoch with Vector Clock ➜ Data Race detected!` ➜ `Deadlock Check: Lock dependency graph circular loop` ➜ `UBSan runtime check: Integer overflow check (llvm.sadd.with.overflow)` ➜ `LSan Scan: Sweep stack/registers for pointers to allocated blocks ➜ Report memory leak`.
