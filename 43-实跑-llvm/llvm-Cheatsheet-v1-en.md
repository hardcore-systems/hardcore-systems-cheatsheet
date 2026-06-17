# LLVM Compiler Infrastructure Core Principles High-Density Cheatsheet (v1)

*   **L0 Essence**: LLVM is a modern compiler infrastructure and toolchain built on a strongly-typed SSA isomorphic IR, decoupling frontend parsing, middle-end pass optimization, and backend SelectionDAG / Machine IR / register allocation.
*   **L1 Four-Sentence Logic**:
    1.  **SSA Isomorphic Design & Triple Form Representation**: LLVM employs IR in Static Single Assignment form, circulating across three isomorphic states (in-memory C++ objects, on-disk bitcode, and human-readable assembly text) to ensure clean optimization decoupling.
    2.  **Pass-Driven Middle-End & Alloca Promotion**: A unified PassManager schedules analysis and transformation passes, where the core Mem2Reg pass uses dominance frontiers to promote stack-based `alloca` memory variables to SSA registers.
    3.  **Legalization, Selection & Machine IR Flattening**: The backend legalizes types and operations on a SelectionDAG, matches patterns to target physical instructions via TableGen rules, and flattens the graph back into linear MachineIR.
    4.  **Register Allocation, Prolog/Epilogue & MC Emission**: Register allocators resolve infinite virtual registers onto target physical registers via interference graph coloring, PEI reconstructs physical frame layouts, and the MC layer emits standard binary object files.
*   **L2 Core Data Flow Topology**:
    *   `Source Code` ➜ `Clang AST Visitor` ➜ `llvm::IRBuilder` ➜ `Raw LLVM IR (Alloca/Load/Store)` ➜ `llvm::verifyModule check` ➜ `PassManager (Mem2Reg)` ➜ `SSA IR (Registers & Phi)` ➜ `LICM/SimplifyCFG passes` ➜ `Optimized IR` ➜ `SelectionDAGISel construction` ➜ `SelectionDAG Graph` ➜ `Type/Operation Legalize` ➜ `ISel Pattern Match (TableGen .td)` ➜ `Target-Specific DAG` ➜ `ListScheduler Scheduling` ➜ `MachineInstr (MI)` ➜ `Live Interval Calculation` ➜ `RegAllocGreedy (Interference Graph Coloring)` ➜ `Spill check? Yes ➜ Insert spill load/store` ➜ `PEI (Prologue/Epilogue)` ➜ `MCStreamer emitter` ➜ `MCInst Stream` ➜ `ELF Object File / OrcJIT Execution`.

---

## 📂 M1: LLVM IR Core Syntax & Type System (Cards 1-5)

#### Card 1. Static Single Assignment (SSA) Design, Phi Nodes, and Def-Use Chains
The soul of LLVM IR is its Static Single Assignment (SSA) form:
1.  **SSA Rule**: Each virtual register must be assigned exactly once. This drastically simplifies dataflow analysis and algorithms (e.g., constant folding, dead code elimination) in the middle-end.
2.  **Phi Nodes**: To represent variable choices at control flow merge points (e.g., after an `if-else` branch), LLVM introduces the `phi` instruction. It selects a value dynamically based on the predecessor basic block the execution path came from.
3.  **Def-Use & Use-Def Chains**: Every `Value` in memory tracks its users (Def-Use chain) and every `User` tracks its arguments (Use-Def chain). This enables the compiler to perform dead code detection and constant propagation in $O(1)$ time.

#### Card 2. Three Isomorphic Representations of LLVM IR
LLVM guarantees that its IR preserves identical expressive power across three physical forms:
1.  **In-Memory Class Structure**: Composed of C++ objects. The top-level container is `llvm::Module` (a compilation unit), which holds `llvm::Function` (functions), `llvm::GlobalVariable` (globals), and `llvm::BasicBlock` (basic blocks). A basic block is a container for a linear sequence of instructions.
2.  **On-Disk Bitcode (.bc)**: A compact binary encoding optimized for fast parsing and disk space efficiency.
3.  **Human-Readable Assembly Text (.ll)**: Human-readable textual representation used for debugging, optimization inspection, and verification.

