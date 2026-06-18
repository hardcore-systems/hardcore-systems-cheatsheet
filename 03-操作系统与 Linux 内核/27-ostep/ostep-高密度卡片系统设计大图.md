# OSTEP Internals Design Map

This design map links the 28 planned cards to operating system architecture components and OSTEP textbook chapter references.

---

## 🗺️ Architectural Concept & Chapter Mappings

### M1: CPU Virtualization & Scheduling
*   **Card 1 (Process Switch)**: Maps to OSTEP Chapter 4 (Processes) & Chapter 6 (Direct Execution). Details CPU registers, process control block (PCB), and context switch assembly.
*   **Card 2 (LDE & Trap)**: Maps to OSTEP Chapter 6 (Limited Direct Execution). User/Kernel space transition via system calls, trap tables, and return-from-trap instruction.
*   **Card 3 (MLFQ Scheduling)**: Maps to OSTEP Chapter 8 (Multi-level Feedback Queue). Rules for priority demotion, queues, time slice allocation, and starvations.
*   **Card 4 (CFS Scheduling)**: Maps to OSTEP Chapter 9 (Proportional Share) & Linux CFS. Virtual runtime, scheduling latency, and RB-Tree sorting.

### M2: Memory Virtualization & Paging
*   **Card 5 (Base & Bounds)**: Maps to OSTEP Chapter 15 (Address Translation). Physical registers for memory relocations and protection faults.
*   **Card 6 (Memory Allocator)**: Maps to OSTEP Chapter 17 (Free-space Management). Memory segregation, splitting, coalescing, and list policies.
*   **Card 7 (Paging & Page Tables)**: Maps to OSTEP Chapter 18 (Introduction to Paging). Virtual Page Number (VPN), Physical Frame Number (PFN), and page table entries.
*   **Card 8 (TLB Hardware)**: Maps to OSTEP Chapter 19 (Translation Lookaside Buffers). Hardware caches, TLB miss traps, ASIDs, and page-directory page table walks.
*   **Card 9 (Multi-level Page Tables)**: Maps to OSTEP Chapter 20 (Advanced Page Tables). Page Directory (PD), Page Table (PT), and space-saving math.

### M3: Concurrency & Synchronization
*   **Card 10 (Locks & Atomic)**: Maps to OSTEP Chapter 28 (Locks). Spinlocks, Compare-and-Swap (CAS), Test-and-Set, Load-Linked/Store-Conditional.
*   **Card 11 (CV & Semaphores)**: Maps to OSTEP Chapter 30 (Condition Variables) & Chapter 31 (Semaphores). Wait queues, producer-consumer ring buffer, and counters.
*   **Card 12 (Futex & Sleeping)**: Maps to OSTEP Chapter 28 (Futexes). Linux futex syscall, wait queue hash, and kernel-guided sleep transitions.
*   **Card 13 (Deadlock Conditions)**: Maps to OSTEP Chapter 32 (Common Concurrency Bugs). Coffman conditions, dependency cycle graphs, lock ordering, and prevention.

### M4: Storage Devices & File Systems
*   **Card 14 (I/O Drivers)**: Maps to OSTEP Chapter 36 (I/O Devices). DMA transfers, device controllers, memory-mapped I/O, and polling vs interrupts.
*   **Card 15 (RAID Arrays)**: Maps to OSTEP Chapter 38 (RAID). RAID 0 stripe, RAID 1 mirror, RAID 4/5 parity math, reconstruction rebuilds.
*   **Card 16 (VSFS Layout)**: Maps to OSTEP Chapter 40 (File System Implementation). Inode block arrays, block allocation bitmaps, directories, and data blocks.
*   **Card 17 (Path Resolution)**: Maps to OSTEP Chapter 40. Iterative directory lookup, reading inode indexes, resolving paths like `/foo/bar.txt`.

### M5: File System Consistency & Journaling
*   **Card 18 (Crash Consistency)**: Maps to OSTEP Chapter 42 (Crash Consistency). Write order violations, corrupted inodes, orphan data blocks.
*   **Card 19 (FSCK Utility)**: Maps to OSTEP Chapter 42 (FSCK). Post-crash metadata scans, block pointer audits, directory link count updates.
*   **Card 20 (Write-Ahead Journaling)**: Maps to OSTEP Chapter 42 (Journaling). TxBegin, Write Journal metadata/data, TxCommit, checkpoint write to main FS.
*   **Card 21 (Journaling Modes)**: Maps to OSTEP Chapter 42 (Metadata Journaling). Metadata-only logging vs. data logging tradeoffs.
*   **Card 22 (Barriers & Caching)**: Maps to OSTEP Chapter 42. Force write order bypassing disk controller cache reorderings.
*   **Card 23 (LFS File System)**: Maps to OSTEP Chapter 43 (Log-structured File Systems). Write buffering, segment cleaning, inode map (imap), and thread-like sequential logs.

### M6: Distributed Systems & Flash SSDs
*   **Card 24 (NFS Protocol)**: Maps to OSTEP Chapter 49 (NFS). Stateless file servers, File Handles (fh), idempotent RPC requests.
*   **Card 25 (NFS Caching)**: Maps to OSTEP Chapter 49. Attribute caching time-outs, write-back close-to-open consistency, validation loops.
*   **Card 26 (SSD Physics)**: Maps to OSTEP Chapter 44 (Flash-based SSDs). Flash memory pages, erase-before-write blocks, wear endurance constraints.
*   **Card 27 (FTL Translation)**: Maps to OSTEP Chapter 44. Flash Translation Layer logical mapping, block garbage collection, wear leveling, WAF index.
*   **Card 28 (VFS Interface)**: Maps to Linux VFS and OSTEP Chapter 39. `file` and `inode` vfs descriptors, file operations pointer redirection.
