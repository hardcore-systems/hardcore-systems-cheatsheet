# bpftrace-高密度卡片系统设计大图.md

本文件定义了 **bpftrace (性能调优与动态追踪语言)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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

    Card1["Card 1: Lexer & Parser"]:::M1
    Card2["Card 2: Semantic Checking"]:::M1
    Card3["Card 3: AST to LLVM IR"]:::M1
    Card4["Card 4: Variables Scopes"]:::M1
    Card5["Card 5: Builtins & Aggregators"]:::M1
    Card6["Card 6: BTF Type Deduction"]:::M1
    
    Card7["Card 7: LLVM JIT Engine"]:::M2
    Card8["Card 8: BPF Maps Allocation"]:::M2
    Card9["Card 9: Probes Attachment"]:::M2
    Card10["Card 10: Ring Buffer Communication"]:::M2
    Card11["Card 11: Multi-probe Coord"]:::M2
    Card12["Card 12: Verifier Safety"]:::M2

    Card13["Card 13: Maps Printing"]:::M3
    Card14["Card 14: Session Lifecycle"]:::M3
    Card15["Card 15: Helper Calls Mapping"]:::M3
    Card16["Card 16: Stack Backtraces"]:::M3
    Card17["Card 17: Tracepoint Parsing"]:::M3
    Card18["Card 18: USDT Semaphore"]:::M3
    Card19["Card 19: Filters Generation"]:::M3

    Card20["Card 20: Safe Mode Constraints"]:::M4
    Card21["Card 21: Resource Boundaries"]:::M4
    Card22["Card 22: Kernel Adaptation"]:::M4
    Card23["Card 23: Kernel Stack Limit"]:::M4
    Card24["Card 24: Signal Handling"]:::M4

    Card25["Card 25: CLI Front"]:::M5
    Card26["Card 26: BTF Cache"]:::M5
    Card27["Card 27: User-state State Tracking"]:::M5
    Card28["Card 28: Multitenant Isolation"]:::M5

    Card1 --> Card2
    Card2 --> Card3
    Card3 --> Card4
    Card3 --> Card5
    Card2 --> Card6
    
    Card3 --> Card7
    Card7 --> Card8
    Card7 --> Card9
    Card9 --> Card10
    Card9 --> Card11
    Card11 --> Card12

    Card10 --> Card13
    Card13 --> Card14
    Card9 --> Card15
    Card15 --> Card16
    Card9 --> Card17
    Card9 --> Card18
    Card5 --> Card19

    Card20 --> Card3
    Card21 --> Card12
    Card22 --> Card7
    Card23 --> Card12
    Card24 --> Card14

    Card25 --> Card1
    Card26 --> Card6
    Card27 --> Card4
    Card28 --> Card25
```

---

## 📍 bpftrace 物理源码位置映射

本设计大图的知识节点与 bpftrace 核心类库及 Crate 物理源码强关联：
1. **Frontend & AST**: `src/parser.yy`, `src/lexer.l`, `src/ast/`。
2. **Semantic Analysis**: `src/semantic_analyser.cpp`, `src/ast/semantic_analyser_visit.cpp`。
3. **LLVM Codegen & JIT**: `src/codegen_llvm.cpp`, `src/ast/codegen_llvm_visit.cpp`。
4. **Probe Attachment**: `src/attached_probe.cpp`, `src/probe_metadata.cpp`。
5. **BTF & Symbols**: `src/btf.cpp`, `src/symbols.cpp`。
