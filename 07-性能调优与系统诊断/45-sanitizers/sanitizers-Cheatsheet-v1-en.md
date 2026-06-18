# Sanitizers Runtime Memory & Concurrency Detection Suite Core Principles Cheatsheet (v1)

*   **L0 Essence**: The essence of Sanitizers is a dynamic program analysis suite integrating compiler instrumentation and runtime libraries. By using shadow memory mapping to track memory states, vector clocks to maintain happens-before partial orders, and bit-to-bit shadow mapping to trace initialization, it detects memory out-of-bounds/UAF, data races, uninitialized reads, and undefined behavior at runtime with lower overhead than traditional interpreters.
*   **L1 Four-Sentence Logic**:
    1.  **Instrumentation & Interception**: The compiler LLVM Pass inserts checks at the IR level, and runtime interceptors hijack key libc syscalls, collaborating with llvm-symbolizer to achieve high-precision symbolization and stack unwinding.
    2.  **Shadow Poisoning & Quarantine**: ASan establishes an 8:1 shadow memory mapping, intercepts out-of-bounds access by poisoning redzones around variables, and detects use-after-free (UAF) by delaying physical block reclamation in a quarantine queue.
    3.  **Vector Clocks & Race Detection**: TSan tracks happens-before relations via vector clocks, matching each 8-byte application memory to 4 shadow cells (recording thread, epoch, range, write/read) to achieve lock-free concurrency race detection.
    4.  **Bit Tracing & Conservative Sweep**: MSan uses a 1:1 bit-to-bit shadow map to verify initialization states, coupled with origin tracking to trace allocations; LSan scans the root set in a conservative GC manner on exit to detect unreachable memory and report leaks.
*   **L2 Core Data Flow Topology**:
    *   `Compile C/C++ Code` ➜ `LLVM IR` ➜ `ASan/TSan/MSan Pass` ➜ `Insert Instrumentation Checks` ➜ `Link Runtime Library (libclang_rt)` ➜ `Application Startup` ➜ `Interceptors Hijack malloc()/free()` ➜ `Heap Allocation` ➜ `Poison Redzones (Shadow Memory set to 0xf1-0xf8)` ➜ `Application Write/Read` ➜ `Compile-Time Instrumented Check: (Addr >> 3) + Offset` ➜ `Shadow Byte Value != 0?` ➜ `Crash Report (ASan Global/Stack OOB)` ➜ `Free Object` ➜ `Move Page/Block to Quarantine (Poison Shadow Memory to 0xfd)` ➜ `Application Use-After-Free Access` ➜ `Shadow Byte == 0xfd Check` ➜ `Symbolic Stack Unwinding` ➜ `Report ASan UAF` ➜ `Thread A Write` ➜ `Record Vector Clock & Epoch` ➜ `Update TSan Shadow Cell (Thread, Epoch, Lock)` ➜ `Thread B Read (No happens-before link)` ➜ `Compare current Epoch with Vector Clock ➜ Data Race detected!` ➜ `Deadlock Check: Lock dependency graph circular loop` ➜ `UBSan runtime check: Integer overflow check (llvm.sadd.with.overflow)` ➜ `LSan Scan: Sweep stack/registers for pointers to allocated blocks ➜ Report memory leak`.

---

## 📂 M1: Sanitizers Architecture Axioms & Compiler Instrumentation (Cards 1-5)

#### Card 1. Memory and Concurrency Safety Pain Points
C/C++ programs are memory-unsafe by design, leading to critical safety issues:
1.  **Static Analysis Limits**: Static vulnerability scanners (e.g., Clang Static Analyzer) struggle with pointer aliasing analysis and suffer from path explosion, yielding high false-positive and false-negative rates.
2.  **Blackbox and Interpreter Overhead**: Traditional dynamic tools like Valgrind interpret instructions at runtime, incurring 10x-50x CPU slowdown and massive memory footprints, which prohibits their use in performance or concurrency stress testing.
3.  **Dynamic Detection Need**: Low-overhead compilation-phase instrumentation combined with high-precision runtime interception of out-of-bounds, use-after-free (UAF), and concurrent data races forms the core foundation of modern robust systems programming.

