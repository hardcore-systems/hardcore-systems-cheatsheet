# LLVM 核心原理高密度卡片系统设计大图

## 1. 28张卡片依赖拓扑关系图

```mermaid
graph TD
    %% M1: LLVM IR Core Syntax & Type System
    Card1["Card 1: SSA 设计、Phi 节点与 Def-Use/Use-Def 链"] --> Card2["Card 2: LLVM IR 三态同构表示 Module/Function"]
    Card2 --> Card3["Card 3: 强类型系统物理定义与安全验证"]
    Card2 --> Card4["Card 4: 基本块 CFG 约束与终结符 Terminator"]
    Card1 --> Card5["Card 5: alloca 栈变量、load/store 内存访问抽象"]

    %% M2: AST Lowering & IR Gen
    Card6["Card 6: Clang AST 遍历与 IRBuilder 生成翻译"] --> Card7["Card 7: 控制流分支 br 条件降级与基本块拆分"]
    Card6 --> Card8["Card 8: 调用约定 ABI、参数传递与 landingpad 异常注册"]
    Card6 --> Card9["Card 9: 局部变量生存期 lifetime 与 Dwarf 调试信息"]
    Card2 --> Card10["Card 10: IR 验证器 verifyModule 静态完备性校验"]

    %% M3: NewPassManager & Transformations
    Card10 --> Card11["Card 11: NewPassManager 调度与 Analysis/Transform 缓存失效"]
    Card5 --> Card12["Card 12: Mem2Reg 支配边界算法与 SSA 重命名"]
    Card11 --> Card13["Card 13: 循环优化 LoopSimplify/LICM 与 ScalarEvolution"]
    Card11 --> Card14["Card 14: 冗余消除 EarlyCSE/ADCE 与 GVN 等价判定"]
    Card4 --> Card15["Card 15: SimplifyCFG 控制流简化与 Inliner 函数内联"]

    %% M4: SelectionDAG & ISel
    Card2 --> Card16["Card 16: CodeGen 代码生成流水线阶段划分"]
    Card16 --> Card17["Card 17: SelectionDAG 构建与类型/操作合法化"]
    Card17 --> Card18["Card 18: TableGen 匹配表驱动的 ISel 指令选择"]
    Card18 --> Card19["Card 19: ListScheduler 指令调度平铺回 MachineInstr"]
    Card16 --> Card20["Card 20: GlobalISel 快速流水线通用机器指令 gMIR"]

    %% M5: MachineIR & Register Allocation
    Card19 --> Card21["Card 21: MachineIR (MIR) 结构、虚拟/物理寄存器类"]
    Card21 --> Card22["Card 22: RegAllocGreedy 着色干涉图、溢出与 Spill 插入"]
    Card22 --> Card23["Card 23: PEI 栈帧布局重建与 Callee-Saved 恢复"]
    Card18 --> Card24["Card 24: TableGen .td 物理架构寄存器与指令定义"]

    %% M6: MC Layer, Object Emission & JIT
    Card23 --> Card25["Card 25: MC 层 MCStreamer/MCInst 指令流式编码与反汇编"]
    Card25 --> Card26["Card 26: 目标二进制 ELF/Mach-O 段生成与符号重定位表"]
    Card21 --> Card27["Card 27: OrcJIT 异步编译模型、JITLink/RuntimeDyld 动态链接"]
    Card2 --> Card28["Card 28: LTO & ThinLTO 跨模块优化与 Bitcode 局部修剪"]

    %% Cross-module relationships
    Card12 --> Card13
    Card12 --> Card14
    Card15 --> Card16
    Card24 --> Card20
    Card26 --> Card27
```

---

## 2. LLVM 物理源码位置映射锚点

为便于硬核技术速查，以下是 28 张核心卡片对应在 LLVM 官方源码仓 `llvm/llvm-project` 中的核心源码文件及类/函数位置：

*   **LLVM IR 核心语法与类型 (M1)**:
    *   SSA 设计与 Def-Use/Use-Def 链：`llvm/lib/IR/Value.cpp` -> `llvm::Value::use_iterator` 遍历，以及 `llvm/include/llvm/IR/Use.h`
    *   IR 三态表示：`llvm/lib/IR/Module.cpp`、`llvm/lib/IR/Function.cpp`、`llvm/lib/IR/BasicBlock.cpp`
    *   强类型定义：`llvm/lib/IR/Type.cpp` -> `llvm::Type` 核心衍生类（`PointerType`、`StructType` 等）
    *   基本块终结符规范：`llvm/lib/IR/Instructions.cpp` -> `BranchInst`、`ReturnInst`、`SwitchInst` 的实现
    *   内存分配抽象：`llvm/lib/IR/Instructions.cpp` -> `AllocaInst`、`LoadInst`、`StoreInst`