#### Card 3. Strongly-Typed System & Safety Verification
LLVM features a target-independent, strong type system that prevents invalid memory accesses and implicit type coercions early in the compilation flow:
1.  **Primitive Types**: Includes integers of arbitrary bit-widths (e.g., `i1` for boolean, `i8` for byte, `i32`, `i64`), floating-point types (`float`, `double`), and `void`.
2.  **Derived Types**: Includes arrays (`[N x Type]`), structures (`{ Type1, Type2 }`), pointers (`ptr`), and vector types (`<V x Type>`) designed for SIMD vectorization.
3.  **Safety Verification**: All type conversions must be explicit via casting instructions (e.g., `trunc` for truncation, `zext` for zero extension, `bitcast` for pointer casts); implicit truncation or coercion is strictly prohibited.

#### Card 4. Basic Block CFG Constraints & Terminator Instructions
A basic block (BasicBlock) is the fundamental atomic unit of control flow in LLVM IR:
1.  **Basic Block Rules**: Satisfies the Single-Entry, Single-Exit (SESE) Control Flow Graph (CFG) constraints. No instruction other than the first can be jumped to from outside, and no instruction other than the last can branch out.
2.  **Terminator Instructions**: The final instruction in a basic block must be a terminator. It controls flow direction, such as unconditional branch `br`, conditional branch `br i1`, return `ret`, jump table `switch`, or exception unwind terminators like `resume`/`cleanupret`.
3.  **Entry Block Rule**: The first basic block of a function (Entry) must never have any CFG predecessors to ensure a deterministic entry point.

#### Card 5. Stack Allocation via alloca & Memory Access Abstraction
To simplify frontend compiler construction, LLVM does not force the frontend to emit SSA form directly:
1.  **Memory Abstraction**: Frontends can allocate all local variables on the stack frame using the `alloca` instruction.
2.  **Memory Operations**: Accessing local variables is done via `load` and `store` instructions. Since stack variables reside in memory, they do not follow SSA single-assignment rules and can be written to multiple times.
3.  **Middle-End Promotion**: This creates significant memory access redundancy, which is cleaned up by the middle-end `Mem2Reg` pass, promoting memory accesses into register SSA.

---

## 📂 M2: Frontend AST Lowering & IR Generation (Cards 6-10)

#### Card 6. Clang AST Traversal & IRBuilder Generation
Clang parses source code into an AST and generates IR modules using AST visitors:
1.  **Class Roles**: `clang::CodeGen::CodeGenModule` maintains the global code generation state; `clang::CodeGen::CodeGenFunction` manages local variable scopes, current basic blocks, and the stack frame.
2.  **IRBuilder**: Provides the `llvm::IRBuilder<>` helper. During AST traversal (e.g., visiting `Stmt` and `Expr` nodes), it builds LLVM IR instructions and appends them to the current insertion point.
3.  **Constant Folding**: IRBuilder performs immediate constant analysis when creating instructions. If an expression can be folded (e.g., `1+2` folded to `3`), it returns the constant object directly, preventing useless instruction overhead.

#### Card 7. Control Flow Branch Lowering & Basic Block Splitting
Complex control structures (e.g., `if-else`, `while`/`for` loops) must be lowered into flat basic blocks with jumps:
1.  **If-Else Lowering**:
    *   Split into three new basic blocks: `then` block, `else` block, and a `merge` block.
    *   In the current block, compute the condition (`i1`) and branch via `br i1 %cond, label %then, label %else`.
    *   Force a `br label %merge` at the end of both the `then` and `else` blocks.
2.  **Short-Circuit Evaluation**: Logical operations `&&` and `||` are split into cascades of conditional jumps to achieve native short-circuiting.

