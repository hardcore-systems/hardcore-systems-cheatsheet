# iovisor / bcc (eBPF) High-Density Knowledge Map & Cheatsheet

*   **L0 One-Line Essence**: Under the physical limits of not modifying kernel source code and not restarting the system, safe and zero-latency kernel behavior interception, performance observability diagnostics, and high-speed network packet bypassing are achieved by running JIT-compiled bytecode inside a safe kernel-space sandbox.
*   **L1 Four-Line Logic**:
    1. The **Safe VM Verifier** guards access using strict static analysis, verifying DAG loops and register type states to guarantee loaded code never crashes or hangs the kernel.
    2. **Multi-Source Instrumentation Probes** unify kernel kprobes/tracepoints and user-space uprobes/USDT into events, enabling transparent, full-stack observability.
    3. The **High-Speed Network Datapath** utilizes XDP and TC at the network driver layer, executing packet filtering, modification, and Sockmap zero-copy redirecting prior to socket buffer allocations.
    4. The **Lockless Shared Memory Grid** leverages BPF Maps and BPF Ring Buffers to transfer events asynchronously between kernel-space and user-space with minimal overhead and lifecycle pinning.
*   **L2 Eight Kernel Subsystem Topology**:
    *   [C/eBPF Source] $\rightarrow$ [Verifier Validation] $\rightarrow$ [JIT Assembly Generation] $\rightarrow$ [Kernel Hooks (Kprobes/XDP/LSM)] $\rightarrow$ [Lockless Maps/Ring Buffer] $\rightarrow$ [bpffs Pinning] $\rightarrow$ [User-space Consumer (bpftool/BCC)] $\rightarrow$ [BTF CO-RE Relocation]

---

## 🌐 eBPF Kernel Epistemic Filter

- **Epistemology - Invisible Monitoring & Hot Patching**:
  Traditional system diagnostics rely on application instrumentation (highly intrusive) or kernel recompilation (highly costly). eBPF's core epistemology lies in "physical ignorance, behavior control." The system does not need any prefix modifications to obtain internal monitoring data; it can inject behavior-capturing hooks dynamically at runtime, treating the kernel as a hot-pluggable, safely programmable execution environment.
- **VM Sandbox Philosophy - Absolute Distrust & Safety Enforcement**:
  eBPF treats C bytecode written by developers as "potentially malicious instructions." The virtual machine does not assume the programmer has advanced kernel safety awareness. Instead, it utilizes a static validator (Verifier) to block any code that might cause infinite loops, null pointer dereferences, or out-of-bounds memory accesses. Safety is the highest priority (better to reject loading than risk 0.001% kernel crash).
- **Methodology - Kernel Bypass & Zero-Copy**:
  The physical performance limit of traditional network and IO processing lies in the overhead of frequent user-kernel context switches and large memory buffer copies. eBPF bypasses the protocol stack by intercepting packets at the driver layer (XDP) and redirecting socket bytes in-kernel (Sockmap), driving down processing overhead to zero and executing the methodology of "local computation, early filtering, lockless sharing."
- **Ethics - Access Isolation & Observability Boundaries**:
  Observability is a double-edged sword. eBPF's ability to intercept system call arguments means it can silently capture sensitive passwords, SSL plaintexts, and private packets. Therefore, eBPF's safety ethics mandate that loading privileges be strictly restricted to privileged users (`CAP_SYS_ADMIN` or `CAP_BPF`), and BPF LSM is used to prevent tracers themselves from becoming stealthy rootkits.

---

## ⚔️ eBPF Architectural Trade-offs & Fallacies

