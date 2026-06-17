# RISC-V Instruction Set Architecture & Processor Cores Cheatsheet (v1)

*   **L0 Essence**: The essence of the RISC-V architecture is a modular, regular instruction set definition that supports implementations ranging from simple in-order pipelines with bypass forwarding to highly complex out-of-order execution (Tomasulo algorithm, ROB in-order commit) processors, maintaining shared data consistency across multiple cores via cache coherence (MESI) and weak memory order (RVWMO) barrier instructions.
*   **L1 Four-Sentence Logic**:
    1.  **Modular Specs & Privilege Control**: RISC-V defines structured general-purpose registers and a CSR-controlled multi-privilege trap handling channel, enabling hardware-level precise capture of instructions and exceptions.
    2.  **Lockstep Stalls & Branch Speculation**: The five-stage pipeline resolves data dependencies using bypass forwarding and reduces control flow jump penalties via dynamic branch predictors (BHT/BTB).
    3.  **Renaming, Reordering & In-Order Commit**: Out-of-order execution resolves false dependencies via Tomasulo reservation stations and register renaming, restoring execution order at the commit stage through a Reorder Buffer (ROB).
    4.  **Snooping Coherence & Weak Ordering**: Cache lines are kept coherent across cores via MESI snooping protocols, and RVWMO weak memory order enforces synchronization using FENCE barriers and LR/SC atomic primitives.
*   **L2 Core Data Flow Topology**:
    *   `PC fetch` ➜ `Instruction Cache read` ➜ `Decode (ID)` ➜ `Register Rename (Map Table lookup)` ➜ `Dispatch to Reservation Station (RS)` ➜ `Wakeup & Select` ➜ `Read operands from Physical Register File (PRF)` ➜ `ALU Execute` ➜ `Data Hazard? No, Bypass Forwarding activated` ➜ `Memory Load/Store Buffer queue` ➜ `L1 Data Cache look up` ➜ `Cache Miss? Yes ➜ TLB virtual to physical walk` ➜ `MESI Bus Request` ➜ `State Invalid ➜ Broadcast Read Invalid` ➜ `Fill L1 Cache Line ➜ State Modified` ➜ `Store-to-Load Forwarding` ➜ `Writeback result to ROB` ➜ `ROB in-order commit` ➜ `Commit to Architectural Register File`.

---

## 📂 M1: Instruction Set Architecture (ISA) Axioms, Privileges & Pipeline Basics (Cards 1-5)

#### Card 1. RISC-V Architecture Axioms
RISC-V redefines systems architecture through an open, modular, and minimalist instruction set design:
1.  **Modular Extension Scheme**: Unlike bloated CISC instruction sets, RISC-V centers on a tiny base integer spec (RV32I / RV64I, only ~40 instructions) and expands via standardized extension building blocks such as M (multiply/divide), A (atomic), F/D (single/double float), and C (compressed 16-bit instructions).
2.  **Minimalist Load-Store Architecture**: Only dedicated memory access instructions (e.g., `ld`/`sd`) interact with physical memory; all arithmetic and logic instructions operate strictly on general-purpose registers, decoupling address calculations from execution logic.
3.  **Regular Field Layout**: Instructions align fields (rs1, rs2, rd) in space across R/I/S/B/U/J formats, minimizing decoder hardwiring fan-in/fan-out latency.

#### Card 2. General-Purpose Registers and ABI Calling Convention
RISC-V defines a highly aligned register file to streamline compiler code generation:
1.  **Register File**: RV64I provides 32 general-purpose registers (`x0`-`x31`). Register `x0` is hardwired to zero (Hardware Zero Register); writes to it are discarded, and reads always return 0, facilitating zero-overhead `nop`, clear, and comparison operations.
2.  **ABI Calling Conventions**:
    *   `ra` (`x1`): Holds the return address (Return Address).
    *   `sp` (`x2`): Stack pointer (Stack Pointer), strictly 16-byte aligned.
    *   `gp` (`x3`) / `tp` (`x4`): Global pointer and thread pointer.
    *   `t0`-`t6` (`x5`-`x7`, `x28`-`x31`): Temporary registers (Caller-saved).
    *   `a0`-`a7` (`x10`-`x17`): Function arguments and return values (with `a0`/`a1` holding return values, Caller-saved).
    *   `s0`-`s11` (`x8`-`x9`, `x18`-`x27`): Saved registers (Callee-saved, with `s0` / `fp` acting as frame pointer).

