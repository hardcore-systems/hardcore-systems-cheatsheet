# 《OSTEP-Operating-Systems》 High-Density Knowledge Map & Cheatsheet

*   **L0 One-Line Essence**: An operating system is a resource manager that isolates processes using hardware privilege levels and timer interrupts, virtualizes physical memory through multi-level page table translation and TLB caching, manages multi-threaded shared resources via lock and condition variable synchronization, and ensures storage integrity via write-ahead logging and recovery scans.
*   **L1 Four-Line Logic**:
    1.  **CPU Virtualization & Scheduling** (M1) performs context switches during timer interrupts, distributing execution time using Multi-Level Feedback Queues (MLFQ) and Completely Fair Scheduling (CFS).
    2.  **Memory Virtualization & Address Translation** (M2) translates virtual addresses to physical frames via multi-level page tables backed by MMU hardware, caching mappings in the TLB to bypass page table walks.
    3.  **Concurrency & Lock Synchronization** (M3) builds mutexes and condition variables using atomic hardware instructions (CAS, Test-and-Set) to prevent race conditions and deadlock cycles.
    4.  **Storage I/O & File Consistency** (M4-M5) mitigates data corruption during unexpected crashes via write-ahead journaling and memory write barriers under VFS abstractions.
*   **L2 Storage Engine Data Flow Topology**:
    *   [User Mode Syscall] $\rightarrow$ [Privileged Trap Exception] $\rightarrow$ [Save CPU Registers to Kernel Stack] $\rightarrow$ [Page Table Base Register CR3 Switch] $\rightarrow$ [TLB Mapping Translation] $\rightarrow$ [VFS Routing] $\rightarrow$ [Page Cache Dirty Buffer] $\rightarrow$ [Write-Ahead Journal Write] $\rightarrow$ [DMA Controller Disk Flush]

---

## 🌐 Modern Operating System Virtualization & Concurrency Epistemic Filter

- **Epistemology - Hardware Privilege Isolation & Context Interruption as Logical Starting Points**:
  OSTEP's core philosophy is that "sandboxed software isolation is a mirage without hardware enforcement." The separation of CPU execution levels (Kernel Mode Ring 0 vs. User Mode Ring 3) is the fundamental starting point of an operating system. Under the Limited Direct Execution (LDE) protocol, user applications cannot directly access physical memory or hardware I/O; they must execute system calls to trap into the kernel. The kernel configures trap vectors during boot and regains CPU control via timer interrupts to perform context switches. This system of hardware enforcement and cooperative interception ensures security in multi-tasking environments.
- **Consistency Theory - Concurrent Thread Protection & File System Crash Consistency**:
  Isolation introduces complex synchronization requirements across memory and disk space. For concurrency in shared address spaces, memory reordering and cache-incoherent states threaten program correctness. The OS utilizes atomic Compare-and-Swap or Test-and-Set instructions to build mutexes and thread sleep queues in the kernel, ensuring thread safety. On physical block storage, crash-safety is similarly threatened by partial writes to data blocks, inodes, and free bitmaps. Consistency theory dictates using Write-Ahead Journaling: transactions must commit to log blocks before main file systems checkpoint data, allowing crash recovery via log replay.
- **Methodology - Performance-Enhancing Caches & Multi-Level Indexing for Space Efficiency**:
  Operating systems hide hardware latencies using caches and trade space complexity for index efficiency. Linear page tables for 64-bit address spaces consume gigabytes of memory; the kernel bypasses this by nesting mappings in multi-level page tables, leaving unallocated regions unmapped to save physical RAM. To offset the memory overhead of multi-level walks, the MMU caches mappings in the TLB. This methodology applies throughout system design: NFS protocols use client-side attribute caches to reduce network RPC latency, and SSDs utilize a Flash Translation Layer (FTL) to logically remap sectors, extending lifespan via wear leveling.

---

## ⚔️ Operating System Virtualization, Concurrency & Persistence Trade-offs

