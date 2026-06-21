# 机器人运动规划与 SLAM 系统工程速查海报

## J-Ladder 阶梯逻辑模型

### L0 一句话本质
机器人系统工程的本质是通过状态估计（SLAM）获取空间姿态，通过路径规划（RRT/Minimum Snap）生成安全曲线，并通过闭环反馈控制（CTC/Impedance）输出关节力矩，实现物理实体高精度自主作业。

### L1 四句话逻辑
1. **空间映射与定位**：机器人通过激光雷达与相机采集特征，借助 EKF 及图优化 SLAM 构建高精度空间地图并实现亚厘米级自定位。
2. **全局与局部规划**：利用 A* 算法在大图上搜索全局骨架，通过 RRT* 或 Minimum Snap 生成平滑且无碰撞的物理轨迹曲线。
3. **关节控制与驱动**：基于正逆运动学（IK）解算末端坐标，利用计算力矩（CTC）或阻抗控制输出电机脉冲，精确追踪规划曲线。
4. **中间件分布式协同**：通过 ROS 2 的 DDS 实时通信总线，将感知、规划与底盘控制解耦，保障分布式节点的低时延数据同步。

### L2 核心数据流转拓扑
`传感器输入` ➜ `EKF 状态估计 / SLAM` ➜ `全局 Costmap A* 规划` ➜ `Minimum Snap 轨迹优化` ➜ `逆运动学（IK）解算` ➜ `计算力矩控制（CTC）` ➜ `电机执行器`

---

## 📂 核心知识卡片 (Cards 1-28)