#### Card 3. Classic Five-Stage Pipeline Architecture
Classic in-order pipelines utilize temporal parallelism to partition instruction lifecycles into overlapping stages:
1.  **Pipeline Stages**:
    *   **Instruction Fetch (IF)**: Fetches the instruction pointed to by the Program Counter (PC) from the Instruction Cache and increments PC by 4.
    *   **Instruction Decode (ID)**: Decodes the opcode, operands, and immediates, and reads source operands from the Register File in parallel.
    *   **Execute (EX)**: The Arithmetic Logic Unit (ALU) performs computation, checks branch jump conditions, or aggregates memory addresses.
    *   **Memory Access (MEM)**: Performs read/write operations on the Data Cache for memory instructions, while non-memory instructions pass through.
    *   **Write-Back (WB)**: Writes the ALU computation or retrieved Cache data back to the Register File.
2.  **Clock-Edge Triggered**: Inter-stage registers latch data on the rising edge of each CPU clock cycle, pushing clock frequencies to their limits.

#### Card 4. Control and Status Registers (CSRs) Access Control
Processor settings, privilege modes, and interrupt configurations are managed through CSRs:
1.  **Dedicated Address Space**: A separate 12-bit address space supports up to 4096 CSRs, physically isolated from general-purpose registers and protected by privilege check limits.
2.  **Atomic Read-Modify-Write Instructions**: RISC-V implements dedicated instructions to modify CSR states without race conditions:
    *   `csrrw` (CSR Read/Write): Atomically swaps a general-purpose register with a target CSR.
    *   `csrrs` / `csrrc` (Read and Set/Clear): Atomically sets or clears specific bits in a CSR based on a register mask.
    *   `csrrwi` / `csrrsi` / `csrrci`: Immediate operand variants of atomic operations.
3.  **Low-Overhead Monitoring**: Enables OS code to read system clocks and cycle counters directly without triggering expensive kernel traps.

#### Card 5. Multi-Privilege Modes & Trap Handling Paths
RISC-V secures system boundaries via layered hardware privilege rings:
1.  **Three Privilege Modes**: User (U, apps), Supervisor (S, OS kernel), and Machine (M, highest firmware/monitor mode, mandatory for all chips).
2.  **Trap Handling**: When a lower privilege mode triggers an exception or makes a system call (`ecall`):
    *   The hardware saves the current PC to the target mode's `xepc` (e.g., `mepc` / `sepc`).
    *   It logs the exception code to `xcause` and logs helper addresses to `xtval`.
    *   The privilege mode is raised, and the PC jumps to the vector entry defined in `xtvec`.
3.  **Execution Resumption**: Once the trap handler completes, the software executes `xret` (e.g., `mret`), restoring the PC from `xepc` and returning to the original privilege level.

---

## 📂 M2: Pipeline Hazards, Bypass Forwarding & Speculation (Cards 6-10)

#### Card 6. Pipeline Hazards and Hardware Stalls
A hazard occurs when subsequent instructions in an overlapping pipeline cannot execute in their designated clock cycles:
1.  **Hazard Classifications**:
    *   **Structural Hazards**: Hardware resource conflicts where multiple instructions request the same block (e.g., simultaneous memory read and write on a single-ported Cache).
    *   **Data Hazards**: Instructions depend on operand values that have not yet been written back by preceding instructions. Types include Read-After-Write (RAW), Write-After-Read (WAR), and Write-After-Write (WAW). In in-order pipelines, only RAW (true dependency) causes stalls.
    *   **Control Hazards**: The next PC address cannot be determined until branch target computations complete.
2.  **Hardware Stalls**: When the ID stage detects a RAW conflict that cannot be resolved via forwarding, the pipeline controller freezes the IF/ID registers and inserts NOP bubbles into the EX stage.

