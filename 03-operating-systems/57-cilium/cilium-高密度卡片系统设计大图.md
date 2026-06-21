# cilium-高密度卡片系统设计大图.md

本文件定义了 **cilium (基于 eBPF 的高性能云原生网络与安全网关)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

---

## 🗺️ 28 张卡片依赖拓扑图 (Mermaid)

```mermaid
graph TD
    classDef default fill:#151d30,stroke:#24324f,color:#e2e8f0;
    classDef M1 fill:#4B5F7A,stroke:#2f3d52,color:white;
    classDef M2 fill:#6B8272,stroke:#4b5c50,color:white;
    classDef M3 fill:#9C6666,stroke:#704949,color:white;
    classDef M4 fill:#7A7A7A,stroke:#595959,color:white;
    classDef M5 fill:#9A825A,stroke:#6e5c40,color:white;
    classDef M6 fill:#755B77,stroke:#534054,color:white;

    Card1["Card 1: eBPF Load"]:::M1
    Card2["Card 2: LLVM eBPF Code"]:::M1
    Card3["Card 3: libbpf Verifier"]:::M1
    Card4["Card 4: CO-RE BTF"]:::M1
    Card5["Card 5: BPF Map Hash"]:::M1
    Card6["Card 6: Cilium Agent Comp"]:::M1
    
    Card7["Card 7: XDP Intercept"]:::M2
    Card8["Card 8: tc Hook"]:::M2
    Card9["Card 9: Header Parse"]:::M2
    Card10["Card 10: Conntrack Maps"]:::M2
    Card11["Card 11: Kernel NAT Routing"]:::M2
    Card12["Card 12: Tunnel Encapsulation"]:::M2

    Card13["Card 13: sockops Event"]:::M3
    Card14["Card 14: SOCKMAP Hash"]:::M3
    Card15["Card 15: msg_redirect Short"]:::M3
    Card16["Card 16: Local Pod Acceleration"]:::M3
    Card17["Card 17: Socket Lock Eliminate"]:::M3
    Card18["Card 18: UDS Compare"]:::M3

    Card19["Card 19: Maglev Hashing"]:::M4
    Card20["Card 20: VIP Mapping"]:::M4
    Card21["Card 21: LRU Session Affinity"]:::M4
    Card22["Card 22: DSR Return Routing"]:::M4

    Card23["Card 23: Identity Label"]:::M5
    Card24["Card 24: IP-to-Identity Trie"]:::M5
    Card25["Card 25: L3/L4/L7 Policy"]:::M5
    Card26["Card 26: IPsec & WireGuard"]:::M5

    Card27["Card 27: Envoy Sidecarless"]:::M6
    Card28["Card 28: Hubble Tracing"]:::M6

    Card1 --> Card3
    Card2 --> Card1
    Card3 --> Card4
    Card4 --> Card5
    Card6 --> Card2
    
    Card1 --> Card7
    Card1 --> Card8
    Card7 --> Card9
    Card8 --> Card9
    Card9 --> Card10
    Card10 --> Card11
    Card11 --> Card12

    Card8 --> Card13
    Card13 --> Card14
    Card14 --> Card15
    Card15 --> Card16
    Card16 --> Card17
    Card17 --> Card18

    Card11 --> Card19
    Card19 --> Card20
    Card20 --> Card21
    Card21 --> Card22

    Card9 --> Card23
    Card23 --> Card24
    Card24 --> Card25
    Card25 --> Card26

    Card25 --> Card27
    Card28 --> Card27
```

---

## 📍 Cilium 物理源码位置映射

本设计大图的知识节点与 Cilium 核心类库及 Crate 物理源码强关联：
1. **Cilium Agent**: `daemon/` 目录下的核心守护进程。
2. **eBPF Datapath C Code**: `bpf/` 目录下的内核 eBPF 代码，包括 `bpf_lxc.c`（LXC 数据路径钩子）和 `bpf_xdp.c`。
3. **Maglev Load Balancer**: `pkg/loadbalancer/` 与 `bpf/lib/lb.h`。
4. **Identity Policy Manager**: `pkg/policy/` 和 `bpf/lib/policy.h`。