| Developer Intuition (⚠) | System Physical Reality (✗) | Best Practice & Architectural Standard (✓) |
| :--- | :--- | :--- |
| **Spinlocks are faster than mutexes in user-space because they avoid context switch overhead in all locking scenarios.** | Under high lock contention or single-CPU systems, waiting threads spin in CPU-busy loops, wasting processing cycles. This behavior can lead to priority inversion or thread starvation anomalies. | Use spinlocks only for short sections of code in multi-core kernel code. For general user-space thread synchronization, prefer `pthread_mutex`, which uses Futex to suspend blocked threads and free up the CPU. |
| **Larger page sizes improve TLB hit ratios, so production systems should configure all memory pages as huge pages.** | Although 2MB or 1GB huge pages reduce page table levels and improve TLB cache efficiency, they cause internal memory fragmentation and increase write amplification during swap file disk I/O. | Enable Transparent Huge Pages (THP) only for large, contiguous memory allocations (e.g., Redis databases or JVM heaps). Keep the default 4KB page size for random-access workloads (e.g., PostgreSQL buffer pools). |
| **To ensure data durability, application write operations must execute fsync to write data to disk immediately.** | High-frequency `fsync` calls block database threads. They bypass disk drive write caches and cause heavy random disk head movements, reducing application write throughput by several orders of magnitude. | Use batched writes combined with periodic background flushing. Store modifications in a memory WAL buffer and call `fsync` at regular intervals to balance data durability with system throughput. |

---

## 🗺️ Clickable Map of 6 Core Modules & 28 Cards

### M1: CPU Virtualization & Scheduling (Slate Blue)

#### Card 1. Process Descriptors, Kernel Stacks & Context Switching (Process Lifecycle & Kernel Stack)
*   **Process Control Block**: The OS defines process properties, file descriptors, and page table root pointers in a descriptor struct (e.g., `struct proc` or `task_struct` in Linux).
*   **Context Switch Assembly**: During a context switch, the scheduler saves current CPU registers (EIP, ESP) to the process's kernel stack, updates the stack pointer (ESP) to the target process's stack, and pops registers to resume execution.

#### Card 2. Limited Direct Execution (LDE) and Trap Handling (Limited Direct Execution)
*   **Hardware Ring Levels**: The CPU isolates memory space using Mode Bits (Kernel Mode Ring 0 vs. User Mode Ring 3). User processes cannot execute privileged commands or modify CR3 registers directly.
*   **Syscall Trap Path**: User requests for system resources execute a `syscall` instruction, which forces a trap that jumps to an entry in the kernel's trap table and transitions execution to Ring 0.

#### Card 3. Multi-Level Feedback Queue (MLFQ) Scheduling (MLFQ Scheduling)
*   **Dynamic Priority Rules**: MLFQ routes new jobs to the highest priority queue (with small time slices). If a job consumes its entire slice, its priority is demoted. If it yields the CPU before expiration, its priority is preserved.
*   **Starvation Prevention**: To prevent background batches from starving, the scheduler monitors a global period $S$. When $S$ expires, the scheduler boosts all active processes to the highest priority queue and resets their slices.

#### Card 4. Proportional Share and Completely Fair Scheduler (CFS) (CFS Scheduling)
*   **Random Lottery Shares**: Proportional share schedulers allocate tickets to processes and run a lottery draw for each slice, ensuring runtime matches ticket allocation ratios.
*   **CFS Virtual Runtime**: Linux CFS tracks a `vruntime` metric for each process, scheduling the thread with the lowest virtual runtime first. The scheduler manages runnable processes in an RB-Tree, ensuring $O(\log N)$ updates.

---

### M2: Memory Virtualization & Paging (Moss Green)

#### Card 5. Base & Bounds Address Relocation (Base and Bounds Translation)
*   **Hardware Relocation**: Early operating systems isolated memory spaces using two CPU registers: `Base` and `Bounds`.
*   **Translation Bounds**: When an instruction references memory, the CPU adds the `Base` offset to the virtual address. If the resulting address exceeds the `Bounds` range, the CPU raises a segmentation fault signal.