#### Card 2. Compiler-Level Instrumentation Mechanism
The core architecture of LLVM Sanitizers is divided into compilation-time transformation (LLVM Pass) and runtime library support (compiler-rt):
1.  **Compilation Instrumentation**: After Clang parses source code into LLVM Intermediate Representation (IR), Sanitizers Pass traverses the IR's Basic Blocks. For every memory load/store instruction, the Pass automatically inserts check logic beforehand.
2.  **Runtime Binding**: The instrumented code links against the compiler-rt static or dynamic library. The runtime manages memory allocations, maintains shadow memory mapping, and initializes system call interceptors.
3.  **Cooperative Running**: The compiler-time pass defines "where to check" while the runtime library determines "how to record and intercept," minimizing dynamic analysis overhead.

#### Card 3. System Function Interception and Wrapping
Dynamic verification requires tracking the application's interactions with standard library (libc) functions:
1.  **Hijacking (LD_PRELOAD)**: On Linux, the runtime library uses dynamic linker loading priorities or the `LD_PRELOAD` environment variable to ensure its custom wrapper functions are resolved first.
2.  **Weak Symbols Override**: For static linking, the runtime library overrides the weak symbol definitions of libc standard functions with its own strong symbol implementations.
3.  **Wrappers (Interceptors)**: Interceptors for functions like `memcpy`, `strcpy`, and `pthread_create` inspect the shadow memory status of target buffer ranges before forwarding calls to the original libc implementations via `dlsym(RTLD_NEXT, ...)`.

#### Card 4. Runtime Symbolization and Stack Unwinding
When a Sanitizer catches a safety violation, it must reconstruct and print a human-readable stack trace:
1.  **Stack Unwinding**: The runtime library captures the instruction pointer register (PC) at the error point. To extract call frames, it uses a fast unwinder (relying on the RBP frame pointer chain, requiring `-fno-omit-frame-pointer`) or a slow unwinder (parsing DWARF `.eh_frame`).
2.  **Symbolizer Engine**: The runtime library communicates via a pipe with a background daemon process, `llvm-symbolizer`. It sends the captured PC addresses to the engine, which queries ELF DWARF debug symbols.
3.  **Formatted Report**: The engine translates PCs to source filenames, line numbers, and function names, outputting a structured stack trace.

#### Card 5. Performance and Optimization Trade-off
To balance detection accuracy and execution speed, compilation instrumentation must be paired with appropriate optimizer settings:
1.  **Overhead Sources**: Shadow address translation, state loading, and branching logic inserted before each load/store instruction disrupt CPU out-of-order execution, stalling pipelines.
2.  **Optimization Options**:
    *   `-O0`: No optimization. The Pass instrument all memory reads and writes indiscriminately, causing severe performance degradation.
    *   `-O1` / `-O2`: The compiler applies Dead Code Elimination (DCE) and redundant check merging (e.g., check once for multiple accesses to the same pointer within a loop), reducing CPU overhead to ~2x.
3.  **Debug Trade-off**: High optimization levels may inline functions, leading to omitted call frames or minor line offsets in the printed error stack.

---

## 📂 M2: AddressSanitizer (ASan) Shadow Memory & Memory Corruption (Cards 6-10)

#### Card 6. Shadow Memory Mapping Algorithm
AddressSanitizer maps the status of the application memory space onto a dedicated Shadow Memory region:
1.  **Eight-to-One Linear Mapping**: Every 8 bytes of application memory are mapped to 1 byte of shadow memory, yielding an 8:1 mapping ratio.
2.  **Mapping Formula**:
    $$\text{ShadowAddr} = (\text{AppAddr} \gg 3) + \text{Offset}$$
    For Linux x86_64, the Offset constant is `0x7fff8000`.
3.  **Shadow Byte Encoding**:
    *   `0x00`: All 8 application bytes are clean and addressable.
    *   `0x01` - `0x07`: The first $k$ bytes are addressable, while the remaining $8-k$ bytes are poisoned (forbidden).
    *   Negative / Special values (e.g., `0xf1` - `0xf8`): Redzones separating allocated regions.
    *   `0xfd`: Freed memory blocks currently in the heap quarantine.