#### Card 7. Data Bypass Forwarding Mechanism
Most Read-After-Write (RAW) hazards can be resolved by forwarding values directly to execution units, bypassing register file writeback:
1.  **Bypass Paths**: Multiplexers are added to the ALU inputs. The results of the current ALU calculation (in the EX/MEM pipeline register) or data retrieved from Cache (in the MEM/WB pipeline register) are routed directly to the ALU inputs.
2.  **Forwarding Control Logic**: When the controller detects that a source register in the ID/EX stage (`rs1`/`rs2`) matches the destination register (`rd`) of an active instruction in EX/MEM or MEM/WB, it switches the input multiplexer to use the forwarded value.
3.  **Load-Use Limit**: If a Load instruction is immediately followed by an ALU instruction that requires its loaded value, the data is not ready until the end of the MEM stage. The hardware must insert a 1-cycle stall.

#### Card 8. Control Hazards and Pipeline Flushes
Branch instructions introduce control flow uncertainty into deeply pipelined microarchitectures:
1.  **Pipeline Flush**: In a standard five-stage pipeline, branch targets and directions are resolved in the EX stage (stage 3). If a branch is taken, the IF stage has already fetched two invalid instructions. The controller must zero out the control registers in the IF/ID and ID/EX stages, transforming the instructions into NOPs (flushing), which incurs a 2-cycle penalty.
2.  **Branch Delay Slot**: To eliminate this penalty, legacy architectures like MIPS executed the instruction immediately following the branch. RISC-V rejects delay slots because they complicate out-of-order execution, relying instead on dynamic branch prediction.

#### Card 9. Branch Predictors and Speculative Execution
Branch predictors guess branch directions during the fetch stage to minimize control hazard penalties:
1.  **Branch Target Buffer (BTB)**: A hardware cache queried during the IF stage. If the current PC matches an entry, the BTB outputs the previously recorded target PC, bypassing decoding and branch offset addition.
2.  **Branch History Table (BHT)**: An array of 2-bit saturating counters. States are: 00 (strongly not taken), 01 (weakly not taken), 10 (weakly taken), and 11 (strongly taken). The counter increments or decrements based on the actual branch outcome, preventing single loop-exit transitions from misaligning predictions.
3.  **gshare Predictor**: XORs the Global Branch History Register (GBHR) with the PC to index the BHT, capturing conditional correlation across multiple branches.

#### Card 10. Precise Exception Handling and Rollback
Processors must protect register states during exceptions (e.g., page faults) or external interrupts:
1.  **Precise Exception Definition**:
    *   All instructions preceding the exception instruction $I_p$ must execute and commit their register changes.
    *   The exception instruction $I_p$ and all subsequent instructions in the pipeline must be flushed, leaving no trace in the register file.
    *   The PC is saved to `xepc` (e.g., `mepc`), pointing back to $I_p$ for execution resumption after the trap handler finishes.
2.  **Hardware Flags**: Exceptions are flagged in registers at each stage and propagated down the pipeline. The exception is only handled when the offending instruction reaches the writeback/commit boundary, ensuring precise state capture.

---

## 📂 M3: Out-of-Order (OoO) Execution, Renaming & Scheduling (Cards 11-15)

#### Card 11. Out-of-Order Execution and the Tomasulo Algorithm
In-order cores halt execution for long-latency instructions like Cache misses, stalling unrelated downstream instructions. Out-of-order execution resolves this:
1.  **Tomasulo Algorithm**: Tracks operand dependencies using Reservation Stations (RS) and broadcasts results over a Common Data Bus (CDB).
2.  **Reservation Stations (RS)**: Decoded instructions enter an available RS slot. If an operand is ready in the register file, it is copied to the RS; if not, the RS records the tag of the virtual execution unit (or physical register) that will produce the value.
3.  **CDB Broadcast**: When an execution unit completes a calculation, it broadcasts the result and its tag over the CDB. RS slots listen to the bus, grabbing values matching their pending tags. Once all operands are ready, the instruction is dispatched for execution.

#### Card 12. Register Renaming Mechanism
Register renaming eliminates false data dependencies to enable out-of-order execution:
1.  **Eliminating False Dependencies**: WAR (Write-After-Read) and WAW (Write-After-Write) hazards are names conflicts caused by compiler register reuse, not true dataflow.
2.  **Physical Register File (PRF) and Map Table**:
    *   The hardware includes a large PRF (e.g., 128 physical registers) mapped to 32 architectural registers (x0-x31).
    *   **Renaming Process**: For every write destination (`rd`), the decoder pulls an unused physical register $P_k$ from the Free List and maps the logical register `rd` to $P_k$ in the Map Table. Subsequent instructions reading this register are redirected to $P_k$.