#### Card 6. Free Space Memory Allocation Strategies (Free-space Management)
*   **Free List Tracking**: Memory allocators (e.g., `malloc` library backends) track unallocated memory segments in a free list.
*   **Allocation Policies**:
    1.  **First-Fit**: Selects the first free block large enough for the request. It is fast but can cause fragmentation at the head of the list.
    2.  **Best-Fit**: Scans the list to find the block closest in size to the request, minimizing wasted space but creating tiny, unusable memory holes.
    3.  **Worst-Fit**: Allocates from the largest block available, leaving large remaining blocks but quickly exhausting big contiguous regions.

#### Card 7. Paging Systems and Page Table Entry (PTE) Flags (Paging & PTE Flags)
*   **Page to Frame Mapping**: Paging divides virtual memory into fixed pages (usually 4KB) and physical memory into page frames, tracking mappings in page tables.
*   **PTE Flags**: Page Table Entries contain the Physical Frame Number (PFN) and control bits: `P` (Present in memory), `R/W` (Read/Write permissions), `U/S` (User/Supervisor privilege limit), and `D` (Dirty bit for page replacement).

#### Card 8. Translation Lookaside Buffers (TLB) and ASID Flags (TLB Cache & ASID)
*   **Translation Cache**: To avoid performing memory walks for every read, the CPU caches virtual-to-physical address mappings in the TLB cache.
*   **ASID Context Preservation**: To prevent flushing the TLB on context switches, entries include an Address Space Identifier (ASID) tag to verify ownership, allowing multiple process mappings to coexist.

#### Card 9. Multi-Level Page Tables Space Savings (Multi-Level Page Tables)
*   **Directory-Level Indexing**: Multi-level page tables divide the linear page table into page-sized chunks, using a Page Directory (PD) to index the page table pages.
*   **On-Demand Allocation**: If a directory entry (PDE) has its Present bit set to 0, the corresponding secondary page table page is omitted, saving memory in sparse address spaces.

---

### M3: Concurrency Control & Synchronization (Plum Rose)

#### Card 10. Mutex Locks and Atomic Hardware Instructions (Mutex & Atomic Hardware)
*   **Data Race Prevention**: Standard load and store instructions are not atomic. Concurrent writes create data races.
*   **Atomic Instructions**:
    1.  **Test-and-Set**: Writes a value to memory and returns the old value atomically, allowing mutex implementation.
    2.  **Compare-and-Swap (CAS)**: Updates a memory location only if its current value matches a target parameter, enabling lock-free data structures.

#### Card 11. Condition Variables, Semaphores & Producer-Consumer (CV & Semaphores)
*   **Thread Block Queues**: Condition variables allow threads to release a mutex and block in a wait queue via `pthread_cond_wait` until another thread calls `pthread_cond_signal`.
*   **Semaphore Counters**: Semaphores manage resource access using an integer counter. `sem_wait` decrements the counter and blocks if the value is negative; `sem_post` increments the counter and wakes a waiting thread.

#### Card 12. Lock Contention and Futex Queues (Futex Queue)
*   **Spinning Overhead**: If a lock is held for long durations, waiting threads spin in CPU loops, wasting processing cycles.
*   **Futex Thread Suspension**: Linux uses `futex` (Fast Userspace Mutex) to handle lock contention. Uncontended locks are acquired in user-space; contested locks trigger a `sys_futex` system call to suspend the thread in a kernel queue.

#### Card 13. Coffman Deadlock Conditions and Prevention (Deadlock Coffman Conditions)
*   **Four Deadlock Conditions**: Deadlocks require all four conditions to hold:
    1.  **Mutual Exclusion** (exclusive resource access).
    2.  **Hold-and-Wait** (threads holding resources request new ones).
    3.  **No Preemption** (allocated resources cannot be forced away).
    4.  **Circular Wait** (threads form a dependency cycle).
*   **Deadlock Prevention**: The most effective prevention is breaking the circular wait condition by enforcing a strict global lock acquisition ordering.

---

### M4: Storage Devices & File Systems (Terracotta)

