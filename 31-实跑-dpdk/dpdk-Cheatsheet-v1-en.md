# DPDK (Data Plane Development Kit) High-Density Core Card System (English Version)

## L0 ~ L2 Knowledge Ladder

*   **L0 One-Line Essence**: DPDK is a zero-copy, ultra-low latency data plane packet processing development kit that bypasses the kernel, avoids context switches using hugepages and lockless ring buffers, and polls NIC descriptors using dedicated PMD cores to achieve physical line-rate throughput.
*   **L1 Four-Line Logic**:
    1.  **Kernel Bypass & Userspace Driving**: Uses UIO/VFIO to expose NIC registers and memory space to userspace via BAR mapping, allowing direct DMA write to host RAM and completely skipping kernel-space networking stacks.
    2.  **Hugepage Memory & NUMA Isolation**: Allocates memory pool buffers and registers on hugepages (2MB/1GB) to achieve nearly 100% TLB hits, and locks execution structures onto local NUMA memory sockets to eliminate cross-bus latency.
    3.  **Dedicated Polling & Hardware Burst**: Pins CPU cores in a tight loop via Poll Mode Drivers (PMD) to read RX descriptors, eliminating interrupt storms, and processes packets in batches (Burst of 32) to amortize instruction overhead.
    4.  **Hardware Offloading & Flow Steering**: Distributes workload using RSS symmetric Toeplitz hashing and FDIR steering rules directly in hardware, offloading L3/L4 checksum computations and TCP segmentation (TSO) to save host cycles.
*   **L2 Core Data Flow Topology**:
    *   `Physical Fiber` ➜ `NIC RX Ring` (DMA to Hugepages) ➜ `rte_eth_rx_burst(rx_pkts, 32)` (PMD on isolcpus) ➜ `rte_mbuf Parsing` ➜ `Run-to-completion / Pipeline` (via rte_ring) ➜ `NIC TX Ring` ➜ `Physical Fiber`.

---

## ⚔️ DPDK Memory & Queue Synchronization Trade-offs Matrix (Page 1 Bottom)

*   **Developer Intuition (⚠)**: Utilize Single-Producer Single-Consumer (SPSC) queues everywhere for maximum lockless throughput. ➜ **Underlying Physics (★)**: SPSC only works between single cores. If multiple threads enqueue concurrently, you must use Multi-Producer Multi-Consumer (MPMC) with atomic CAS operations, or you will corrupt indices and crash the core.
*   **Developer Intuition (⚠)**: Allocate physically contiguous virtual memory and resolve physical addresses dynamically under all conditions. ➜ **Underlying Physics (★)**: Using IOVA as PA requires root privileges to read `/proc/self/pagemap`. In non-privileged containers, you must use IOVA as VA mode, relying on VFIO/IOMMU hardware page tables to safely map DMA targets.
*   **Developer Intuition (⚠)**: To avoid allocation overheads, fetch empty buffers directly from the global Mempool ring inside the packet loop. ➜ **Underlying Physics (★)**: Global mempool accesses trigger atomic CAS lock contentions. You must use Lcore Local Cache buffer arrays, replenishing only when empty, to avoid severe core inter-locking.
*   **Developer Intuition (⚠)**: To guarantee packet queue ordering, insert full-memory synchronization barriers immediately after writing buffers. ➜ **Underlying Physics (★)**: You only need write barriers like `rte_smp_wmb()` to ensure packet data writes precede pointer updates. Full barriers (e.g. `mfence`) drain the CPU's Store Buffer, stalling pipelines and dropping throughput by over 40%.

---

## 📂 M1: Kernel Bypass & Userspace Drivers (Cards 1-4)

