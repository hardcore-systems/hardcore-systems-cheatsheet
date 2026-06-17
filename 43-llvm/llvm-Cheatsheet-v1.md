# LLVM 编译器基础设施核心原理高密卡片速查表 (v1)

*   **L0 一句话本质**: LLVM 是一个基于强类型 SSA 同构 IR 构建的、将前端解析、中端 Pass 独立优化、后端 SelectionDAG / 机器 IR / 寄存器分配解耦的现代编译器基础设施与工具链。
*   **L1 四句话逻辑**:
    1.  **SSA 同构设计与多流表示**: LLVM 采用静态单赋值形式的 IR，以同构的内存类、磁盘 bitcode、汇编文本三态流通，确保优化分析无缝解耦。
    2.  **Pass 驱动中端与解耦提升**: 统一的 PassManager 调度分析与转换 Pass，核心 Mem2Reg 基于支配边界将栈上 alloca 提升为 SSA 虚拟寄存器，驱动后续死代码消除和循环优化。
    3.  **合法选择与平铺机器指令**: 后端通过 SelectionDAG 将 IR 进行类型和操作合法化，TableGen 规则映射物理指令，指令调度器将其重新平铺为线性 MachineIR 序列。
    4.  **分配着色与物理文件发射**: 寄存器分配器根据活跃区间干涉图着色并处理溢出，PEI 构造真实物理栈帧，最终在 MC 层以段布局发射 ELF/Mach-O 二进制文件。
*   **L2 核心数据流转拓扑**:
    *   `Source Code` ➜ `Clang AST Visitor` ➜ `llvm::IRBuilder` ➜ `Raw LLVM IR (Alloca/Load/Store)` ➜ `llvm::verifyModule check` ➜ `PassManager (Mem2Reg)` ➜ `SSA IR (Registers & Phi)` ➜ `LICM/SimplifyCFG passes` ➜ `Optimized IR` ➜ `SelectionDAGISel construction` ➜ `SelectionDAG Graph` ➜ `Type/Operation Legalize` ➜ `ISel Pattern Match (TableGen .td)` ➜ `Target-Specific DAG` ➜ `ListScheduler Scheduling` ➜ `MachineInstr (MI)` ➜ `Live Interval Calculation` ➜ `RegAllocGreedy (Interference Graph Coloring)` ➜ `Spill check? Yes ➜ Insert spill load/store` ➜ `PEI (Prologue/Epilogue)` ➜ `MCStreamer emitter` ➜ `MCInst Stream` ➜ `ELF Object File / OrcJIT Execution`.

---

## 📂 M1: LLVM IR 核心语法与类型系统 (Cards 1-5)

#### Card 1. 静态单赋值 (SSA) 设计哲学、Phi 节点与 Def-Use 链
LLVM IR 的灵魂是静态单赋值（Static Single Assignment, SSA）形式：
1.  **SSA 规范**: 每个虚拟寄存器仅能在物理上被赋值一次。这极大地简化了中端的数据流分析与优化算法（如常量折叠与公共子表达式消除）。
2.  **Phi 节点**: 为了表达控制流汇合点（如 `if-else` 分支后）的变量选择，引入 `phi` 指令。它根据控制流“前驱基本块（Predecessor）”的来源来动态决定当前的变量值。
3.  **Def-Use & Use-Def 链**: 内存中每个 `Value` 类都维护着由其使用者组成的链表（Def-Use 链），而 `User` 类维护其引用的参数列表（Use-Def 链）。这使得编译器可以在 $O(1)$ 的时间内完成死代码判定与常量传播。

#### Card 2. LLVM IR 的三种同构表示 Module/Function
LLVM 保证其 IR 在编译的整个生命周期中具有完全同构的表达能力，支持三种等价形态：
1.  **内存表示 (In-memory Class)**: 由 C++ 对象组成。最顶层是 `llvm::Module`（表示一个编译单元），包含 `llvm::Function`（函数列表）、`llvm::GlobalVariable`（全局变量列表）以及 `llvm::BasicBlock`（基本块）。基本块是线性指令序列的容器。
2.  **磁盘 Bitcode (.bc)**: 紧凑的二进制紧缩编码格式，便于磁盘持久化与编译器快速读取加载。
3.  **文本汇编 (.ll)**: 人类可读的文本表示，便于调试和 Pass 分析排查。

