# dae Kernel-Layer eBPF Transparent Proxy Cheatsheet

## M1: eBPF Hooks & Network Architecture

### Card 1: tc BPF Hook & Ingress Interception
- **Mechanism**: dae attaches eBPF programs to the Linux Traffic Control (TC) clsact queue on physical interfaces, capturing packets at the early Ingress stage.
- **Speed**: Inspects and filters packets before standard TCP/IP stack evaluation, bypassing traditional iptables overhead.
- **Trade-off**: Requires eBPF-based IP reassembly to process fragmented packets, adding implementation complexity.

### Card 2: sockops Socket Event Redirection
- **Pathway**: Attaches to the `sockops` hook to intercept TCP events (ESTABLISHED, state changes).
- **Redirection**: Automatically captures socket details (IP/Port pairs) and registers them in a SOCKMAP.
- **Locality**: Bypasses file descriptor lookups and socket lock evaluations, reducing latency for local proxies.

### Card 3: XDP Driver-Level Packet Filtering
- **Firewall**: Integrates XDP to process packets directly inside network card driver queues.
- **Drop/TX**: Drops packets via `XDP_DROP` or bounces malicious traffic using `XDP_TX` without invoking softirqs.
- **Limitation**: Requires native driver support; fallback to generic mode loses kernel bypass speed.

### Card 4: SOCKMAP Table Architecture
- **Structure**: A SOCKMAP maps connection keys (IP/Port tuples) to physical socket pointers (`struct sock`).
- **Short-circuit**: Links incoming client connections to local proxy ports, routing packet streams directly.
- **Trade-off**: Requires configuring map sizes; write collisions slow down connection setups.

### Card 5: bpf_msg_redirect Short-circuit
- **Memory Copy**: Intercepts TCP transmissions via SOCKMAP hooks and redirects payloads via the `bpf_msg_redirect` helper.
- **Bypass**: Copies payloads directly from sender write buffers to receiver read buffers, bypassing the TCP/IP stack.
- **Benefit**: Achieves near-memory-speed latency for local loopback proxying.

---

## M2: Transparent Proxy (TProxy) & Routing

### Card 6: Linux TProxy Mechanism
- **TProxy**: Leverages `IP_TRANSPARENT` options to redirect incoming client packets to local proxies without editing destination IPs.
- **Decoupled**: Clients remain unaware of proxy presence, communicating with target servers transparently.
- **Trade-off**: Still allocates local file descriptors, consuming system socket resources.

### Card 7: Policy Routing & ip rule Tables
- **Routing**: Configures dedicated policy routing tables (e.g. table 100) and links firewall marks.
- **Interception**: Implements `ip rule add fwmark 1 lookup 100` to route marked packets to the loopback interface.
- **Warning**: Missing or misconfigured policy rules route packets normally, bypassing the proxy.

### Card 8: conntrack BPF Hash Map
- **Conntrack**: Standard conntrack suffers from global spinlock contention. dae uses a custom BPF hash map.
- **State tracking**: Logs connection lifecycles (SYN, ACK, FIN, RST) directly in eBPF maps.
- **Benefit**: Eliminates system conntrack locks, increasing concurrent connection rates.

### Card 9: NetNS Crossing Route
- **Containers**: In containerized systems, microservices reside in separate Network Namespaces (NetNS).
- **Interception**: Attaches eBPF to veth interfaces to route packets across namespace boundaries without bridge copy overhead.
- **Usage**: Requires scanning active veth interfaces at startup to attach BPF instances.

### Card 10: Loopback Interface Short-circuit
- **Loopback**: TProxy traffic is routed through the loopback interface (`lo`).
- **Short-circuit**: eBPF redirects packets at the loopback interface, bypassing standard routing.
- **Speed**: Reduces local proxy hop latency by over 15%.

---

## M3: DNS Spoofing & Split Routing

### Card 11: Local DNS Spoofing
- **Interception**: Monitors port 53 (DNS queries) and intercepts A/AAAA record lookups at Ingress.
- **Spoofing**: Constructs A-record DNS replies in the kernel to return Fake IPs in microseconds.
- **Trade-off**: Modifying packets in the kernel requires recomputing checksums, depending on checksum offloading.

### Card 12: Domain Split Prefix Tree
- **Split Routing**: Manages domain rules inside memory-optimized Trie (Prefix Tree) structures.
- **Matching**: Matches query domains against the Trie to decide routing (Direct, Proxy, Block).
- **Overhead**: Searching large Tries consumes CPU. Match results are cached in IP Sets to avoid redundant checks.

### Card 13: IP Set Cache Map
- **Cache Map**: Target IPs resolved from proxy domains are registered in a BPF `ip_set` map.
- **Direct Lookup**: Subsequent TCP connections bypass Trie lookups by checking the IP set directly.
- **Cleanup**: Implements TTL on BPF map items to purge stale IPs automatically.

