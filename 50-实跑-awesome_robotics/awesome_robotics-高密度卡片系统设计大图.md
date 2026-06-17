# awesome_robotics-高密度卡片系统设计大图.md

本文件定义了 **awesome-robotics (机器人系统与运动规划)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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

    Card1["Card 1: DH Parameters"]:::M1
    Card2["Card 2: Inverse Kinematics"]:::M1
    Card3["Card 3: Jacobian Matrix"]:::M1
    Card4["Card 4: Singularity Analysis"]:::M1
    Card5["Card 5: Rigid Body Dynamics"]:::M1
    
    Card6["Card 6: EKF Sensor Fusion"]:::M2
    Card7["Card 7: V-SLAM Front-end"]:::M2
    Card8["Card 8: Bundle Adjustment BA"]:::M2
    Card9["Card 9: ICP & NDT Laser SLAM"]:::M2
    Card10["Card 10: Loop Closure BoW"]:::M2

    Card11["Card 11: Dijkstra & A* Planning"]:::M3
    Card12["Card 12: RRT & RRT* Sampling"]:::M3
    Card13["Card 13: Minimum Snap Trajectory"]:::M3
    Card14["Card 14: GJK Collision Detection"]:::M3
    Card15["Card 15: Model Predictive Control MPC"]:::M3

    Card16["Card 16: PID Anti-Windup"]:::M4
    Card17["Card 17: Computed Torque Control"]:::M4
    Card18["Card 18: Impedance Control"]:::M4
    Card19["Card 19: Odometry Calibration"]:::M4

    Card20["Card 20: ROS 2 DDS QoS"]:::M5
    Card21["Card 21: tf2 Coordinate Trees"]:::M5
    Card22["Card 22: URDF & Gazebo Sim"]:::M5
    Card23["Card 23: Nav2 Stack Costmaps"]:::M5
    Card24["Card 24: RTPS latency Tuning"]:::M5

    Card25["Card 25: Cam-LiDAR Calibration"]:::M6
    Card26["Card 26: Reinforcement Learning RL"]:::M6
    Card27["Card 27: Point Cloud RANSAC"]:::M6
    Card28["Card 28: Mobile Robot Kinematics"]:::M6

    Card1 --> Card2
    Card2 --> Card3
    Card3 --> Card4
    Card4 --> Card5
    Card5 --> Card17
    Card6 --> Card7
    Card7 --> Card8
    Card8 --> Card10
    Card9 --> Card8
    Card11 --> Card13
    Card12 --> Card13
    Card13 --> Card15
    Card14 --> Card12
    Card15 --> Card16
    Card16 --> Card17
    Card17 --> Card18
    Card20 --> Card21
    Card21 --> Card23
    Card22 --> Card23
    Card23 --> Card24
    Card25 --> Card6
    Card27 --> Card9
    Card28 --> Card19
```

---

## 📍 Awesome-Robotics 物理源码位置映射

本设计大图的知识节点与自主机器人核心软件系统物理源码模块强关联：
1. **Kinematics & Dynamics**: ROS Orocos KDL (`orocos_kdl/src/`) / Pinocchio C++ 动力学库。
2. **SLAM Back-end**: g2o (`g2o/core/`) / Ceres Solver (优化后端)。
3. **Motion Planning**: OMPL (`ompl/src/ompl/geometric/planners/`) / MoveIt! 库。
4. **Control Core**: `ros2_control/` 硬件控制循环抽象接口。
5. **DDS & Middleware**: FastDDS (`fastdds/src/cpp/rtps/`) QoS 通信实现。
6. **Mobile Kinematics**: Nav2 Controller Server (`navigation2/nav2_controller/`)。
