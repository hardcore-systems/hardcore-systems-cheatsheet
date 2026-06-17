# CUDA Parallel Computing & GPGPU Programming High-Density Cheatsheet

---

## 🪜 J-Ladder Knowledge Progression Hierarchy

*   **L0 Essence**: The essence of GPU parallel computing is to achieve high-throughput data parallel processing and tensor computation acceleration through massive lightweight threads mapping under a 3D grid-block-thread topology, executing on SIMT (Single Instruction Multiple Threads) hardware architecture coupled with high-bandwidth memory (HBM/GDDR).
*   **L1 Four-Sentence Logic**:
    1. **Grid Topology Mapping**: The software logical layer abstracts computation tasks into Grid and Block, where millions of lightweight threads are mapped to 1D/2D physical data arrays in a 3D coordinate system via `threadIdx` and `blockIdx`.
    2. **SIMT Hardware Scheduling**: The hardware execution layer maps thread Blocks to Streaming Multiprocessors (SMs), where thread warps (32 threads) serve as the basic physical scheduling and instruction issue unit.
    3. **High-Throughput Storage Hierarchy**: A multi-tiered high-throughput memory hierarchy consisting of thread-private registers, Block-shared memory (SRAM), and Global Memory (GDDR/HBM) is constructed, optimized via coalesced global memory access and bank-conflict-free shared memory access.
    4. **Async Pipeline Concurrency**: Intra-block thread synchronization is enforced via `__syncthreads()`, while tasks and memory copies are overlapped asynchronously using CUDA Streams and CUDA Graphs to maximize hardware utilization.
*   **L2 Core Data Flow Topology**:
    `CPU Host Memory` ➜ (PCIe / NVLink Copy HtoD) ➜ `GPU Global Memory` ➜ (Grid Launch) ➜ `SM Cache / Registers` ➜ (Shared Memory Intra-Block Reduction) ➜ `Warp Registers for Arithmetic (FMA / Tensor Core)` ➜ (Shared Memory / Global Memory Writeback) ➜ (PCIe / NVLink Copy DtoH) ➜ `CPU Host Memory Result`

---

## 📂 GPGPU Performance Design & Memory Access Trade-off Matrix

| Design Dimension | Mainstream Mode A (Global Memory Coalesced) | Alternative Mode B (Shared Memory Tile) | Extreme Mode C (Warp Shuffle Direct) | Physical Constraint & Bottom Trade-off |
| :--- | :--- | :--- | :--- | :--- |
| **Memory Bandwidth** | Low to Moderate (100 - 300 GB/s) | High (1.5 - 2.5 TB/s) | Extremely High (Intra-register swap, zero bus latency) | Physical memory bus pins and on-chip SRAM bandwidth limits. Global Coalesced is still bound by L2 throughput; Shared Memory degrades under Bank Conflicts. |
| **On-chip Memory Overhead** | 0 Bytes (No explicit SRAM requested) | 16KB - 96KB / Block | 0 Bytes (Directly reuse general Register File) | On-chip SRAM size per SM is physically fixed (96KB-228KB). Mode B over-requesting will heavily limit active Blocks per SM, causing Occupancy to drop. |
| **Sync Overhead** | None (Independent thread execution) | Moderate (Requires `__syncthreads()` barrier) | Extremely Low (Only warp-level sync) | Synchronization barriers freeze pipeline instruction issue and drain executing execution units, hindering latency hiding; Warp-level synchronization has negligible overhead. |
| **Development Complexity** | Extremely Low (Simple 1D/Multi-D pointer index) | Moderate to High (Requires manual tiling, collaborative load, sync) | High (Requires bitwise shifts, masks, and warp alignment) | The lower the software abstraction layer, the more critical the register allocation and Warp alignment become, demanding complex mask and indexing logic. |

---

## 🗂️ 28 Core Knowledge Cards Dictionary

### M1: CUDA Programming Model & Grid Topology (M1.1 - M1.5)