#### Card 8. Calling Conventions, Parameter Passing ABI, and Exception landingpad
Function calls deal with target ABI constraints and exception propagation:
1.  **ABI Lowering**: Clang translates arguments according to the target's Calling Convention (e.g., `x86_64-sysv`). Large struct parameters may be passed as pointers (`byval` attribute) or split into multiple integer registers.
2.  **Exception Handling**: `try-catch` blocks are lowered using specialized unwind instructions:
    *   Calls that might throw exceptions must be invoked using `invoke` rather than `call`.
    *   `invoke` specifies the normal return block and an `unwind` destination block.
    *   The `unwind` block must start with a `landingpad` instruction, declaring the types (TypeInfo) it can catch. Uncaught exceptions propagate via `resume`.

#### Card 9. Variable Lifetimes & Dwarf Debug Metadata
Frontends retain metadata for variables and debuggers without hindering middle-end optimizations:
1.  **Lifetimes**: Intrinsics `llvm.lifetime.start` and `llvm.lifetime.end` mark variable scopes. Middle-end optimization passes (like `StackColoring`) reuse stack slots of non-overlapping variables, reducing stack frame sizes.
2.  **DIBuilder (Debug Info Builder)**: Generates metadata attached to IR nodes in Dwarf format. Metadata like `!DILocalVariable` holds source lines and names, mapped via `llvm.dbg.declare` or `llvm.dbg.value` to track variables after SSA transformations.

#### Card 10. The IR Verifier (verifyModule) for Static Correctness
Before handing IR over to middle-end passes, or immediately after a pass completes, the IR verifier must be run:
1.  **Static Verification**: `llvm::verifyModule()` performs static semantic checks across all functions and basic blocks in the Module.
2.  **Verification Criteria**:
    *   **SSA Dominance**: Any `Use` of an SSA variable must be dominated by its `Def` (the execution path must pass through the Def block first).
    *   **Terminator Rule**: Every basic block must end with exactly one `Terminator` instruction; none can appear mid-block.
    *   **Type Consistency**: Verifies matching types (e.g., binary `add` operands must have identical types; adding `i32` directly to `i64` is invalid).

---

## 📂 M3: Middle-End Pass Manager & Classic Optimization Passes (Cards 11-15)

#### Card 11. NewPassManager Scheduling & Analysis/Transform Invalidation
The modern LLVM Pass Manager (NPM) maximizes pipeline efficiency:
1.  **NPM Model**: Separates passes into `AnalysisPass` (e.g., `DominatorTreeAnalysis`, queries IR without modifying it) and `TransformPass` (e.g., dead code elimination, modifies IR).
2.  **Dependency Cache**: NPM maintains an analysis cache. When a TransformPass runs, it returns a `PreservedAnalyses` result:
    *   If instructions are modified but the CFG remains unchanged, `DominatorTreeAnalysis` is preserved.
    *   If the CFG is altered, the cache is invalidated, and subsequent passes will trigger a lazy recalculation of the Dominator Tree.

#### Card 12. The Mem2Reg Algorithm & SSA Renaming
`Mem2Reg` is the gateway pass that promotes stack-allocated `alloca` variables to registers:
1.  **Dominance Frontiers**:
    *   Analyzes variables and computes the dominance frontiers of all basic blocks containing a `store` to that variable.
    *   Dominance frontiers represent merge points. A `phi` node is inserted at the top of these frontier basic blocks, allocating new virtual registers.
2.  **SSA Renaming**:
    *   Traverses the Dominator Tree in a Depth-First Search (DFS), maintaining a stack of active definitions.
    *   Replaces local `load` operations with the register at the top of the definition stack; removes `store` instructions and pushes the new value onto the stack.
3.  **Cleanup**: Eliminates useless `alloca` stack memory frames, turning memory accesses into clean register dataflows.

#### Card 13. Loop Optimizations: LoopSimplify, LICM, and ScalarEvolution
Loops are primary optimization targets, managed by a robust suite of loop passes:
1.  **LoopSimplify**: Reconstructs loop structures to guarantee a single Preheader, a single Latch, and a dedicated Exit block, simplifying the CFG.
2.  **Loop Invariant Code Motion (LICM)**: Hoists instructions whose operands do not change inside the loop body (and have no side effects/are guaranteed to execute on loop exit) into the `Preheader` to prevent redundant execution.
3.  **Scalar Evolution (SCEV)**: Formulates closed-form expressions for induction variables (e.g., loop index `i`). SCEV represents variables as polynomials like $\{Start, +, Step\}_L$ to compute loop trip counts and align vectorization.