3.  **Dependency Decoupling**: Renaming breaks name conflicts, allowing independent instruction streams to execute concurrently.

#### Card 13. Instruction Dispatch and Issue Queue Scheduling
Renamed instructions are placed in the Issue Queue (IQ) or Reservation Stations to wait for dispatch:
1.  **Wakeup-Select Loop**:
    *   **Wakeup Phase**: In each cycle, the CDB broadcasts completed physical tags. The IQ matches these tags against the source operands of pending instructions, marking ready operands.
    *   **Select Phase**: The scheduler picks ready instructions based on priority (e.g., Oldest First) and dispatches them to execution units.
2.  **Critical Path Bottleneck**: The Wakeup-Select loop is a tight feedback loop. Its gate delay scales quadratically with dispatch width (e.g., 4-wide, 8-wide), representing a major limitation on CPU clock speeds.

#### Card 14. Reorder Buffer (ROB) and In-Order Commit
Out-of-order execution writes results to registers out of order, which complicates precise exceptions and branch misprediction recovery. The ROB restores program order:
1.  **In-Order Commit**:
    *   As instructions decode, they are allocated entries in the ROB in program order (using a FIFO queue).
    *   When an instruction executes, its result is written to its ROB entry and marked ready, but **it does not modify the architectural register file (ARF)**.
    *   Only when the instruction at the head of the ROB (the oldest instruction) is marked ready does the processor commit it, writing the result to the ARF.
2.  **State Recovery**: If a branch misprediction or exception occurs at the ROB head, the processor flushes all subsequent entries in the queue and rolls back the Map Table.

#### Card 15. Memory Disambiguation and Store-to-Load Forwarding
Memory instructions can have overlapping addresses, meaning their execution order must be strictly managed:
1.  **Store Buffer Isolation**: To prevent branch mispredictions from corrupting memory, Store instructions write their target addresses and data to a Store Buffer instead of Cache. They only write to Cache at the commit stage.
2.  **Store-to-Load Forwarding**: When a Load executes, its physical address is matched against the Store Buffer and Cache in parallel. If a pending Store matches the Load address, the data is forwarded directly from the Store Buffer to the Load, bypassing Cache.
3.  **Load Queue Verification**: If a Load executes out of order and reads stale data before a preceding conflicting Store address is calculated, the Load Queue flags the conflict and flushes the pipeline.

---

## 📂 M4: Cache Hierarchy, Cache Coherence & Address Translation (Cards 16-19)

#### Card 16. Cache Set-Associative Structure
Cache architectures exploit program locality to bridge the latency gap between CPU execution and main memory:
1.  **Set-Associative Cache**: Caches are partitioned into $S$ sets, each containing $N$ cache lines (or ways, e.g., 4-way/8-way).
2.  **Physical Address Parsing**: Addresses are parsed into three fields:
    *   **Offset**: Locates bytes within a cache line (e.g., 64-byte width).
    *   **Index**: Selects the target Set.
    *   **Tag**: Holds the physical page frame number.
3.  **Hit Verification**: The MMU hashes the address Index to select a Set and compares the physical Tag against all $N$ ways in parallel. If a matching tag is valid, the cache hit is resolved.
4.  **Replacement Policy**: If a set is full, the cache controller evicts a line using algorithms like Least Recently Used (LRU).

#### Card 17. Cache Write Policies and Coherence Bottlenecks
Cache write strategies dictate data sync policies and balance bus traffic overhead:
1.  **Write Strategies**:
    *   **Write-Through**: Every write operation updates both the cache line and main memory. While simple and safe, it consumes bus bandwidth and stalls execution on memory writes.
    *   **Write-Back**: Writes update only the local L1 Cache, marking the line as dirty (Dirty bit = 1). The line is only written back to main memory when it is evicted.
2.  **Coherence Conflicts**: In multi-core systems with private L1 Caches, if Core 0 modifies variable $X$ in its L1 and Core 1 reads its own stale copy of $X$ from its private L1, it causes cache inconsistency.