#### Card 3. 强类型系统物理定义与安全验证
LLVM 拥有一个与目标平台无关的强类型系统，在编译早期就规避了非法内存访问与类型隐式转换：
1.  **基本类型**: 包含任意宽度的整型（如 `i1` 对应布尔，`i8` 对应字节，`i32`/`i64` 等）、浮点型（`float`/`double`）与 `void` 类型。
2.  **派生类型**: 包含数组 (`[N x Type]`)、结构体 (`{ Type1, Type2 }`)、指针 (`ptr`) 及面向 SIMD 向量化优化的向量类型 (`<V x Type>`)。
3.  **安全验证**: 任何类型转换必须显式使用转换指令（如整型截断 `trunc`、无符号扩展 `zext`、指针转换 `bitcast` 等），禁止任何隐式截断。

#### Card 4. 基本块 CFG 约束与终结符 Terminator
基本块（BasicBlock）是 LLVM IR 流程控制的基本原子物理单位：
1.  **基本块结构**: 满足“单入口，单出口”的控制流图（CFG）约束。每个基本块除了第一条指令外，其余指令均不可被外部直接跳转跳转；除最后一条指令外，其他指令也无法跳转出去。
2.  **终结符指令 (Terminator)**: 必须是基本块的最后一条指令。用于控制流程流向，包含无条件跳转 `br`、条件分支 `br i1`、函数返回 `ret`、跳转表 `switch`、或异常抛出 `resume`/`cleanupret`。
3.  **Entry 块约束**: 函数的第一个基本块（Entry）绝对不能拥有任何 CFG 前驱节点，以保证入口的确定性。

#### Card 5. alloca 栈变量、load/store 内存访问抽象
为了简化前端编译器的实现，LLVM 并不要求前端直接生成 SSA 格式的 IR：
1.  **内存分配抽象**: 前端可以将所有的局部变量平铺在栈上，通过 `alloca` 指令分配内存。
2.  **访存接口**: 读写局部变量使用 `load` 和 `store` 访存指令。由于栈变量是在内存中，不属于 SSA 虚拟寄存器，因此可以多次被 store 写入，绕开了 SSA 的“单次赋值”限制。
3.  **中端提升**: 这一步会产生大量冗余的内存读写，需要依赖中端的 `Mem2Reg` Pass 将其重命名提升为寄存器 SSA 结构。

---

## 📂 M2: 前端 AST 降级与 IR 生成 (Cards 6-10)

#### Card 6. Clang AST 遍历与 IRBuilder 生成翻译
Clang 前端解析生成 AST 树，随后通过 AST Visitor 访问模式生成 IR 模块：
1.  **类角色**: `clang::CodeGen::CodeGenModule` 维护全局生成上下文；`clang::CodeGen::CodeGenFunction` 维护函数内变量作用域、当前生成基本块与栈空间。
2.  **IRBuilder**: 提供 `llvm::IRBuilder<>` 工具类。在 AST 遍历（如语句节点 `Stmt`、表达式节点 `Expr`）时，流式地创建 LLVM IR 指令，并自动向当前插入点（Insertion Point）追加指令节点。
3.  **常量折叠**: IRBuilder 在创建指令时，会对操作数进行即时常量分析，若能进行折叠（如 `1+2` 自动转为 `3`）则直接返回常量对象，避免在中端制造垃圾指令。

#### Card 7. 控制流分支 br 条件降级与基本块拆分
高层语言复杂的流程控制结构（如 `if-else`、`while` / `for` 循环）需要被翻译为扁平的基本块跳转关系：
1.  **If-Else 降级**:
    *   拆分为三个新基本块：`then` 基本块、`else` 基本块和 `merge` 汇合块。
    *   在当前块根据条件表达式计算出 `i1` 结果，调用 `br i1 %cond, label %then, label %else` 进行条件跳转。
    *   在 `then` 和 `else` 基本块的末尾，强制插入 `br label %merge` 无条件跳转汇合。
2.  **短路求值**: 逻辑与 `&&`、逻辑或 `||` 表达式会被流式拆解成多个级联的条件跳转基本块，实现早期求值短路保护。