#### Card 14. Redundancy Elimination: EarlyCSE, ADCE, and GVN
Eliminating redundant calculations is the most direct way to improve execution speed:
1.  **EarlyCSE (Common Subexpression Elimination)**: Performs a pre-order traversal of the Dominator Tree, maintaining a hash table of previously computed expressions (e.g., `%3 = add i32 %1, %2`). Sub-nodes with identical operands are replaced with `%3`, avoiding recomputation.
2.  **ADCE (Aggressive Dead Code Elimination)**: Assumes all instructions are dead unless they have side effects (e.g., writing to memory, returns, syscalls). Traces backward along Use-Def chains from side-effect roots, preserving only active dependency chains and erasing the rest.
3.  **GVN (Global Value Numbering)**: Assigns a unique global value number to all expressions. Unlike EarlyCSE, it can determine equivalence across control flow branches by analyzing value graphs.

#### Card 15. SimplifyCFG & The Inliner
Simplifying control flow graphs and reducing calling overheads occur frequently in the middle-end:
1.  **SimplifyCFG**:
    *   Merges basic blocks that have a single predecessor/successor.
    *   Deletes unreachable basic blocks.
    *   Promotes branches with a single `phi` (e.g., if-then-else returning constants) to a branchless `select` instruction, leveraging hardware conditional moves.
2.  **Inliner**:
    *   Calculates an Inline Cost based on the size of the target function, call frequency (e.g., loop nesting), and branch hotness.
    *   If the cost falls below a threshold, the callee's IR is copied directly into the caller, eliminating stack parameter pushing, calls, and return overheads while enabling cross-function optimizations.

---

## 📂 M4: Target-Independent CodeGen (SelectionDAG) (Cards 16-20)

#### Card 16. CodeGen Pipeline Stages
The LLVM backend (CodeGen) acts as a multi-stage factory transforming target-independent IR to target-specific assembly:
1.  **Stage 1: SelectionDAGISel**: Builds a target-independent SelectionDAG representation from LLVM IR.
2.  **Stage 2: Type/Operation Legalization**: Lowers types and operations unsupported by physical hardware.
3.  **Stage 3: Instruction Selection (ISel)**: Maps DAG nodes to target physical instructions via pattern matching.
4.  **Stage 4: Instruction Scheduling**: Orders DAG nodes into linear MachineIR sequences.
5.  **Stage 5: Register Allocation (RegAlloc)**: Maps infinite virtual registers to physical registers.
6.  **Stage 6: PEI & MC Emission**: Constructs stack frame prologues/epilogues and writes output assembly or binary objects.

#### Card 17. SelectionDAG Construction & Legalization
SelectionDAG represents data and control dependencies within a basic block as a Directed Acyclic Graph:
1.  **SelectionDAG Structure**: Nodes (`SDNode`) represent operations, edges (`SDValue`) represent data flow. A special "Chain Edge" enforces ordering on side-effect instructions (like Load/Store) to prevent invalid reordering.
2.  **Type Legalization**:
    *   **Promote**: Zero-extends unsupported narrow types (e.g., `i8` to `i32` if the hardware only supports 32-bit registers).
    *   **Split**: Divides wide types into supported widths (e.g., splitting `i64` into two `i32` nodes on 32-bit targets).
3.  **Operation Legalization**: Replaces unsupported operations with runtime library calls (e.g., replacing `sdiv` with a call to `__divdi3` if the target CPU lacks hardware division).

#### Card 18. TableGen-Driven Instruction Selection (ISel)
Instruction Selection maps target-independent SelectionDAG nodes to actual machine instructions:
1.  **TableGen**: Developers define declarative pattern matches in target description (`.td`) files (e.g., defining an x86 `ADD32rr` matching DAG `(add i32:$src1, i32:$src2)`).
2.  **Build-time Compilation**: The TableGen compiler (`llvm-tblgen`) parses `.td` files into a compact byte-mapped pattern matcher table.
3.  **Runtime Matching**: The ISel engine runs a state machine matching DAG nodes against the table, replacing generic `SDNode` instances with physical instruction nodes.