#### Card 1. UIO Simple Bypass vs VFIO Secure Isolation Bypass
*   **Technical Mechanism**: UIO (`igb_uio`) exposes interrupts and device memory to user space but lacks IOMMU hardware page isolation, allowing raw DMA to write anywhere and risking memory corruption. VFIO (`vfio-pci`) utilizes hardware IOMMU page tables, restricting DMA access only to mapped hugepage regions.
*   **Application Strategy**: VFIO is mandatory in secure virtualized environments and multi-tenant containers. UIO should only be used as a fallback for legacy hardware lacking IOMMU support.

#### Card 2. PCI BAR (Base Address Register) Mapping & Userspace PCI Access
*   **Technical Mechanism**: Physical NIC registers are exposed as PCI BAR space. During initialization, DPDK probes the device sysfs resource path (`/sys/bus/pci/devices/.../resourceX`) and maps the BAR descriptor directly to user space memory via `mmap()`.
*   **Application Strategy**: Userspace drivers write directly to device registers using memory pointers (e.g. updating RX/TX head/tail indices), avoiding costly `ioctl()` syscall transitions.

#### Card 3. DPDK EAL (Environment Abstraction Layer) Boot Sequence
*   **Technical Mechanism**: Initialized via `rte_eal_init()`, which parses configuration flags, probes and maps hugepage segments, scans the PCI bus, binds device drivers (e.g. `vfio-pci`), and spawns and pins worker threads (Lcores) to dedicated CPU cores.
*   **Application Strategy**: Common boot failures are caused by missing hugepages, insufficient permissions to access `/dev/hugepages`, or core mask overlap.

#### Card 4. Userspace DMA vs Kernel sk_buff Double Copy Overhead
*   **Technical Mechanism**: In the kernel, packets are DMA-transferred to kernel memory, wrapped in `sk_buff` structs, processed, and then copied to userspace buffers via syscalls. DPDK maps pre-allocated hugepage buffers directly to the NIC's DMA engine, enabling direct userspace DMA writes.
*   **Application Strategy**: For links exceeding 10Gbps, eliminating copy overheads is required to maintain line-rate throughput.

---

## 📂 M2: Lockless Ring Buffer & Memory Pool (Cards 5-9)

#### Card 5. rte_ring SPSC (Single-Producer Single-Consumer) Pointer Sliding
*   **Technical Mechanism**: The SPSC model utilizes independent pointers for `prod.head`/`prod.tail` and `cons.head`/`cons.tail`. With only one thread writing and one reading, no atomic CAS (Compare-And-Swap) overhead is required, relying only on lightweight memory barriers.
*   **Application Strategy**: Use SPSC when data exchange is strictly locked between two dedicated core threads to save 30% overhead compared to MPMC rings.

#### Card 6. rte_ring MPMC (Multi-Producer Multi-Consumer) Atomic CAS Reservation
*   **Technical Mechanism**: Multiple producers execute an atomic CAS (`__atomic_compare_exchange_n()`) to reserve free slots by advancing `prod.head`. Threads write data in parallel within their reserved slots, then spin-wait until all preceding writes complete before updating `prod.tail`.
*   **Application Strategy**: Ideal for core aggregation or thread distribution. The spin-waiting step to align tail pointers is the primary bottleneck.

#### Card 7. rte_ring Memory Barrier & Write Consistency
*   **Technical Mechanism**: Enqueuing involves writing data to the mbuf array and advancing the ring pointer. If the CPU reorders these steps, a consumer core might read garbage data. A write barrier `rte_smp_wmb()` must be inserted between data writes and pointer updates.
*   **Application Strategy**: DPDK's modern C11 atomics handle this automatically. Custom ring implementations must explicitly place barriers to ensure read-after-write consistency.

#### Card 8. rte_mempool Ring Backend & Fast Packet Allocation
*   **Technical Mechanism**: `rte_mempool` is a fixed-size block allocator. It utilizes a `rte_ring` as its default backend to store unused mbuf memory block addresses. Allocating a packet equates to deqeueuing a pointer, and freeing a packet equates to enqueuing it.
*   **Application Strategy**: Mempool size must be sized at boot to be 2-3x the sum of RX/TX descriptors and local cache size to prevent pool exhaustion under high traffic.

