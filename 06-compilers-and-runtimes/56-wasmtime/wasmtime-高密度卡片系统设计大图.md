# wasmtime-高密度卡片系统设计大图.md

本文件定义了 **wasmtime (WebAssembly 高效 JIT 虚拟机与轻量沙箱运行时)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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
    classDef M6 fill:#755B77,stroke:#534054,color:white;

    Card1["Card 1: Engine & Config"]:::M1
    Card2["Card 2: Store Space"]:::M1
    Card3["Card 3: Module Compile"]:::M1
    Card4["Card 4: Linker Route"]:::M1
    Card5["Card 5: Instance Lifecycle"]:::M1
    Card6["Card 6: Host-to-Guest Trampoline"]:::M1
    
    Card7["Card 7: Cranelift SSA CLIF"]:::M2
    Card8["Card 8: Regalloc2 Allocator"]:::M2
    Card9["Card 9: Translation Loop"]:::M2
    Card10["Card 10: Cranelift Opt"]:::M2
    Card11["Card 11: Dynamic Trampolines"]:::M2
    Card12["Card 12: JIT Serialization Cache"]:::M2

    Card13["Card 13: Linear Memory Mapping"]:::M3
    Card14["Card 14: Guard Pages Protection"]:::M3
    Card15["Card 15: Page Fault Signal Intercept"]:::M3
    Card16["Card 16: memory.grow Dynamics"]:::M3
    Card17["Card 17: Sandbox OOM Hardware Control"]:::M3
    Card18["Card 18: VM Pools Allocation"]:::M3
    Card19["Card 19: Multi-Memory Spaces"]:::M3

    Card20["Card 20: WASI Preview 1 Syscalls"]:::M4
    Card21["Card 21: WASI Preview 2/3 Components"]:::M4
    Card22["Card 22: WIT Canonical ABI Bridge"]:::M4
    Card23["Card 23: WASI Async Streams"]:::M4
    Card24["Card 24: VFS & Net Sandbox Isolation"]:::M4

    Card25["Card 25: Signal Trap Router"]:::M5
    Card26["Card 26: Stack Backtrace Unwinding"]:::M5
    Card27["Card 27: Fuel & Epoch Preemption"]:::M5
    Card28["Card 28: call_indirect Verification"]:::M5

    Card1 --> Card2
    Card1 --> Card3
    Card2 --> Card5
    Card3 --> Card4
    Card4 --> Card5
    Card5 --> Card6
    Card6 --> Card25
    
    Card3 --> Card7
    Card7 --> Card9
    Card9 --> Card8
    Card8 --> Card10
    Card10 --> Card11
    Card11 --> Card12

    Card5 --> Card13
    Card13 --> Card14
    Card14 --> Card15
    Card15 --> Card25
    Card16 --> Card13
    Card17 --> Card13
    Card18 --> Card13
    Card19 --> Card13

    Card5 --> Card20
    Card20 --> Card21
    Card21 --> Card22
    Card21 --> Card23
    Card22 --> Card24

    Card25 --> Card26
    Card27 --> Card25
    Card28 --> Card25
```

---

## 📍 Wasmtime 物理源码位置映射

本设计大图的知识节点与 Wasmtime 核心类库及 Crate 物理源码强关联：
1. **Engine / Store / Instance**: `wasmtime/src/runtime/` 下的 `engine.rs`, `store.rs`, `instance.rs`。
2. **Cranelift Compiler & Regalloc2**: `cranelift/codegen/`（Cranelift 编译器核心）与 `regalloc2/`。
3. **Linear Memory & Signal Handler**: `wasmtime-runtime/src/` 中的 `memory.rs`（线性内存映射）与 `traphandlers/`（Signal 异常陷阱）。
4. **WASI Component Model**: `wasmtime-wasi/` 与 `wasmtime-environ/src/component/`。