#### Card 18. MESI Cache Coherence Protocol
The MESI protocol maintains cache coherence across cores via bus snooping:
1.  **Four Cache States**:
    *   **Modified (M)**: The line is dirty. It exists only in the current core's cache and contains data that differs from main memory.
    *   **Exclusive (E)**: The line matches main memory and exists only in the current core's cache.
    *   **Shared (S)**: The line matches main memory and may exist in other cores' caches.
    *   **Invalid (I)**: The line's data is stale and unreadable.
2.  **Bus Snooping**: When Core 0 writes to a Shared line, it broadcasts a invalidate signal (BusUpgrade / BusRdX) on the bus. The snoop controllers of other cores intercept this signal and mark their local copies as Invalid.

#### Card 19. Address Translation and TLB Miss Handling
Virtual-to-physical address translation requires hardware acceleration to minimize performance penalties:
1.  **Page Table Walker (PTW)**: In SV39 (three-level page tables), translating an address requires querying three levels of page tables in memory, which adds translation overhead to memory operations.
2.  **Translation Lookaside Buffer (TLB)**: A high-speed cache inside the MMU that stores recently translated virtual-to-physical address mappings.
3.  **PTW Miss Walking**: When a TLB miss occurs, the PTW walks the page tables in memory starting from the root page table (in the `satp` register). If a Page Table Entry (PTE) is invalid or violates read/write/execute permissions, the PTW triggers a Page Fault Trap; otherwise, it loads the translation into the TLB.

---

## 📂 M5: Memory Consistency Models & Concurrency Primitives (Cards 20-24)

#### Card 20. Memory Consistency Models
Memory consistency models define the execution order and visibility of reads and writes across different memory locations:
1.  **Coherence vs. Consistency**: Cache coherence ensures multiple copies of a single memory location match; memory consistency defines order rules across distinct memory locations.
2.  **Sequential Consistency (SC)**: SC requires all cores to see a single, globally interleaved sequence of reads and writes that matches the Program Order (PO) of each individual core.
3.  **Reordering Trade-offs**: To maximize performance, processors use write buffers and out-of-order execution, which violates SC. Maintaining SC would require stalling execution on every memory access. Modern architectures resolve this using weak memory models.

#### Card 21. RISC-V Weak Memory Order (RVWMO)
RISC-V adopts a relaxed memory model (RVWMO) to allow aggressive pipeline optimizations:
1.  **Instruction Reordering**: RVWMO allows the processor to reorder memory accesses to different addresses, provided it does not violate single-thread data or control dependencies:
    *   Read-Read, Read-Write, Write-Write, Write-Read.
2.  **Write Buffer Effects**: Writes by Thread A may sit in its local Store Buffer before being committed to main memory, making them temporarily invisible to Thread B.
3.  **Software Synchronization**: This model shifts the responsibility of ordering memory accesses to software, which must use explicit barrier instructions.

#### Card 22. FENCE and FENCE.I Synchronization
Software inserts barrier instructions to enforce memory ordering when required by application logic:
1.  **FENCE Instruction**: Syntax is `fence predecessor, successor`, where predecessor and successor specify combinations of memory operations: `I` (input/read), `O` (output/write), `R` (read), and `W` (write). E.g.:
    `fence r, rw`
    Enforces that all reads preceding the fence must complete before any reads or writes following the fence can execute.
2.  **FENCE.I Instruction**: Synchronizes the Instruction Cache (I-Cache) and Data Cache (D-Cache). For self-modifying code (e.g., JIT compilers writing code to data pages), executing `fence.i` flushes the I-Cache to ensure subsequent fetches load the newly written instructions from memory.

#### Card 23. Atomic Memory Operations (AMO)
Atomic operations provide hardware-level primitives to build mutual exclusion locks and lock-free queues:
1.  **AMO Instructions**: Syntax is `amoadd.w rd, rs2, (rs1)`. These instructions perform a read-modify-write operation directly in memory or cache without loading data into general-purpose registers.
2.  **Atomic Execution Paths**:
    *   **Cache Hit**: If the target cache line is Modified (M), the core updates it in its private L1 Cache while locking the line's MESI state, preventing other cores from modifying it.
    *   **Cache Miss**: The bus controller locks the bus, preventing other cores from accessing memory until the read-modify-write operation completes.

