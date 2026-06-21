# Wasmtime High-Performance JIT VM & Lightweight Sandbox Runtime Cheatsheet

## J-Ladder Hierarchical Model

### L0 One-Line Essence
Wasmtime compiles WebAssembly bytecode into native machine code using the Cranelift JIT compiler and establishes a zero-overhead lightweight security sandbox using virtual address space mapping and hardware Guard Pages.

### L1 Four-Sentence Logic
1. **Static Validation & SSA Translation**: Performs single-pass validation of Wasm bytecodes before translating them into SSA-based Cranelift Intermediate Representation (CLIF).
2. **Zero-Overhead Hardware Guarding**: Allocates a large virtual space (4GB + 2GB Guard Pages) per instance, leveraging the OS page fault mechanism to catch out-of-bounds accesses instead of inserting software branch checks.
3. **Component-Based Semantic Interoperability**: Implements the Wasm Component Model and WebAssembly Interface Type (WIT) IDL to bridge complex types and async streams across host-guest boundaries.
4. **Hardware Signal Trap Redirection**: Registers custom thread signal handlers to capture hardware faults (e.g. division by zero, page access violation) and safely unwind them into structured Wasm Traps.

### L2 Core Data Flow
`Wasm Bytecode Load` ➜ `Cranelift JIT Compilation (CLIF SSA ➜ Native Code)` ➜ `Virtual Address Reservation (4GB + 2GB Guard)` ➜ `Linker Imports Injection` ➜ `Instantiation & Run` ➜ `Out-of-Bounds Access` ➜ `OS SIGSEGV/Page Fault` ➜ `Signal Handler Unwinding` ➜ `Structured Wasm Trap Output`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: Wasmtime Engine & Config Initialization
*   **Theory**: Immutable compilation and execution engine configuration core. The Engine acts as the thread-safe global context container while Config acts as the builder.
*   **Details**: Host configures compilation levels, JIT/AOT modes, memory reservation strategies, and limits via `Config`. Instantiating `Engine::new(config)` freezes the configuration, making it a read-only, shared coordinator for modules and threads.
*   **Trade-off**: Heavy JIT optimizations (e.g. Speed level 2) increase cold startup compilation latency. Utilize pre-compiled AOT modules and disk caching to bypass JIT compilation overhead during cold starts.

### Card 2: Store Space & Data Isolation
*   **Theory**: Isolation boundary containing active runtime data. Every Store encapsulates instance states, custom host data, resource limits, and GC root sets.
*   **Details**: Wasmtime enforces a strict "no-shared-memory" model between Stores. Host and Guest exchange information safely via `StoreContext`. The Store acts as the GC root set tracker for Wasm Garbage Collection references.
*   **Trade-off**: Stores are single-threaded and do not implement `Send`. Multi-threaded applications must spin up separate Stores per thread and coordinate state updates in the application layer.

### Card 3: Module Compile & Validation
*   **Theory**: Secure compilation unit guaranteeing execution safety and local optimization. Translates Wasm bytecode into an ELF-formatted executable.
*   **Details**: `Module::new` validates Wasm bytecode structures, control-flow jumps, and instruction stacks. Valid modules are split into parallel segments compiled concurrently by Cranelift compiler threads, generating an ELF-formatted JIT package in memory.
*   **Trade-off**: Real-time compilation creates CPU usage spikes. Mitigate this by pre-compiling Wasm modules into AOT files and caching them.

### Card 4: Linker & Import Resolution
*   **Theory**: Connects Guest import declarations with Host implementations prior to instantiation.
*   **Details**: The `Linker` maintains a hash-based type-safe registry mapping module and function names. It validates imports against corresponding Host functions, Wasm instances, and memories before setting up invocation routes.
*   **Trade-off**: Dynamic lookup adds small overhead during instantiation. Use static pre-linking templates to cache imports and speed up setup.