#### Card 1. Grid-Block-Thread Hierarchy Model
*   **Core Definition**: The CUDA software programming model organizes threads hierarchically into a Grid of thread Blocks. Each Block contains up to 1024 threads, sharing on-chip memory and supporting barrier synchronization.
*   **Bottom Mechanism**: Thread indices map to physical data coordinates. The 1D mapping index formula is: `int tid = blockIdx.x * blockDim.x + threadIdx.x;`. For 2D spatial layouts, coordinates are mapped using: `int x = blockIdx.x * blockDim.x + threadIdx.x; int y = blockIdx.y * blockDim.y + threadIdx.y;`. This model governs GPU execution mapping.

#### Card 2. Kernel Device Life Cycle
*   **Core Definition**: A Kernel is a parallel function declared to execute on the GPU device, launched asynchronously from the host CPU.
*   **Bottom Mechanism**: When a CPU thread calls a Kernel with configuration parameters `<<<grid, block, sharedMem, stream>>>`, the launch request is pushed into a hardware command queue (CUDA Stream). The CPU returns immediately, while the GPU hardware scheduler performs Grid Launch to dispatch Blocks.

#### Card 3. Streaming Multiprocessor (SM) Physical Architecture
*   **Core Definition**: The Streaming Multiprocessor (SM) is the fundamental hardware block of the GPU's processing power.
*   **Bottom Mechanism**: Each SM houses multiple SPs (Single Precision CUDA Cores), DP Cores (Double Precision), SFUs (Special Function Units for trans transcendental math like sin/cos), and a massive Register File (RF). The warp scheduler inside the SM dispatches instruction packets to execution units.

#### Card 4. SIMT Architecture & Warp Scheduling Logic
*   **Core Definition**: Single Instruction Multiple Threads (SIMT) is the hardware execution model where the GPU manages and executes threads in groups of 32 called Warps.
*   **Bottom Mechanism**: A Warp is the smallest execution and scheduling unit in the SM. The 32 threads in a Warp share a single Program Counter (PC) and execute the same instruction in lockstep. The Warp Scheduler switches active Warps instantly to hide execution stalls.

#### Card 5. Warp Divergence Mitigation
*   **Core Definition**: Warp Divergence occurs when threads in a single Warp branch to different execution paths due to conditional statements (`if-else`).
*   **Bottom Mechanism**: Because threads in a Warp share the same instruction pipeline, different execution branches must be serialized. The hardware uses an Active Mask register to disable non-participating threads for each branch. Developers must align branch boundaries to 32-thread intervals to avoid serial stalls.

---

### M2: GPU Hardware Architecture & Warp Scheduling (M2.1 - M2.5)

#### Card 6. GPU Latency Hiding & Occupancy Optimization
*   **Core Definition**: Occupancy is the ratio of active Warps per SM to the maximum theoretical active Warps supported by the hardware.
*   **Bottom Mechanism**: GPUs lack complex branch predictors or massive caches; instead, they rely on zero-overhead context switching. When a Warp stalls (e.g., waiting for global memory), the scheduler instantly switches to a ready Warp, hiding latency. High occupancy ensures a large pool of ready Warps.

#### Card 7. GPU Memory Hierarchy & Latency
*   **Core Definition**: The memory hierarchy ranges from registers, local memory, shared memory, constant memory, texture memory, L1/L2 caches, to global memory.
*   **Bottom Mechanism**: Registers feature 1-cycle access; on-chip Shared Memory takes tens of cycles; Global Memory takes 400 - 800 cycles. High-performance CUDA design centers on staging data from global memory into high-speed on-chip SRAM (Shared Memory) for reuse.

#### Card 8. Global Memory Coalescing
*   **Core Definition**: When threads in a Warp access global memory, the hardware merges 32 memory requests into a single or minimum number of memory transactions.
*   **Bottom Mechanism**: Memory controllers fetch data in aligned segments of 32, 64, or 128 bytes. If thread accesses are contiguous and aligned, one transaction serves the Warp. Uncoalesced accesses force up to 32 separate transactions, bottlenecking the memory bus.

#### Card 9. Shared Memory & 32-Bank Organization
*   **Core Definition**: Shared Memory is a fast on-chip SRAM allocated per Block, structured physically into 32 independent memory banks.
*   **Bottom Mechanism**: Successive 32-bit (or 64-bit) words are mapped cyclicly to Bank 0 through Bank 31. If the 32 threads of a Warp access 32 distinct banks in the same cycle, the access executes in parallel. This bank architecture yields multi-terabyte-per-second aggregate bandwidth.

