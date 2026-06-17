# sre_book-高密度卡片系统设计大图 (SRE 运维与架构稳定性圣经)

本大图描绘了 `google / sre-book` 库中 28 张核心卡片的相互依赖和数据/控制流关系，涵盖了可用性管理、自动化告警响应、过载雪崩级联防御、瞬态重试自愈以及运维自动化效率指标等高可用核心拼图。

## 1. 卡片依赖拓扑 (Mermaid Diagram)

```mermaid
graph TD
    %% M1: 可用性与错误预算
    C1["Card 1: SLI 指标定义"] --> C2["Card 2: SLO 目标确立"]
    C2 --> C3["Card 3: 错误预算与释放冲突"]
    C1 --> C4["Card 4: 白盒与黑盒监控"]
    
    %% M2: 告警与故障响应
    C3 --> C5["Card 5: 多窗口多燃烧率告警"]
    C5 --> C6["Card 6: 告警降噪 Paging vs Ticket"]
    C6 --> C7["Card 7: 故障响应 On-call 与 MTTR"]
    C7 --> C8["Card 8: 无指责事后剖析与 5 Whys"]
    
    %% M3: 过载控制与级联防御
    C2 --> C9["Card 9: 级联故障与队列积压"]
    C9 --> C10["Card 10: 谷歌客户端自适应限流"]
    C9 --> C11["Card 11: 服务端优雅降级 (Load Shedding)"]
    C11 --> C12["Card 12: 调用超时预算与死线传导"]
    C11 --> C13["Card 13: 慢调用与冷启动热机 (Warm-up)"]
    
    %% M4: 瞬态失效与降级重试
    C10 --> C14["Card 14: 重试风暴与重试熔断额度"]
    C14 --> C15["Card 15: 指数退避与随机扰动 Jitter"]
    C15 --> C16["Card 16: 幂等性设计与全局锁"]
    C11 --> C17["Card 17: 特性开关 (Feature Toggles) 降级"]
    
    %% M5: 容量规划与负载均衡
    C2 --> C18["Card 18: 全局负载均衡 (GSLB)"]
    C18 --> C19["Card 19: 容量规划与极限压测水位"]
    C18 --> C20["Card 20: 客户端负载与连接子集 Subsetting"]
    C19 --> C21["Card 21: 优雅停机与流量排干 (Draining)"]
    
    %% M6: 运维自动化与琐事消除
    C22["Card 22: 运维琐事 (Toil) 研发硬约束"] --> C23["Card 23: IaC 与 GitOps 自愈"]
    C23 --> C24["Card 24: 灰度发布 Canary 与自动回滚"]
    C24 --> C25["Card 25: 自动恢复脚本防反馈风暴"]
    C24 --> C26["Card 26: 灾难演练 (DiRT) 主动防御"]
    C22 --> C27["Card 27: 亚线性扩展与 SRE 衡量指标"]
    C22 --> C28["Card 28: 变更管理小步迭代与回滚"]
    
    %% Style Classes
    classDef m1 fill:#7A8B99,stroke:#333,stroke-width:1px,color:#fff;
    classDef m2 fill:#7D8F7B,stroke:#333,stroke-width:1px,color:#fff;
    classDef m3 fill:#9E828A,stroke:#333,stroke-width:1px,color:#fff;
    classDef m4 fill:#B58A7D,stroke:#333,stroke-width:1px,color:#fff;
    classDef m5 fill:#5F7582,stroke:#333,stroke-width:1px,color:#fff;
    classDef m6 fill:#BFA88F,stroke:#333,stroke-width:1px,color:#fff;
    
    class C1,C2,C3,C4 m1;
    class C5,C6,C7,C8 m2;
    class C9,C10,C11,C12,C13 m3;
    class C14,C15,C16,C17 m4;
    class C18,C19,C20,C21 m5;
    class C22,C23,C24,C25,C26,C27,C28 m6;
```

---

## 2. 核心架构算法与物理公式映射

*   **Error Budget 消耗速度与多燃烧率告警 (Multi-burn-rate Alerting)**：
    *   SRE 告警的核心在于监控错误预算消耗速度（即燃烧率 Burn Rate）。
    *   燃烧率定义为消耗全部错误预算的理论速度倍数。如一个月的 $99\%$ SLO，错误预算为 $1\%$。如果系统在 1 小时内发生 $100\%$ 的全部故障，则燃烧率 $B = 30 \times 24 = 720$。
    *   多燃烧率告警规则定义了在特定窗口内发生高燃烧率时立刻 paging。例如：如果在 1 小时内燃烧率 $\ge 14.4$（消耗了 $2\%$ 预算）或 6 小时内燃烧率 $\ge 6$（消耗了 $5\%$ 预算），则触发紧急 paging。
*   **Google 客户端自适应限流算法 (Adaptive Client Throttling)**：
    *   当后端过载开始报错时，为避免流量持续积压拖死服务，客户端应就地丢弃一部分请求。
    *   丢弃概率公式定义为：
        $$P_{\text{reject}} = \max\left(0, \frac{\text{requests} - K \times \text{accepts}}{\text{requests} + 1}\right)$$
        *   `requests` 是客户端发起的总请求数；
        *   `accepts` 是后端成功处理并返回的请求数；
        *   `K` 是自适应激进系数（通常为 $1.1$ 到 $2.0$，如 $2$ 表示允许客户端发起超出接受量两倍的超载试探）。
    *   该公式使客户端在后端成功率下跌时迅速呈指数级自动本地抛弃请求，在后端恢复时逐步放行。
*   **指数退避与随机扰动 (Exponential Backoff with Jitter)**：
    *   重试间隔计算公式：
        $$\text{sleep} = \min(\text{max\_sleep}, \text{base} \times 2^{\text{attempt}})$$
    *   为防止重试时所有客户端同时发包形成 thundering herd，必须加入随机扰动（Jitter）：
        $$\text{sleep} = \text{random}(0, \min(\text{max\_sleep}, \text{base} \times 2^{\text{attempt}}))$$
