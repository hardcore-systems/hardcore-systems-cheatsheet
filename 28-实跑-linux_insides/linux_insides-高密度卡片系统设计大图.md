# linux-insides Internals Design Map

This design map links the 28 planned cards to specific Linux kernel source code file paths and core architectures.

---

## 🗺️ Codebase File Mappings

### M1: Booting & Kernel Initialization
*   **Card 1 (Mode Switching)**: Maps to `arch/x86/boot/main.c` and `arch/x86/boot/header.S`. Loading kernel, transitioning to 32-bit protected and 64-bit long modes.
*   **Card 2 (汇编入口 head_64.S)**: Maps to `arch/x86/kernel/head_64.S` and `init/main.c` (`start_kernel`). Assembly bootstrapping, BSS clearing.
*   **Card 3 (setup_arch)**: Maps to `arch/x86/kernel/setup.c`. E820 physical RAM parsing, memblock reservations.
*   **Card 4 (init_IRQ & Page Table)**: Maps to `arch/x86/kernel/irq.c` and `arch/x86/mm/init_64.c`. Page table switchover, interrupts setup.

### M2: Interrupts, Traps & CFS Scheduler
*   **Card 5 (IDT & Trap Table)**: Maps to `arch/x86/kernel/idt.c` and `arch/x86/include/asm/desc.h`. IDT gates registration.
*   **Card 6 (system_call Entry)**: Maps to `arch/x86/entry/entry_64.S` and `arch/x86/entry/common.c`. IA32_LSTAR MSR syscall vectors.
*   **Card 7 (softirq & workqueue)**: Maps to `kernel/softirq.c` and `kernel/workqueue.c`. Bottom half mechanisms.
*   **Card 8 (CFS Scheduler)**: Maps to `kernel/sched/fair.c` (`update_curr`, `pick_next_task_fair`). Virtual runtime RB-Tree management.
*   **Card 9 (switch_to Context Switch)**: Maps to `arch/x86/entry/entry_64.S` and `kernel/sched/core.c` (`context_switch`). Kernel stack redirection.

### M3: Physical Memory & Buddy System
*   **Card 10 (NUMA & Zones)**: Maps to `include/linux/mmzone.h` (`pg_data_t`, `struct zone`). DMA, Normal, HighMem zones.
*   **Card 11 (struct page)**: Maps to `include/linux/mm_types.h` (`struct page`). Page metadata descriptors, mapping reference counters.
*   **Card 12 (Buddy Allocation)**: Maps to `mm/page_alloc.c` (`__alloc_pages`, `free_area`). Power-of-two block allocation and buddy merges.
*   **Card 13 (Compaction)**: Maps to `mm/compaction.c` (`compact_zone`). Page migration scanner, defragmentation.

### M4: Slab/SLUB Object Allocator
*   **Card 14 (Slab Concept)**: Maps to `mm/slab.c` and `mm/slab_common.c`. Small object caching and recycling.
*   **Card 15 (kmem_cache Layering)**: Maps to `include/linux/slub_def.h` and `mm/slub.c`. Per-CPU CPU caches (`kmem_cache_cpu`) vs. node caches (`kmem_cache_node`).
*   **Card 16 (SLUB freelist)**: Maps to `mm/slub.c` (`slab_alloc_node`). Inline next pointers, lock-free allocations.
*   **Card 17 (kmalloc)**: Maps to `include/linux/slab.h` and `mm/slab_common.c`. Pre-allocated fixed-size slab lists.

### M5: Page Cache & Swapping
*   **Card 18 (address_space Page Cache)**: Maps to `include/linux/fs.h` (`struct address_space`) and `include/linux/xarray.h`. Radix Tree/xarray page indexes.
*   **Card 19 (Writeback thread)**: Maps to `mm/page-writeback.c`. `sysctl` writeback thresholds, background flushes.
*   **Card 20 (PFRA Page Reclaim)**: Maps to `mm/vmscan.c` (`shrink_zone`, `shrink_lruvec`). Active/Inactive page lists scanning.
*   **Card 21 (LRU 置换 & swappiness)**: Maps to `mm/vmscan.c` (`get_scan_count`). Swap tendencies, double scanning.
*   **Card 22 (OOM Killer)**: Maps to `mm/oom_kill.c` (`oom_badness`, `out_of_memory`). Out-of-memory processes scoring and killing.
*   **Card 23 (Swap Space)**: Maps to `mm/swapfile.c` and `mm/swap_state.c`. Swap maps tracking, page slots allocation.

### M6: VFS & Block Device I/O
*   **Card 24 (VFS Structures)**: Maps to `fs/super.c`, `fs/inode.c`, `fs/dcache.c`. Superblock, inode, dentry, and file descriptor linkings.
*   **Card 25 (file_operations)**: Maps to `include/linux/fs.h` (`struct file_operations`). VFS syscall redirections.
*   **Card 26 (BIO Structure)**: Maps to `include/linux/bio.h` (`struct bio`, `struct bio_vec`). Block request slices.
*   **Card 27 (I/O Schedulers)**: Maps to `block/` (e.g., `block/bfq-iosched.c`, `block/kyber-iosched.c`). Elevator request merging.
*   **Card 28 (mmap & Page Fault)**: Maps to `mm/mmap.c` and `arch/x86/mm/fault.c` (`do_page_fault`). File memory mapping.