#### Card 7. Redzones and Compiler-Time Instrumentation
To detect buffer overflows, ASan structures unaddressable buffer boundaries known as Redzones:
1.  **Variable Padding**: The LLVM Pass modifies the memory layout of stack and global variables during compilation. It inserts unaddressable padding (Redzones) and poisons their corresponding shadow bytes (e.g., stack left redzone is `0xf1`, middle is `0xf2`, right is `0xf3`).
2.  **Out-of-Bounds Assertion**: For any memory load/store instruction:
    ```c
    char *p = malloc(10);
    // Inserted by compiler:
    char *shadow_addr = (p >> 3) + 0x7fff8000;
    if (*shadow_addr != 0) {
        ReportError(p);
    }
    ```
    If an index calculation shifts the pointer into a Redzone, the shadow check hits a non-zero poisoned value, triggering an immediate crash report.

#### Card 8. Use-After-Free and Quarantine Protection
Use-After-Free (UAF) occurs when dangling pointers write back to reclaimed heap blocks:
1.  **Deallocation Hijacking**: The runtime wrapper intercepts `free(ptr)`. Instead of returning the memory to the OS, it poisons the corresponding shadow bytes to `0xfd` (freed heap).
2.  **Quarantine Queue**: The freed block is pushed into a First-In-First-Out (FIFO) quarantine queue (e.g., limited to 256MB). While in quarantine, the block cannot be recycled or reallocated.
3.  **Violation Interception**: If a dangling pointer accesses the quarantined address, ASan identifies the `0xfd` shadow byte, asserts a UAF violation, and halts. Older blocks are evicted from quarantine and reclaimed when the size limit is exceeded.

#### Card 9. Heap Allocation Hijacking
ASan uses a custom allocator, `asan_allocator`, to manage physical heap memory:
1.  **Reconstructed Allocation**: When `malloc(size)` is called, the allocator reserves a contiguous memory region of size `Header_Size + Left_Redzone + size + Right_Redzone`, aligned to 16-byte boundaries.
2.  **Header Metadata**: The block header (poisoned) stores vital allocation metadata, including the allocation thread ID and a hash of the allocation stack trace.
3.  **Boundary Isolation**: The allocator poisons the Left and Right Redzones (e.g., `0xfa` / `0xfb`). This layout isolates buffer boundaries, ensuring even large out-of-bounds writes are blocked before corrupting neighboring allocations.

#### Card 10. Stack Use-After-Return (UAR) and Use-After-Scope (UAS)
Stack variables are deallocated when their declaring function or block exits, but escaping pointers can cause bugs:
1.  **UAR Detection**: With `ASAN_OPTIONS=detect_stack_use_after_return=1` enabled, the LLVM Pass alters stack frames. Instead of physical stack allocation, stack variables are allocated dynamically on a heap-backed "Fake Stack." Upon return, these fake frames are poisoned to `0xf5`. Subsequent accesses by escaping pointers are intercepted.
2.  **UAS Detection**: When execution exits a block scope (e.g., `{ char tmp[10]; ... }`), the compiler inserts a call to `__asan_poison_stack_memory`, poisoning the corresponding shadow bytes to `0xf8`. Accessing `tmp` after the scope block triggers an error.

---

## 📂 M3: ThreadSanitizer (TSan) Vector Clocks & Data Races (Cards 11-15)

#### Card 11. Data Races and the Happens-Before Relation
A data race is a dangerous concurrent access pattern occurring when multiple threads read and write to the same memory without synchronization:
1.  **Race Definition**: Two or more threads concurrently access the same memory address, at least one access is a write, and no synchronization locks establish a relative execution order.
2.  **Happens-Before Relation ($\rightarrow$)**: A partial order relation defining temporal dependency. If event $A \rightarrow B$, the results of $A$ are guaranteed visible to $B$.
3.  **Race-Free Condition**: For any concurrent accesses $E_1$ and $E_2$ to the same memory, they must satisfy $E_1 \rightarrow E_2$ or $E_2 \rightarrow E_1$; otherwise, TSan flags a data race.

#### Card 12. Vector Clocks State Tracking
TSan tracks happens-before relations across threads using Vector Clocks:
1.  **Vector Definition**: Each thread $i$ maintains an integer array $VC_i$ of size $N$ (total threads). $VC_i[j]$ denotes the latest logical time (epoch) of thread $j$ known to thread $i$.
2.  **Local Tick**: On performing local reads/writes, thread $i$ increments its own component:
    $$VC_i[i] = VC_i[i] + 1$$
3.  **Synchronization Merge**: When thread $i$ releases mutex $L$, the lock copies the clock: $VC_L = VC_i$. When thread $j$ subsequently acquires $L$, it merges the clocks:
    $$VC_j = \max(VC_j, VC_L)$$
    This transfers the happens-before dependency across threads.