#### Card 8. 调用约定 ABI、参数传递与 landingpad 异常注册
函数调用涉及到 ABI（Application Binary Interface）的底座对齐与运行时异常捕获：
1.  **ABI 降级**: Clang 需要根据目标平台的 Calling Convention（如 `x86_64-sysv`）决定参数如何布局。超出物理寄存器宽度的结构体参数可能会被隐式转换为指针（`byval` 属性），或者拆解为多个整型进行参数传递。
2.  **异常捕获**: C++ 等高级语言的 `try-catch` 需要翻译为 LLVM 的异常处理指令：
    *   调用可能抛出异常的函数不能使用普通的 `call`，必须使用 `invoke`。
    *   `invoke` 指定正常流跳转的基本块，以及异常分支的汇合块 `unwind`。
    *   在 `unwind` 汇合块首行，必须声明 `landingpad` 指令，注册当前能捕获的类型元数据（TypeInfo），未匹配的异常继续向外传播 `resume`。

#### Card 9. 局部变量生存期 lifetime 与 Dwarf 调试信息
前端需要在 IR 中保留调试信息与高级生存期分析锚点，以备后端优化和调试器使用：
1.  **生存期标志**: 编译器在作用域入口和出口处分别插入内置函数调用 `llvm.lifetime.start` 与 `llvm.lifetime.end`。中端优化 Pass（如 `StackColoring`）据此对生命周期不重叠的栈变量进行合并，大幅压缩物理栈帧大小。
2.  **DIBuilder (Debug Information Builder)**: 用于生成元数据，它以 Dwarf 格式向 IR 节点挂载额外的说明。如局部变量元数据 `!DILocalVariable` 标明了源码行号、变量名，并用 `llvm.dbg.declare` 或 `llvm.dbg.value` 追踪变量在 SSA 变换后的物理位置。

#### Card 10. IR 验证器 verifyModule 静态完备性校验
在前端将 IR 交给中端优化链、或者中端 Pass 刚运行完毕后，必须强制运行 IR 验证器：
1.  **静态验证**: `llvm::verifyModule()` 会拓扑遍历 Module 内所有的函数与基本块，执行语义检查。
2.  **验证维度**:
    *   **SSA 支配性约束 (Dominance)**: 任何 SSA 变量 `Use` 指令的物理位置必须被其定义 `Def` 所在的指令所“支配”（即执行路径必然先经过 Def 块）。
    *   **终结符合法性**: 每个基本块的最后一条指令必须且仅能是一条 `Terminator` 指令，禁止中途出现跳转。
    *   **类型完备**: 校验操作数类型一致性（如二进制加法 `add` 两侧的类型必须完全一致，严禁 `i32` 与 `i64` 直接相加）。

---

## 📂 M3: 中端 Pass 管理器与经典优化 Pass (Cards 11-15)

#### Card 11. NewPassManager 调度与 Analysis/Transform 缓存失效
LLVM 现代 Pass 管理器通过流水线隔离，将编译器优化的执行效率推向极致：
1.  **NPM 模型**: 将 Pass 分为分析 Pass（`AnalysisPass`，如支配树 `DominatorTree`，不改变 IR 仅提取信息）与转换 Pass（`TransformPass`，如死代码消除，会修改 IR 结构）。
2.  **依赖与失效**: NPM 维护一个缓存中心。当 TransformPass 运行完毕时，它必须返回一个 `PreservedAnalyses` 结果：
    *   如果只修改了指令而没有改变 CFG，可以保留 `DominatorTreeAnalysis` 缓存。
    *   如果修改了控制流，NPM 会自动标记支配树失效，后续需要使用支配树的 Pass 会触发自动重算，实现精准惰性计算。

#### Card 12. Mem2Reg 支配边界算法与 SSA 重命名
`Mem2Reg` 是 LLVM 优化的开山核心 Pass，负责将前端平铺的 `alloca` 提升为纯 SSA 寄存器结构：
1.  **支配边界 (Dominance Frontiers)**:
    *   分析各局部变量，计算其所有 `store` 所在基本块的支配树边界。
    *   支配边界基本块代表多个定义分支汇聚的节点。在这些边界基本块的首部，插入 `phi` 节点，为变量分配新的虚拟寄存器。
2.  **SSA 重命名**:
    *   以深度优先搜索（DFS）遍历支配树，维护变量的最新定义栈。
    *   将局部的 `load` 操作直接替换为栈顶对应的最新虚拟寄存器；遇到 `store` 时，更新定义栈顶并将 store 指令安全删除。
3.  **零碎消除**: 最终将无用的 `alloca` 栈帧全部物理消除，内存操作彻底转化为寄存器数据流。

