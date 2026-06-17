# Sanitizers 运行时内存与并发检测套件高密卡片系统设计大图

## 1. 28 张卡片依赖拓扑图 (Mermaid)

```mermaid
graph TD
    %% Module 1: Compiler & Runtime Base
    C1[Card 1: 内存不安全痛点] --> C2[Card 2: LLVM Pass 插桩机理]
    C2 --> C3[Card 3: Interceptors 劫持劫断]
    C2 --> C4[Card 4: 栈回溯与符号化]
    C2 --> C5[Card 5: 优化选项与性能折衷]

    %% Module 2: AddressSanitizer (ASan)
    C2 --> C6[Card 6: ASan 影子内存映射]
    C6 --> C7[Card 7: 红色区域 Redzones 毒化]
    C7 --> C8[Card 8: Use-After-Free 与 Quarantine 隔离区]
    C3 --> C9[Card 9: 堆内存分配劫持 asan_allocator]
    C9 --> C8
    C2 --> C10[Card 10: Stack Use-After-Return / Scope]

    %% Module 3: ThreadSanitizer (TSan)
    C11[Card 11: 数据竞争与 Happens-Before] --> C12[Card 12: 向量时钟 Vector Clock]
    C12 --> C13[Card 13: 影子状态 Shadow State 压缩]
    C13 --> C14[Card 14: 互斥锁与屏障状态同步]
    C14 --> C15[Card 15: 锁依赖图死锁检测]

    %% Module 4: MemorySanitizer (MSan)
    C16[Card 16: 未初始化读取威胁] --> C17[Card 17: 比特级影子映射 Shadow Bit-Map]
    C17 --> C18[Card 18: 来源追踪 Origin Tracking]
    C18 --> C19[Card 19: 分支判定与系统调用边界]

    %% Module 5: UBSan & LSan
    C20[Card 20: 内存泄漏成因] --> C21[Card 21: LSan 根集合保守扫描]
    C22[Card 22: UBSan 算术溢出与空指针] --> C23[Card 23: UBSan 边界检查与对齐校验]
    C23 --> C24[Card 24: UBSan vptr 校验与 CFI 完整性]

    %% Module 6: Ops & Diagnostics
    C25[Card 25: 运行时配置与 Suppressions] --> C26[Card 26: 故障报告堆栈分析]
    C26 --> C27[Card 27: 未插桩库引发误报排查]
    C27 --> C28[Card 28: Fuzzing 与 CD 生产集成]

    %% Cross-Module Connections
    C3 --> C21
    C4 --> C26
    C8 --> C26
    C13 --> C27
    C19 --> C27
```

---

## 2. LLVM 与 compiler-rt 物理源码路径映射

### M1: Sanitizers 体系公理与编译器插桩机理
*   **LLVM Pass 插桩框架**: [llvm/lib/Transforms/Instrumentation/](file:///D:/llvm-project/llvm/lib/Transforms/Instrumentation) 
    *   ASan Pass: `AddressSanitizer.cpp`
    *   TSan Pass: `ThreadSanitizer.cpp`
    *   MSan Pass: `MemorySanitizer.cpp`
*   **运行时拦截器 (Interceptors)**: [compiler-rt/lib/sanitizer_common/sanitizer_common_interceptors.inc](file:///D:/llvm-project/compiler-rt/lib/sanitizer_common/sanitizer_common_interceptors.inc)
*   **符号化引擎**: [compiler-rt/lib/sanitizer_common/sanitizer_symbolizer_libcdep.cpp](file:///D:/llvm-project/compiler-rt/lib/sanitizer_common/sanitizer_symbolizer_libcdep.cpp)

### M2: AddressSanitizer (ASan) 影子内存与越界释放检测
*   **影子内存映射公式与状态定义**: [compiler-rt/lib/asan/asan_mapping.h](file:///D:/llvm-project/compiler-rt/lib/asan/asan_mapping.h)
*   **堆内存分配器劫持**: [compiler-rt/lib/asan/asan_allocator.cpp](file:///D:/llvm-project/compiler-rt/lib/asan/asan_allocator.cpp)
*   **内存隔离区 (Quarantine)**: [compiler-rt/lib/sanitizer_common/sanitizer_quarantine.h](file:///D:/llvm-project/compiler-rt/lib/sanitizer_common/sanitizer_quarantine.h)
*   **拦截器实现**: [compiler-rt/lib/asan/asan_interceptors.cpp](file:///D:/llvm-project/compiler-rt/lib/asan/asan_interceptors.cpp)

### M3: ThreadSanitizer (TSan) 状态机、向量时钟与数据竞争
*   **向量时钟更新与时钟逻辑**: [compiler-rt/lib/tsan/rtl/tsan_clock.cpp](file:///D:/llvm-project/compiler-rt/lib/tsan/rtl/tsan_clock.cpp)
*   **互斥锁同步拦截器**: [compiler-rt/lib/tsan/rtl/tsan_mutex.cpp](file:///D:/llvm-project/compiler-rt/lib/tsan/rtl/tsan_mutex.cpp)
*   **影子状态更新与数据竞争碰撞判定**: [compiler-rt/lib/tsan/rtl/tsan_rtl_report.cpp](file:///D:/llvm-project/compiler-rt/lib/tsan/rtl/tsan_rtl_report.cpp)

### M4: MemorySanitizer (MSan) 未初始化内存读取检测
*   **比特影子图映射与 Origin 追踪**: [compiler-rt/lib/msan/msan.cpp](file:///D:/llvm-project/compiler-rt/lib/msan/msan.cpp)
*   **MSan 专用拦截器**: [compiler-rt/lib/msan/msan_interceptors.cpp](file:///D:/llvm-project/compiler-rt/lib/msan/msan_interceptors.cpp)

### M5: UBSan 与 LSan 运行时检测与分析
*   **LSan 保守 GC 扫描与根集合检索**: [compiler-rt/lib/lsan/lsan_common.cpp](file:///D:/llvm-project/compiler-rt/lib/lsan/lsan_common.cpp)
*   **UBSan 算术溢出与对齐检测 Handlers**: [compiler-rt/lib/ubsan/ubsan_handlers.cpp](file:///D:/llvm-project/compiler-rt/lib/ubsan/ubsan_handlers.cpp)
*   **Clang 前端编译期溢出插桩点**: [clang/lib/CodeGen/CGExprScalar.cpp](file:///D:/llvm-project/clang/lib/CodeGen/CGExprScalar.cpp)

### M6: Sanitizers 物理部署、运维排查与故障字典
*   **解析 Suppressions 规则**: [compiler-rt/lib/sanitizer_common/sanitizer_suppressions.cpp](file:///D:/llvm-project/compiler-rt/lib/sanitizer_common/sanitizer_suppressions.cpp)
*   **报告格式化输出**: [compiler-rt/lib/asan/asan_report.cpp](file:///D:/llvm-project/compiler-rt/lib/asan/asan_report.cpp)