### Card 1: Denavit-Hartenberg (D-H) 参数化连杆建模规范
*   **核心原理**: D-H 法是建立机器人连杆坐标系的经典手段。通过四个几何参数：连杆长度 $a_i$、连杆转角 $\\alpha_i$、连杆偏置 $d_i$ 和关节角 $\\theta_i$，定义从第 $i-1$ 到第 $i$ 关节的齐次变换矩阵。
*   **技术细节**: 齐次变换矩阵通过绕 Z 轴旋转与平移、绕 X 轴旋转与平移四个步骤级联而成：$T_i^{i-1} = Rot_z(\\theta_i) \cdot Trans_z(d_i) \cdot Trans_x(a_i) \cdot Rot_x(\\alpha_i)$。
*   **源码位置**: [kdl/models.cpp:L30-65](file:///orocos_kdl/src/models.cpp#L30-65)

### Card 2: 逆运动学（IK）解析解与数值迭代解法
*   **核心原理**: 逆运动学（IK）是从期望的末端执行器位姿解算出各关节角度的过程。解析解使用代数或几何法（若满足 Pieper 准则，即三相邻关节轴线交于一点，存在封闭解）；数值解采用迭代法。
*   **技术细节**: 数值迭代法通常采用 Newton-Raphson 法，在当前关节角 $\\theta$ 下，利用伪逆雅可比矩阵迭代修正关节角增量：$\Delta\\theta = J^{\\dagger}(\\theta) \cdot \Delta x$，直到误差收敛。
*   **源码位置**: [kdl/chainiksolverpos_nr.cpp:L50-90](file:///orocos_kdl/src/chainiksolverpos_nr.cpp#L50-90)

### Card 3: 机器人雅可比矩阵（Jacobian）与速度/力传导映射
*   **核心原理**: 雅可比矩阵 $J(\\theta)$ 描述了从关节空间速度 $\\dot{q}$ 到操作空间末端速度 $\\dot{x}$ 的一阶微分映射关系：$\\dot{x} = J(\\theta)\\dot{q}$。
*   **技术细节**: 根据虚功原理，雅可比转置矩阵亦决定了关节驱动力矩 $\\tau$ 与末端静外力 $F$ 的传递关系：$\\tau = J^T(\\theta)F$。雅可比计算常采用几何雅可比迭代求取。
*   **源码位置**: [kdl/chainjnttojacsolver.cpp:L20-75](file:///orocos_kdl/src/chainjnttojacsolver.cpp#L20-75)

### Card 4: 雅可比行列式奇异状态（Singularity）判定与可操作度
*   **核心原理**: 当雅可比矩阵 $J$ 降秩（$\det(J) = 0$）时，机器人处于奇异状态，此时末端在某些物理方向上丢失自由度，且该方向所需的关节速度趋近于无穷大。
*   **技术细节**: 奇异状态通过计算可操作度（Manipulability）指标 $w = \sqrt{\det(JJ^T)}$ 评估，接近 0 时采用阻尼最小二乘法（Damped Least Squares）截断奇异值避免失控。
*   **源码位置**: [kdl/chainiksolvervel_wdls.cpp:L80-130](file:///orocos_kdl/src/chainiksolvervel_wdls.cpp#L80-130)

### Card 5: 刚体动力学 Newton-Euler 递推公式与 Euler-Lagrange 动能建模
*   **核心原理**: Euler-Lagrange 方法基于能量观点建模，计算关节空间动能和势能，生成形式简洁的封闭动力学方程；Newton-Euler 采用前向递推速度加速度，后向递推力矩，具有 $O(N)$ 计算效率。
*   **技术细节**: 多连杆动力学标准方程写为：$M(q)\\ddot{q} + C(q,\\dot{q})\\dot{q} + G(q) = \\tau$，其中 $M$ 为质量矩阵，$C$ 为科氏力及向心力矩阵，$G$ 为重力项。
*   **源码位置**: [kdl/chaindynparam.cpp:L40-95](file:///orocos_kdl/src/chaindynparam.cpp#L40-95)

### Card 6: 扩展卡尔曼滤波（EKF）在多传感器融合定位中的协方差传播
*   **核心原理**: EKF 通过在一阶泰勒展开点对非线性非均匀系统状态方程和观测方程进行局部线性化，预测机器人当前姿态并更新协方差矩阵。
*   **技术细节**: 状态预测：$X_{k|k-1} = f(X_{k-1}, u_k)$，协方差预测：$P_{k|k-1} = F_k P_{k-1} F_k^T + Q$，引入传感器观测后，通过计算卡尔曼增益 $K_k$ 修正状态量。
*   **源码位置**: [robot_localization/ekf.cpp:L50-110](file:///robot_localization/src/ekf.cpp#L50-110)

### Card 7: Visual SLAM 前端特征检测与光流追踪
*   **核心原理**: 前端负责从相邻相机图像中估计瞬时相机运动（里程计）。通过检测图像特征点（如 ORB，利用 FAST 算子与 BRIEF 描述子匹配）或使用光流法追踪图像强度梯度。
*   **技术细节**: KLT 光流法不计算描述子，直接在图像间最小化灰度平方和来追踪特征点运动：$\sum [I_{t+1}(x+u, y+v) - I_t(x, y)]^2 \rightarrow \min$。
*   **源码位置**: [orb_slam3/Frame.cc:L100-160](file:///orb_slam3/src/Frame.cc#L100-160)

### Card 8: 视觉后端 Bundle Adjustment (BA) 最小二乘与 g2o 图优化
*   **核心原理**: 后端旨在从多帧图像和路标点中消除累积漂移。BA 通过最小化所有三维路标点在相机成像面上的重投影误差来联合优化相机位姿与路标点三维坐标。
*   **技术细节**: 目标函数：$E = \sum \|x_{ij} - \pi(T_i, p_j)\|^2_{H}$。图优化中，位姿与路标作为图节点，重投影误差作为边，利用 g2o 库采用 Levenberg-Marquardt 和稀疏 Schur 补求解。
*   **源码位置**: [g2o/core/sparse_optimizer.cpp:L40-85](file:///g2o/core/sparse_optimizer.cpp#L40-85)

### Card 9: LiDAR SLAM 的 ICP 与 NDT 扫描匹配算法
*   **核心原理**: 激光里程计通过对齐当前帧点云（Scan）与局部地图（Map）估算相对位移。ICP（迭代最近点）最小化对应点几何欧氏距离；NDT（正态分布变换）最大化点在网格高斯分布下的概率。
*   **技术细节**: ICP 迭代公式：$E(R, T) = \sum \|p_i - (R q_i + T)\|^2$，通过奇异值分解（SVD）直接求解出旋转矩阵 $R$ 与平移向量 $T$。
*   **源码位置**: [pcl/registration/icp.hpp:L80-140](file:///pcl/registration/icp.hpp#L80-140)

### Card 10: 回环检测与 Bag of Words (BoW) 词袋模型检索
*   **核心原理**: 回环检测判定机器人是否回到了历史到过的位置。DBoW（如 DBoW2）通过将图像特征点聚类成视觉词汇，转换为稀疏向量，以树状检索加速图像匹配。
*   **技术细节**: 计算当前帧向量与历史帧向量的余弦相似度（TF-IDF 权重），相似度超过阈值则代表检测到回环，随后利用 Sim3/SE3 求解闭环约束并更新全局因子图。
*   **源码位置**: [orb_slam3/LoopClosing.cc:L30-90](file:///orb_slam3/src/LoopClosing.cc#L30-90)

### Card 11: 图搜索算法：Dijkstra 路径开销计算与 A* 启发式函数估算
*   **核心原理**: Dijkstra 算法通过累加路径边权重搜索从起点到全图所有节点的最短路径；A* 引入启发式距离函数 $h(n)$，引导算法优先朝目标方向扩展以减少盲目搜索量。
*   **技术细节**: A* 估价公式：$f(n) = g(n) + h(n)$。其中 $g(n)$ 为起点到当前节点的实际代价值，$h(n)$ 常采用曼哈顿距离（Manhattan）或欧氏距离（Euclidean）。
*   **源码位置**: [nav2_astar_planner/astar_planner.cpp:L70-120](file:///navigation2/nav2_astar_planner/src/astar_planner.cpp#L70-120)

### Card 12: 采样规划算法：RRT 随机采样树构建与 RRT* 渐进最优剪枝
*   **核心原理**: RRT 算法通过向空间随机撒播采样点，并向其延展最近的树节点，快速避障生成骨架；RRT* 在生成路径时引入“邻域重接”（Rewire）机制，使路径长度收敛于全局最优值。
*   **技术细节**: Rewire 步骤：在当前新插入的节点 $x_{new}$ 邻域内寻找潜在父节点，若通过新节点连接能缩短历史邻近节点的累积路径 $g(x)$，则重置这些邻近节点的父指针。
*   **源码位置**: [ompl/geometric/planners/rrt/RRTstar.cpp:L100-160](file:///ompl/geometric/planners/rrt/RRTstar.cpp#L100-160)

### Card 13: 轨迹生成：五次多项式样条曲线与 Minimum Snap 凸优化求解
*   **核心原理**: 路径点只是几何离散点，需要生成对时间连续的一阶（速度）、二阶（加速度）、三阶（Jerk）及四阶导数（Snap）平滑的连续轨迹。
*   **技术细节**: Minimum Snap 将规划表示为多段多项式插值问题：$p(t) = c_0 + c_1 t + \dots + c_n t^n$，在给定走廊边界约束下，将 Snap 平方积分最小化表示为二次规划（QP）求解。
*   **源码位置**: [planning/minimum_snap.cpp:L20-75](file:///planning/minimum_snap.cpp#L20-75)

### Card 14: GJK 凸多面体碰撞检测算法
*   **核心原理**: GJK 算法用于极速判定两个凸多面体是否发生穿透。它通过求取两个凸包的闵可夫斯基差（Minkowski Difference），将碰撞检测转换为寻找差集多面体距原点的最近距离。
*   **技术细节**: 利用 Support 函数迭代构建单纯形（点、线段、三角形或四面体），若原点包含在生成的单纯形内部，则代表两凸包发生了碰撞交叠。
*   **源码位置**: [fcl/narrowphase/detail/gjk.cpp:L30-80](file:///fcl/narrowphase/detail/gjk.cpp#L30-80)

### Card 15: 模型预测控制（MPC）在轨迹追踪中的滚动时域优化
*   **核心原理**: MPC 在每个决策时刻基于系统动力学状态方程，在向前预测时域内求解带约束的最优控制问题，但仅将计算出的第一步控制输入施加给系统。
*   **技术细节**: 目标函数最小化末端追踪误差与控制量惩罚之和：$\sum \|x_{k+i} - x_{ref}\|^2_Q + \|u_{k+i}\|^2_R$，并在线滚动求解 QP 或非线性规划，以应对轮式滑移与动力学饱和。
*   **源码位置**: [nav2_mppi_controller/mppi.cpp:L100-150](file:///navigation2/nav2_mppi_controller/src/mppi.cpp#L100-150)

### Card 16: PID 控制器积分饱和（Windup）抑制与前馈补偿机制
*   **核心原理**: 关节执行器（如伺服电机）存在最大物理输出限制。当系统存在持续偏差时，PID 中的积分项会无限累加导致积分饱和，导致超调过大及系统振荡。
*   **技术细节**: 抑制机制包括：积分限幅（Clamping）法（当驱动器饱和时暂停积分运算）或逆推抗饱和法（Anti-windup Back-calculation）。重力等可预测分量则通过前馈（Feedforward）直接加到控制器输出中。
*   **源码位置**: [control_toolbox/pid.cpp:L40-80](file:///control_toolbox/src/pid.cpp#L40-80)

### Card 17: 计算力矩控制（CTC）多关节解耦与动力学前馈
*   **核心原理**: CTC 是一种针对非线性多关节机械臂的经典反馈线性化控制算法。它通过实时估算关节重力、离心力与惯性，抵消动力学非线性，将系统转化为等效的双积分器线性系统。
*   **技术细节**: 控制力矩公式：$\\tau = M(q)(\\ddot{q}_d + K_p e + K_d \\dot{e}) + C(q,\\dot{q})\\dot{q} + G(q)$。其中 $e = q_d - q$。在伺服周期内，CTC 实现了各关节的完全解耦控制。
*   **源码位置**: [ros2_control/joint_trajectory_controller.cpp:L120-170](file:///ros2_control/joint_trajectory_controller.cpp#L120-170)

### Card 18: 阻抗控制（Impedance Control）与导纳控制力反馈交互
*   **核心原理**: 机械臂执行打磨、装配等接触作业时，不能采用刚性位置控制（容易损坏工件）。阻抗控制通过调节机器人的动态质量 $M_{d}$、阻尼 $B_{d}$ 和刚度 $K_{d}$ 参数，模拟弹簧-阻尼-质量行为。
*   **技术细节**: 动力学表现建模为：$M_d \\ddot{e} + B_d \\dot{e} + K_d e = F_{ext}$。阻抗控制直接根据位置误差输出外力阻抗，导纳控制则根据测得的外力 $F_{ext}$ 重新计算生成期望的位置修正轨迹。
*   **源码位置**: [control/impedance_controller.cpp:L20-65](file:///control/impedance_controller.cpp#L20-65)

### Card 19: 轮式里程计校准与滑移误差建模
*   **核心原理**: 移动机器人通过编码器累加左右轮转角计算位姿，但轮径误差和轮距（Track）误差会导致严重的里程计累计漂移，需对其运动学参数进行在线或离线校准。
*   **技术细节**: 对两轮差速底盘，左/右轮线速度为 $v_l, v_r$。底盘线速度 $v = (v_r + v_l)/2$，角速度 $\omega = (v_r - v_l)/b$（$b$ 为轮距）。滑移率 $s = (v_w - v_r)/v_w$ 引入非完整性约束偏置修正。
*   **源码位置**: [diff_drive_controller/diff_drive_controller.cpp:L150-210](file:///diff_drive_controller/diff_drive_controller.cpp#L150-210)

### Card 20: ROS 1/2 通信总线机制与 DDS 中间件服务等级 QoS
*   **核心原理**: ROS 2 取代了 ROS 1 中依赖 Master 的 XML-RPC 通信架构，直接在工业级 DDS (Data Distribution Service) 分布式实时总线上建立无中心节点的对等网络。
*   **技术细节**: ROS 2 DDS 支持服务质量（QoS）策略，包括：Reliability（可靠级/尽力而为）、Durability（暂存历史数据/只发新数据）、Liveliness（活跃度心跳监控）和 Deadline（规定时延内必须有数据包）。
*   **源码位置**: [rmw_fastrtps_cpp/rmw_node.cpp:L40-95](file:///rmw_fastrtps_cpp/src/rmw_node.cpp#L40-95)

### Card 21: tf2 实时坐标变换树计算机制与四元数线性插值
*   **核心原理**: 机器人各部位（雷达、相机、底盘、世界坐标）存在大量坐标系。tf2 维护了一棵分布式多叉坐标变换树，支持查询任意两个坐标系在历史指定时间戳下的相对位姿。
*   **技术细节**: tf2 内部采用四元数进行三维旋转插值。对于位姿树插值，使用 Slerp (球面线性插值) 保证旋转速度均匀：$Slerp(q_0, q_1, t) = \frac{\sin((1-t)\theta)}{\sin\theta}q_0 + \frac{\sin(t\theta)}{\sin\theta}q_1$。
*   **源码位置**: [tf2/src/buffer_core.cpp:L10-60](file:///tf2/src/buffer_core.cpp#L10-60)

### Card 22: URDF 物理碰撞模型定义与 Gazebo 动力学物理引擎集成
*   **核心原理**: URDF 文件使用 XML 描述机器人的关节类型（joint: continuous/revolute/fixed）及连杆的物理属性：几何外观（visual）、碰撞边界（collision）和转动惯量（inertial）。
*   **技术细节**: 物理引擎（如 ODE, Bullet）通过解析 URDF 中的惯性张量矩阵 $I$ 和质量 $m$ 对其施加刚体运动方程约束，模拟现实中的碰撞反弹及摩擦滑移。
*   **源码位置**: [urdf/urdf_parser/src/urdf_sensor.cpp:L10-50](file:///urdf/urdf_parser/src/urdf_sensor.cpp#L10-50)

### Card 23: ROS Navigation Stack (Nav2) 架构与分层 Costmap 地图
*   **核心原理**: Nav2 采用行为树（Behavior Trees）控制导航逻辑，将全局规划器（Planner）与局部控制器（Controller）解耦。地图处理使用分层 Costmap。
*   **技术细节**: Costmap 包括三层：Static Layer (静态栅格地图)、Obstacle Layer (传感器实时避障层，使用三维体素 Voxel 网格更新) 和 Inflation Layer (膨胀层，防止连杆边缘擦碰)。
*   **源码位置**: [nav2_costmap_2d/costmap_2d.cpp:L450-510](file:///navigation2/nav2_costmap_2d/src/costmap_2d.cpp#L450-510)

### Card 24: DDS 实时发布/订阅协议（RTPS）在跨网络机器人控制中的延时调优
*   **核心原理**: DDS 底层采用有线可靠实时发布/订阅协议（RTPS）。当跨局域网传输高频传感器或控制报文时，默认的 IP 广播易导致延迟激增或丢包。
*   **技术细节**: 优化机制包括：关闭 multicast 组播路由改用 unicast 单播 Peers 直连列表；调整数据段大小以避免大 IP 包的分片重组；设置专属的 RTPS 数据发送接收线程优先级。
*   **源码位置**: [fastdds/rtps/participant/RTPSParticipantImpl.cpp:L20-100](file:///fastdds/src/cpp/rtps/participant/RTPSParticipantImpl.cpp#L20-100)

### Card 25: 相机与 LiDAR 时间戳硬同步与外参标定优化
*   **核心原理**: 多传感器融合要求相机快门触发与雷达扫描处于同一时间线（通过硬件 PTP/PPS 硬同步）。外参标定通过最小化 3D 棋盘格角点与重投影线的联合残差进行求解。
*   **技术细节**: 误差损失函数：$\sum \|X_{img} - P \cdot (T_{calib} \cdot X_{lidar})\|^2$，其中 $P$ 为相机内参投影矩阵，$T_{calib}$ 为待求的雷达至相机 3D 齐次标定矩阵，采用非线性最小二乘求解。
*   **源码位置**: [calibration/extrinsic_calibration.cpp:L10-45](file:///calibration/extrinsic_calibration.cpp#L10-45)

### Card 26: 强化学习在控制中的应用与 Sim-to-Real 迁移防御
*   **核心原理**: 深度强化学习（PPO, SAC）支持机器人自主学习高难度操控动作。但仿真器（Sim）与现实世界（Real）存在摩擦力、时延等物理鸿沟，需要防御 Sim-to-Real 的失效。
*   **技术细节**: 主要采用“域随机化”（Domain Randomization）方法，在训练中随机抖动摩擦系数、重力大小、物理链路质量及通信延时包，迫使神经网络学出泛化控制策略。
*   **源码位置**: [rl/ppo_control.py:L50-110](file:///rl/ppo_control.py#L50-110)

### Card 27: 点云滤波下采样（Voxel Grid）与 RANSAC 平面拟合分割
*   **核心原理**: 点云数据量巨大，常规算法无法做到高频处理。Voxel Grid 将三维空间网格化，保留每个体素网格内点云的重心；RANSAC 通过随机一致性采样，快速分割出地面等平面障碍。
*   **技术细节**: RANSAC 算法：随机选 3 个点拟合平面 $Ax+By+Cz+D=0$，统计该平面容差半径内的局内点（Inliers）数量，通过数千次迭代选出包含局内点最多的平面并将其剥离。
*   **源码位置**: [pcl/filters/voxel_grid.cpp:L100-140](file:///pcl/filters/voxel_grid.cpp#L100-140)

### Card 28: 移动底盘运动学约束：差速驱动、阿克曼转向与麦克纳姆轮建模
*   **核心原理**: 移动机器人底盘分为非完整性约束底盘（差速转向底盘、Ackermann 转向底盘）与全向移动底盘（Mecanum 轮底盘，依靠轮面上 45 度斜向辊子的受力分解实现侧移）。
*   **技术细节**: 麦克纳姆轮运动学逆解：由机器人整体速度 $v_x, v_y, \omega_z$ 映射求 4 轮转速 $\omega_i = \frac{1}{r}(v_x \pm v_y \pm (a+b)\omega_z)$，此关系在运动控制器中高频运行。
*   **源码位置**: [nav2_controller/kinematics.cpp:L10-40](file:///navigation2/nav2_controller/src/kinematics.cpp#L10-40)

---

## 📂 机器人控制系统与运动规划强度折衷矩阵

| 设计维度 | 运动学高精度追踪模式 (Strong Kinematics Pattern) | 自主快速探索模式 (Exploration Optimized Pattern) | 关键折衷分析 (Trade-off Matrix Analysis) |
| :--- | :--- | :--- | :--- |
| **机械臂控制** | 计算力矩控制 (CTC) 反馈线性化 | 经典单关节独立 PID 控制 | CTC 需要对机械臂连杆质量、惯性张量拥有极其精确的动力学先验模型，能够完全消除关节间多自由度非线性耦合，在高速运动下跟踪精度极高，但对动力学参数建模误差敏感；独立 PID 不需要动力学模型，鲁棒性高，但高速或重载下各关节干涉会导致轨迹失真。 |
| **避障路径规划** | 渐进最优采样规划 (RRT*) | 快速随机采样规划 (RRT) | RRT* 具备渐进最优性，规划生成的折线轨迹接近理论最短曲线，但需要海量重接 (Rewire) 计算，消耗 CPU；传统 RRT 只要找到可行路径便立刻返回，生成速度极快（~毫秒级），但路径极度冗余弯曲，难以供物理底盘直接追踪。 |
| **多传感器融合** | 基于图优化 (Factor Graph) 的 SLAM | 经典卡尔曼滤波 (EKF) 里程计 | 图优化维护了一个包含多帧观测和回环因子边的全局滑窗图谱，精度极高且具有回环自愈性，但计算资源开销极大，不适合微型控制器；EKF 里程计采用马尔可夫链单步递推，计算快且内存占用极低，但随着时间推移，无回环校验会导致累积误差无限发散。 |
| **三维避障地图** | 体素网格 (Voxel Grid) 3D Costmap | 二维栅格 (2D Grid) 2D Costmap | 体素网格能够精准感知悬挂障碍物及斜坡地形，但极耗显存与内存；2D 网格将全部障碍物强行压缩至单一平面，处理开销极小，但在室外三维起伏地段易发生盲区擦碰。 |