#### Card 13. 循环优化 LoopSimplify/LICM 与 ScalarEvolution
循环是程序优化的兵家重地，LLVM 具有一套完备的循环优化 Pass：
1.  **规整化 (LoopSimplify)**: 为循环强行构筑单独的入口前驱块（Preheader）、唯一的出口块（Exit Block）和唯一的返回跳转边（Latch Block），消除杂乱 CFG。
2.  **循环不变代码外提 (LICM)**: 遍历循环体。如果某条指令的操作数在循环内部从未发生变化，且该指令无副作用、其所在分支在循环出口必定执行，则将其提升移动到 `Preheader`，避免循环内重复执行。
3.  **标量演进分析 (Scalar Evolution, SCEV)**: 用于分析循环诱导变量（如循环中的索引 `i`）的数学闭式表达式。SCEV 将变量表示为 $\{Start, +, Step\}_L$ 的多项式形式，便于判定迭代次数、执行矢量化对齐。

#### Card 14. 经典冗余消除 EarlyCSE/ADCE 与 GVN 等价判定
冗余计算消除是提升执行效率的最直接手段：
1.  **EarlyCSE (公共子表达式消除)**: 基于支配树的前序遍历，维护一个哈希表记录当前已执行的表达式（如 `%3 = add i32 %1, %2`）。当支配树子节点中再次出现相同的 add 表达式时，直接用已有的寄存器 `%3` 替换，消除多余运算。
2.  **ADCE (侵略性死代码消除)**: 假设所有指令都是死的，除非它有外部副作用（如内存写入、函数返回、系统调用等）。以副作用指令为起点，沿 Use-Def 链反向逆推，只保留存活的数据流依赖链，其余无关指令一键删除。
3.  **GVN (全局数值编号)**: 为程序中所有表达式计算一个唯一的全局编号。与 EarlyCSE 相比，它支持跨控制流分支判定等价性（如通过值图分析，发现不同分支计算出的最终表达式数学值等价，予以合并）。

#### Card 15. SimplifyCFG 控制流简化与 Inliner 函数内联
精简控制流图与减少函数调用开销在中端执行非常频繁：
1.  **SimplifyCFG**: 
    *   合并只有一个后继块的无条件跳转链。
    *   将只有一个前驱的悬空基本块物理消除。
    *   将只有一个 `phi` 节点的分支结构（如 `if-then-else` 且两端皆是常量）提升为无分支的 `select` 指令，利用 CPU 的条件传送指令优化分支预测。
2.  **Inliner**:
    *   根据被调用函数的尺寸、调用频次（是否在循环内）以及控制流热度，计算一个内联代价估算值（Inline Cost）。
    *   若代价低于阈值，则将被调用函数的 IR 复制并展开在调用处，物理消除函数调用指令、参数压栈及返回开销，同时打通跨函数调用的数据流优化。

---

## 📂 M4: 后端独立代码生成 SelectionDAG (Cards 16-20)

#### Card 16. CodeGen 代码生成流水线阶段划分
LLVM 后端（CodeGen）是一个将高层 IR 变换为具体硬件指令的多阶段物理工厂：
1.  **阶段一**: `SelectionDAGISel` 将 LLVM IR 输入，翻译为目标无关的 SelectionDAG 选择图表示。
2.  **阶段二**: **类型/操作合法化 (Legalization)**。将目标硬件不支持的数据类型和操作降级合并。
3.  **阶段三**: **指令选择 (Instruction Selection)**。通过模式匹配将 DAG 节点翻译为目标物理指令。
4.  **阶段四**: **指令调度 (Scheduling)**。平铺 DAG，将其变回线性 MachineIR 序列。
5.  **阶段五**: **寄存器分配 (RegAlloc)**。将无限的虚拟寄存器映射为物理寄存器。
6.  **阶段六**: **PEI 与 MC 层发射**。构建物理栈帧并输出汇编/二进制文件。

#### Card 17. SelectionDAG 构建与类型/操作合法化
SelectionDAG 是一种用有向无环图（DAG）表达基本块内数据依赖和控制依赖的物理结构：
1.  **SelectionDAG 结构**: 节点（`SDNode`）表示操作，边（`SDValue`）表示数据流，此外还包含特有的“链边（Chain Edge）”以强制约束有副作用指令（如 Load/Store）的执行顺序，防止指令被错误重排。
2.  **类型合法化 (Type Legalize)**:
    *   **提升 (Promote)**: 如果硬件只有 `i32` 但 IR 有 `i8`，则将 `i8` 零扩展为 `i32` 进行运算。
    *   **拆分 (Split)**: 如果硬件是 32 位但 IR 存在 `i64`，将 `i64` 拆解为两个 `i32` 节点计算。