#### Card 19. Instruction Scheduling & MachineInstr Flattening
SelectionDAGs must be linearized before register allocation and emission:
1.  **Constraint**: SelectionDAGs are graph structures, whereas physical execution pipelines are linear.
2.  **Scheduling**: Schedulers (e.g., `ScheduleDAGMILive`) use heuristics (considering pipeline latencies, register pressure, etc.) to traverse the DAG, producing an ordered sequence that minimizes CPU stalls and limits register lifetimes.
3.  **MachineInstr (MI)**: The scheduler deconstructs the DAG back into a linear doubly-linked list of `MachineInstr` objects, reintroducing virtual registers.

#### Card 20. GlobalISel: Fast Pipeline via Generic Machine IR (gMIR)
To resolve the high memory overhead and execution speed bottlenecks of SelectionDAG's multiple legalization phases, LLVM introduced GlobalISel:
1.  **gMIR (Generic MIR)**: Bypasses SelectionDAG entirely, translating IR directly to generic machine instructions (e.g., `%vreg0 = G_ADD i32 %vreg1, %vreg2`), preserving a linear MIR representation throughout.
2.  **Global Passes**: Legality, RegBankSelect, and InstructionSelect are decoupled into standard `MachineFunctionPass` classes. They analyze code across basic blocks, offering superior global optimization opportunities.
3.  **Efficiency**: Minimizes memory allocation overhead, yielding compilation throughput increases of several fold, ideal for JIT engines and debug builds.

---

## 📂 M5: Target-Specific Machine IR & Register Allocation (Cards 21-24)

#### Card 21. MachineIR (MIR) Structure, Virtual/Physical Registers
MachineIR is the final representation container, matching physical assembly syntax while maintaining virtual registers before register allocation:
1.  **Hierarchy**: Outer wrapper is `MachineFunction`, which contains `MachineBasicBlock` (MBB) and `MachineInstr` (MI).
2.  **MachineOperand**: Operands inside instructions can be virtual registers (`%stack_val`), physical registers (`$rax`), immediates, or memory symbols.
3.  **RegisterClass**: Virtual registers must bind to a register class (e.g., x86 `GR32` for 32-bit general-purpose registers), which defines the pool of physical registers available for allocation.

#### Card 22. The RegAllocGreedy Coloring & Spilling Algorithm
LLVM employs a greedy register allocator (RegAllocGreedy) to map virtual registers to physical registers:
1.  **Live Intervals**: Computes the instruction range (definition to last use) for which each virtual register remains active.
2.  **Interference Graph**: Builds an interference graph where nodes are intervals and edges connect overlapping intervals (meaning they cannot share a physical register).
3.  **Coloring & Allocation**:
    *   Sorts virtual registers by priority (shorter intervals and loop-nest variables have higher priority) and assigns free physical registers.
    *   If physical registers are exhausted, a "Spill" is triggered.
4.  **Spill Insertion**: Chooses the lowest-cost spill candidate, allocates a stack slot, and inserts a `store` after its definition and a `load` before its uses, splitting the live interval for reallocation.

#### Card 23. PEI Frame Layout Reconstruction
After register allocation, all virtual registers are replaced by physical ones, but the stack frame layout is not yet complete:
1.  **PrologEpilogInserter (PEI)**:
    *   Computes the final size of the stack frame (aggregating spill slots and local `alloca` arrays).
    *   Determines stack alignment (e.g., x86_64 mandates 16-byte aligned stack pointers).
2.  **Physical Insertion**:
    *   Inserts Prologue assembly (e.g., adjusting SP `subq $32, %rsp` and saving FP) at the function entry basic block.
    *   Inserts Epilogue assembly (restoring SP and callee-saved registers) before all function return `ret` blocks.
    *   **Callee-Saved Registers**: The ABI specifies registers that must be preserved (e.g., `%rbx`, `%r12`). If modified, PEI pushes them in the Prologue and pops them in the Epilogue.

