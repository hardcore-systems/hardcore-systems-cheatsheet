# dae-高密度卡片系统设计大图

## 1. 卡片依赖拓扑图 (Mermaid)
```mermaid
graph TD
    classDef default fill:#1A202C,stroke:#24324F,color:#E2E8F0;
    classDef M1 fill:#4B5F7A,stroke:#2D3748,color:white;
    classDef M2 fill:#6B8272,stroke:#2D3748,color:white;
    classDef M3 fill:#9C6666,stroke:#2D3748,color:white;
    classDef M4 fill:#7A7A7A,stroke:#2D3748,color:white;
    classDef M5 fill:#9A825A,stroke:#2D3748,color:white;
    classDef M6 fill:#755B77,stroke:#2D3748,color:white;

    C1[Card 1: tc BPF Hook]:::M1 --> C2[Card 2: sockops Redirect]:::M1
    C2 --> C4[Card 4: SOCKMAP Table]:::M1
    C1 --> C3[Card 3: XDP Driver Filter]:::M1
    C4 --> C5[Card 5: bpf_msg_redirect Core]:::M1

    C5 --> C6[Card 6: Linux TProxy Mechanism]:::M2
    C6 --> C7[Card 7: Policy Routing ip rule]:::M2
    C6 --> C8[Card 8: conntrack BPF Hash]:::M2
    C7 --> C9[Card 9: NetNS Crossing Route]:::M2
    C8 --> C10[Card 10: Loopback Short-circuit]:::M2

    C10 --> C11[Card 11: Local DNS Spoofing]:::M3
    C11 --> C12[Card 12: Domain Split Trie]:::M3
    C12 --> C13[Card 13: IP Set Cache Map]:::M3
    C11 --> C14[Card 14: Fake IP Map Engine]:::M3
    C13 --> C15[Card 15: Dual Stack AAAA Route]:::M3

    C14 --> C16[Card 16: TCP/UDP Stream Splicing]:::M4
    C16 --> C17[Card 17: Outbound Interface Select]:::M4
    C16 --> C18[Card 18: Tunnel encapsulation VLESS]:::M4
    C18 --> C19[Card 19: UDP Over TCP Tunnel]:::M4
    C17 --> C20[Card 20: MTU Segment GSO]:::M4

    C18 --> C21[Card 21: cgo Dual Lang Bind]:::M5
    C21 --> C22[Card 22: Hot Reload config Map]:::M5
    C21 --> C23[Card 23: CLI Controller RPC]:::M5
    C22 --> C24[Card 24: Cgroup Intercept Link]:::M5

    C24 --> C25[Card 25: Ring Buffer Logs]:::M6
    C25 --> C26[Card 26: bpftool decompilation]:::M6
    C26 --> C27[Card 27: trace_pipe prints]:::M6
    C27 --> C28[Card 28: Performance Tunings]:::M6
```

## 2. 源码符号映射
- `dae/pkg/ebpf/c/dae.c` (Card 1, 2, 5) - eBPF 内核 C 语言主逻辑，TC 过滤及 SOCKMAP 路由。
- `dae/pkg/ebpf/c/dns.h` (Card 11) - DNS 过滤、Fake IP 重写规则。
- `dae/pkg/tproxy/tproxy.go` (Card 6, 7) - TProxy 用户态透明套接字接收。
- `dae/pkg/proxy/tunnel.go` (Card 18, 19) - 代理隧道建立及网络包封装协议。
