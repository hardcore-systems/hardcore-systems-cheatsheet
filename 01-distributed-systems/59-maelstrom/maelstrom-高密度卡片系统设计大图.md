# maelstrom-高密度卡片系统设计大图

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

    C1[Card 1: Clojure JVM 运行时]:::M1 --> C2[Card 2: STDIN/STDOUT 文本协议]:::M1
    C2 --> C3[Card 3: 节点间消息通信]:::M1
    C3 --> C4[Card 4: 状态循环与并发]:::M1
    C3 --> C5[Card 5: JSON Schema RPC]:::M1

    C5 --> C8[Card 8: 事件生成器流水线]:::M2
    C6[Card 6: Knossos 线性化校验]:::M2 --> C9[Card 9: Checker 归纳引擎]:::M2
    C7[Card 7: Elle 事务一致性分析]:::M2 --> C9
    C8 --> C10[Card 10: 客户端代理物理并发]:::M2

    C9 --> C11[Card 11: 因果一致性验证]:::M3
    C11 --> C12[Card 12: 读未提交与已提交]:::M3
    C12 --> C13[Card 13: 可重复读]:::M3
    C13 --> C14[Card 14: 串行化一致性]:::M3
    C11 --> C15[Card 15: 单调读写约束]:::M3

    C14 --> C16[Card 16: Lin-KV 强一致存储]:::M4
    C14 --> C17[Card 17: Seq-KV 顺序一致存储]:::M4
    C17 --> C18[Card 18: Lww-Register 冲突合并]:::M4
    C17 --> C19[Card 19: PN-Counter 状态合并]:::M4
    C16 --> C20[Card 20: 副本日志同步]:::M4

    C20 --> C21[Card 21: 网络分区混沌注入]:::M5
    C21 --> C22[Card 22: 丢包率动态模拟]:::M5
    C21 --> C23[Card 23: 时延膨胀与乱序]:::M5
    C22 --> C24[Card 24: Nemesis 故障引入器]:::M5
    C23 --> C25[Card 25: 恢复重连与路由恢复]:::M5

    C25 --> C26[Card 26: Gnuplot 吞吐时延图]:::M6
    C25 --> C27[Card 27: History 历史日志追踪]:::M6
    C26 --> C28[Card 28: 命令行参数调优字典]:::M6
```

## 2. 源码符号映射
- `maelstrom.core` (Card 1, 2) - Clojure 主入口，标准输入输出解析器。
- `maelstrom.node` (Card 3, 4) - 节点间通信状态机，消息缓冲队列。
- `jepsen.checker` (Card 9, 11) - 一致性校验接口及 Elle 核心分析模块。
- `knossos.linear` (Card 6, 14) - Knossos WGL 算法进行线性化校验。
- `maelstrom.db` (Card 16, 17) - Lin-KV 与 Seq-KV 物理抽象桥接。
- `maelstrom.nemesis` (Card 21, 24) - 故障注入管理器（分区、丢包）。
