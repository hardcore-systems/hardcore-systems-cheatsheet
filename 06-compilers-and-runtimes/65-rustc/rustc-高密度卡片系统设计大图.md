# rustc-高密度卡片系统设计大图.md

本文件定义了 **rustc (Rust 官方编译器内核、HIR/MIR 变换与 NLL 借用检查器)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

---

## 🗺️ 28 张卡片依赖拓扑图 (Mermaid)

```mermaid
graph TD
    classDef default fill:#151d30,stroke:#24324f,color:#e2e8f0;
    classDef M1 fill:#4B5F7A,stroke:#2f3d52,color:white;
    classDef M2 fill:#6B8272,stroke:#4b5c50,color:white;
    classDef M3 fill:#9C6666,stroke:#704949,color:white;
    classDef M4 fill:#7A7A7A,stroke:#595959,color:white;
    classDef M5 fill:#9A825A,stroke:#6e5c40,color:white;

    Card1["Card 1: Tokenizer & Lexer"]:::M1
    Card2["Card 2: Macro & Resolution"]:::M1
    Card3["Card 3: AST to HIR Lowering"]:::M1
    Card4["Card 4: HM Type Inference"]:::M1
    Card5["Card 5: Trait Coherence"]:::M1
    Card6["Card 6: Method Resolution"]:::M1
    
    Card7["Card 7: HIR to MIR Lowering"]:::M2
    Card8["Card 8: MIR Dataflow Analysis"]:::M2
    Card9["Card 9: NLL Region Inference"]:::M2
    Card10["Card 10: Borrow Checker Validation"]:::M2
    Card11["Card 11: Drop Elaboration"]:::M2
    Card12["Card 12: MIR Optimizations"]:::M2

    Card13["Card 13: Monomorphization"]:::M3
    Card14["Card 14: Dynamic VTables"]:::M3
    Card15["Card 15: Memory Layout"]:::M3
    Card16["Card 16: LLVM IR Translation"]:::M3
    Card17["Card 17: Linker & LTO"]:::M3
    Card18["Card 18: Incremental Query"]:::M3
    Card19["Card 19: Unwinding Panic"]:::M3

    Card20["Card 20: Safe/Unsafe Boundaries"]:::M4
    Card21["Card 21: Miri Interpreter"]:::M4
    Card22["Card 22: CTFE Const Eval"]:::M4
    Card23["Card 23: Send/Sync Concurrency"]:::M4
    Card24["Card 24: Macro Hygiene"]:::M4

    Card25["Card 25: Diagnostics Engine"]:::M5
    Card26["Card 26: Query Cache"]:::M5
    Card27["Card 27: Feature Gates"]:::M5
    Card28["Card 28: Metadata Serialization"]:::M5

    Card1 --> Card2
    Card2 --> Card3
    Card3 --> Card4
    Card4 --> Card5
    Card4 --> Card6
    
    Card3 --> Card7
    Card7 --> Card8
    Card8 --> Card9
    Card9 --> Card10
    Card10 --> Card11
    Card11 --> Card12

    Card12 --> Card13
    Card12 --> Card14
    Card13 --> Card15
    Card14 --> Card15
    Card15 --> Card16
    Card16 --> Card17
    Card18 --> Card26
    Card19 --> Card17

    Card10 --> Card20
    Card20 --> Card21
    Card21 --> Card22
    Card15 --> Card23
    Card2 --> Card24

    Card26 --> Card18
    Card27 --> Card3
    Card28 --> Card2
```

---

## 📍 rustc 物理源码位置映射

本设计大图的知识节点与 rustc 核心类库及 Crate 物理源码强关联：
1. **Frontend / Resolution**: `compiler/rustc_lexer/`, `compiler/rustc_parse/`, `compiler/rustc_resolve/`。
2. **Type Checking & Traits**: `compiler/rustc_hir/`, `compiler/rustc_typeck/`, `compiler/rustc_trait_selection/`。
3. **MIR & Borrow Check (NLL)**: `compiler/rustc_mir_build/`, `compiler/rustc_mir_dataflow/`, `compiler/rustc_borrowck/`。
4. **Codegen & LLVM**: `compiler/rustc_codegen_ssa/`, `compiler/rustc_codegen_llvm/`。
5. **CTFE & Miri**: `compiler/rustc_const_eval/`。