### Card 5: Instance Instantiation Lifecycle
*   **Theory**: Activation of the virtual execution environment by binding compiled modules to active Stores.
*   **Details**: Instantiation allocates linear memories, initializes global states, copies Wasm Data segments into linear memory, and runs the Module's `start` initialization entrypoint.
*   **Trade-off**: Allocating resources and running initialization code adds microsecond-level latency. Utilize virtual memory pooling to pre-allocate memory buffers and avoid runtime mmap calls.

### Card 6: Host-to-Guest Trampoline Dynamic Invocation
*   **Theory**: Bridging layer resolving the differences between Host ABI calling standards (Rust/C) and Wasm JIT register allocation rules.
*   **Details**: JIT code executes using raw assembly rules (Guest ABI). `func.call()` invokes assembly-written trampolines that capture host arguments, place them into CPU registers, align the call stack, and jump to compiled Wasm code.
*   **Trade-off**: Trampolines introduce microsecond register-shuffling overhead. High-frequency functions should use `TypedFunc` wrappers to compile optimized direct calls.

### Card 7: Cranelift SSA CLIF Intermediate Representation
*   **Theory**: Architecture-independent Compiler Intermediate Representation designed around Static Single Assignment (SSA).
*   **Details**: Cranelift CLIF groups instructions into basic blocks. Variable registers are assigned exactly once; variable flows merging from multiple paths are handled via block parameters, avoiding complex register allocation conflicts.
*   **Trade-off**: SSA representations consume more memory than stack-based representations during compilation. Cranelift uses a fast, indexed arena vector representation to minimize memory usage.

### Card 8: Regalloc2 Allocation Logic
*   **Theory**: Fast, robust register allocator mapping infinite virtual SSA values to limited CPU registers.
*   **Details**: Wasmtime utilizes `Regalloc2` to perform single-pass life-cycle scans across basic blocks, placing critical variables in physical registers and inserting spills/reloads for cold paths.
*   **Trade-off**: Allocator performance is highly critical in JIT compilation. Regalloc2 trades marginal compilation density for massive speedups compared to LLVM's register allocation algorithms.

### Card 9: Wasm Bytecode Translation Loop
*   **Theory**: High-speed, single-pass decoder translating stack-based WebAssembly bytecode into Cranelift SSA IR.
*   **Details**: The Cranelift translation loop scans Wasm bytecodes sequentially. It emulates the Wasm operand stack in-memory, compiling Wasm instructions (e.g. `i32.add`) directly into SSA IR nodes (`v2 = iadd v0, v1`).
*   **Trade-off**: Single-pass decoding is memory efficient but generates redundant SSA code, requiring subsequent lightweight optimization passes to cleanup JIT assembly.

### Card 10: Cranelift Compiler Optimizations
*   **Theory**: Low-latency compiler optimization techniques tuned for real-time JIT compilation.
*   **Details**: Cranelift bypasses time-consuming loop optimizations in favor of high-yield passes: constant folding, global value numbering (GVN), dead code elimination (DCE), and local inline optimizations.
*   **Trade-off**: Heavy optimization passes delay startup execution. Utilize `OptLevel::None` during development or debugging to speed up startup times.

### Card 11: Trampoline Dynamic Machine Code Generation
*   **Theory**: Generates machine code adapters on-the-fly for host closures whose function signatures are unknown at compile time.
*   **Details**: For dynamic host functions registered in the Linker, Wasmtime dynamically compiles a custom adapter that parses Wasm register parameters into serialized Rust `Val` arrays.
*   **Trade-off**: Generating code dynamically requires modifying memory execution permissions (`mprotect`). Cache wrappers or pre-compile closures to prevent system page table contention.

### Card 12: JIT Serialization & Compilation Cache
*   **Theory**: Speeds up module loading by saving compiled JIT code to disk as ELF files.
*   **Details**: Wasmtime serializes compiled modules to files. Upon reload, Wasmtime uses `mmap` to map the ELF file as read-only and executable, reducing startup times to microseconds.
*   **Trade-off**: Serialized cache files are tied to specific CPU architectures and Wasmtime versions. Upgrades invalidate caches to prevent SIGILL crashes.