#### Card 9. rte_mempool Lcore Local Cache & Lock Elimination
*   **Technical Mechanism**: To avoid multiple cores competing on the global mempool ring CAS, each core (Lcore) is allocated a local cache block (`struct rte_mempool_cache`). Allocation and freeing occur on the local cache first, eliminating atomic spinlocks.
*   **Application Strategy**: Cache size should be set to 32 or 64 in the fast path to bypass the global lock and ensure fast access.

---

## 📂 M3: Hugepages & NUMA Affinity (Cards 10-13)

#### Card 10. Hugepages (2MB / 1GB) & TLB Miss Elimination
*   **Technical Mechanism**: Standard 4KB pages yield large page table structures and frequent TLB (Translation Lookaside Buffer) misses. By scaling pages to 2MB or 1GB, the page table size decreases, enabling translation entries to be permanently cached in the CPU TLB.
*   **Application Strategy**: Production DPDK setups should reserve 1GB hugepages at boot time (`default_hugepagesz=1G hugepages=16` in grub) to boost memory translation speed by 10%-15%.

#### Card 11. Physical Contiguous Mapping (IOVA as VA vs IOVA as PA)
*   **Technical Mechanism**: IOVA (I/O Virtual Address) is the address space used by the NIC DMA engine. IOVA as PA maps memory using actual physical addresses, requiring root access to read page tables. IOVA as VA matches userspace virtual layout, managed safely by VFIO/IOMMU.
*   **Application Strategy**: IOVA as VA is recommended for non-root containers and cloud deployments to bypass security access issues.

#### Card 12. NUMA (Non-Uniform Memory Access) Socket Affinity
*   **Technical Mechanism**: Cross-socket memory access via UPI/QPI interconnects doubles latency. DPDK enforces strict socket affinity: the NIC PCI lane, hugepage allocation memory segment, and polling core thread must reside on the same NUMA node.
*   **Application Strategy**: Allocate structures using socket-aware interfaces like `rte_malloc_socket()` and pass the matching NUMA ID using `rte_eth_dev_socket_id()`.

#### Card 13. Cacheline Alignment (__rte_cache_aligned) & False Sharing Elimination
*   **Technical Mechanism**: CPU cache coherence invalidates the entire cacheline (64 bytes) when a core writes to it. If two cores concurrently write to different variables residing in the same cacheline, the line bounces between L1 caches, causing false sharing.
*   **Application Strategy**: Apply `__rte_cache_aligned` and padding to shared counters or core-specific variables to isolate writes on separate cachelines.

---

## 📂 M4: Poll Mode Driver (PMD) & Burst I/O (Cards 14-17)

#### Card 14. Linux NAPI Interrupt Merging vs DPDK PMD Polling
*   **Technical Mechanism**: Traditional OS interrupts on high-speed links trigger high context-switch overheads and軟中断 softirq latency (receive livelock). DPDK PMD disables hardware interrupts and runs a continuous loop on dedicated cores to poll hardware descriptors.
*   **Application Strategy**: A 100% CPU utilization reading for PMD cores is expected. Polling is significantly more efficient than interrupt handling under loads exceeding 1 Mpps.

#### Card 15. Descriptor Ring Buffer & Write-Back State Management
*   **Technical Mechanism**: The descriptor ring is a physical memory array shared between the NIC and the CPU. The NIC writes packet addresses and sets the Descriptor Done (DD) bit. The CPU polls the DD bit, extracts the packet, and updates the descriptor.
*   **Application Strategy**: Configure ring buffer depth (e.g. 1024, 2048) based on latency budgets to handle traffic bursts without dropping DMA transfers.

#### Card 16. rte_eth_rx_burst / rte_eth_tx_burst API & Instruction Prefetching
*   **Technical Mechanism**: `rte_eth_rx_burst` fetches packets in batches (default size 32). While processing packet N, the driver executes compiler assembly prefetch instructions (`rte_prefetch0()`) to load packet N+1's header into the L1 CPU Cache, hiding latency.
*   **Application Strategy**: Ensure packet processing loops make use of batching and prefetching to hide memory bus latency during packet analysis.

