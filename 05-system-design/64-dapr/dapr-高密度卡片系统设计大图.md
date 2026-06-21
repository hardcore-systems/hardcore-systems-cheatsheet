# dapr-高密度卡片系统设计大图

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

    C1[Card 1: App-to-Sidecar Bridge]:::M1 --> C2[Card 2: gRPC/HTTP API Abstraction]:::M1
    C2 --> C4[Card 4: Sidecar Process Proxy]:::M1
    C1 --> C3[Card 3: Port Lifecycle Hook]:::M1
    C4 --> C5[Card 5: Seamless App Integration]:::M1

    C5 --> C6[Card 6: Pluggable State Stores]:::M2
    C6 --> C7[Card 7: ETag Optimistic Locking]:::M2
    C6 --> C8[Card 8: State API Transactions]:::M2
    C7 --> C9[Card 9: Distributed Lock API]:::M2
    C8 --> C10[Card 10: State Bulk Operations]:::M2

    C10 --> C11[Card 11: CloudEvents Protocol]:::M3
    C11 --> C12[Card 12: Pluggable Msg Broker]:::M3
    C12 --> C13[Card 13: Subscription Registry]:::M3
    C11 --> C14[Card 14: Retry Backoff Delivery]:::M3
    C13 --> C15[Card 15: Dead Letter Queue DLQ]:::M3

    C14 --> C16[Card 16: Virtual Actor Lifecycle]:::M4
    C16 --> C17[Card 17: Actor Placement Engine]:::M4
    C16 --> C18[Card 18: Actor Consistency State]:::M4
    C18 --> C19[Card 19: Timers & Reminders]:::M4
    C17 --> C20[Card 20: Concurrent Execution Lock]:::M4

    C18 --> C21[Card 21: Input Binding Listen]:::M5
    C21 --> C22[Card 22: Output Binding Trigger]:::M5
    C21 --> C23[Card 23: Declarative Component Config]:::M5
    C22 --> C24[Card 24: Bidirectional Data Stream]:::M5

    C24 --> C25[Card 25: mTLS Mutual Auth]:::M6
    C25 --> C26[Card 26: SPIFFE Identity Mgmt]:::M6
    C26 --> C27[Card 27: Circuit Breakers & Retry]:::M6
    C27 --> C28[Card 28: OpenTelemetry Tracking]:::M6
```

## 2. 源码符号映射
- `dapr/pkg/runtime/runtime.go` (Card 1, 2, 4) - Dapr Sidecar 运行时初始化、端口管理、API 服务装载。
- `dapr/pkg/components/state/registry.go` (Card 6, 8) - 可插拔状态管理组件中心注册、状态数据流序列化。
- `dapr/pkg/messaging/pubsub/pubsub.go` (Card 11, 12) - 消息发布订阅组件交互、CloudEvents 标准事件解析。
- `dapr/pkg/actors/placement.go` (Card 16, 17) - 虚拟 Actor 多实例位置 Placement 位置调度逻辑。