3.  **操作合法化 (Operation Legalize)**: 如果目标 CPU 缺少硬件除法，则在 DAG 中将 `sdiv` 节点替换为运行时辅助函数调用（如 `__divdi3`）。

#### Card 18. TableGen 匹配表驱动的 ISel 指令选择
指令选择（ISel）将目标无关的 SelectionDAG 节点折叠为真实机器指令的物理过程：
1.  **TableGen**: 开发者在目标描述文件（`.td`）中编写声明式模式匹配规则（如定义一个 X86 的 `ADD32rr` 指令，说明它匹配 DAG 的 `(add i32:$src1, i32:$src2)` 模式）。
2.  **编译期生成**: TableGen 工具自动将这些规则编译为一个字节码模式匹配表。
3.  **运行期匹配**: ISel 阶段运行一个状态机匹配引擎，遍历 SelectionDAG。当匹配到对应子图模式时，自动用物理指令节点（如 `ADD32rr`）替换原先目标无关的 `SDNode`，实现指令折叠。

#### Card 19. 指令调度 DAG Scheduling 与 MachineInstr 平铺
SelectionDAG 在选择完成后需要被平铺为线性的执行序列以供 CPU 执行：
1.  **物理矛盾**: SelectionDAG 是一个有向无环图，而 CPU 的执行流是线性顺序的。
2.  **指令调度**: 调度器（如 `ScheduleDAGMILive`）使用启发式策略（基于流水线延迟、寄存器压力等因素），遍历 DAG 节点，寻找一个能使 CPU 气泡最少、同时寄存器生存周期最短的线性序列。
3.  **MachineInstr (MI)**: 调度完成后，DAG 节点解构，转换为线性双向链表组织的通用机器指令 `MachineInstr` 形式，重新引入虚拟寄存器。

#### Card 20. GlobalISel 快速流水线通用机器指令 gMIR
为了解决 SelectionDAG 构建及两次 Legalize 导致内存占用大、编译速度缓慢的弊端，LLVM 引入了 GlobalISel（全局指令选择）：
1.  **gMIR (Generic MIR)**: 不构建复杂的 SelectionDAG，直接将中端 IR 翻译为通用机器指令 gMIR（如 `%vreg0 = G_ADD i32 %vreg1, %vreg2`），在整个 CodeGen 中保持线性 MIR 状态流通。
2.  **全局 Pass 设计**: Legalize、RegBankSelect 和 InstructionSelect 被解耦为标准的 MachineFunctionPass。它们可以跨基本块进行全局分析与优化，相比 SelectionDAG 仅能在基本块内处理，表现出更强的全局优化能力。
3.  **编译效率**: 大幅降低内存分配开销，编译吞吐量相较 SelectionDAG 有数倍的提升，尤其适用于 JIT 和 Debug 快速编译场景。

---

## 📂 M5: 目标特定机器 IR 与寄存器分配 (Cards 21-24)

#### Card 21. MachineIR (MIR) 结构、虚拟/物理寄存器类
MachineIR 是后端优化的核心容器，它无限逼近物理汇编代码，但在寄存器分配前依然拥有无限虚拟寄存器：
1.  **结构层次**: 最外层是 `MachineFunction`，包含 `MachineBasicBlock` (MBB) 和 `MachineInstr` (MI)。
2.  **MachineOperand**: 指令内的操作数可以是虚拟寄存器（`%stack_val`）、物理寄存器（`$rax`）、立即数或内存寻址符号。
3.  **RegisterClass**: 虚拟寄存器必须绑定到一个寄存器类（如 x86 的 `GR32` 对应 32 位通用寄存器）。寄存器类定义了该虚拟寄存器在分配时可选用的物理寄存器集合。

#### Card 22. RegAllocGreedy 着色干涉图、溢出与 Spill 插入
LLVM 默认采用高度优化的贪婪寄存器分配器（RegAllocGreedy）将虚拟寄存器映射到物理硬件寄存器：
1.  **活跃区间分析 (Live Intervals)**: 计算每个虚拟寄存器在程序中从定义到最后一次使用的有效活跃指令行范围。
2.  **干涉图构建**: 判断不同虚拟寄存器的活跃区间是否重叠。如果重叠，则在干涉图中建立一条边，表示两者不能共享同一个物理寄存器。
3.  **着色与分配**: 
    *   以优先级排序（活跃长度越短、循环内引用越多的变量优先级越高），依次为其分配可用的物理寄存器。
    *   如果物理寄存器已被用尽导致图无法着色，则判定为发生“溢出（Spill）”。