#### Card 17. CPU Thread Pinning & isolcpus Isolation
*   **Technical Mechanism**: Although CPU affinity binds threads, the kernel scheduler can still dispatch system timers, interrupts, or other tasks onto PMD cores, causing micro-second latency spikes.
*   **Application Strategy**: Configure `isolcpus=2-7 rcu_nocbs=2-7 nohz_full=2-7` in the boot grub loader to isolate worker cores from system task scheduling.

---

## 📂 M5: mbuf Layout & Packet Processing Pipeline (Cards 18-23)

#### Card 18. rte_mbuf Metadata Layout & Cacheline Optimization
*   **Technical Mechanism**: `struct rte_mbuf` spans exactly two cachelines (128B). Cacheline 0 contains critical data path parameters (`buf_addr`, `buf_iova`, `data_len`, `pool`) to ensure a single cache miss loads all necessary info during receipt.
*   **Application Strategy**: Do not insert custom variables into Cacheline 0. Use the predefined `udata64` field or store metadata in the packet data headroom.

#### Card 19. mbuf Headroom & Tailroom Layout
*   **Technical Mechanism**: The mbuf buffer layout is structured as: `[rte_mbuf] -> [Headroom] -> [Packet Data] -> [Tailroom]`. Headroom (default 128B) is reserved space to prepend protocol encapsulations (like VXLAN or GRE headers) without moving data.
*   **Application Strategy**: Use `rte_pktmbuf_prepend()` to adjust headroom pointers, avoiding memory moves when adding network wrappers.

#### Card 20. mbuf Packet Cloning (rte_pktmbuf_clone) & Reference Counting
*   **Technical Mechanism**: Cloning creates a new mbuf metadata struct pointing to the same data segment inside hugepage memory, incrementing the data block's reference counter (`refcnt`).
*   **Application Strategy**: The clone is freed via `rte_pktmbuf_free()`, which decrements `refcnt`. Memory is recycled only when `refcnt` drops to 0, enabling zero-copy multicasting.

#### Card 21. Run-to-completion Processing Model
*   **Technical Mechanism**: A single Lcore handles polling, packet parsing, routing lookup, offload configuration, and transmission. This model avoids inter-thread communication, locking, or queue overheads, keeping packet context in local L1/L2 caches.
*   **Application Strategy**: Use this model for lightweight packet processing to achieve the highest throughput and lowest latency.

#### Card 22. Pipeline Core Distribution Model & rte_ring bottlenecks
*   **Technical Mechanism**: Divides complex logic (e.g. firewall, crypto, routing) into pipeline stages running on different cores. Packet pointers travel between cores via lockless `rte_ring` instances.
*   **Application Strategy**: Cross-core queue moves introduce L3 cache migration and CAS contention. Map workloads to minimize inter-core ring hops.

#### Card 23. DPDK ACL (Access Control List) Trie Match Algorithm
*   **Technical Mechanism**: The `librte_acl` library compiles multi-dimensional firewall rules (5-tuple) into a multi-bit Trie trie structure. Searching rules scales to O(1), bypassing linear loop iterations.
*   **Application Strategy**: Prefer DPDK's optimized ACL implementation for userspace firewalls and packet gateways to handle large rule sets at wire speed.

---

## 📂 M6: Hardware Offloading & Flow Steering (Cards 24-28)

#### Card 24. RSS (Receive Side Scaling) Symmetric Toeplitz Hash Splitting
*   **Technical Mechanism**: RSS calculates a Toeplitz hash over packet IP/Port values in hardware, mapping packets to specific RX descriptor queues on the host to distribute incoming load.
*   **Application Strategy**: Configure symmetric RSS keys to ensure bi-directional packet flows (e.g., source and destination flipped) land on the same RX queue and worker thread.