#### Card 24. Load-Reserved / Store-Conditional (LR/SC)
LR/SC instructions provide an alternative primitive to construct atomic operations (like Compare-And-Swap):
1.  **lr.w rd, (rs1) (Load-Reserved)**: Loads a value from address `rs1` into `rd` and registers a reservation on the target address in the core's hardware monitor.
2.  **sc.w rd, rs2, (rs1) (Store-Conditional)**: Conditionally writes `rs2` to address `rs1`.
    *   **Success Rule**: The store succeeds if no other core has written to the reserved address since the `lr.w` instruction, and no interrupts or context switches have occurred. It writes `0` to `rd`.
    *   **Failure Rule**: If the reservation is lost, the store fails, and the instruction writes a non-zero error code to `rd` without updating memory.
3.  **Optimistic Concurrency**: Provides a lock-free mechanism that avoids the ABA problem.

---

## 📂 M6: Vector Extensions (RVV) & Hardware Debugging (Cards 25-28)

#### Card 25. RISC-V Vector Extension (RVV) Axioms
Vector extensions accelerate data-parallel workloads (e.g., AI inference and image processing):
1.  **Vector Length Agnostic (VLA)**: Unlike SIMD architectures (e.g., Intel AVX-512, ARM NEON) that hardcode register widths in instructions, RVV instructions do not specify vector widths. Width is determined by the hardware (VLEN, e.g., 128 to 65536 bits), ensuring binary compatibility across chips.
2.  **Register File**: RVV defines 32 vector registers (`v0`-`v31`).
3.  **Portability**: The same RVV binary can run on a high-throughput vector supercomputer or a low-power IoT SoC, adapting to the target hardware's vector width.

#### Card 26. Vector Configuration and Execution Modes
RVV allows software to configure vector parameters dynamically to optimize execution:
1.  **vsetvli Instruction**: Configures vector execution parameters for a loop:
    *   **SEW (Selected Element Width)**: Sets the width of individual elements (8, 16, 32, or 64 bits).
    *   **LMUL (Vector Register Grouping)**: Groups multiple vector registers into single logical registers (e.g., `LMUL=8` groups `v0-v7` to increase instruction throughput).
2.  **Vector Length Control (`vl`)**: The hardware calculates the maximum elements processed per instruction (`vl`) based on SEW, LMUL, and VLEN. The loop increments by `vl` on each iteration.

#### Card 27. Hardware Performance Monitors and Debugging
On-chip monitoring and debugging primitives are essential for microarchitectural tuning:
1.  **Hardware Performance Monitors (HPM)**: Read-only registers in the CSR space that track core events. The three base counters are:
    *   `cycle`: Counts CPU clock cycles since reset.
    *   `time`: Tracks real-time clock cycles.
    *   `instret` (Instructions Retired): Counts committed instructions, excluding flushed instructions.
2.  **CPI Calculation**: Measuring `cycle` and `instret` across code segments yields the Cycles Per Instruction (CPI).
3.  **Hardware Debugging**: The JTAG interface provides access to TAP controllers for boundary scans, breakpoints, register inspection, and single-step execution.

#### Card 28. Spike and QEMU Instruction Set Simulators
Before silicon tape-out, software stack development and ISA verification rely on simulators:
1.  **Spike Simulator**: The official golden model simulator. It interprets RISC-V instructions sequentially, maintaining precise CSR, page table, and privilege states for ISA verification.
2.  **QEMU Dynamic Translation (TCG)**: Spike is accurate but slow. QEMU uses JIT compilation (Tiny Code Generator) to translate RISC-V instruction blocks into host instructions (e.g., x86_64), running fast enough to boot Linux and run large software packages.

---

## 📂 RISC-V Pipeline Architecture & Performance Trade-off Matrix