#### Card 10. Shared Memory Bank Conflict & Broadcast
*   **Core Definition**: A Bank Conflict occurs when multiple threads in a Warp access different addresses within the same memory bank in the same cycle.
*   **Bottom Mechanism**: Accesses to the same bank must be serialized (an n-way conflict increases latency by n times). Exception: if all threads in a Warp access the *same* address in a bank, the hardware performs a Broadcast, serving all threads in 1 cycle without conflict. Use odd strides to avoid conflicts.

---

### M3: CUDA Storage Hierarchy & Cache Tuning (M3.1 - M3.5)

#### Card 11. Register Spilling Overhead
*   **Core Definition**: Register Spilling occurs when a thread requests more local variables than the hardware register allocation limit per thread.
*   **Bottom Mechanism**: The compiler forces excess variables into Local Memory (physically located in Global Memory). Although cached in L1, Local Memory access is hundreds of times slower than register access, leading to severe throughput degradation.

#### Card 12. Constant & Texture Memory Caching
*   **Core Definition**: Constant and Texture memories are read-only memory spaces cached at the SM level.
*   **Bottom Mechanism**: Constant memory excels when all threads of a Warp read the same address simultaneously (single-cycle broadcast). Texture memory features 2D spatial locality caching and dedicated hardware for filtering/clamping, ideal for graphics and spatial data.

#### Card 13. L1 / L2 Cache Configuration
*   **Core Definition**: The L1 cache and Shared Memory share a physical SRAM pool within each SM, allowing dynamic allocation sizing.
*   **Bottom Mechanism**: Developers can configure the SRAM partitioning (e.g., `96KB Shared / 32KB L1` or `64KB Shared / 64KB L1`) via APIs. The L2 cache is a unified global cache that buffers non-coalesced accesses, mitigating memory bus congestion.

#### Card 14. Unified Memory Page Migration
*   **Core Definition**: Unified Memory (UM) provides a single, unified virtual address space (UVA) allowing the CPU and GPU to share pointers with automatic data movement.
*   **Bottom Mechanism**: When the GPU accesses an address that is not physically present in VRAM, it triggers a page fault. The driver intercepts the fault and asynchronously migrates the corresponding 4KB virtual pages over PCIe/NVLink.

#### Card 15. Barriers & Memory Fences
*   **Core Definition**: Synchronizations govern execution ordering and memory visibility across GPU threads.
*   **Bottom Mechanism**: `__syncthreads()` acts as an execution barrier, halting all threads in a Block until everyone reaches it. `__threadfence()` acts as a memory fence; it does not block execution but flushes outstanding writes to ensure global visibility before proceeding.

---

### M4: Thread Sync, Cooperative Groups & Atomic Operations (M4.1 - M4.4)

#### Card 16. Cooperative Groups Programming Model
*   **Core Definition**: Cooperative Groups provide a clean abstraction for defining and synchronizing custom groups of threads (sub-blocks or grid-wide).
*   **Bottom Mechanism**: It breaks the block-wide constraint of `__syncthreads()`. Developers can group threads by Warp tiling (e.g., groups of 8/16) or synchronize globally across the entire Grid using `this_grid()`, enabling safe multi-block iteration.

#### Card 17. Warp Shuffle Register Data Exchange
*   **Core Definition**: Warp Shuffle operations allow threads within a Warp to read registers from other threads in the same Warp directly.
*   **Bottom Mechanism**: Registers are read directly via crossbar routing inside the Register File, bypassing Shared Memory and L1 cache entirely. This eliminates the need for execution barriers and is highly optimized in primitives like `__shfl_sync` and `__shfl_down_sync`.

#### Card 18. GPU Atomic Operations & L2 ALU Acceleration
*   **Core Definition**: Atomic operations (e.g., `atomicAdd`) guarantee that read-modify-write actions on shared or global memory are executed atomically.
*   **Bottom Mechanism**: Modern GPUs integrate hardware atomic ALUs directly into the L2 cache controllers. Global atomics are executed at the L2 cache line level, avoiding costly round-trips to registers and minimizing lock contention in high-concurrency writes.