| Developer Intuition (⚠) | Objective Technical Reality (✓) | Code & Architectural Defenses (★) |
| :--- | :--- | :--- |
| **Intuition 1**: Using `uprobe` in user space to instrument high-frequency calls (like `malloc` or light JSON parsers) has negligible performance impact. | **Reality**: `uprobe` relies on soft-interrupt breakpoints, forcing two user-kernel context switches per execution. Instrumenting hot paths can drop application throughput by over 50%. | Avoid setting `uprobe` on paths with QPS > 10,000. Use **USDT** (static probes) for hot paths, or aggregate data in-app and share it via BPF Maps. |
| **Intuition 2**: `Perf Ring Buffer` is the fastest event channel to user space in multi-core setups; as long as the buffer size is large enough, no events will be lost. | **Reality**: `Perf Buffer` allocates memory per-CPU. A spike on one core while its consumer thread is delayed causes buffer overflow and event drops on that core. Events are also out-of-order in user space. | Deprecate `Perf Buffer`. Upgrade to the globally shared **BPF Ring Buffer** in production. It uses a lockless MPSC ring to balance loads across cores and supports zero-copy `reserve`/`commit` APIs. |
| **Intuition 3**: BPF supports loops, so developers can write complex multi-dimensional array traversals or string matching loops for business logic filtering. | **Reality**: The verifier strictly limits program complexity. Even after bounded loops were introduced in 5.3+, deep loop branches can exceed the verifier's `instruction limit` (1M instructions) due to path state explosion. | Pre-process complex structures in user space. Keep in-kernel operations to basic fixed-size binary searches or hash lookups. Unroll loops using `#pragma unroll` and keep exits simple. |
| **Intuition 4**: BCC dynamic compilation (passing C code as Python strings and compiling on-the-fly using Clang/LLVM) is the only standard for eBPF tools. | **Reality**: BCC requires the target machine to install Clang/LLVM and kernel headers (>100MB), causing a severe CPU spike on startup due to on-target compilation. | Adopt **BPF CO-RE (Compile Once - Run Everywhere)**. Generate BTF relocation metadata at build time, producing lightweight (<50KB) binaries that load in <10ms with zero target dependencies. |
| **Intuition 5**: `bpf_probe_read_user()` can read any user-space pointer; hence, tracing code can read arbitrary user-space strings at any time. | **Reality**: User memory is demand-paged and can be swapped out (Page Fault). When BPF reads a swapped-out page, BPF cannot trigger page faults in tracing context, causing the read to fail silently. | Ensure pages are resident before reading. Read immediately after a syscall returns, or use user-space triggers to wake pages up. Always check helper return codes. |

---

## 🧠 28 High-Density Cards (Card 01 - 28)

### Module 1: Sandbox & VM Architecture

#### Card 01. eBPF Registers & CPU Model
*   **Register Constraints**: eBPF virtual CPU defines 11 64-bit hardware registers. `R0` stores helper return values and exit status; `R1-R5` pass arguments to helper functions; `R6-R9` are callee-saved (restored via BPF stack); `R10` is a read-only Frame Pointer pointing to the stack base.
*   **Physical Limits**: The stack size is strictly limited to **512 bytes**. Exceeding this limit causes immediate verifier rejection. Large objects or complex contexts must be stored in BPF Maps instead of local stack variables.

#### Card 02. Bytecode & JIT Compilation
*   **Compilation Flow**: C source code is compiled via LLVM with `-target bpf` to produce an ELF binary containing BPF bytecode. Upon loading, the kernel **JIT (Just-In-Time) Compiler** (e.g., x86_64 JIT) translates bytecode directly into native instructions (e.g., x86 `MOV/JMP`).
*   **Production Check**: Ensure `/proc/sys/net/core/bpf_jit_enable` is set to `1` (JIT enabled) and `bpf_jit_harden` is set to `2` (harden against JIT spraying) to guarantee native execution speeds.

#### Card 03. Verifier Static Analysis
*   **Verification Flow**: The verifier works in two phases. Phase 1 runs a Depth First Search (DFS) on the Control Flow Graph (CFG) to reject loops and unreachable code. Phase 2 simulates execution to track register "type states" (e.g., `SCALAR_VALUE`, `PTR_TO_STACK`) and bounds.
*   **Defense Strategy**: Any pointer arithmetic without NULL checks or out-of-bounds stack access will trigger rejection. "State pruning" is used to merge evaluation paths and reduce CPU overhead during verification.

#### Card 04. Helper Functions
*   **Interface Bounds**: eBPF programs cannot call arbitrary kernel functions (e.g., `printk` or internal symbols) to protect the kernel call stack. All external interactions must go through static, pre-defined Helper Functions (e.g., `bpf_map_lookup_elem`, `bpf_ktime_get_ns`).
*   **Permission Control**: Each program type (e.g., `BPF_PROG_TYPE_KPROBE` vs `BPF_PROG_TYPE_SOCKET_FILTER`) has a restricted whitelist of allowed helpers. Blocking or sleeping helpers are strictly prohibited in driver-level packet hooks.

