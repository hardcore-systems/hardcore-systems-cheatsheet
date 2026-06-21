# Step1-判题材-java_design_patterns.md: 企业级经典设计模式与并发架构审计大纲

本审计项目聚焦于 Java 语言经典 GoF 设计模式、企业级设计架构与 JVM 底层高并发编程规范的系统工程。我们将通过 28 张核心知识卡片，深度解构创建型/结构型/行为型模式、J2EE 经典分层、JMM 内存模型、AQS 线程等待队列、锁状态膨胀升级与响应式编程，建立 L0-L2 梯级模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: 经典 GoF 创建型与结构型设计模式** (Cards 1–5) - `#4B5F7A` (Slate Blue)
    - 单例模式（DCL 与类加载）、工厂方法与抽象工厂、代理模式（JDK 动态代理与 CGLIB）、适配器与装饰器、外观与享元模式。
*   **M2: 经典 GoF 行为型设计模式** (Cards 6–10) - `#6B8272` (Muted Sage)
    - 观察者模式、策略与状态模式、模板方法与命令模式、责任链模式、中介者与备忘录模式。
*   **M3: 企业级 J2EE 与微服务设计模式** (Cards 11–15) - `#9C6666` (Tea Red)
    - DTO 与业务对象隔离、DAO 与 Repository 抽象、拦截过滤器与前端控制器、断路器与故障转移降级、CQRS 与事件溯源（Event Sourcing）。
*   **M4: JVM 核心并发基础与线程模型** (Cards 16–19) - `#7A7A7A` (Iron Grey)
    - Java 线程物理模型、JMM 重排序与 volatile 内存屏障、ThreadPoolExecutor 线程池复用与饱和拒绝、ThreadLocal 线程局部变量与内存泄漏防御。
*   **M5: 高级同步、无锁设计与锁优化** (Cards 20–24) - `#9A825A` (Dusty Gold)
    - ReentrantLock 基于 AQS 队列同步、CAS 原子操作与 ABA 冲突防御、ConcurrentHashMap 分段锁结构、偏向-轻量-重量锁状态膨胀、死锁检测与哲学家就餐防御。
*   **M6: 高级响应式与响应式流编程** (Cards 25–28) - `#755B77` (Muted Grape)
    - Reactive Streams 规范背压机制、Reactor 库 Flux 与 Mono 流、CompletableFuture 多任务异步依赖链路、ForkJoinPool 工作窃取算法。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    经典模式与并发架构的本质是通过多态接口规范对象交互（GoF 模式）以及利用 JVM 底层内存屏障与操作系统原语（AQS/锁升级）保障多线程数据一致性，实现健壮、松耦合的软件系统。
*   **L1 四句话逻辑**：
    1. **GoF 接口松耦合**：利用创建型、结构型和行为型 GoF 模式，通过继承与多态机制跨越业务变动，降低代码组件耦合。
    2. **企业级架构分层**：基于 J2EE 拦截过滤器及 CQRS 事件溯源，规范企业级多端数据交互流与微服务隔离层边界。
    3. **JVM 并发内存一致**：依托 JMM volatile 读写屏障，消除指令重排与缓存同步滞后，实现线程级强一致数据传递。
    4. **底层同步与锁竞争自愈**：利用 AQS 等待队列和重量锁膨胀机制，解决多物理线程高并发抢占临界资源的数据冲突与自旋损耗。
*   **L2 核心数据流转拓扑**：
    `业务请求触发` ➜ `拦截过滤器拦截校验` ➜ `JDK 动态代理转发` ➜ `Repository 仓储库交互` ➜ `多任务 CompletableFuture 异步并行` ➜ `AQS / CAS 原子同步抢占` ➜ `JVM 线程池工作线程执行`

---

## 🗂️ 28 张核心知识卡片大纲
1. 单例（Singleton）模式：双重检查锁定（DCL）中 volatile 防止指令重排机制及类加载器线程安全保障。
2. 工厂（Factory）与抽象工厂模式：解耦对象创建与多产品族扩展规范。
3. 代理（Proxy）模式：JDK 动态代理类字节码生成与 CGLIB 子类代理原理。
4. 适配器（Adapter）与装饰器（Decorator）模式：组合与继承结构装配开销。
5. 外观（Facade）与享元（Flyweight）模式：利用共享池减少 JVM 堆内存占用。
6. 观察者（Observer）模式与发布-订阅机制事件解耦逻辑。
7. 策略（Strategy）与状态（State）模式：多分支判定解耦与状态转换时序控制。
8. 模板方法（Template Method）与命令（Command）模式：事务流标准化抽象。
9. 责任链（Chain of Responsibility）模式：网关请求拦截过滤器管道构建。
10. 中介者（Mediator）与备忘录（Memento）模式：多组件通信解耦与回滚设计。
11. 数据传输对象（DTO）与业务实体解耦设计模式及数据装配。
12. 数据访问对象（DAO）与仓储（Repository）模式：数据访问隔离设计。
13. 拦截过滤器（Intercepting Filter）与前端控制器（Front Controller）模式。
14. 微服务断路器（Circuit Breaker）与故障转移降级设计模式。
15. CQRS（读写分离）与事件溯源（Event Sourcing）高扩展一致性模型。
16. Java 线程模型：OS 内核线程（1:1）射及线程栈帧变量生命周期。
17. Java 内存模型（JMM）：volatile 关键字读写屏障与指令重排安全策略。
18. 线程池（ThreadPoolExecutor）工作线程复用、工作队列排队及饱和拒绝策略。
19. ThreadLocal 线程局部变量存储原理与弱引用 key 导致内存泄漏防御。
20. ReentrantLock 显式锁：基于 AQS（AbstractQueuedSynchronizer）CLH 双向队列同步状态机。
21. CAS（Compare-And-Swap）无锁原子自旋操作与 ABA 冲突防范（版本号机制）。
22. ConcurrentHashMap 并发容器：分段锁、Node 数组 CAS 抢占与红黑树重平衡。
23. Synchronized 锁优化：偏向锁、轻量级自旋锁与重量级 OS 互斥锁膨胀升级过程。
24. 死锁（Deadlock）检测：等待关系图（Wait-for Graph）环路判定与预防规避。
25. 响应式编程流规约（Reactive Streams）：有界背压（Backpressure）双向反馈机制。
26. Reactor 框架：异步冷流 Flux 与单流 Mono 调度器切换（publishOn/subscribeOn）。
27. CompletableFuture 异步编排：多任务异步流水线组合与异常拦截。
28. ForkJoinPool 并行框架：工作窃取（Work-Stealing）算法双端队列与分治计算。
