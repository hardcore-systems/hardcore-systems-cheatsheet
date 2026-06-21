# ros2-高密度卡片系统设计大图.md

本文件定义了 **ros2 (机器人 DDS 实时中间件与共享内存)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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

    Card1["Card 1: Node Lifecycle"]:::M1
    Card2["Card 2: RMW Abstraction"]:::M1
    Card3["Card 3: DDS Discovery"]:::M1
    Card4["Card 4: DDS QoS Policies"]:::M1
    Card5["Card 5: Intra-Process Comm"]:::M1
    Card6["Card 6: Zero-Copy Shm"]:::M1
    Card7["Card 7: Topics & CDR"]:::M2
    Card8["Card 8: Async Services"]:::M2
    Card9["Card 9: Action Framework"]:::M2
    Card10["Card 10: Executor Engine"]:::M2
    Card11["Card 11: Dynamic Parameters"]:::M2
    Card12["Card 12: Lifecycle Nodes"]:::M2
    Card13["Card 13: Launch System"]:::M3
    Card14["Card 14: MCAP/Bag Recording"]:::M3
    Card15["Card 15: Client C Library (rcl)"]:::M3
    Card16["Card 16: RMW C Library (rmw)"]:::M3
    Card17["Card 17: Coordinate Tree (tf2)"]:::M3
    Card18["Card 18: Real-Time Guarantee"]:::M3
    Card19["Card 19: Secure DDS"]:::M3
    Card20["Card 20: DDS Interoperability"]:::M4
    Card21["Card 21: Component Nodes"]:::M4
    Card22["Card 22: Time Sync"]:::M4
    Card23["Card 23: C++ API (rclcpp)"]:::M4
    Card24["Card 24: Python API (rclpy)"]:::M4
    Card25["Card 25: QoS Compatibility"]:::M5
    Card26["Card 26: Diagnostics & Monitor"]:::M5
    Card27["Card 27: micro-ROS (XRCE)"]:::M5
    Card28["Card 28: Gazebo Integration"]:::M5

    Card1 --> Card10
    Card2 --> Card16
    Card3 --> Card7
    Card4 --> Card25
    Card5 --> Card6
    Card6 --> Card7
    Card7 --> Card14
    Card8 --> Card9
    Card10 --> Card23
    Card10 --> Card24
    Card12 --> Card1
    Card15 --> Card23
    Card16 --> Card15
    Card7 --> Card17
    Card18 --> Card10
    Card19 --> Card3
    Card20 --> Card2
    Card21 --> Card5
    Card22 --> Card17
    Card25 --> Card7
    Card26 --> Card21
    Card27 --> Card2
    Card28 --> Card17
```

---

## 📂 核心代码物理映射锚点

在 `ros2` 源码库中，核心设计原则与卡片知识点在底层代码中有清晰的物理位置映射，供深入开发和审计参考：

*   `rclcpp::Node`: C++ 客户端库核心节点管理，驱动参数绑定及等待集分发。
*   `rcl::rcl`: C 语言客户端层，负责中间件解耦，处理时间、执行器状态调度。
*   `rmw::rmw`: 中间件接口层，封装 DDS 订阅发布动作，维护底层的网络发现状态。
*   `rcutils::rcutils`: 统一底层工具集，负责零内存分配内存池、日志排版格式控制。
*   `tf2::BufferCore`: tf2 位姿树后台存储核心，解算分布式机器人的高频坐标转换。
*   `rclcpp::executors::MultiThreadedExecutor`: 多线程调度器核心，分配 Callback Group 并管理内核级线程锁绑定。

---

## 🔬 Zone T2: 机器人分布式中间件与运行错误字典

*   `dds_qos_incompatible_match`: 发布端与订阅端的 QoS 策略不匹配（Offered vs Requested 违规），导致话题数据无法成功传输。
*   `shared_memory_segment_overflow`: iceoryx 共享内存数据段不足或环形缓冲区耗尽，引发零拷贝发布端借用失败（Loan Refused）。
*   `executor_deadlock_callback_stall`: 在同一个单线程 Executor 中发起了相互等待的重入式服务调用，导致调度线程死锁。
*   `realtime_priority_inversion`: 高优先级运动控制线程由于获取了低优先级节点持有的锁，引发优先级翻转而导致时间抖动。
*   `tf2_extrapolation_out_of_range`: 请求解算的坐标变换时间戳超出了缓冲区窗口，引发 tf2 外推边界超时异常。
*   `lifecycle_invalid_state_transition`: 试图在非受控节点状态下触发非法生命周期跳变，被 Lifecycle 状态机硬件抛出拒绝警报。