#### Card 13. Shadow State Compression
TSan records memory access histories using compressed shadow memory representations:
1.  **Four Shadow Cells**: Every 8 bytes of application memory map to 4 shadow cells (each cell is 8 bytes, totaling 32 bytes, for a 4:1 mapping ratio).
2.  **Access Ring Buffer**: These 4 cells act as a circular buffer, tracking the 4 most recent read or write events on that 8-byte application memory chunk.
3.  **64-Bit Compressed Encoding**: Each cell stores:
    *   `Thread ID` (13 bits): The thread performing the access.
    *   `Epoch` (16 bits): The logical epoch of the thread during access.
    *   `Access Range` (8 bits): The relative byte offset and size within the 8-byte block.
    *   `IsWrite` (1 bit): Flag indicating a read or write operation.

#### Card 14. Lock Synchronization and State Transition
Interceptors are the bridge that transfers vector clock state during synchronization:
1.  **Synchronization Hijacking**: TSan intercepts `pthread_mutex_lock/unlock`, `pthread_cond_wait`, and atomic operations.
2.  **Clock Bridge**:
    *   On `pthread_mutex_unlock`, the thread's vector clock $VC_{\text{thread}}$ is copied to the lock's shadow state $VC_{\text{lock}}$, and the thread's epoch is incremented.
    *   On `pthread_mutex_lock`, the thread's $VC$ is merged with $VC_{\text{lock}}$.
3.  **Conflict Checking**: For each memory access, TSan compares the current epoch against those stored in the 4 historical shadow cells. If the historical epoch is not less than the current thread's vector clock component, a concurrent race is detected and reported.

#### Card 15. Deadlock Dynamic Detection
Static graph analysis of runtime locking relationships allows deadlocks to be predicted before they block execution:
1.  **Lock Order Graph**: TSan maintains a global directed graph representing active locking dependencies. Vertices represent locks, and edges represent lock acquisitions.
2.  **Edge Insertion**: If a thread holding lock $L_1$ attempts to acquire lock $L_2$, TSan inserts a directed edge: $L_1 \rightarrow L_2$.
3.  **Cycle Analysis**: Upon inserting any dependency edge, TSan runs a cycle detection algorithm. If a cycle is detected (e.g., $L_1 \rightarrow L_2 \rightarrow L_1$), TSan immediately issues a deadlock warning, exposing lock inversion bugs.

---

## 📂 M4: MemorySanitizer (MSan) Uninitialized Memory Reads (Cards 16-19)

#### Card 16. Uninitialized Memory Read Vulnerabilities
Reading uninitialized memory is a subtle yet severe class of systems programming bugs:
1.  **Residual Dirty Data**: Recycled physical pages reallocated for new variables stack/heap space retain previous memory states. Variables declared without explicit initializers read this dirty data.
2.  **Information Leakage**: Transmitting uninitialized structures (which contain alignment padding bytes) via sockets leaks stack/heap fragments over the network.
3.  **Control Flow Hijacking**: If uninitialized data affects an `if` branch condition or is cast to a function pointer, an attacker can manipulate stack layouts to hijack control flow.

#### Card 17. Shadow Memory Bit-Map Mapping
To track variable initialization at a single-bit level, MSan uses a 1:1 bit-to-bit shadow mapping:
1.  **1:1 Byte Mapping**: Every 1 byte of application memory corresponds to 1 byte of shadow memory, yielding a 1:1 mapping ratio.
2.  **Mapping Formula**:
    $$\text{ShadowAddr} = \text{AppAddr} \oplus 0x200000000000$$
3.  **Bit Initialization Flags**: Each bit in a shadow byte indicates the initialization state of the corresponding application bit:
    *   `0`: The bit has been written to, indicating it is Initialized.
    *   `1`: The bit has not been written, indicating it is Uninitialized (dirty).
4.  **Shadow Propagation**: Arithmetic operations (e.g., `a = b + c`) propagate shadow bits. If `b` or `c` is uninitialized, their dirty shadow bits are propagated to the shadow representation of `a`.

