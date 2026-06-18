# LLVM 核心原理高密卡片系统题材判定

## 1. 核心模块与 28 张卡片大纲

### M1: LLVM IR 核心语法与类型系统 (Cards 1-5)
*   **Card 1**: 静态单赋值 (SSA) 设计哲学、Phi 节点物理定义与 Def-Use/Use-Def 链遍历机制。
*   **Card 2**: LLVM IR 的三种同构表示：内存类结构 (`llvm::Module`/`Function`/`BasicBlock`/`Value`)、磁盘 Bitcode (`.bc`) 与文本汇编 (`.ll`)。
*   **Card 3**: 强类型系统物理定义：基本类型（浮点、整型）、派生类型（指针、结构体、数组、向量）与类型安全验证。
*   **Card 4**: 基本块 (Basic Block) 与控制流图 (CFG) 约束、终结符指令 (Terminator) 与 Entry 块无前驱约束。
*   **Card 5**: 内存访问抽象与 SSA 的矛盾：`alloca` 栈帧变量分配、`load`/`store` 内存交互与 SSA 寄存器重命名基础。

### M2: 前端 AST 降级与 IR 生成 (Cards 6-10)
*   **Card 6**: 语法树降级模式：从 Clang AST 遍历到 LLVM IR 生成器 (`llvm::IRBuilder`) 的翻译流转。
*   **Card 7**: 控制流结构降级：分支指令 (`br`, `indirectbr`) 的条件展开、短路求值及基本块的拆分合并。
*   **Card 8**: 表达式与函数调用降级：调用约定 (Calling Convention) 映射、参数传递 ABI 规范（整型扩展、结构体值传递）与异常处理 `landingpad` 注册。
*   **Card 9**: 高级编译标志与元数据：Clang 局部变量生存期标记 (`llvm.lifetime.start/end`) 与调试信息元数据 (Dwarf & `llvm::DIBuilder`)。
*   **Card 10**: IR 验证器 (`llvm::verifyModule`)：对类型匹配、SSA 支配性约束 (Dominance)、基本块终结符的静态完备性校验。

### M3: 中端 Pass 管理器与经典优化 Pass (Cards 11-15)
*   **Card 11**: 新一代 Pass 管理器 (`NewPassManager`) 架构：分析 Pass (`AnalysisPass`)、转换 Pass (`TransformPass`) 与缓存失效判定机制。
*   **Card 12**: Mem2Reg 核心算法：基于支配边界 (Dominance Frontiers) 的内存变量提升、Phi 节点精简插入与 SSA 重命名。
*   **Card 13**: 循环优化体系：循环规整化 (LoopSimplify)、循环不变代码外提 (LICM)、循环展开 (LoopUnroll) 与标量演进分析 (Scalar Evolution)。
*   **Card 14**: 经典冗余消除：公共子表达式消除 (EarlyCSE)、静态死代码消除 (ADCE) 与全局数值编号 (GVN) 的等价性判定。
*   **Card 15**: 控制流与过程间优化：基本块合并与简化 (SimplifyCFG)、函数内联 (Inliner) 决策因子与过程间常量传播 (IPCP)。

### M4: 后端独立代码生成 (DAG / SelectionDAG) (Cards 16-20)
*   **Card 16**: 代码生成器 (CodeGen) 核心流水线：从中端 IR 降级到机器相关汇编与目标二进制文件的物理阶段划分。
*   **Card 17**: 选择图 (SelectionDAG) 的构建与合法化 (Type/Vector/Operation Legalization)：将 IR 平铺转换为目标无关的有向无环图。
*   **Card 18**: 指令选择 (ISel) 机制：利用 TableGen (`.td`) 自动生成的匹配表对 DAG 节点进行目标指令模式匹配的匹配引擎。
*   **Card 19**: 指令调度 (DAG Scheduling) 与寄存器分配前置准备：通过启发式算法（如 ListScheduler）将 SelectionDAG 平铺回 SSA MachineInstr (MI) 序列。
*   **Card 20**: GlobalISel 架构：跳过 DAG 构建的新一代全局指令选择器，直接在通用机器指令 (gMIR) 上执行合法化、选择与翻译的快速流水线。

### M5: 目标特定机器 IR 与寄存器分配 (Cards 21-24)
*   **Card 21**: MachineIR (MIR) 核心物理结构：物理寄存器、虚拟寄存器、寄存器类 (RegisterClass) 与 MachineBasicBlock 物理布局。
*   **Card 22**: 寄存器分配 (RegAlloc) 贪婪算法：活跃区间 (Live Intervals) 计算、干涉图构建、着色简化堆栈、溢出权重度量与 Spill 内存指令插入。
*   **Card 23**: 函数栈帧与栈帧重建 (Prologue/Epilogue Insertion)：栈帧布局确定、栈指针/帧指针修正、Callee-Saved 寄存器保存与恢复。
*   **Card 24**: 目标描述文件 (TableGen `.td`)：物理架构定义语言，描述寄存器集、指令编码特征、流水线模型及匹配规则。