4.  **Spill 指令插入**: 挑选溢出代价最小的寄存器，在物理栈帧上为其分配一个插槽（Stack Slot），并在该变量的定义点后插入 `store`（写入内存栈），在其使用点前插入 `load`（从栈重载入内存），并分割其活跃区间重新进行分配。

#### Card 23. PEI 栈帧布局重建与 Callee-Saved 恢复
在寄存器分配结束后，所有的虚拟寄存器均被替换为物理寄存器，但栈帧尚未真正构筑：
2.  **栈帧重建 (PrologEpilogInserter, PEI)**:
    *   计算当前函数实际使用的栈帧大小（包含溢出的 Spill 插槽、局部 `alloca` 分配的数组空间）。
    *   根据目标架构的对齐要求（如 x86_64 要求栈指针 16 字节对齐）确定栈指针偏移量。
2.  **物理生成**:
    *   在函数的入口基本块（Entry）首部，插入 Prologue 汇编（如调整 SP 指针 `subq $32, %rsp`，并保存帧指针 FP）。
    *   在函数的所有退出 `ret` 块前，插入 Epilogue 汇编（恢复 SP 并恢复 Callee-Saved 寄存器）。
    *   **Callee-Saved 寄存器**: 根据 ABI 规范，如果函数改变了被调用者保存寄存器（如 `%rbx`、`%r12`），必须在 Prologue 中将其 push 压栈，在 Epilogue 中 pop 弹栈恢复，保证调用者环境不被污染。

#### Card 24. TableGen .td 物理架构寄存器与指令定义
TableGen 是整个 LLVM 代码生成器在异构物理架构上扩展的灵魂：
1.  **TableGen 语言**: 是一种强类型的记录式声明语言，文件后缀为 `.td`。
2.  **描述对象**:
    *   **寄存器集**: 描述目标 CPU 有哪些物理寄存器，它们的名称、别名、重叠关系（如 `EAX` 是 `RAX` 的低 32 位）。
    *   **指令集**: 描述指令的助记符（如 `add`）、二进制机器编码特征、操作数列表（输入/输出类型）、执行延迟及隐式寄存器改写副作用。
    *   **调用约定**: 描述参数传递时优先使用哪些物理寄存器（如 x86_64 的 `%rdi`, `%rsi`, `%rdx`），超出时如何排布在栈上。
3.  **CodeGen 自动生成**: LLVM 构建时，TableGen 编译器（`llvm-tblgen`）读取 `.td` 文件，在编译期自动生成大量的 C++ 模式匹配表和类型判定代码，无缝挂载到 CodeGen 的各个环节。

---

## 📂 M6: MC 层、目标文件发射与 JIT 引擎 (Cards 25-28)

#### Card 25. MC 层 MCStreamer/MCInst 指令流式编码与反汇编
MC（Machine Code）层是 LLVM 工具链中负责最终生成目标文件与汇编文本的终极网关：
1.  **MCInst**: 相比 MachineInstr，MCInst 被剥离了所有编译器特有的控制流和虚拟寄存器属性，仅仅包含物理寄存器 ID、立即数和机器指令码，是物理指令的最纯粹抽象。
2.  **MCStreamer**: 提供了一套统一的流式发射器接口。如果编译输出汇编文件，它会选择 `MCAsmStreamer` 格式化打印人类可读文本；如果编译输出二进制目标文件，它会选择 `MCObjectStreamer` 将指令流传递给编码器。
3.  **流式汇编/反汇编**: 编码器流式计算指令字节码；反汇编器则接收物理二进制数据流并根据 `.td` 生成的匹配树重构为 `MCInst` 打印，实现双向映射。

#### Card 26. 目标二进制 ELF/Mach-O 段生成与符号重定位表
MC 层的 ObjectWriter 负责将 MCInst 流封包为操作系统能识别的标准执行格式（Linux 的 ELF，macOS 的 Mach-O，Windows 的 COFF）：
1.  **段生成**: 将代码区转换为 `.text` 二进制数据段，将只读常量放入 `.rodata` 段，将全局变量写入 `.data` 段。
2.  **符号表重建**: 提取函数、全局变量的符号名称，填充到 `.symtab` 物理段中，并记录每个符号在对应段内的字节偏移。
3.  **重定位表 (Relocations)**:
    *   由于在单模块编译期，调用外部函数（如 `printf`）或者访问全局变量的物理绝对地址是未知的。
    *   编译器会在调用处先填充占位符，并在 `.rela.text` 重定位段中生成一条记录，说明“在模块的哪一个字节偏移处，引用了哪一个外部符号，在链接时必须使用何种寻址模式（如相对寻址 `R_X86_64_PC32`）进行地址修正”。