#### Card 05. BPF LSM Security Hooks
*   **Security Framework**: BPF LSM allows developers to attach eBPF programs to Linux Security Module (LSM) hooks (e.g., `path_mkdir`, `file_permission`, `task_alloc`).
*   **Fine-Grained Auditing**: When a system operation triggers, the BPF program inspects subject/object metadata and returns `0` (allow) or non-zero (e.g., `-EACCES`, deny), acting as a dynamic, hot-swappable security engine.

---

### Module 2: Kernel Tracing

#### Card 06. Kprobes & Kretprobes
*   **Instrumentation**: `kprobe` dynamically rewrites the first instruction of a target kernel function to a breakpoint instruction (e.g., `int3` on x86), trapping execution to the BPF handler. `kretprobe` hijacks the return address to capture exit parameters and function execution time.
*   **Overhead Penalty**: Due to soft interrupts, each probe costs thousands of CPU cycles. Do not use dynamic `kprobes` on hot scheduling paths (`schedule`) or high-frequency hardware interrupt handlers.

#### Card 07. Tracepoints
*   **Static Probes**: Stable hooks pre-defined by kernel developers at critical sub-system locations (e.g., `sched_switch`, `sys_enter_read`). They offer stable ABI structures across different kernel versions.
*   **Zero-Disabled Cost**: When inactive, a tracepoint is merely a NOP instruction (zero CPU overhead). Once enabled, it jumps to execute the attached BPF program, calling much faster than `kprobe`.

#### Card 08. Raw Tracepoints
*   **Performance Bypassing**: Standard `tracepoint` structures utilize format adapters to serialize parameters into readable structures. `raw_tracepoint` bypasses this formatting layer, passing raw arguments (e.g., `struct pt_regs` or raw pointers) directly to BPF.
*   **High-Throughput Use**: Ideal for massive traffic networks or disk IO tracing, saving parameter conversion costs and raising tracing throughput limits by over 15%.

#### Card 09. fentry / fexit & BPF Trampoline
*   **Direct Call**: Introduced in kernel 5.5. BPF Trampoline dynamically generates assembly bridges to call BPF programs directly before/after kernel functions.
*   **Performance Jump**: By completely avoiding the soft-interrupt context-saving overhead of `kprobe` (no `int3` trap), `fentry/fexit` executes as fast as static tracepoints.

---

### Module 3: User-Space Tracing

#### Card 10. Uprobes & Uretprobes
*   **Instrumentation**: Modifies virtual memory of user-space processes (e.g., `malloc` in `libc.so`) by injecting breakpoint instructions. Execution shifts to kernel space when the instruction is hit.
*   **Context Switch Cost**: Since it triggers "user $\rightarrow$ kernel $\rightarrow$ user" mode switches, each `uprobe` call incurs a 1-2 microsecond delay. Never place `uprobe` hooks inside high-frequency loops in C++/Rust programs.

#### Card 11. USDT Probes
*   **User-Space Static Tracing**: Statically defined tracepoints embedded in applications (e.g., MySQL, JVM, Node.js) via `DTRACE_PROBE` macros.
*   **Dynamic Patching**: Compiled as a NOP instruction and cataloged in the ELF `.note.stapsdt` section. Once tracing starts, the kernel patches the NOP to a breakpoint trap, capturing events with minimal application overhead.

#### Card 12. Symbol Resolution & DWARF
*   **Symbol Mapping**: BPF trace captures user stack frames as virtual memory addresses (e.g., `0x7fff81a2f10b`). Tracing tools (BCC/Golang) must parse the process's ELF `.symtab`/`.dynsym` sections to map addresses to function names.
*   **Debug Info**: If the binary is `stripped`, external DWARF debug symbols are required. For JITed runtimes (Java, V8 JIT), parse `/tmp/perf-<pid>.map` generated by the runtime.

#### Card 13. Memory Leak Tracing
*   **Leak Detection**: In BCC's `memleak`, we trace memory allocations (`malloc`/`calloc`) using `uprobes` (or `fentry` for kernel-level slabs) and record pointer addresses and callstacks into a BPF Hash Map.
*   **Trace Delta**: We trace `free` using another probe and delete the pointer key from the map. Residual keys in the map after execution represent leaking memory buffers and their allocation traces.

---

### Module 4: High-Performance Networking