#### Card 14. Disk I/O Controllers, DMA, and Hardware Interrupts (Disk I/O & DMA)
*   **Polling Overhead**: If the CPU handles data transfers directly through I/O registers, it wastes cycles polling device status registers.
*   **DMA Controller Bypass**: A Direct Memory Access (DMA) controller manages block transfers between memory and disk directly. The CPU initiates the transfer and continues other work until the DMA raises an interrupt upon completion.

#### Card 15. RAID Array Topologies and Trade-offs (RAID Architectures)
*   **RAID 0 (Striping)**: Spreads blocks across disks to maximize read/write performance. It lacks redundancy; a single disk failure corrupts the entire array.
*   **RAID 1 (Mirroring)**: Duplicate writes to duplicate disks provide 100% redundancy, doubling read throughput but reducing capacity utilization to 50%.
*   **RAID 5 (Distributed Parity)**: Rotates parity blocks across disks to survive a single disk failure, offering a storage efficiency of $(N-1)/N$.

#### Card 16. VSFS File System Disk Layout (Very Simple File System)
*   **Block-Based Segmentation**: VSFS partitions storage space into fixed 4KB blocks.
*   **Functional Layout**:
    1.  **Superblock**: Stores file system metadata (e.g., block counts, inode limits).
    2.  **Bitmaps**: Tracks free inodes and data blocks.
    3.  **Inode Table**: Contains inode definitions, mapping file properties and data block pointers.
    4.  **Data Blocks**: Stores file contents and directory trees.

#### Card 17. Path Resolution and Directory Lookups (Path Resolution Walk)
*   **Iterative Lookup**: Resolving a path like `/foo/bar.txt` starts at the root inode (usually inode 2).
*   **Inode Traversal**: The kernel reads the root directory's data blocks to find the inode number for the directory `foo`, reads `foo`'s data blocks to find `bar.txt`'s inode, and uses that inode to read the file contents.

---

### M5: File System Consistency & Journaling (Indigo)

#### Card 18. Crash Consistency and Split Writes (Crash Consistency)
*   **Multi-Block Updates**: Appending data to a file requires updating three structures: the inode, the data bitmap, and the data block.
*   **Inconsistency Anomalies**: Since disk writes cannot update all three locations simultaneously, a crash can leave the file system in an inconsistent state (e.g., an inode pointing to unallocated space).

#### Card 19. FSCK Consistency Scanner (FSCK Utility)
*   **Post-Crash Verification**: FSCK (File System Consistency Check) scans the file system after an unclean shutdown.
*   **Scan Latency**: FSCK audits all inodes, bitmaps, and directory structures. On modern large-capacity drives, this scan can take hours or days, making it unsuitable for high-availability systems.

#### Card 20. Write-Ahead Journaling Transaction Flow (Write-Ahead Journaling)
*   **Transactional Writes**: To prevent metadata corruption, file changes are written to a dedicated disk journal space first.
*   **Journal States**:
    1.  **Journal Write**: Writes block updates to the journal.
    2.  **Journal Commit**: Appends a transaction commit block; once written, the transaction is committed.
    3.  **Checkpoint**: Writes changes to their final file system locations.
    4.  **Free**: Clears the transaction from the journal.

#### Card 21. Data vs. Metadata Journaling Modes (Metadata Journaling)
*   **Data Journaling**: Logs both file data and metadata. It offers the highest safety but doubles disk write volume, reducing performance.
*   **Metadata Journaling**: Logs only metadata changes (inodes and bitmaps); file data is written directly to the data blocks. This mode reduces I/O volume while maintaining system consistency.

#### Card 22. Write Barriers and Controller Reordering (Write Barriers)
*   **Out-of-Order Disk Writes**: Disk controllers reorder write queues to optimize head movements, which can cause a commit block to write before its journal transactions.
*   **Barrier Synchronization**: The OS sends a `Write Barrier` command to force all pending writes to disk before executing subsequent write operations, preserving log order.