#### Card 18. Origin Tracking Technology
When an uninitialized read occurs, knowing only the crash location is insufficient; developers need to trace where the uninitialized memory was allocated:
1.  **Origin Memory Mapping**: Compiled with `-fsanitize-memory-track-origins`, MSan reserves extra memory. Every 4 bytes of application memory map to a 4-byte Origin space.
2.  **32-Bit ID Metadata**: The Origin space holds a 32-bit ID referencing a global allocation stack trace registry.
3.  **Origin Propagation**: When uninitialized memory is copied (e.g., via `memcpy`), the corresponding Origin ID is copied alongside it. When an uninitialized read eventually triggers a crash, MSan prints the full propagation chain back to the original allocation site.

#### Card 19. Uninitialized Read False-Positive Suppression
To prevent excessive noise, MSan defers reporting uninitialized data until it reaches critical boundaries:
1.  **Deferred Reporting**: MSan does not report errors on simple memory loads or copies (copying uninitialized padding bytes is valid). Reports are triggered only at:
    *   **Conditional Branches**: When an uninitialized value determines an `if` / `while` branch transition.
    *   **Syscall Parameters**: When uninitialized data is passed to a syscall (e.g., `write`, `send`).
2.  **Uninstrumented Code Blind Spots**: MSan can only track code compiled at the LLVM IR level. Inline assembly and uninstrumented third-party libraries (e.g., precompiled dynamic libraries) do not update shadow memory, leading to potential false positives or false negatives.

---

## 📂 M5: UBSan & LSan Runtime Verification & Analysis (Cards 20-24)

#### Card 20. Memory Leaks and Conservative GC Scanning
Memory leaks degrade system reliability and eventually lead to Out-Of-Memory (OOM) process termination:
1.  **Leak Definition**: Memory allocated via `malloc` loses all references to its start address before being freed, creating an unreachable memory island.
2.  **Conservative GC Approach**: LeakSanitizer (LSan) acts as the mark phase of a conservative garbage collector (GC), avoiding heavy reference-counting overhead.
3.  **Reachability Axiom**: If no pointer in the application's root set (registers, stack, globals) references the address of an allocated memory block, that block is unreachable and considered leaked.

#### Card 21. LSan Root Set Conservative Sweep
LSan sweeps the application memory space using a Stop-The-World (STW) mechanism to detect leaked blocks:
1.  **Root Set Definition**: Includes global writable segments (.data and .bss), active CPU registers, and the stack frames of all running threads.
2.  **Pointer Scanning**: LSan pauses all threads (via `ptrace`) and scans the root set. It treats raw 8-byte aligned memory chunks as potential pointers. If a value falls within the `[start_addr, end_addr]` range of an allocated heap block, it marks the block as "Reachable."
3.  **Recursive Coloring**: LSan recursively scans reachable heap blocks for pointers to other blocks. At the end of the sweep, any allocated block not marked reachable is flagged as a leak, and its allocation stack trace is printed.

#### Card 22. UBSan Arithmetic and Pointer Safety Checks
UndefinedBehaviorSanitizer (UBSan) targets runtime actions that violate language specifications:
1.  **Signed Integer Overflow**: In C/C++, signed overflow is undefined. The compiler replaces normal arithmetic instructions with overflow-checking intrinsics (e.g., `llvm.sadd.with.overflow`). If an overflow occurs at runtime, it triggers a UBSan report.
2.  **Null Pointer Dereferences**: UBSan inserts null checks `if (ptr == nullptr)` before dereferences, preventing hard segfaults and emitting informative error locations instead.
3.  **Misaligned Access**: Casting an odd memory address to an aligned pointer (e.g., `int*`) violates alignment requirements. UBSan checks `(uintptr_t)ptr % alignof(type) != 0` before accesses.

#### Card 23. UBSan Bounds and Vptr Validation
Array bound checking and C++ polymorphism protection guard against out-of-bounds corruption:
1.  **Array Bounds Checking**: UBSan extracts static array bounds during compilation or dynamically calculates Variable-Length Array (VLA) sizes at runtime. For each index `i`, it asserts `if (i >= array_size)` to prevent buffer overflows.
2.  **Vptr Validation**: Before dynamic C++ virtual function calls, UBSan verifies the virtual table (vtable) pointer (`vptr`). It checks if the `vptr` points to a valid vtable layout defined in the application's metadata, preventing control flow hijacking via vtable tampering.

