# Rocket Chip RISC-V SoC Generator & TileLink Bus Cheatsheet

## J-Ladder Hierarchical Model

### L0 One-Line Essence
Rocket Chip is a Scala/Chisel-based parameterized SoC generator that negotiates hardware specifications via diplomacy graphs and emits RISC-V RTL code featuring TileLink cache coherency.

### L1 Four-Sentence Logic
1. **Diplomacy Parameter Negotiation**: Resolves bit-widths, address ranges, and interrupt mappings automatically on directed graphs before physical hardware instantiation.
2. **TileLink MSI Coherency**: Enforces hardware cache coherency across cores and L2 interfaces using TileLink's decoupled A-E physical channel state transitions.
3. **Non-Blocking Cache & MSHR**: Utilizes Miss Status Holding Registers (MSHR) to track pending D-Cache misses, keeping the CPU pipeline running for independent instructions.
4. **Hardware PMP Isolation & RoCC**: Enforces hard physical memory protections (PMP) at the pipeline boundary while offering RoCC interfaces for custom coprocessors.

### L2 Core Data Flow
`Chisel Source` ➜ `Scala Instantiation (Diplomacy)` ➜ `AST Emission` ➜ `FIRRTL Compile` ➜ `Verilog RTL Emit` ➜ `SoC Boot` ➜ `CPU Memory Load (D$)` ➜ `Cache Miss` ➜ `TileLink Channel A Request` ➜ `L2 Arbitration` ➜ `TileLink Channel D Grant` ➜ `MSHR Resolution` ➜ `Resume Pipeline`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: Chisel Language Hardware Modeling
*   **Theory**: Generates structural RTL circuits via Scala-embedded object-oriented abstractions.
*   **Details**: Chisel compiles hardware builders into FIRRTL intermediate trees. It represents physical wires, registers, and ports using typed Scala values.
*   **Trade-off**: Chisel is structural RTL, not software HLS. Designers must map syntax constructs directly to physical gate-level structures.

### Card 2: Configuration & Parameterization System
*   **Theory**: Dynamically builds sub-systems using Scala Traits (Cake Pattern) and implicit parameters context.
*   **Details**: Passes global parameters (e.g. cache configurations, CPU features) down module trees via implicit Parameter objects. Builds subsystems by mixing Traits.
*   **Trade-off**: Highly complex Scala type systems; parameter collisions (e.g., overlapping address maps) cause compile-time aborts rather than runtime faults.

### Card 3: Rocket Core 5-Stage Pipeline
*   **Theory**: In-order execution featuring IF/ID/EX/MEM/WB stages and bypass logic.
*   **Details**: Implements an in-order pipeline, resolving RAW dependencies using EX-to-ID bypassing.
*   **Trade-off**: Simple, low-power design, but stalls on long-latency events (e.g., FPU or Cache Misses).

### Card 4: FPU & MMU Virtual Memory Translation
*   **Theory**: IEEE-754 hardware floating-point and SV39 Page Table Walker with TLBs.
*   **Details**: Implements SV39 virtual memory translation featuring private ITLBs/DTLBs and hardware page table walkers that traverse memory pages on TLB misses.
*   **Trade-off**: Hardware page walking accelerates translation but increases core size and control logic complexity.

### Card 5: Instruction Cache & ITim
*   **Theory**: Single-port RAM instruction cache with prefetching and tightly-integrated memory.
*   **Details**: Employs single-port SRAM with prefetch buffers for zero-latency instruction dispatch. ITim maps cache regions to physical scratchpads for deterministic timing.
*   **Trade-off**: Single-port RAM is compact but halts CPU fetches during cache line refilling.

### Card 6: Data Cache & MSHR Architecture
*   **Theory**: Non-blocking HellaCache using Miss Status Holding Registers (MSHR).
*   **Details**: HellaCache handles D-Cache misses by recording pending transactions in MSHRs and issuing bus requests, allowing independent CPU execution.
*   **Trade-off**: MSHRs increase instruction throughput but require complex state tracking logic and consume extra area.

### Card 7: TileLink Five-Channel Model
*   **Theory**: Decoupled physical channels A, B, C, D, E for request/response and coherency.
*   **Details**: Channel A handles Master requests, B tracks Slave probes, C routes L1 releases/writebacks, D transmits Slave responses, and E acknowledges Grants.
*   **Trade-off**: Eliminates cyclic channel dependencies to prevent deadlocks, but increases physical routing overhead.

### Card 8: MSI Cache Coherency Controller
*   **Theory**: L1 MSI state machine transitions driven by L2 probe/release routing.
*   **Details**: When a core requests write access (Exclusive), the L2 controller broadcasts Probes (channel B) to other cores, which downgrade and release lines (channel C).
*   **Trade-off**: Guarantees cache consistency across cores but introduces massive probe bus overhead under high contention.

### Card 9: SystemBus Subsystem Interconnect
*   **Theory**: Diplomacy-negotiated subsystem bus routing topologies.
*   **Details**: Subsystem networks (SystemBus, PeripheralBus, ControlBus) are mapped using Diplomacy nodes, compiling into optimized bus decoders.
*   **Trade-off**: Automates correct connectivity but adding bus hierarchies increases latency for peripheral I/O access.