#### Card 24. TableGen .td Target Descriptions
TableGen is the core mechanism enabling LLVM to easily extend to new hardware architectures:
1.  **TableGen Language**: A strongly-typed, record-oriented description language (.td).
2.  **Description Focus**:
    *   **Register Sets**: Declares physical registers, names, aliases, and overlapping sub-registers (e.g., `EAX` as the low 32 bits of `RAX`).
    *   **Instruction Sets**: Declares instruction mnemonics (`add`), binary instruction encoding formats, operand input/output types, latencies, and implicit register side effects.
    *   **Calling Conventions**: Defines register priority for argument passing (e.g., `%rdi`, `%rsi`, `%rdx` on x86_64) and stack offset rules.
3.  **CodeGen Auto-generation**: During build time, `llvm-tblgen` compiles `.td` files into C++ pattern matching tables and type-helper code loaded into CodeGen.

---

## 📂 M6: MC Layer, Object Emission & JIT Engine (Cards 25-28)

#### Card 25. MC Layer: MCStreamer & MCInst Instructions
The MC (Machine Code) layer acts as the final assembler/disassembler engine:
1.  **MCInst**: Unlike MachineInstr, MCInst is stripped of compiler-specific control flow and virtual registers, holding only physical register IDs, immediates, and opcode bytes.
2.  **MCStreamer**: Provides a uniform streaming interface. It selects `MCAsmStreamer` to write text assembly, or `MCObjectStreamer` to forward the stream to the binary encoder.
3.  **Assembly & Disassembly**: The encoder serializes instructions to machine code; the disassembler reads binary arrays, builds an `MCInst` tree via table-mapped decoders, and outputs text.

#### Card 26. Object File Emission: ELF/Mach-O Sections and Relocation Tables
The MC ObjectWriter packages instruction streams into system-executable object files:
1.  **Section Generation**: Converts code into the `.text` binary section, read-only constants into `.rodata`, and global variables into `.data`.
2.  **Symbol Tables**: Extracts function and global variable symbols into a `.symtab` section, recording byte offsets within their parent sections.
3.  **Relocations**:
    *   Calls to external functions (e.g., `printf`) or accesses to global variables have unknown absolute addresses during single-module compilation.
    *   The compiler fills these sites with placeholders and writes relocation records into `.rela.text` instructing the linker how to patch the addresses (e.g., via PC-relative `R_X86_64_PC32` relocations).

#### Card 27. OrcJIT Engine & JITLink Linker
LLVM's JIT (Just-In-Time) engine compiles IR dynamically into memory for immediate execution:
1.  **OrcJIT Architecture**: Employs a layered architecture. The high-level `LLJIT` interface sits atop an `IRCompileLayer` (compiles IR to objects) and an `ObjectLinkingLayer` (manages memory allocation).
2.  **On-Demand Compilation**: OrcJIT registers modules lazily. Actual compilation and machine code generation are deferred to background worker threads until a function's symbol is first resolved.
3.  **JITLink**: A modern linker replacing the legacy `RuntimeDyld`. It parses object files in memory and replicates operating system linking: allocating `r-x` memory for code and `r--` for data, executing relocations directly into active memory buffers, and transferring control.

#### Card 28. Link-Time Optimization (LTO) & ThinLTO
LLVM pioneered Link-Time Optimization (LTO) at scale:
1.  **Limitation**: Traditional compilers optimize single modules, leaving linkers to copy symbols without cross-module inlining.
2.  **Monolithic LTO**: Compiles source files to LLVM IR bitcode instead of assembly. During linking, the LTO driver reads all bitcode into a single massive memory module, executing global optimizations, though at high memory and time costs.
3.  **ThinLTO**:
    *   **Phase 1**: Emits thin bitcode containing local IR alongside a lightweight summary table (storing function size, import/export relations).
    *   **Phase 2**: During linking, the ThinLTO driver loads summaries globally. If module A needs to inline a function in B, it imports only that function's IR, allowing optimization passes to run fully in parallel across cores with low memory overhead.

---

## 📂 LLVM Optimizations & CodeGen Trade-off Matrix