#### Card 24. UBSan Downcasting and Control Flow Integrity (CFI)
Securing class hierachy casting and enforcing control-flow bounds are vital for exploit mitigation:
1.  **Downcast Verification**: Casting a base class pointer to a derived class pointer (`static_cast` or `dynamic_cast`) is undefined if the underlying object is not an instance of the target derived class. UBSan queries Run-Time Type Information (RTTI) to prevent Type Confusion exploits.
2.  **Control Flow Integrity (CFI)**: CFI constructs a directed Control Flow Graph (CFG) at compile time, identifying all valid targets for indirect calls (function pointers, virtual functions). At runtime, it checks target PCs before jumps. If a pointer has been overwritten and points to an unauthorized address, CFI terminates the process immediately.

---

## 📂 M6: Sanitizers Operations, Diagnostics & Deployment (Cards 25-28)

#### Card 25. Runtime Configuration and Suppressions
Sanitizers support fine-grained runtime control to facilitate deployment on legacy systems:
1.  **Environment Variables**:
    *   ASan: `ASAN_OPTIONS=quarantine_size_mb=64:detect_odr_violation=1` manages quarantine sizes and checks for One Definition Rule violations.
    *   TSan: `TSAN_OPTIONS=history_size=7` retains longer thread history records.
2.  **Suppressions File**: Used to ignore known warnings in uninstrumented third-party libraries (e.g., `race:libcrypto.so`).
3.  **Compilation Ignorelist**: Configured via `-fsanitize-ignorelist=ignorelist.txt` to bypass instrumentation for specific source files or performance-critical functions.

#### Card 26. ASan Diagnostic Stack Trace Reconstruction
ASan reports follow a highly structured layout for rapid diagnostic analysis:
1.  **Header Summary**: Displays the error type (e.g., `heap-use-after-free`), the action (READ/WRITE), the thread ID, and the crash stack trace.
2.  **Lifecycle Trace**: For UAF errors, ASan outputs the stack trace where the heap block was allocated, followed by the stack trace where it was freed.
3.  **Shadow Memory Grid**: Prints the shadow bytes surrounding the offending address. The corrupted address is highlighted in brackets (e.g., `[fd]` for freed heap, `[fa]` for left redzone), providing physical evidence of the memory state.

#### Card 27. TSan and MSan Integration Troubleshooting
Uninstrumented libraries and low-level asynchronous primitives are the primary sources of diagnostic issues:
1.  **TSan False Positives**: Custom locking mechanisms implemented via assembly or raw atomic instructions bypass TSan's thread synchronization detection. This results in false data race warnings. To resolve this, developers insert explicit synchronization annotations (e.g., `ANNOTATE_HAPPENS_BEFORE`).
2.  **MSan Unpoisoning**: Writing to memory via syscalls (e.g., `ioctl`) or uninstrumented libraries (e.g., OpenSSL) does not update shadow memory. This causes subsequent reads to trigger uninitialized memory errors. Developers must manually call `__msan_unpoison(addr, size)` after such writes.

#### Card 28. Fuzzing Integration and Gold Canary Deployment
Combining Sanitizers with fuzz testing (Fuzzing) is the industry standard for vulnerability discovery:
1.  **Fuzzing Compilation**: Code is instrumented with ASan/UBSan and compiled as a binary target for fuzzing engines like libFuzzer or AFL++.
2.  **Crash-Driven Exploration**: The fuzzer mutates inputs to find edge cases. When a vulnerability is triggered, the Sanitizer halts the process, allowing the fuzzer to log the crashing input.
3.  **Canary Deployments**: In CI/CD pipelines, canary containers compiled with Sanitizers are run under real staging traffic to intercept memory leaks and concurrency errors before production deployment.

---

## 📂 Sanitizers Performance Overhead & Detection Accuracy Trade-off Matrix