#### Card 19. CUDA Streams & Concurrency Model
*   **Core Definition**: A CUDA Stream represents a queue of commands (kernels, memcpys) that execute sequentially on the GPU.
*   **Bottom Mechanism**: The default stream (Stream 0) is synchronous and blocks operations on other streams. Custom non-default streams execute concurrently and out-of-order, allowing overlap of memory copy and kernel execution across different streams.

---

### M5: Async Streams, Overlap & Heterogeneous Computing (M5.1 - M5.5)

#### Card 20. Host-Device Synchronization & Events
*   **Core Definition**: GPU operations are asynchronous to the CPU; Events monitor stream execution progress and measure time intervals.
*   **Bottom Mechanism**: `cudaEventRecord` places an event marker in a stream. When the GPU reaches it, the event is marked complete. The host queries the event status via `cudaEventQuery` or blocks CPU execution using `cudaEventSynchronize` until the stream drains.

#### Card 21. CUDA Graphs Task Graph Launch
*   **Core Definition**: CUDA Graphs model workflow launches (kernels, memcpys) as a static Directed Acyclic Graph (DAG) instead of individual API calls.
*   **Bottom Mechanism**: Traditional launch sequences incur high CPU overhead (driver validation and launch latency of ~3-5 microseconds per kernel). A captured Graph packages these nodes, sending a single execution token to the GPU, removing host launch overhead.

#### Card 22. Page-locked (Pinned) Memory & DMA Transfer
*   **Core Definition**: Pinned memory is host RAM allocated such that the OS cannot swap it out to virtual memory.
*   **Bottom Mechanism**: Allocated using `cudaMallocHost`, this memory is locked in physical RAM. The GPU's DMA (Direct Memory Access) engine can bypass the CPU and read host RAM directly, reaching the physical limits of the PCIe/NVLink bus width.

#### Card 23. Multi-GPU Peer-to-Peer (P2P) NVLink Mesh
*   **Core Definition**: Peer-to-Peer (P2P) enables GPUs in a system to directly read and write each other's VRAM without copying to host RAM.
*   **Bottom Mechanism**: Using high-bandwidth NVLink or NVSwitch interconnections, peer GPUs bypass the PCIe slot and CPU altogether, communicating directly at high speed. This forms the backbone of modern distributed large-model training clusters.

#### Card 24. Tensor Cores MMA Hardware Acceleration
*   **Core Definition**: Tensor Cores are specialized execution units designed to accelerate matrix multiply-accumulate operations in hardware.
*   **Bottom Mechanism**: In a single clock cycle, a Tensor Core performs a small matrix multiplication ($D = A \times B + C$, e.g., 16x16x16 shape) at the ASIC level. By hardwiring arithmetic arrays, they process massive Multiply-Accumulate operations far faster than standard SP cores.

---

### M6: Tensor Cores & Performance Tuning (M6.1 - M6.4)

#### Card 25. CUDA Dynamic Parallelism
*   **Core Definition**: Dynamic Parallelism enables a GPU Kernel to launch other child Kernels directly on the device without CPU intervention.
*   **Bottom Mechanism**: The GPU's internal device-side scheduler handles child kernel execution parameters directly. This allows highly efficient recursive algorithms, quad-tree refinement, and adaptive mesh refinement inside device threads.

#### Card 26. GPU Performance Optimization Matrix
*   **Core Definition**: Strategies to diagnose and solve Memory-bound and Compute-bound bottlenecks.
*   **Bottom Mechanism**: Use Roofline analysis to measure arithmetic intensity (FLOPs per byte transferred). Memory-bound kernels require coalescing and shared memory tiling. Compute-bound kernels require loop unrolling, fast-math intrinsics, and Tensor Core utilization.

#### Card 27. GPGPU Parallel Patterns (Reduction/Scan/GEMM)
*   **Core Definition**: Classical algorithmic paradigms for massive parallel computing.
*   **Bottom Mechanism**: In Parallel Reduction, warp-level unrolling and tree-reduction are used to avoid block-wide synchronization. In Histogram generation, Shared Memory local atomics are used to buffer writes, resolving heavy contention on Global Memory.