#### Card 23. Log-Structured File Systems (LFS) and Cleaner Threads (LFS & Segment Clean)
*   **Random Write Seek Penalty**: Traditional file systems write updates to different disk locations, causing latency on mechanical drives.
*   **Segment Logging**: LFS buffers writes in memory and writes them sequentially in large `Segments` at the end of the disk log. It uses a background cleaner thread to relocate active blocks and reclaim space.

---

### M6: Distributed Systems & Flash SSDs (Antique Gold)

#### Card 24. Network File System (NFS) Stateless Architecture (NFS Protocol)
*   **Remote File Mounting**: NFS uses RPC protocols to mount remote file systems over networks.
*   **Stateless Recovery**: The server does not track client states (e.g., opened files). Clients include a `File Handle` (volume ID, inode, generation count) with every RPC request, allowing seamless server failovers.

#### Card 25. NFS Client Caching and Attribute Consistency (NFS Cache Consistency)
*   **Latency Mitigation**: NFS clients cache file properties and data blocks locally to minimize network RPC overhead.
*   **Caching Policies**:
    1.  **Attribute Cache Timeout**: Clients verify file attributes with the server at fixed intervals (e.g., every 3 seconds).
    2.  **Close-to-Open**: Clients flush dirty pages on `close()` and verify file attributes on `open()`, providing basic multi-client consistency.

#### Card 26. SSD Physics: Page Writes and Block Erasures (SSD Physics)
*   **Granular Boundaries**: Solid-state drives read and write data in `Pages` (e.g., 4KB) but erase data in larger `Blocks` (e.g., 128 pages).
*   **Overwrite Limitations**: SSD cells cannot be overwritten directly. A page must be erased before it can be written to, requiring a block-level erase operation.

#### Card 27. Flash Translation Layer (FTL) and Wear Leveling (FTL & Wear Leveling)
*   **Logical Mapping**: The SSD controller runs FTL firmware to map logical block addresses from the OS to physical flash pages.
*   **Garbage Collection**: FTL uses out-of-place writes to avoid in-place erasures. Outdated pages are marked invalid, and a garbage collector consolidates active pages to erase blocks. FTL uses wear leveling to distribute writes and prevent premature cell wear.

#### Card 28. Virtual File System (VFS) Redirection (VFS Abstraction)
*   **File Abstraction Layer**: The operating system provides a VFS layer to expose standard syscalls (e.g., `open`, `read`, `write`) to user applications.
*   **Function Table Routing**: VFS defines generic structures like `struct file` and `struct inode`. It routes read/write calls to file-system-specific implementations using function pointer tables.

---

## 🔬 Zone T: Production Diagnosis & Configuration Reference

### T1 Linux OS Kernel Tuning Parameters

| Parameter / /etc/sysctl.conf | Recommended Value | Description |
| :--- | :--- | :--- |
| `vm.swappiness` | `10 - 30 (Lower on servers)` | Adjusts how aggressively the kernel swaps memory pages to disk. Set to 10 to keep applications in physical RAM and minimize swap I/O. |
| `fs.file-max` | `2097152 (2 Million+)` | The maximum limit of open file descriptors system-wide. Increase this limit on high-concurrency systems to prevent file limit errors. |
| `vm.dirty_background_ratio` | `5 - 10` | The percentage of system memory that can contain dirty pages before background flush daemons start flushing them to disk. |
| `vm.dirty_ratio` | `20 - 30` | The max percentage of memory containing dirty pages. Reaching this limit forces user writes to block until pages are written to disk. |

### T2 Linux Diagnostic and Tracing Command Reference

*   **1. Diagnose Context Switches and I/O Wait**
    ```bash
    # Print system statistics every 1 second. Inspect 'cs' (context switches) and 'wa' (CPU time spent waiting for I/O)
    vmstat 1
    ```
*   **2. Analyze Process Virtual Memory Page Mappings**
    ```bash
    # Display the memory map of a process (PID) to identify physical memory allocation and trace leaks
    pmap -x [PID]
    ```
*   **3. Trace System Call Durations and Call Counts**
    ```bash
    # Gather statistics on system calls (e.g., read, futex) executed by a process, tracking counts, errors, and elapsed time
    strace -c -p [PID]
    ```
