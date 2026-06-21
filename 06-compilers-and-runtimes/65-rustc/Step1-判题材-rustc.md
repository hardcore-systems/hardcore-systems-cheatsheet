# Step1-判题材-rustc.md: Rust 官方编译器内核、HIR/MIR 变换与 NLL 借用检查器审计大纲

本审计项目聚焦于 Rust 官方编译器 `rustc` 的内部机制。我们将通过 28 张高密度核心知识卡片，深度解构 rustc 语法解析与宏展开、AST 至 HIR 的降低（Lowering）、Hindley-Milner 类型推导与 Trait 协同性判定、HIR 至 MIR 的转换、Non-Lexical Lifetimes (NLL) 区域推导、借用检查器（Borrow Checker）冲突判定、生命周期周期分析、泛型单态化（Monomorphization）与虚表（VTable）生成、以及 LLVM 机器码生成桥接。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: AST, HIR 与 Lowering** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - Tokenizer 与 Lexer 词法解析、宏展开与名称解析、AST 至 HIR 降级、类型推论与 HM 算法、Trait 协同性与一致性校验、方法解析与 Deref 强转。
*   **M2: MIR 降低与 NLL 借用检查器** (Cards 7–12) - `#6B8272` (Muted Sage)
    - HIR 至 MIR 降级（CFG 构建）、MIR 数据流分析、NLL 非词法生命周期域推导、借用校验与生命周期冲突判定、Drop 细化与清理路径、MIR 级编译优化。
*   **M3: 代码生成与 LLVM 桥接** (Cards 13–19) - `#9C6666` (Tea Red)
    - 泛型静态分派单态化、动态分派与虚表（VTable）生成、内存物理布局与 Auto Traits 边界、LLVM IR 代码翻译、链接器桥接与 LTO、增量编译与 Query 系统、堆栈回溯与 Panic 运行时。
*   **M4: 安全边界与常量求值** (Cards 20–24) - `#7A7A7A` (Iron Grey)
    - Safe/Unsafe 静态边界检查、未定义行为与 Miri 解释器、CTFE 常量求值引擎、线程安全 Send/Sync 约束性判定、宏卫生性与 proc-macro 桥接。
*   **M5: 诊断与底层架构** (Cards 25–28) - `#9A825A` (Dusty Gold)
    - 诊断引擎与 suggestions 渲染、Query 缓存与增量底座、Unstable Feature Gate 栅栏、Crate 元数据序列化。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    rustc 的本质是将高阶声明式的 Rust 源码在编译期通过多级中间表示（AST ➜ HIR ➜ MIR ➜ LLVM IR）递降解构，并在 MIR 阶段通过基于 CFG 点集求值和非词法区间（NLL）拓扑推导的借用检查器，实施零运行时开销的内存与并发安全静态规约。
*   **L1 四句话逻辑**：
    1. **多级表示平滑递降**：通过 AST（语法树）、HIR（高阶脱糖树）、MIR（中阶控制流图）逐步剥离语法糖，暴露清晰的基本块（Basic Blocks）与局部变量读写。
    2. **静态借用与生命周期规约**：NLL 检查器根据控制流图的活跃点计算引用的生命周期区间，检测是否存在“读写冲突”或“悬空指针”，将安全隐患拦截在编译期。
    3. **零成本泛型与动态分派**：编译器通过单态化（复制膨胀泛型）实现极致的静态零成本抽象，或通过生成物理 VTable 表及指针偏移实现显式的动态分派。
    4. **全局查询驱动的增量缓存**：采用基于红黑依赖图（Dependency Graph）的全局 Query 架构，将编译拆分为细粒度的原子查询并提供磁盘级缓存，实现闪电级的增量二次编译。
*   **L2 核心数据流转拓扑**：
    `Rust 源码` ➜ `Lexing/Parsing (AST)` ➜ `Name Resolution/Macro Expansion` ➜ `Lowering (HIR)` ➜ `Type Checking & Trait Selection` ➜ `Lowering (MIR CFG)` ➜ `NLL Region Inference` ➜ `Borrow Checking (Lifetimes Validation)` ➜ `Monomorphization (Concrete IR)` ➜ `Codegen (LLVM IR)` ➜ `Linker (Native Target Binary)`

---

## 🗂️ 28 张核心知识卡片大纲
1. Tokenizer 与 Lexer 词法解析：源码到 Token 流的增量切片与 Span 物理定位。
2. 宏展开与名称解析：宏嵌套展开、名称导入域绑定与符号索引构建。
3. AST 至 HIR 降级：语法糖剥离、闭包捕获分析与高阶脱糖树表示。
4. 类型推论与 HM 算法：类型变量声明、合一化（Unification）与类型方程求解。
5. Trait 协同性与一致性：特化（Specialization）与孤儿规则等重叠规则静态校验。
6. 方法解析与 Deref 强转：隐式 Deref 链搜寻、候选方法匹配与方法签名绑定。
7. HIR 至 MIR 降级：控制流图（CFG）构建、临时变量分配与语句块扁平化。
8. MIR 数据流分析：定义-使用（Def-Use）链、活跃变量分析与控制流汇合状态求值。
9. NLL 非词法生命周期：控制流图上的点集生存期计算与生命周期自适应收缩。
10. 借用校验与冲突判定：读写锁模型（独占写/共享读）在生命周期内的重叠校验与悬空引用拦截。
11. Drop 细化与清理路径：Drop 标志插入、移动语义边界识别与异常栈清理分支。
12. MIR 级编译优化：局部常量传播、死代码消除、函数内联与复制消除。
13. 泛型静态分派单态化：泛型函数根据具体类型参数复制膨胀与符号去重。
14. 动态分派与虚表生成：Trait Object 内部双指针结构、VTable 生成与函数指针偏移跳转。
15. 内存物理布局与 Auto Traits：Struct 字段重排优化、对齐校验与 Send/Sync 约束性静态分析。
16. LLVM IR 代码翻译：MIR 转换至 LLVM IR、类型系统映射与 Intrinsic 处理。
17. 链接器桥接与 LTO：多 Crate LLVM 字节码链接、跨模块优化与目标平台连接器调用。
18. 增量编译与 Query 系统：编译过程拆分为原子 Query、缓存索引与增量失效。
19. 堆栈回溯与 Panic 运行时：Landing Pads 异常捕获、Unwind 栈帧析构与 Panic 中断处理。
20. Safe/Unsafe 静态边界检查：Unsafe 块作用域划分、原始指针解引用与静态条件防线。
21. 未定义行为与 Miri 解释器：Stacked Borrows 别名模型、未定义行为运行时检测。
22. CTFE 常量求值引擎：编译期虚拟机执行、常量函数求值与类型擦除常量缓存。
23. 线程安全 Send/Sync 约束：多线程数据共享与多线程所有权转移在编译期的严苛审查。
24. 宏卫生性与 proc-macro 桥接：名称捕获防范、进程间通信（IPC）ProcMacro 编译器桥。
25. 诊断引擎与 suggestions 渲染：多样式 Span 高亮、编译错误码输出与一键自动修复建议。
26. Query 缓存与增量底座：依赖依赖图构建、红黑节点重打标与脏数据失效算法。
27. Unstable Feature Gate 栅栏：未稳定 API 限制、编译器通道控制与 feature 门控编译。
28. Crate 元数据序列化：Crate 导出元数据编码（rmeta）、依赖包解析与跨跨包加载。