#### Card 28. Nsight Compute & Systems Diagnostics
*   **Core Definition**: NVIDIA's official profiling suite for system-level and kernel-level performance tuning.
*   **Bottom Mechanism**: Nsight Systems tracks global timelines, showing CPU-GPU stream overlaps and API delays. Nsight Compute gathers detailed Performance Monitor (PM) counters, reporting SM occupancy, Speed of Light (SOL), and pipeline stalls.

---

## 🛠️ Zone T1: Device/Host Keywords & Primitive API Quick Reference

### 1. Device Kernel and Memory Qualifiers
```cuda
__global__ void myKernel(float *d_out, const float *d_in) {
    // Device-side kernel, launched asynchronously from CPU. Return must be void.
}

__device__ float helperFunc(float x) {
    // Device-side inline function, callable only from GPU device code.
}

__shared__ float s_tile[32][32]; 
// Shared memory variable allocated on-chip in fast SRAM, shared within a Block.

__constant__ float c_mask[256]; 
// Constant memory variable cached in Constant Cache, globally visible.
```

### 2. Thread Index Built-in Variables
```cuda
int thread_id_x = threadIdx.x; // Thread index within the current Block (X-axis)
int block_id_x  = blockIdx.x;  // Block index within the Grid (X-axis)
int block_dim_x = blockDim.x;  // Number of threads in a Block (X-axis)
int grid_dim_x  = gridDim.x;   // Number of Blocks in the Grid (X-axis)
```

### 3. Host Memory Management & Stream APIs
```cpp
// Allocate device global memory
cudaError_t err = cudaMalloc((void**)&d_ptr, sizeBytes);

// Allocate pinned (page-locked) host memory for DMA copy acceleration
cudaMallocHost((void**)&h_pinned_ptr, sizeBytes);

// Synchronous data copy (blocks host CPU)
cudaMemcpy(d_ptr, h_ptr, sizeBytes, cudaMemcpyHostToDevice);

// Asynchronous stream copy (non-blocking, requires pinned host memory and a stream)
cudaMemcpyAsync(d_ptr, h_pinned_ptr, sizeBytes, cudaMemcpyHostToDevice, stream);

// Custom non-blocking stream creation
cudaStream_t stream;
cudaStreamCreate(&stream);
myKernel<<<grid, block, 0, stream>>>(d_ptr, d_in); // Async kernel launch
cudaStreamDestroy(stream);
```

### 4. Intra-Block Sync & Intra-Warp Shuffle Primitives
```cuda
// Synchronize all threads within a Block (execution barrier)
__syncthreads();

// Memory fence: ensures writes before the fence are visible, does not block execution
__threadfence();

// Warp Shuffle: Thread 0 pulls 'val' directly from Thread 31's register
float received = __shfl_sync(0xFFFFFFFF, val, 31);

// Warp Shuffle Down: Thread pulls register value from ThreadIdx + 4
float offsetVal = __shfl_down_sync(0xFFFFFFFF, val, 4);
```

---

## 🔍 Zone T2: Runtime Errors & Hardware Stall Diagnosis Dictionary

### 1. Common CUDA Runtime Error Codes
*   `cudaErrorMemoryAllocation` (Error 2): **Out of Memory (OOM)**. The GPU global memory allocation request exceeded the physical capacity.
*   `cudaErrorIllegalAddress` (Error 700): **Illegal Address Access**. A thread attempted to read/write an out-of-bounds pointer or unallocated memory address.
*   `cudaErrorLaunchOutOfResources` (Error 701): **Launch Out of Resources**. The kernel requested more registers or shared memory than available per SM, preventing Block dispatch.
*   `cudaErrorInvalidConfiguration` (Error 9): **Invalid Configuration**. Typically thrown when block size exceeds the 1024-thread physical limit.

### 2. Nsight Compute Stall Reasons Dictionary
*   **Stall Warps Waiting for Memory (LG Throttle / MIO Stall)**: Warps are stalled waiting for global memory transactions. Remedy: align and coalesce memory accesses, or use Shared Memory tiling.
*   **Stall Warps Barrier (Sync Stall)**: Warps are waiting at a `__syncthreads()` barrier. Indicates load imbalance or Warp Divergence inside the Block.
*   **Stall Instruction Fetch**: Instruction cache miss. Common in massive kernels with complex control flow. Remedy: break down or simplify the kernel structure.