### Card 10: Bus Adapters & Cross-Clock Bridges
*   **Theory**: Protocol conversion, width adjustment, and asynchronous boundary crossings.
*   **Details**: Translates internal TileLink signals to AXI4/APB formats. Manages width widgets and clock domain crossings using asynchronous synchronization registers.
*   **Trade-off**: Bridges provide broad IP compatibility but introduce 2~4 clock cycles of synchronization delay.

### Card 11: Crossbars & Bus Arbitration
*   **Theory**: Address decoders and multiplexer arbiters with deadlock-free queuing.
*   **Details**: TLXbar generates bus crossbars containing address decoders and priority arbiters (e.g. Round-Robin), using FIFO buffers to prevent blocking.
*   **Trade-off**: Arbitration choices impact core latency. Large crossbars complicate physical wire routing in backend layouts.

### Card 12: Memory Mapped I/O (MMIO) Registers
*   **Theory**: Automatically generates address decoding and register maps.
*   **Details**: Peripherals implementing `RegMapper` register ranges during Diplomacy parameter negotiation, compiling into collision-free address decoders.
*   **Trade-off**: Address alterations require rebuilding Verilog from Scala configs.

### Card 13: JTAG Debug Module
*   **Theory**: Implements RISC-V Debug spec 0.13 via JTAG and DMI.
*   **Details**: Resolves JTAG shift streams into Debug Module Interface (DMI) register read/write commands, allowing debuggers to halt cores.
*   **Trade-off**: Debug modules are crucial for physical testing but must be physically fused or locked to prevent hardware backdoors.

### Card 14: BootROM & Reset Logic
*   **Theory**: Fixed reset vector execution and devicetree blob (DTB) setup.
*   **Details**: On reset, the PC jumps to the BootROM. BootROM code passes the DTB to memory and invokes OpenSBI.
*   **Trade-off**: Hardcoded reset vectors are robust but lock boot sequences; upgrades must rely on secondary software bootloaders.

### Card 15: PLIC & CLINT Interrupt Controllers
*   **Theory**: Aggregates external interrupts and manages core timer ticks.
*   **Details**: CLINT handles timer/software interrupts. PLIC routes external signals through an arbitration tree based on priority thresholds.
*   **Trade-off**: Centralized PLIC is easy to route, but multi-tile systems face routing congestion, requiring decentralized CLIC approaches.

### Card 16: FIRRTL Compiler Infrastructure
*   **Theory**: Lowers Chisel AST structures to low-level Verilog code.
*   **Details**: FIRRTL compiles high-level Chisel trees through High, Middle, and Low compiler passes, applying optimizations before emitting Verilog.
*   **Trade-off**: Multi-pass lowering optimizes netlists but lengthens compile times for large SoC designs.

### Card 17: Verilog Codegen & Simulation Harness
*   **Theory**: Emits synthesis-ready Verilog and compiles Harness programs.
*   **Details**: Generates synthesizable Verilog. Verilator translates this to C++ classes, and the simulation harness injects software binaries into memory blocks.
*   **Trade-off**: Speeds up verification but software simulation runs 6~7 orders of magnitude slower than physical hardware.

### Card 20: RoCC Coprocessor Interface
*   **Theory**: Integrates custom accelerators using custom0-3 opcodes.
*   **Details**: Under custom0-3 opcodes, the ID stage dispatches commands and registers to the RoCC bus, halting execution until the coprocessor returns data.
*   **Trade-off**: Simple accelerator integration, but coprocessors share L1 D$ and MMU resources, making debugging difficult if conflicts occur.

### Card 22: Physical Memory Protection (PMP)
*   **Theory**: Enforces physical address access checks at the memory interface.
*   **Details**: Compares address registers during the MEM stage, raising exceptions (Access Faults) on unauthorized reads, writes, or executes.
*   **Trade-off**: Critical for security but parallel address comparators add combinational logic delay.

---

## 🔬 Zone T1: Rocket Chip Core APIs
*   `freechips::rocketchip::rocket::RocketCore`: Declares the 5-stage Rocket CPU pipeline module.
*   `freechips::rocketchip::tilelink::TLBundle`: Interfaces definition for TileLink's A, B, C, D, E physical lines.
*   `freechips::rocketchip::subsystem::BaseSubsystem`: System coordinator tracking tiles list and diplomacy bindings.
*   `freechips::rocketchip::config::Parameters`: Parameter wrapper passed implicitly across compilation steps.
*   `freechips::rocketchip::tile::LazyRoCC`: Base class defining custom coprocessors and D-Cache attachments.
*   `freechips::rocketchip::diplomacy::LazyModule`: Diplomacy coordinator separating parameter negotiation from instantiation.
*   `freechips::rocketchip::devices::tilelink::PLIC`: Generates PLIC arbitration and priority threshold registers.

## 🔬 Zone T2: Compilation & Execution Errors
*   `tilelink_protocol_deadlock`: Occurs when total lockups in the A-E channels halt total bus operations.
*   `pipeline_hazard_data_stall`: Occurs when pipeline hazard bypass fails or long-latency memory writes stall execution.
*   `cache_coherency_mismatch`: Occurs when timing faults allow multiple tiles to modify the same cache address.
*   `pmp_access_violation`: Triggered when memory accesses violate PMP register privileges.
*   `firrtl_lowering_compilation_error`: Raised during FIRRTL compile if ports are left open or double-driven.
*   `debug_module_jtag_timeout`: Triggered when JTAG connections fail or DTM response times out.