#### Card 25. FDIR (Flow Director) Flow Rules & Direct Core Steering
*   **Technical Mechanism**: FDIR provides precise ASIC matching rules (e.g. routing source IP 10.0.0.1 TCP port 80 to Queue 4). The NIC redirects matched traffic, bypassing CPU-based software filtering.
*   **Application Strategy**: Configure flow rules via the `rte_flow` API to isolate specific traffic classes directly on dedicated worker cores.

#### Card 26. IP/TCP/UDP Checksum Offloading & L3/L4 Pre-calculation
*   **Technical Mechanism**: Instead of processing checksum additions in software, set offload flags (e.g. `RTE_MBUF_F_TX_IP_CKSUM`) on the mbuf. The hardware NIC recalculates and writes checksums during the DMA step.
*   **Application Strategy**: Clear IP header checksums to 0 in software and set mbuf offload flags to bypass software loop checksum calculations.

#### Card 27. TSO (TCP Segmentation Offload) & DMA Segmentation
*   **Technical Mechanism**: CPU submits a large TCP frame (up to 64KB) with TSO enabled. The NIC DMA hardware segments the packet into MTU-sized frames, duplicating the IP/TCP headers on the fly.
*   **Application Strategy**: Enable TSO in the device tx capabilities to offload TCP packet segmentation, saving significant CPU cycles on outbound flows.

#### Card 28. SR-IOV (Single Root I/O Virtualization) & PF/VF Direct Pass-through
*   **Technical Mechanism**: Splitting a physical function (PF) NIC into virtual functions (VF). Each VF behaves as a standalone PCIe device, passed directly into VMs or containers.
*   **Application Strategy**: Use SR-IOV to bypass hypervisor virtual switch software layers, achieving near-native line rate performance within VMs.

---

## 🛠️ Zone T: DPDK Diagnostics & Performance Tuning (Page 2 Bottom)

### T1: Kernel Tuning Parameters for DPDK Core Isolation

*   `isolcpus=2-7`: Boot parameter. Isolates CPU cores 2-7, preventing the OS scheduler from running ordinary tasks on them.
*   `nohz_full=2-7`: Boot parameter. Enables tickless mode on isolated cores, removing kernel clock tick interrupts to prevent thread scheduling jitter.
*   `rcu_nocbs=2-7`: Boot parameter. Prevents RCU callback threads from executing on isolated cores to preserve dedicated CPU cycles.
*   `default_hugepagesz=1G hugepages=16`: Boot parameter. Reserves 16 contiguous 1GB hugepage segments at system boot time.
*   `vm.zone_reclaim_mode=0`: `/etc/sysctl.conf`. Disables reclamation on memory zones to prevent high-latency page allocation sweeps.
*   `vm.max_map_count=1048576`: `/etc/sysctl.conf`. Expands memory mapping limits to allow large scale page allocations.

### T2: DPDK Diagnostic CLI Utilities

*   `dpdk-devbind.py`: Manages network interface driver binding.
    *   *Check status*: `./usertools/dpdk-devbind.py --status`
    *   *Bind interface*: `./usertools/dpdk-devbind.py -b vfio-pci 0000:01:00.0`
*   `dpdk-hugepages.py`: Manages hugepage mappings.
    *   *Setup allocations*: `./usertools/dpdk-hugepages.py -p 2M --setup 2048`
*   `dpdk-pdump`: Captures and records traffic traversing DPDK ports.
    *   *Capture packets*: `dpdk-pdump -- --pdump 'device_id=0000:01:00.0,queue=*,rx-dev=/tmp/rx.pcap,tx-dev=/tmp/tx.pcap'`
*   `dpdk-proc-info`: Queries statistics and configurations of a running primary DPDK process.
    *   *Get stats*: `dpdk-proc-info -- --stats --xstats`
*   `lspci -vvv -s <pci_id>`: Check PCIe link speed, width (LnkSta), and hardware memory configuration of targeted interface.