#### Card 27. OrcJIT 异步编译模型、JITLink/RuntimeDyld 动态链接
LLVM 提供的 JIT (Just-In-Time) 引擎支持在进程运行期间动态将 IR 编译并加载到内存中执行：
1.  **OrcJIT 架构**: 采用基于 Layer（层）的模块化堆叠设计。最高层是 `LLJIT` 易用接口，底层是负责接收 IR 的 `IRCompileLayer`，再到底层接收机器码并分配内存的 `ObjectLinkingLayer`。
2.  **异步按需编译**: OrcJIT 允许注册模块而不立即编译。只有当程序第一次调用某符号的函数指针时，才触发后台工作线程的按需编译与加载，极大地缩短了启动时间。
3.  **JITLink**: 新一代 JIT 内存链接器，取代了旧版的 `RuntimeDyld`。它直接在内存中执行目标文件的解析，并模拟操作系统的动态链接过程：在物理内存中为代码区申请 `r-x`（可读可执行）页，为数据区申请 `r--` 页，然后根据重定位表直接将内存地址填入指令中，最后安全交付执行。

#### Card 28. LTO & ThinLTO 跨模块优化与 Bitcode 局部修剪
LLVM 率先在业界大规模普及并商用了链接时优化（Link-Time Optimization, LTO）技术：
1.  **传统编译瓶颈**: 传统编译器以模块（`.cpp`）为单位独立优化，链接器仅做符号拷贝，无法跨模块内联。
2.  **Monolithic LTO**: 各模块不直接编译为汇编，而是输出 LLVM IR Bitcode。在链接期，LTO 驱动器将所有 bitcode 读入同一个庞大的内存 Module 中，执行全局死代码擦除与跨模块内联，但内存和时间消耗极大。
3.  **ThinLTO (轻量级 LTO)**: 
    *   **第一阶段**: 在单模块编译期，为各模块生成 bitcode 的同时，生成一个轻量级的摘要符号表（包含函数尺寸、全局导入导出关系）。
    *   **第二阶段**: 在链接期，ThinLTO 驱动器快速加载所有摘要表并分析。如果发现 A 模块需要内联 B 模块的某函数，只将 B 模块的该函数 IR 进行局部导入，而无须合并整个 B 模块。这使得跨模块优化可以在多个 CPU 核心上完全并行执行，其内存和编译时间开销直逼常规链接，而性能几乎等同于 Monolithic LTO。

---

## 📂 LLVM 优化与代码生成策略折衷矩阵

| 设计维度 | 方案 A | 方案 B | 折衷平衡 |
| :--- | :--- | :--- | :--- |
| **局部变量表达形式** | 局部栈分配访存模型 (Alloca + Load/Store) | 纯静态单赋值寄存器模型 (Pure SSA Registers) | 前端代码生成极为简单，无需考虑复杂的控制流变量合并与作用域分析 ➜ 但制造了海量内存冗余读写，数据流完全断裂；纯 SSA 在生成期就要通过复杂的算法推导分支合并并插入 Phi，算法复杂 ➜ 但有利于极致的中端分析。[LLVM 选择在前端使用 Alloca 规避前端复杂性，在中端通过开山 Pass Mem2Reg 提升为纯 SSA 寄存器，实现各阶段的最优平衡] |
| **指令选择流水线** | 传统的 SelectionDAG 机制 (DAG-based ISel) | 新一代的 GlobalISel 机制 (gMIR-based ISel) | SelectionDAG 将基本块转换为图，类型和操作的合法化极其精准，匹配度极高 ➜ 但每次构造 DAG 会产生数倍的内存开销与图遍历耗时，且无法跨基本块全局优化；GlobalISel 在编译全程使用线性的通用 MachineIR (gMIR) 流通，内存极小，支持全局 Pass ➜ 但早期目标架构 TableGen 规则支持度较低 [对于快速 JIT 与调试编译，选用 GlobalISel；对于高阶静态代码生成优化，选用 SelectionDAG] |
| **寄存器分配策略** | 基于图着色的 Kempe 算法 (Graph Coloring RegAlloc) | 基于局部代价分析的贪婪算法 (Greedy RegAlloc) | 图着色算法将分配映射为 NP-Complete 的图着色问题，在变量较少时能给出理论最优分配 ➜ 但在大型程序中，由于其需要迭代重构干涉图，编译速度较慢且易发生死锁；贪婪分配器采用启发式区间拆分、溢出开销评估并结合局部扫描，编译速度快且易于扩展 ➜ 可能会在个别边界情况损失少量寄存器利用率 [LLVM 默认采用 Greedy 算法，大幅优化了大型 C++ 项目的后端编译性能体验] |
| **跨模块优化深度** | 全量合并链接优化 (Monolithic LTO) | 摘要驱动轻量链接优化 (ThinLTO) | Monolithic LTO 全局合并所有 Bitcode，信息绝对完备，跨模块函数内联及死代码消除最彻底 ➜ 但多模块合并会导致链接期内存爆炸，耗时巨大，无法并行；ThinLTO 通过轻量摘要表，仅局部按需导入关联的 IR 数据，支持全核心并行编译 ➜ 损失了极其微弱的超全局死代码分析精度 [牺牲了极限状态下的全量分析，换取了链接时间暴跌与生产环境可用性的飞跃] |