| Design Dimension | Option A (AddressSanitizer - ASan) | Option B (ThreadSanitizer - TSan) | Trade-off Balance |
| :--- | :--- | :--- | :--- |
| **Detection Goal & Core Algorithm** | Shadow memory mapping and redzone poisoning (8:1 Shadow Map & Redzone/Quarantine) | Vector clocks and shadow cell partial order comparisons (Vector Clocks & 4x Shadow Cells) | ASan targets spatial and temporal safety (out-of-bounds, UAF) in a single-threaded context ➜ cannot detect multi-threaded data races. TSan targets happens-before order violations in multi-threaded code ➜ cannot detect buffer overflows or memory leaks [In development pipelines, run ASan first to ensure single-threaded memory safety, then compile with TSan to identify concurrency races]. |
| **Execution Overhead Control** | 8:1 shadow memory shift and static padding (CPU slowdown 1.5x - 2.5x, Memory footprint 2x) | 4:1 compressed shadow cells and vector clock operations (CPU slowdown 5x - 10x, Memory footprint 4x - 9x) | ASan requires only simple bit shifts for address validation, making it suitable for continuous fuzzing and integration testing ➜ but quarantine queues consume extra memory. TSan must update vector clocks and slide circular buffers for every memory access, causing a large slowdown [Use ASan for routine CI testing; isolate TSan runs for concurrent stress test suites]. |
| **Bit Mapping & Origin Tracking** | 1:1 bit-to-bit shadow mapping and Origin chains (MemorySanitizer - MSan) | Conservative GC root scanning (LeakSanitizer - LSan) | MSan's 1:1 bit map provides high accuracy for uninitialized reads, tracing allocation sites via Origin ID chains ➜ but adds 3x CPU slowdown and doubles memory usage. LSan performs root sweeps on process exit, requiring no compile-time instrumentation and adding zero runtime overhead ➜ but can miss leaks if pointers remain in memory [Use MSan during debugging to eliminate uninitialized reads; keep LSan enabled inside ASan to audit memory leaks]. |
| **Instrumentation Depth & Boundaries** | Compile-time IR-level instrumentation (LLVM IR-level Instrumentation) | Runtime interceptors (Runtime Interceptors) | IR instrumentation is highly accurate and yields zero false positives ➜ but cannot inspect assembly or uninstrumented libraries. Runtime interceptors hijack dynamic library calls without source code ➜ but can break if library interfaces drift [Instrument all application source code, and use suppression files to filter false positives arising from uninstrumented third-party binaries]. |

---

## 🔬 Zone T: Compilation Flags, Environment Variables & Suppression Reference

### T1: Core Sanitizers Compilation and Runtime CLI Commands
*   `clang++ -fsanitize=address -fno-omit-frame-pointer -g main.cpp -o app` : Compiles the application with ASan enabled. `-fno-omit-frame-pointer` ensures fast stack unwinding, and `-g` preserves DWARF debugging symbols for `llvm-symbolizer`.
*   `clang++ -fsanitize=thread -g -O1 main.cpp -o app` : Compiles the application with TSan. Enabling optimization (at least `-O1`) allows the LLVM Pass to prune redundant check instructions, reducing CPU slowdown from 10x to ~5x.
*   `clang++ -fsanitize=memory -fsanitize-memory-track-origins=2 -g -O2 main.cpp -o app` : Compiles the application with MSan. `track-origins=2` records the allocation stack trace and tracks the history of uninitialized memory copying.
*   `clang++ -fsanitize=undefined main.cpp -o app` : Compiles the application with UBSan to inspect undefined behaviors (overflows, misalignment, null dereferences). Errors are reported to stderr without halting by default.

### T2: ASan/TSan/MSan Suppression Rules & Runtime Diagnostic Dictionary
To ignore warnings in uninstrumented third-party libraries or suppress specific errors, define suppression files and load them at runtime:
*   **ASan Suppression Configuration (asan_supp.txt)**:
    ```bash
    # Suppress ODR violation errors in a specific shared library
    odr_violation:libthirdparty.so
    # Suppress memory leak detection for a specific legacy function
    leak:MyLegacyAllocFunc
    ```
    *   *Usage*: Run the binary with: `export ASAN_OPTIONS=suppressions=asan_supp.txt:detect_leaks=1`.
*   **TSan Suppression Configuration (tsan_supp.txt)**:
    ```bash
    # Suppress data race warning for a specific global variable
    race:g_shared_unlocked_var
    # Suppress data race warning inside an uninstrumented dynamic library
    race:libcrypto.so.1.1
    # Suppress deadlock warning for a specific mutex class call
    deadlock:MyWeirdMutexManager::Lock
    ```
    *   *Usage*: Run the binary with: `export TSAN_OPTIONS=suppressions=tsan_supp.txt`.
*   **Common Runtime Option Combinations**:
    *   `ASAN_OPTIONS=halt_on_error=0:log_path=./asan.log` : Prevents the process from exiting immediately on the first error (`halt_on_error=0`) and redirects ASan stack traces to log files.
    *   `MSAN_OPTIONS=poison_in_free=1` : Poisons heap blocks immediately upon release, enabling MSan to detect temporal violations in multithreaded environments.