#### Card 14. XDP (eXpress Data Path)
*   **Early Ingest**: XDP runs at the earliest point in the network driver RX loop (before allocating `sk_buff` structures or invoking the host network stack).
*   **Verdict Codes**: Decisions: `XDP_DROP` (discard immediately, perfect for DDoS mitigation), `XDP_TX` (bounce back out the same interface), `XDP_REDIRECT` (bypass routing to send to another NIC or user space `AF_XDP` socket) within nanoseconds.

#### Card 15. TC (Traffic Control) Clsact
*   **Dual Ingress/Egress**: BPF attached to the TC `clsact` queuing discipline can process packets in both inbound (Ingress) and outbound (Egress) directions.
*   **Access Detail**: TC programs parse and rewrite the full `struct __sk_buff` metadata, including VLAN tags and headers, making it the base for container firewalls, load balancers, and QoS shapers.

#### Card 16. Socket Filters
*   **Socket Attachment**: Attached directly to sockets via `setsockopt(sock, SOL_SOCKET, SO_ATTACH_BPF, ...)`.
*   **Application Sandbox**: The classic cBPF interface. It passively mirrors and filters packet payloads without modifying them, commonly used for tcpdump-like traffic mirroring and auditing.

#### Card 17. Sockmap & Sockhash Socket Redirect
*   **Loopback Shortcuts**: `BPF_MAP_TYPE_SOCKMAP` stores active socket file descriptors. When attached to `sk_msg`, data written to one socket's transmit queue is copied directly into the read queue of the destination socket.
*   **Stack Bypass**: Bypasses the IP and TCP layers. For local loopback traffic (`localhost` / inter-container calls), it reduces latency by over 50%.

#### Card 18. Container Network Acceleration
*   **Virtual Path Bottleneck**: Standard Kubernetes networking routes packets through veth-pairs twice (Pod stack $\rightarrow$ Host stack $\rightarrow$ Destination Pod), degrading performance.
*   **Cilium Solution**: Cilium identifies CNI topologies using eBPF and maps container sockets. Packets bypass virtual adapters and routing tables, jumping directly between socket ring buffers.

---

### Module 5: Shared Storage & Data Flow

#### Card 19. BPF Maps
*   **Type Selection**:
  * `BPF_MAP_TYPE_HASH`: Dynamic KV map for general sparse lookups;
  * `BPF_MAP_TYPE_ARRAY`: Fixed-size pre-allocated array with lowest lookup latency;
  * `BPF_MAP_TYPE_LRU_HASH`: Evicts oldest elements, preventing memory exhaustion under traffic spikes.
*   **Concurrency**: Under high-concurrency writes, use `Per-CPU` maps (e.g., `BPF_MAP_TYPE_PERCPU_HASH`). Each CPU core owns a private data segment, enabling lockless writes and preventing CPU cache thrashing.

#### Card 20. Perf Ring Buffer
*   **Per-CPU Ring**: The traditional channel for sending events to user space. The kernel allocates circular buffers per CPU core.
*   **Event Drop Penalty**: If one CPU core spikes and the user-space thread lag, its local buffer overflows, discarding events. Events are also out-of-order in user space due to the multi-ring structure.

#### Card 21. BPF Ring Buffer
*   **Global MPSC**: Introduced in kernel 5.8. A globally shared, lockless Multi-Producer Single-Consumer (MPSC) ring buffer across all CPU cores.
*   **Zero-Copy API**: Supports `bpf_ringbuf_reserve` to allocate space directly in the shared buffer. The program writes data in place and commits with `bpf_ringbuf_commit`, avoiding copying from kernel stack to ring.

#### Card 22. Map Operations & User Sync
*   **System Call Bounds**: User-space processes interact with maps via the `bpf(BPF_MAP_LOOKUP_ELEM, ...)` system call. Polling maps inside a loop induces heavy syscall overhead.
*   **Event-Driven**: BPF Ring Buffers have built-in epoll fds. The user-space consumer sleeps on `epoll_wait` and wakes up only when new data crosses the ring buffer's watermark, keeping idle CPU usage near zero.

#### Card 23. Map Pinning & bpffs
*   **FD Lifecycle**: By default, BPF maps are bound to the program fd. Once the daemon exits and the eBPF program is unloaded, maps and their historical metrics are cleaned up.
*   **State Pinning**: Pin maps to the virtual BPF filesystem `bpffs` (typically `/sys/fs/bpf/`). This allows maps to survive restarts and enables different eBPF programs or user daemons to share state.

---

### Module 6: Production Operations & CO-RE