---

## 🔬 Zone T: LLVM 关键编译参数与 IR 汇编特征速查字典

### T1: 核心 LLVM/Clang 命令行控制与运行期调试标志
*   `clang++ -O3 -mllvm -print-after-all main.cpp` ：追踪中端所有 Pass 变化。在每个中端 Pass 运行完毕后，在标准输出打印出当前最新的整个 IR 文本，便于在数百个优化 Pass 中精准定位是哪个 Pass 污染了代码或者制造了 Bug。
*   `clang++ -Xclang -ast-dump -fsyntax-only main.cpp` ：强制输出 Clang 语法树（AST）。在终端流式输出彩色缩进的抽象语法树，包含节点的内存地址、作用域范围、类型定义（如 `ParmVarDecl`），用于排查降级生成 IR 前的前端解析正确性。
*   `opt -passes='mem2reg,simplifycfg' -S input.ll -o output.ll` ：独立调用中端优化器。使用新一代 NPM 执行指定的优化组合（本例为先提升 SSA 再简化控制流），输出优化后的文本格式 `.ll` IR 文件。
*   `llc -O2 -march=x86-64 -mcpu=skylake -print-after-isel input.ll` ：调试后端代码生成。将 LLVM IR 翻译为 X86 汇编，并输出 ISel 模式匹配完成后、机器指令平铺前的底层机器 DAG 节点和通用机器指令序列。

### T2: LLVM IR 汇编指令特征与 SSA 结构模式
在 LLVM 文本汇编中，可以清晰观测到中端优化的核心指令模式：
*   **栈变量定义与 Mem2Reg 优化对比**:
    在未开启优化的 IR 中，局部变量（如 C 中的 `int a = 5;`）表现为频繁的访存：
    ```llvm
    %a = alloca i32, align 4              ; 在当前物理栈上分配 4 字节
    store i32 5, ptr %a, align 4          ; 将立即数 5 写入栈内存 %a 处
    %1 = load i32, ptr %a, align 4         ; 从栈内存 %a 读取到虚拟寄存器 %1
    %add = add nsw i32 %1, 10             ; 执行加法运算
    ```
    经过 `Mem2Reg` 优化后，上述代码会被彻底重命名，消除栈空间分配和读写：
    ```llvm
    ; alloca, store, load 指令被全量抹去
    %add = add nsw i32 5, 10               ; 常量直接通过 SSA 关系下传，直接生成常量表达式
    ```
*   **Phi 节点与控制流汇聚特征**:
    在存在分支的结构中，不同路径可能对同一个逻辑变量 `%val` 进行了多次 store。在合并块 `%merge` 的首部，通过 `phi` 指令将不同的前驱来源收拢：
    ```llvm
    then:
      %val_then = add i32 %x, 1
      br label %merge

    else:
      %val_else = sub i32 %x, 1
      br label %merge

    merge:
      ; phi 语法: 根据前驱基本块的实际执行路径，挑选对应的寄存器值
      %val = phi i32 [ %val_then, %then ], [ %val_else, %else ]
      ret i32 %val
    ```