| Dimension | Option A | Option B | Trade-off / Compromise |
| :--- | :--- | :--- | :--- |
| **Local Variable Representation** | Local Stack访存 (Alloca + Load/Store) | Pure SSA Register Model | Option A simplifies frontends by avoiding variable-to-register mapping and branch phi placement, though it creates heavy load/store redundancies. Option B requires complex frontend SSA calculation, though it simplifies middle-end passes. [LLVM generates Alloca in frontends, then immediately promotes them to SSA via Mem2Reg in the middle-end to combine the best of both worlds] |
| **ISel CodeGen Pipeline** | SelectionDAG (DAG-based ISel) | GlobalISel (gMIR-based ISel) | SelectionDAG uses basic-block graph structures, legalizing types and operations with high precision, but consumes heavy memory and compilation time. GlobalISel uses generic machine instructions linearly, executing global passes across basic blocks with small footprints. [SelectionDAG is chosen for static release builds; GlobalISel is used for fast JIT and debug compiles] |
| **Register Allocation** | Graph Coloring (Kempe's Algorithm) | Greedy Allocation (RegAllocGreedy) | Graph coloring views register mapping as an NP-complete coloring problem, providing optimal allocation for small functions but running slowly on large structures. Greedy allocation splits intervals and computes spilling cost with fast linear heuristics. [LLVM defaults to Greedy allocation, optimizing compilation speed for large C++ codebases] |
| **Cross-Module Optimization** | Monolithic LTO | ThinLTO (Summary-based LTO) | Monolithic LTO aggregates all bitcode into a single module, executing comprehensive inlining and dead code deletion at the expense of high link-time memory and single-threaded execution. ThinLTO runs parallel codegen by only importing relevant function IR based on summary tables. [ThinLTO compromises extreme dead code deletion for multifold improvements in link time and memory consumption] |

---

## 🔬 Zone T: LLVM Parameters & IR Patterns Dictionary

### T1: Core LLVM/Clang CLI Commands & Debug Flags
*   `clang++ -O3 -mllvm -print-after-all main.cpp` : Prints the complete IR text after every single middle-end pass, allowing developers to isolate which pass introduced an optimization bug.
*   `clang++ -Xclang -ast-dump -fsyntax-only main.cpp` : Outputs Clang's Abstract Syntax Tree (AST) to standard output, verifying parser correctness before lowering to LLVM IR.
*   `opt -passes='mem2reg,simplifycfg' -S input.ll -o output.ll` : Runs the new Pass Manager (NPM) stand-alone, applying the specified pass pipeline (promoting alloca to SSA and simplifying control flow) to output textual `.ll` IR.
*   `llc -O2 -march=x86-64 -mcpu=skylake -print-after-isel input.ll` : Compiles LLVM IR to x86 assembly, printing out target machine instructions immediately after Instruction Selection (ISel) is resolved.

### T2: LLVM IR Assembly Features & SSA Patterns
*   **Stack Allocation vs. SSA Form Promotion**:
    Unoptimized IR stores local variables (e.g., `int a = 5;`) on the stack:
    ```llvm
    %a = alloca i32, align 4              ; Allocates 4 bytes on stack frame
    store i32 5, ptr %a, align 4          ; Stores 5 into local stack memory
    %1 = load i32, ptr %a, align 4         ; Loads stack memory back to virtual register %1
    %add = add nsw i32 %1, 10             ; Performs addition
    ```
    After running `Mem2Reg` optimization, stack accesses are removed:
    ```llvm
    ; alloca, store, load instructions are completely erased
    %add = add nsw i32 5, 10               ; Constant propagated directly to SSA instruction
    ```
*   **Phi Node CFG Convergence**:
    When code paths merge, the `phi` instruction resolves multiple definition inputs:
    ```llvm
    then:
      %val_then = add i32 %x, 1
      br label %merge

    else:
      %val_else = sub i32 %x, 1
      br label %merge

    merge:
      ; phi syntax: selects register based on the actual predecessor block executed
      %val = phi i32 [ %val_then, %then ], [ %val_else, %else ]
      ret i32 %val
    ```