| Design Dimension | Option A (In-Order Five-Stage Pipeline - e.g., Rocket Core) | Option B (Out-of-Order Superscalar Pipeline - e.g., BOOM Core) | Trade-off Balance |
| :--- | :--- | :--- | :--- |
| **Pipeline Scheduling Policy** | In-order execution with lockstep stalls (In-order Execution & Stall) | Out-of-order execution with Tomasulo algorithm (Out-of-order Execution & RS/ROB) | In-order cores require simple bypass forwarding to resolve RAW hazards ➜ but stall on Cache misses. Out-of-order cores bypass false dependencies via reservation stations and renaming to exploit instruction-level parallelism (ILP) ➜ but require complex Map Tables, Issue Queues, and ROBs, increasing silicon area and power consumption [In-order pipelines are suited for low-power embedded IoT; out-of-order designs are preferred for high-performance servers]. |
| **Register File & Renaming** | Direct architectural register access (32 registers, direct addressing) | Physical register renaming (Map Table & 128+ physical register file) | Direct register reads require no renaming logic, completing in 1 cycle with minimal latency ➜ but restrict out-of-order scheduling. Renaming resolves WAR/WAW conflicts via spatial registers ➜ but Map Table queries and Free List management add silicon area and timing constraints [Use direct register access for low-latency, low-power cores; use renaming for high-IPC out-of-order cores]. |
| **Branch Prediction Control** | Static or simple 2-bit counter prediction (Static/2-bit Predictor) | Global history correlation with gshare (Global History & gshare Predictor) | 2-bit counters require minimal hardware overhead ➜ but suffer high misprediction rates in complex loops, causing pipeline flushes. gshare tracks execution history to achieve >95% accuracy ➜ but the history logic and XOR gates add fetch-stage latency [Use 2-bit branch predictors for short in-order pipelines; use gshare or TAGE predictors for deep out-of-order pipelines]. |
| **Memory Sync Model** | Write-through or write-back L1 caches (Write-Through / Write-Back L1) | Store Buffer with Store-to-Load Forwarding (Store Buffer & Forwarding) | In-order memory access requires no Store Buffers and simplifies address checking ➜ but cannot hide memory latency. Out-of-order memory access buffers stores to isolate cache before commit and forwards values to loads, hiding latency ➜ but requires complex load-queue collision checks to prevent data corruption [Deploy forwarding for data-intensive processing cores; use simple write-back caches for standard MCUs]. |

---

## 🔬 Zone T: Assembly Reference, Control Registers & Exceptions Reference

### T1: Core RISC-V Assembly Instructions and Privilege CSR Operations
*   `csrrw t0, mepc, x0` : Atomic Read/Write. Reads the Machine Exception Program Counter `mepc` into register `t0` and writes `x0` (0) to `mepc` (reading and clearing the CSR).
*   `csrrs t1, mstatus, t2` : Atomic Read and Set. Reads the Machine Status register `mstatus` into `t1` and sets bits in `mstatus` that are high in `t2` (commonly used to enable global interrupts).
*   `lr.w x5, (x10)` : Load-Reserved. Loads a 4-byte word from the memory address in `x10` into `x5` and registers a reservation on that address in the core's hardware monitor.
*   `sc.w x6, x7, (x10)` : Store-Conditional. Writes `x7` to address `x10` if the reservation on `x10` is still valid. On success, writes `0` to `x6`; on failure, discards the store and writes a non-zero error code.
*   `vsetvli t0, a0, e32, m8, ta, ma` : Vector Configuration. Configures vector parameters (32-bit element width `e32`, register grouping factor `m8`) for a maximum elements count `a0`, returning the active vector length to `t0`.

### T2: Processor Exception Codes and Trap Control Dictionary
When a trap occurs, the Machine Cause register `mcause` (or `scause` in Supervisor mode) records the exception code. Below is a dictionary of core Exception Codes:
*   **Exception Code = 0 (Instruction Address Misaligned)** : Instruction fetch address misalignment. Triggered if the PC jumps to an address that is not aligned to a 2/4-byte boundary.
*   **Exception Code = 2 (Illegal Instruction)** : Decodes an invalid opcode, or attempts to execute a privileged instruction at an insufficient privilege level (e.g., executing `mret` in User mode).
*   **Exception Code = 3 (Breakpoint)** : Triggered by the `ebreak` instruction, commonly used by debuggers to halt execution.
*   **Exception Code = 5 (Load Address Misaligned) / 6 (Store/AMO Address Misaligned)** : Memory address misalignment. Triggered when executing a memory load or store to an unaligned address on cores that do not support unaligned accesses.
*   **Exception Code = 8 (Environment Call from U-mode)** : User-mode system call. Executing `ecall` in User mode shifts execution to Supervisor mode (OS kernel) to handle system services.
*   **Exception Code = 12 (Instruction Page Fault) / 13 (Load Page Fault) / 15 (Store/AMO Page Fault)** : Page fault exceptions. Triggered when the PTW walks page tables for translation and encounters an invalid PTE (Valid=0) or a read/write/execute permission violation.