### M6: MC 层、目标文件发射与 JIT 引擎 (Cards 25-28)
*   **Card 25**: MC (Machine Code) 核心层：统一的流式指令编码器 (MCStreamer)、汇编器/反汇编器与目标指令集定义 (MCInst)。
*   **Card 26**: 目标二进制文件发射：ELF/Mach-O 物理格式段生成、符号表重建与未定位符号重定位项 (Relocation Entries) 填充。
*   **Card 27**: LLVM 即时编译 (JIT)：新一代 OrcJIT 异步编译模型、动态链接器 JITLink / RuntimeDyld 与符号解析器运作细节。
*   **Card 28**: 链接时优化 (LTO) 与 ThinLTO：全局代码跨模块合并、序列化 Bitcode 导入导出、并行流水线与局部/全局符号链接修剪。

---

## 2. 莫兰迪色系设计配置 (Morandi Palette)
为了实现视觉设计上的风格化与一致性，LLVM 编译器基础设施的 cheatsheet 与 HTML 知识大图采用以下精选莫兰迪色系：

*   **M1**: 莫兰迪蓝灰色 (`#4B5F7A` / Slate Blue) - LLVM IR 核心语法与类型
*   **M2**: 莫兰迪绿灰色 (`#6B8272` / Muted Sage) - 前端 AST 降级与 IR 生成
*   **M3**: 莫兰迪铁灰色 (`#7A7A7A` / Iron Grey) - 中端 Pass 管理器与经典优化
*   **M4**: 莫兰迪茶红色 (`#9C6666` / Tea Red) - 后端独立代码生成 SelectionDAG
*   **M5**: 莫兰迪金黄色 (`#9A825A` / Dusty Gold) - 机器 IR 与寄存器分配
*   **M6**: 莫兰迪紫灰色 (`#755B77` / Muted Grape) - MC 层、目标文件与 JIT 引擎
*   **L0-L2 梯级底色**: 暗夜黑 (`#0B0F19` / Dark Charcoal) 与 炭灰色 (`#111827` / Carbon)

---

## 3. 梯级架构层级 (J-Ladder)
*   **L0 (一句话本质)**: LLVM 是一个基于强类型 SSA 同构 IR 构建的、将前端解析、中端 Pass 独立优化、后端 SelectionDAG / 机器 IR / 寄存器分配解耦的现代编译器基础设施与工具链。
*   **L1 (四句话逻辑)**:
    1.  **SSA 同构设计与多流表示**: LLVM 采用静态单赋值形式 of IR，以同构的内存类、磁盘 bitcode、汇编文本三态流通，确保优化分析无缝解耦。
    2.  **Pass 驱动中端与解耦提升**: 统一的 PassManager 调度分析与转换 Pass，核心 Mem2Reg 基于支配边界将栈上 alloca 提升为 SSA 虚拟寄存器，驱动后续死代码消除和循环优化。
    3.  **合法选择与平铺机器指令**: 后端通过 SelectionDAG 将 IR 进行类型和操作合法化，TableGen 规则映射物理指令，指令调度器将其重新平铺为线性 MachineIR 序列。
    4.  **分配着色与物理文件发射**: 寄存器分配器根据活跃区间干涉图着色并处理溢出，PEI 构造真实物理栈帧，最终在 MC 层以段布局发射 ELF/Mach-O 二进制文件。
*   **L2 (核心数据流转拓扑)**:
    *   `Source Code` ➜ `Clang AST Visitor` ➜ `llvm::IRBuilder` ➜ `Raw LLVM IR (Alloca/Load/Store)` ➜ `llvm::verifyModule check` ➜ `PassManager (Mem2Reg)` ➜ `SSA IR (Registers & Phi)` ➜ `LICM/SimplifyCFG passes` ➜ `Optimized IR` ➜ `SelectionDAGISel construction` ➜ `SelectionDAG Graph` ➜ `Type/Operation Legalize` ➜ `ISel Pattern Match (TableGen .td)` ➜ `Target-Specific DAG` ➜ `ListScheduler Scheduling` ➜ `MachineInstr (MIR)` ➜ `Live Interval Calculation` ➜ `RegAllocGreedy (Interference Graph Coloring)` ➜ `Spill check? Yes ➜ Insert spill load/store` ➜ `PEI (Prologue/Epilogue)` ➜ `MCStreamer emitter` ➜ `MCInst Stream` ➜ `ELF Object File / OrcJIT Execution`.