### Card 13: Wasm Linear Memory Virtual Address Mapping
*   **Theory**: Isolates sandbox memory by decoupling Wasm offset addresses from physical host RAM.
*   **Details**: Instantiation reserves a contiguous block of virtual memory using `mmap` or `VirtualAlloc`. Under static memory strategies, Wasmtime reserves 4GB of virtual memory but commits physical pages only as needed.
*   **Trade-off**: Reserving 4GB of virtual memory uses no physical RAM but consumes virtual address space. Multi-tenant architectures running thousands of instances can exhaust virtual addresses.

### Card 14: Guard Pages Hardware Boundary Protection
*   **Theory**: Bypasses runtime check instructions by placing unmapped memory boundaries around Wasm address spaces.
*   **Details**: Wasmtime reserves 2GB of unmapped, protected memory (Guard Pages) immediately following the 4GB Wasm address space. Out-of-bounds reads or writes fall into the guard pages, triggering a hardware CPU Page Fault.
*   **Trade-off**: Eliminates compiler-inserted software bounds checks, boosting performance by 30% while increasing virtual address space consumption to 6GB per instance.

### Card 15: Page Fault Signal Interception
*   **Theory**: Captures hardware errors to prevent host crashes by turning segfaults into handled Wasm Traps.
*   **Details**: Accessing guard pages triggers a SIGSEGV (Linux) or SEH (Windows) exception. Wasmtime's signal handler intercepts it, validates if the faulting IP is inside JIT code, and safely jumps back to host trap handlers.
*   **Trade-off**: Resolving signals adds system-level context switch costs, making this mechanism suitable for error recovery rather than control flow.

### Card 16: memory.grow Dynamic Growth Coordination
*   **Theory**: Allows instances to dynamically resize their heaps while running.
*   **Details**: Under static memory allocation, growing the heap calls `mprotect` to grant read-write permissions to the next chunk of reserved virtual address space.
*   **Trade-off**: Dynamic memory allocation strategies (without 4GB reservations) require allocating new contiguous memory blocks and copying active data, causing temporary execution pauses.

### Card 17: Sandbox OOM Hardware Limitation
*   **Theory**: Protects host memory from resource exhaustion by limiting guest memory allocation.
*   **Details**: Hosts bind `ResourceLimiter` structures to Stores. Grow requests exceeding configuration bounds return `-1`, prompting the Guest runtime to handle OOM or crash while keeping host memory secure.
*   **Trade-off**: Guest Wasm applications not compiled to handle grow failures will enter infinite loops or crash instantly. Use instruction execution limits (fuel/epoch) to clean up.

### Card 18: VM Pools Allocation
*   **Theory**: Eliminates runtime kernel system call overhead in high-frequency multi-tenant environments.
*   **Details**: `InstanceAllocationStrategy::pooling` pre-allocates a large virtual address space split into fixed size slots (e.g. 6GB). Instantiation simply borrows a slot, avoiding runtime mmap/munmap calls.
*   **Trade-off**: Consumes massive amounts of virtual memory up-front, restricting this strategy to 64-bit systems.

### Card 19: Multi-Memory Independent Space Support
*   **Theory**: Allows Wasm modules to instantiate and manage multiple distinct memory address spaces.
*   **Details**: Supports Wasm multi-memory proposals where Guest modules reference indices beyond memory 0. `memory.copy` instructions execute high-performance zero-copy data moves between sandboxes.
*   **Trade-off**: Multiple memory allocations scale virtual space and Guard Page usage, adding pressure on virtual memory pools.