*   **前端 AST 降级与 IR 生成 (M2)**:
    *   Clang 降级生成入口：`clang/lib/CodeGen/CodeGenModule.cpp` -> `CodeGenModule::EmitTopLevelDecl` 与 `clang/lib/CodeGen/CGExpr.cpp`
    *   控制流分支降级：`clang/lib/CodeGen/CGStmt.cpp` -> `CodeGenFunction::EmitBranch`
    *   异常处理 Landingpad：`clang/lib/CodeGen/CGException.cpp` -> `CodeGenFunction::EmitLandingPad`
    *   调试信息元数据：`llvm/lib/IR/DIBuilder.cpp` -> `DIBuilder` 核心元数据接口
    *   IR 静态验证器：`llvm/lib/IR/Verifier.cpp` -> `llvm::verifyModule()` & `Verifier::visitFunction`
*   **Pass 管理器与经典优化 (M3)**:
    *   PassManager 骨架：`llvm/lib/IR/PassManager.cpp` -> `PassManager::run`
    *   Mem2Reg 提升算法：`llvm/lib/Transforms/Utils/PromoteMemoryToRegister.cpp` -> `llvm::PromoteMemToReg` 与 `llvm/lib/Transforms/Utils/Local.cpp`
    *   LICM 循环不变外提：`llvm/lib/Transforms/Scalar/LICM.cpp` -> `LICM::run` 与 `llvm/lib/Analysis/ScalarEvolution.cpp`
    *   EarlyCSE/GVN 消除：`llvm/lib/Transforms/Scalar/EarlyCSE.cpp` -> `EarlyCSE::run` & `llvm/lib/Transforms/Scalar/GVN.cpp`
    *   SimplifyCFG 与 Inliner：`llvm/lib/Transforms/Scalar/SimplifyCFGPass.cpp` & `llvm/lib/Transforms/IPO/Inliner.cpp`
*   **SelectionDAG 与 ISel 后端 (M4)**:
    *   代码生成流程控制器：`llvm/lib/CodeGen/TargetPassConfig.cpp`
    *   SelectionDAG 选择图构建：`llvm/lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp` & `llvm/lib/CodeGen/SelectionDAG/SelectionDAGBuilder.cpp`
    *   TableGen 指令选择匹配：`llvm/lib/CodeGen/SelectionDAG/SelectionDAGISel.cpp` -> `SelectCode` 逻辑
    *   指令调度 ListScheduler：`llvm/lib/CodeGen/SelectionDAG/ScheduleDAGSDNodes.cpp`
    *   GlobalISel 管道：`llvm/lib/CodeGen/GlobalISel/InstructionSelect.cpp` & `llvm/lib/CodeGen/GlobalISel/Legalizer.cpp`
*   **MachineIR 与寄存器分配 (M5)**:
    *   MachineIR 类表示：`llvm/lib/CodeGen/MachineFunction.cpp`、`llvm/lib/CodeGen/MachineBasicBlock.cpp`、`llvm/lib/CodeGen/MachineInstr.cpp`
    *   贪婪寄存器分配器：`llvm/lib/CodeGen/RegAllocGreedy.cpp` -> `RAGreedy::runOnMachineFunction`
    *   Spill 溢出生成器：`llvm/lib/CodeGen/Spiller.cpp` -> `llvm::createInlineSpiller`
    *   PEI 栈帧重建：`llvm/lib/CodeGen/PrologEpilogInserter.cpp` -> `PEI::runOnMachineFunction`
*   **MC 层、Object 发射与 JIT (M6)**:
    *   MC 流式发射器与指令：`llvm/lib/MC/MCStreamer.cpp` & `llvm/lib/MC/MCInst.cpp`
    *   ELF 二进制文件发射：`llvm/lib/MC/ELFObjectWriter.cpp` -> `ELFObjectWriter::writeHeader`
    *   OrcJIT 异步执行：`llvm/lib/ExecutionEngine/Orc/OrcCore.cpp` -> `AsynchronousSymbolQuery` 与 `llvm/lib/ExecutionEngine/Orc/LLJIT.cpp`
    *   LTO 链接跨模块：`llvm/lib/LTO/LTO.cpp` -> `LTO::run` & `llvm/lib/LTO/ThinLTOCodeGenerator.cpp`