#### Card 24. BPF CO-RE (Compile Once – Run Everywhere)
*   **Offsets Challenge**: Offsets of fields within kernel structures (like `struct task_struct`) change across kernel versions, causing BPF binaries compiled on machine A to read corrupt offsets on machine B.
*   **Relocation Solution**: Compile BPF code with debugging symbols. Relocation macros (`__builtin_preserve_access_index()`) generate relocation records. At load time, `libbpf` matches local kernel BTF definitions and patches offsets in bytecode.

#### Card 25. BTF (BPF Type Format)
*   **Compact Debug Metadata**: BTF is a compact, compressed format for kernel types (100x smaller than DWARF). It encodes structure fields, offset locations, and function definitions.
*   **Header Removal**: Kernels export BTF via `/sys/kernel/btf/vmlinux`. Developers can generate `vmlinux.h` to access all kernel definitions, eliminating dependencies on external kernel headers.

#### Card 26. BCC vs libbpf-bootstrap
*   **BCC Drawback**: BCC embeds compiler runtimes (Clang/LLVM) to compile BPF C code on the target machine, requiring large containers and causing high CPU usage during launch.
*   **libbpf Advantage**: `libbpf-bootstrap` compiles BPF code into static bytecode embedded directly inside the user-space C binary. The tool is lightweight (<100KB), has zero dependencies, and loads in milliseconds.

#### Card 27. bpftool Diagnostics
*   **CLI Utility**: The official CLI tool for managing and inspecting the kernel BPF subsystem.
*   **Key Diagnostics**:
  * List active programs: `bpftool prog show`;
  * Dump JITed CPU instructions: `bpftool prog dump jited id <prog_id>`;
  * Pin maps manually: `bpftool map pin id <map_id> /sys/fs/bpf/<map_name>`.

#### Card 28. eBPF Production Safety & Verifier Tuning
*   **Kernel Shield**: The verifier blocks programs containing loops without static bounds, exceeding stack allocations, or dereferencing pointers without NULL checks, preventing kernel panics.
*   **Tuning Defenses**: Unroll loops with `#pragma unroll`. Always check pointer returns from map lookups before accessing fields. Use `Tail Calls` (limited to 33 nested jumps) to bypass instruction limits by splitting code into multiple programs.

---

## 叁、 Zone T: eBPF Performance & Diagnostic Labs

### T1 eBPF Performance Overhead & Limits
*   **Hook Latency Baselines**:
    *   `Tracepoint` $\rightarrow$ ~10-20 ns/call | `Kprobe` $\rightarrow$ ~100-150 ns/call | `Uprobe` $\rightarrow$ ~1-2 us/call (severe context switch penalty).
*   **Physical Constraints**:
    *   Maximum instruction limit: 1,000,000 instructions in kernels 5.2+ (legacy limit was 4096).
    *   BPF stack size: **512 bytes**.
    *   Tail call limit: **33 nested jumps**.

### T2 bpftool Tuning & Diagnostic Gold Standards
*   **Debug Verifier Failures**:
    `bpftool prog load ./my_prog.o /sys/fs/bpf/my_prog type tracepoint` (Check kernel `dmesg -w` for the verifier trace if it fails)
*   **List BPF Maps & Memory Footprint**:
    `bpftool map show`
*   **Profile Program Cycles**:
    `bpftool prog profile id <prog_id> runs cycles` to evaluate instruction overhead and cache miss ratios.

### T3 eBPF Production Troubleshooting Guide
1.  **Resolving Verifier Rejections**
    *   *Symptom*: `R2 invalid mem access 'mem_or_null'`
    *   *Defense*: Always check map lookup pointers: `if (val == NULL) return 0;` before dereferencing. This registers a safe path branch in the verifier.
2.  **Mitigating Event Loss (Lost Events)**
    *   *Symptom*: Tracing outputs `Lost 1432 events`.
    *   *Defense*: Increase `Perf Buffer` allocation size, or switch to `BPF Ring Buffer` using `bpf_ringbuf_output`. Verify user-space consumer callbacks are not blocked to keep pace with kernel events.
3.  **Fixing User-Space Symbol Resolution Failures**
    *   *Symptom*: User stack traces show only `[unknown]` or raw hex addresses.
    *   *Defense*: Verify the target binary was compiled with debug flags (`-g`) and not `stripped`. For Go/JVM runtimes, ensure `/tmp/perf-<pid>.map` is generated and point symbol search paths to it.