### Card 14: Fake IP Mapping Engine
- **Fake IP**: Allocates synthetic IPs (e.g. `198.18.0.0/15`) for proxy-destined domains.
- **Map**: Maintains a bidirectional mapping table (`Fake IP <-> Domain`) in memory.
- **Routing**: When clients connect to a Fake IP, dae resolves the domain instantly to direct routing.

### Card 15: Dual Stack AAAA Route
- **Dual Stack**: Resolves A records (IPv4) and AAAA records (IPv6) concurrently.
- **Fake IPv6**: Maps IPv6 queries to synthetic IPv6 addresses.
- **Limitation**: Requires full end-to-end IPv6 routing support on the host system to prevent blackhole drops.

---

## M4: Protocol Handling & Tunneling

### Card 16: TCP/UDP Stream Splicing
- **Splicing**: Leverages eBPF stream parsers to append client payloads to proxy tunnel headers in the kernel.
- **Zero-Copy**: Bypasses user-space read/write cycles, saving memory bandwidth.
- **Trade-off**: Increases eBPF instruction counts. Large code segments require tail calls.

### Card 17: Outbound Interface Selection
- **Multi-WAN**: Selects the target interface based on path metrics and latency checks.
- **Bypass**: Employs `bpf_redirect` to route packets to chosen physical interfaces, bypassing normal routing tables.
- **Benefit**: Provides policy-driven interface steering without routing table latency.

### Card 18: Tunnel Encapsulation (VLESS/VMess/Trojan)
- **Tunnels**: Integrates proxy tunnel wrapping (VLESS, VMess, Trojan) inside dae.
- **Encapsulation**: Computes protocol headers in eBPF and appends them to outbound TCP payloads.
- **Throughput**: Batching packets minimizes encapsulation overhead and increases throughput.

### Card 19: UDP Over TCP Tunneling
- **UDP bypass**: Wraps UDP packets in a persistent TCP or HTTP/3 tunnel to bypass ISP throttling.
- **Reconstruction**: The remote proxy unpacks the payload and delivers raw UDP.
- **Trade-off**: TCP packet recovery delays cause HOL blocking, which can affect real-time applications.

### Card 20: MTU & Segment Offloading
- **MTU Limits**: Tunnel headers add bytes, risking packet splits when exceeding MTUs.
- **GSO**: Integrates Linux GSO (Generic Segmentation Offload) to process large frames before driver fragmentation.
- **Trade-off**: Requires driver GSO support; otherwise, packet splits consume CPU resources.

---

## M5: Go/C Integration & Management

### Card 21: cgo Dual-Language Integration
- **Design**: Implements the control plane in Go and the eBPF kernel program in C.
- **Loading**: Go uses the `cilium/ebpf` library to load compiled C binaries into the kernel.
- **Warning**: Go GC must not garbage collect memory referenced by pinned kernel maps.

### Card 22: Configuration Hot Reloading
- **Hot-swaps**: Swapping rules and configurations does not require restarting dae.
- **Incremental**: The Go daemon updates the kernel Trie and IP Set maps via system calls.
- **Benefit**: Modifies configurations on the fly without packet drops or connection resets.

### Card 23: CLI Controller RPC
- **Control**: dae exposes a Unix Domain Socket for RPC communication.
- **CLI**: The `dae-cli` utility queries eBPF map stats and list active connections.
- **Benefit**: Restricts controller communications to local socket pathways for security.

### Card 24: Cgroup Traffic Interception
- **Cgroups**: Attaches to Cgroupv2 directories to restrict proxying to specific application containers.
- **Benefit**: Provides fine-grained container steering.
- **Limitation**: Requires unified Cgroupv2 layouts; not supported on Cgroupv1.

---

## M6: Diagnostics & Diagnostics

### Card 25: eBPF Ring Buffer Telemetry
- **Telemetry**: Stream diagnostics logs to user space using `BPF_MAP_TYPE_RINGBUF`.
- **Lock-Free**: Shares memory ring buffers between the kernel and the Go daemon.
- **Warning**: Slow user-space readers cause the ring buffer to drop incoming kernel logs.

### Card 26: bpftool Decompilation & Maps Dump
- **bpftool**: The primary diagnostic tool for inspecting dae kernel states.
- **Commands**:
  - `bpftool prog show`: List loaded programs and instruction sizes.
  - `bpftool map dump`: Dump the contents of active Tries or conntrack maps.
- **Usage**: Verify that BPF maps are loaded correctly when routing rules fail.

### Card 27: trace_pipe Debug Logging
- **printk**: Logs kernel states (e.g. `ip_src`, `fwmark`) via `bpf_trace_printk`.
- **Command**: Run `cat /sys/kernel/debug/tracing/trace_pipe` to read trace logs.
- **Trade-off**: Debug logging slows down the Reactor loop; disable trace printing in production.

### Card 28: Performance Tuning & Map Sizing
- **Sizing**: BPF maps require static memory allocations at load time.
- **Tuning**: Large maps prevent hash collisions but increase kernel memory footprints.
- **Optimization**: Set `max_entries` based on host memory limits (e.g. router vs server).