### Card 20: WASI Preview 1 File-Descriptor-Based Syscalls
*   **Theory**: Translates basic POSIX system calls from sandbox environments to host operating systems.
*   **Details**: WASI Preview 1 maps Guest calls (e.g. `wasi_snapshot_preview1::fd_write`) to host file descriptor IO functions under sandbox-enforced limits.
*   **Trade-off**: Traditional file descriptor models struggle to abstract complex async operations or high-level type definitions, prompting the move to WASI Preview 2.

### Card 21: WASI Preview 2/3 Component Model Integration
*   **Theory**: Modern architecture isolating component modules while enabling type-safe interoperability.
*   **Details**: WASI Preview 2 uses the Wasm Component Model. It isolates component memory blocks, routing communications via WebAssembly Interface Types (WIT) and Canonical ABI schemas.
*   **Trade-off**: Component boundaries require serializing/deserializing high-level types, adding translation overhead compared to direct linear memory offsets.

### Card 22: WIT Interface Definition & Type Bridging
*   **Theory**: Standardizes complex data structure transfers (Strings, Records) across sandbox barriers.
*   **Details**: Wasm runtimes only understand numeric types. Wasmtime's Canonical ABI writes Host strings into Guest linear memory, passing only pointers and lengths (i32 numbers) to Wasm code.
*   **Trade-off**: High-frequency allocations and data copying add runtime overhead. Use shared buffer structures to optimize performance.

### Card 23: WASI Asynchronous Resource Channels
*   **Theory**: Prevents Wasm network/IO operations from blocking host runtime threads.
*   **Details**: WASI 0.3 exposes async promises to Guest modules. If an IO block occurs, Wasm yields execution back to host event loops (e.g. Tokio), waking when the reactor registers readiness.
*   **Trade-off**: Async WASI requires compiling Guest modules with async support, which expands Wasm file sizes.

### Card 24: VFS & Net Sandbox Isolation
*   **Theory**: Restricts file and network access to secure host paths and addresses.
*   **Details**: WASI virtualizes directory access by mapping root paths to explicit host folders. Virtual paths are resolved relative to mapped boundaries, blockading directory traversal attacks (e.g. `..` escapes).
*   **Trade-off**: Requires developers to specify path mappings and port layouts, which increases runtime configuration complexity.

### Card 25: Signal-Based Trap Routing
*   **Theory**: Prevents guest execution errors from crashing host processes.
*   **Details**: Division-by-zero or illegal instruction exceptions in Wasm are caught by OS signal handlers. Thread-local flags determine if the crash occurred in Wasm, converting errors to Traps rather than aborting.
*   **Trade-off**: Thread-local storage lookups add negligible branching costs in high-frequency context-switching runtimes.

### Card 26: Stack Backtrace Unwinding
*   **Theory**: Translates raw JIT execution pointer addresses into readable Wasm source line numbers.
*   **Details**: Upon a Trap, Wasmtime captures the CPU instruction pointer (IP) and resolves it against the DWARF debug symbols generated during JIT compilation.
*   **Trade-off**: Generating DWARF symbols increases memory and binary footprint. Disable `generate_debug_info` in production to optimize memory.

### Card 27: Fuel & Epoch-Based Execution Interruption
*   **Theory**: Prevents malicious or bugged Wasm code from running infinite loops and hogging CPU threads.
*   **Details**: Fuel injects instruction counters during compilation, trapping when they hit 0. Epoch polling checks a host-incremented version number, yielding if the Wasm instance lags behind.
*   **Trade-off**: Fuel counters add minor instructions to basic blocks, decreasing execution speed by 10%; Epoch checks are faster (less than 1% loss) but operate at millisecond-level resolution.

### Card 28: call_indirect Signature Runtime Verification
*   **Theory**: Prevents control-flow hijacking by validating dynamic indirect call destinations.
*   **Details**: `call_indirect` instructions compare the target function signature hash with the caller's expected signature before execution, trapping on mismatches.
*   **Trade-off**: Adds memory lookups and branch checks to indirect calls, slightly slowing dynamic dispatches.
