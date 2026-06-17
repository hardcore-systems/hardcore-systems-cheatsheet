# Robotics Motion Planning & SLAM System Engineering Cheatsheet

## J-Ladder Architecture Framework

### L0 Essence
The essence of robotic system engineering is to estimate spatial poses via SLAM, generate collision-free smooth trajectories via path planning (RRT*/Minimum Snap), and track these paths using feedback control (CTC/Impedance) to command joint actuators.

### L1 Logical Framework
1. **Mapping & Localization**: Robots extract spatial landmarks using cameras/LiDARs, resolving sub-centimeter poses via EKF and graph-optimizing SLAM.
2. **Global & Local Planning**: Generates global paths over grid maps using A*, optimizing them into dynamically smooth trajectories using Minimum Snap.
3. **Control & Actuation**: Resolves end-effector demands into joint configurations via IK, outputting motor commands using Computed Torque Control (CTC).
4. **Middleware Orchestration**: Coordinates sensors, planners, and drivers asynchronously over a distributed ROS 2 DDS real-time software backbone.

### L2 Core Data Flow Topology
`Sensor Inputs` ➜ `EKF / SLAM` ➜ `A* Costmap Planning` ➜ `Minimum Snap Optimization` ➜ `Inverse Kinematics (IK)` ➜ `Computed Torque Control (CTC)` ➜ `Actuators`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: Denavit-Hartenberg (D-H) Parametrization
*   **Core Principle**: D-H convention models robot links by mapping frame transitions via four parameters: link length $a_i$, link twist $\\alpha_i$, link offset $d_i$, and joint angle $\\theta_i$.
*   **Technical Details**: The transformation matrix combines Z-axis and X-axis translations and rotations: $T_i^{i-1} = Rot_z(\\theta_i) \cdot Trans_z(d_i) \cdot Trans_x(a_i) \cdot Rot_x(\\alpha_i)$.
*   **Source Location**: [models.cpp:L30-65](file:///orocos_kdl/src/models.cpp#L30-65)

### Card 2: Inverse Kinematics (IK)
*   **Core Principle**: Inverse Kinematics (IK) translates spatial poses into joint angles. Analytical methods provide closed-loop equations, while numerical solvers iterate step-by-step.
*   **Technical Details**: Numerical solvers employ Newton-Raphson iterations using the pseudo-inverse Jacobian matrix: $\Delta\\theta = J^{\\dagger}(\\theta) \cdot \Delta x$.
*   **Source Location**: [chainiksolverpos_nr.cpp:L50-90](file:///orocos_kdl/src/chainiksolverpos_nr.cpp#L50-90)

### Card 3: Jacobian Matrix
*   **Core Principle**: The Jacobian $J(\\theta)$ relates joint space velocities $\\dot{q}$ to operational end-effector velocities $\\dot{x}$ via: $\\dot{x} = J(\\theta)\\dot{q}$.
*   **Technical Details**: Based on virtual work, it also maps operational end-effector forces $F$ to joint torques $\\tau$ through: $\\tau = J^T(\\theta)F$.
*   **Source Location**: [chainjnttojacsolver.cpp:L20-75](file:///orocos_kdl/src/chainjnttojacsolver.cpp#L20-75)

### Card 4: Singularity Analysis
*   **Core Principle**: Singularities occur when the Jacobian matrix loses full rank ($\det(J) = 0$), freezing motion along locked axes and causing infinite joint velocities.
*   **Technical Details**: Manipulability is measured via $w = \sqrt{\det(JJ^T)}$. Near singularities, damped least squares (DLS) is applied to bound joint inputs.
*   **Source Location**: [chainiksolvervel_wdls.cpp:L80-130](file:///orocos_kdl/src/chainiksolvervel_wdls.cpp#L80-130)

### Card 5: Rigid Body Dynamics
*   **Core Principle**: Dynamics relates acceleration to torques. Newton-Euler recursively propagates velocities and forces, while Euler-Lagrange derives closed-form equations.
*   **Technical Details**: Multi-joint dynamics equation: $M(q)\\ddot{q} + C(q,\\dot{q})\\dot{q} + G(q) = \\tau$ (M: Inertia matrix, C: Coriolis/centrifugal forces, G: gravity).
*   **Source Location**: [chaindynparam.cpp:L40-95](file:///orocos_kdl/src/chaindynparam.cpp#L40-95)

### Card 6: EKF Sensor Fusion
*   **Core Principle**: EKF estimates vehicle states by linearizing non-linear dynamics and observations using first-order Taylor expansions.
*   **Technical Details**: Predict: $X_{k|k-1} = f(X_{k-1}, u_k)$ and $P_{k|k-1} = F_k P_{k-1} F_k^T + Q$, updating with Kalman gain $K_k$ upon sensor observations.
*   **Source Location**: [ekf.cpp:L50-110](file:///robot_localization/src/ekf.cpp#L50-110)

### Card 7: V-SLAM Front-end
*   **Core Principle**: Visual odometry tracks camera poses over consecutive frames using image feature matches (e.g. ORB) or intensity-based optical flow.
*   **Technical Details**: KLT tracker minimizes intensity differences between frames: $\sum [I_{t+1}(x+u, y+v) - I_t(x, y)]^2 \rightarrow \min$ without computing descriptor vectors.
*   **Source Location**: [Frame.cc:L100-160](file:///orb_slam3/src/Frame.cc#L100-160)

### Card 8: Bundle Adjustment (BA)
*   **Core Principle**: BA refines camera poses and landmark points by minimizing the overall reprojection error across multi-frame projections.
*   **Technical Details**: Minimizes: $E = \sum \|x_{ij} - \pi(T_i, p_j)\|^2_{H}$. Ceres/g2o uses Schur complement to solve the sparse linear system.
*   **Source Location**: [sparse_optimizer.cpp:L40-85](file:///g2o/core/sparse_optimizer.cpp#L40-85)

### Card 9: ICP & NDT Laser SLAM
*   **Core Principle**: Scan matching aligns LiDAR scans with maps. ICP minimizes point-to-point Euclidean distances, while NDT maximizes Gaussian probability over grids.
*   **Technical Details**: ICP minimizes $E(R, T) = \sum \|p_i - (R q_i + T)\|^2$, solving rotation $R$ and translation $T$ using Singular Value Decomposition (SVD).
*   **Source Location**: [icp.hpp:L80-140](file:///pcl/registration/icp.hpp#L80-140)

### Card 10: Loop Closure BoW
*   **Core Principle**: Loop closure recognizes visited places. Bag of Words (BoW) clusters image descriptors into visual words, enabling fast tree-based image queries.
*   **Technical Details**: Computes cosine similarities (TF-IDF weights) between current and past frames, triggering global factor graph optimization upon match confirmation.
*   **Source Location**: [LoopClosing.cc:L30-90](file:///orb_slam3/src/LoopClosing.cc#L30-90)

### Card 11: Dijkstra & A* Planning
*   **Core Principle**: Dijkstra finds shortest paths by compiling cumulative edge costs. A* optimizes search steps by introducing a goal-oriented heuristic cost $h(n)$.
*   **Technical Details**: Total cost: $f(n) = g(n) + h(n)$, where $h(n)$ measures Manhattan or Euclidean distances to the target node.
*   **Source Location**: [astar_planner.cpp:L70-120](file:///navigation2/nav2_astar_planner/src/astar_planner.cpp#L70-120)

### Card 12: RRT & RRT* Sampling
*   **Core Principle**: RRT grows a random tree towards free space samples to find feasible paths. RRT* refines paths incrementally by rewiring tree connections in nearby neighborhoods.
*   **Technical Details**: Rewiring compares whether connecting through $x_{new}$ reduces path costs $g(x)$ of neighboring nodes, redirecting parent pointers accordingly.
*   **Source Location**: [RRTstar.cpp:L100-160](file:///ompl/geometric/planners/rrt/RRTstar.cpp#L100-160)

### Card 13: Minimum Snap Trajectory
*   **Core Principle**: Discrete paths are smoothed into continuous trajectories limiting high-order derivatives: velocity (1st), acceleration (2nd), jerk (3rd), and snap (4th).
*   **Technical Details**: Polynomial trajectory segment: $p(t) = c_0 + c_1 t + \dots + c_n t^n$. Formulated as a Quadratic Programming (QP) problem under collision boundaries.
*   **Source Location**: [minimum_snap.cpp:L20-75](file:///planning/minimum_snap.cpp#L20-75)

### Card 14: GJK Collision Detection
*   **Core Principle**: GJK checks intersections between convex shapes by mapping Minkowski differences, reducing collision tests to finding the closest point to origin.
*   **Technical Details**: Support functions build simplices iteratively. The shapes intersect if the simplex encloses the origin.
*   **Source Location**: [gjk.cpp:L30-80](file:///fcl/narrowphase/detail/gjk.cpp#L30-80)

### Card 15: Model Predictive Control (MPC)
*   **Core Principle**: MPC solves constrained control sequences online based on state dynamics over finite time horizons, executing only the first computed input.
*   **Technical Details**: Minimizes tracking errors: $\sum \|x_{k+i} - x_{ref}\|^2_Q + \|u_{k+i}\|^2_R$ online using QP solvers under motor constraints.
*   **Source Location**: [mppi.cpp:L100-150](file:///navigation2/nav2_mppi_controller/src/mppi.cpp#L100-150)

### Card 16: PID Anti-Windup
*   **Core Principle**: Actuator saturation causes integral terms to accumulate indefinitely, generating heavy overshoot.
*   **Technical Details**: Prevented using Clamping (halting integration upon actuator saturation) or Back-calculation (feeding back saturation errors to the integrator).
*   **Source Location**: [pid.cpp:L40-80](file:///control_toolbox/src/pid.cpp#L40-80)

### Card 17: Computed Torque Control
*   **Core Principle**: CTC linearizes manipulator dynamics by canceling inertia, Coriolis, and gravity terms, reducing the plant to decoupled double integrators.
*   **Technical Details**: Torque command: $\tau = M(q)(\ddot{q}_d + K_p e + K_d \dot{e}) + C(q,\dot{q})\dot{q} + G(q)$ (where $e = q_d - q$).
*   **Source Location**: [joint_trajectory_controller.cpp:L120-170](file:///ros2_control/joint_trajectory_controller.cpp#L120-170)

### Card 18: Impedance Control
*   **Core Principle**: Impedance control models robot contact dynamics as virtual mass-damper-spring systems, regulating forces without causing impact damage.
*   **Technical Details**: Model: $M_d \ddot{e} + B_d \dot{e} + K_d e = F_{ext}$. Admittance controllers compute trajectory adjustments based on sensed external forces.
*   **Source Location**: [impedance_controller.cpp:L20-65](file:///control/impedance_controller.cpp#L20-65)

### Card 19: Odometry Calibration
*   **Core Principle**: Wheel radius and wheel track errors degrade odometry calculations over time, requiring systematic parameter calibration.
*   **Technical Details**: Linear/angular velocities: $v = (v_r + v_l)/2$, $\omega = (v_r - v_l)/b$ (where $b$: track). Wheel slip rates $s = (v_w - v)/v_w$ compensate kinematic drift.
*   **Source Location**: [diff_drive_controller.cpp:L150-210](file:///diff_drive_controller/diff_drive_controller.cpp#L150-210)

### Card 20: ROS 2 DDS QoS
*   **Core Principle**: ROS 2 drops global master architectures, migrating communications onto industrial Data Distribution Service (DDS) networks.
*   **Technical Details**: DDS QoS parameters include: Reliability (reliable/best-effort), Durability (transient-local/volatile), Liveliness, and Deadline.
*   **Source Location**: [rmw_node.cpp:L40-95](file:///rmw_fastrtps_cpp/src/rmw_node.cpp#L40-95)

### Card 21: tf2 Coordinate Trees
*   **Core Principle**: tf2 maintains tree transformations across robot components (camera, LiDAR, chassis, map), allowing coordinate transforms across timestamps.
*   **Technical Details**: Employs spherical linear interpolation (Slerp) for quaternions: $Slerp(q_0, q_1, t) = \frac{\sin((1-t)\theta)}{\sin\theta}q_0 + \frac{\sin(t\theta)}{\sin\theta}q_1$.
*   **Source Location**: [buffer_core.cpp:L10-60](file:///tf2/src/buffer_core.cpp#L10-60)

### Card 22: URDF & Gazebo Sim
*   **Core Principle**: URDF XML defines links (visual, collision, inertial) and joints (revolute, continuous, fixed) modeling mechanical dimensions.
*   **Technical Details**: ODE/Bullet engines parse inertia tensors $I$ and masses $m$ to calculate interactions, friction, and dynamic collisions.
*   **Source Location**: [urdf_sensor.cpp:L10-50](file:///urdf/urdf_parser/src/urdf_sensor.cpp#L10-50)

### Card 23: Nav2 Stack Costmaps
*   **Core Principle**: Nav2 deploys behavior trees (BT) to coordinate navigation tasks, loading Static, Obstacle (Voxel 3D), and Inflation costmap layers.
*   **Technical Details**: Obstacle layers parse sensor feeds to clear/mark grids, while Inflation layers add safety zones around obstacles to prevent chassis damage.
*   **Source Location**: [costmap_2d.cpp:L450-510](file:///navigation2/nav2_costmap_2d/src/costmap_2d.cpp#L450-510)

### Card 24: RTPS Latency Tuning
*   **Core Principle**: High-frequency telemetry via RTPS over local networks can lag. Optimization requires configuring unicast peer direct mappings.
*   **Technical Details**: Tuning limits multicast overheads, adjusts MTU to prevent IP packet fragmentation, and prioritizes RTPS reader/writer socket threads.
*   **Source Location**: [RTPSParticipantImpl.cpp:L20-100](file:///fastdds/src/cpp/rtps/participant/RTPSParticipantImpl.cpp#L20-100)

### Card 25: Cam-LiDAR Calibration
*   **Core Principle**: Joint sensor fusion requires camera-LiDAR hardware PPS sync. Extrinsic parameters optimize projection residuals on checkboards.
*   **Technical Details**: Loss: $\sum \|X_{img} - P \cdot (T_{calib} \cdot X_{lidar})\|^2$. Resolves extrinsic transform $T_{calib}$ using non-linear least squares.
*   **Source Location**: [extrinsic_calibration.cpp:L10-45](file:///calibration/extrinsic_calibration.cpp#L10-45)

### Card 26: Reinforcement Learning RL
*   **Core Principle**: PPO/SAC algorithms train robots to learn complex policies. Sim-to-Real gaps are bridged using domain randomization.
*   **Technical Details**: Randomizes parameters (gravity, friction, latencies, mass) during training, forcing neural networks to generalize over physical mismatches.
*   **Source Location**: [ppo_control.py:L50-110](file:///rl/ppo_control.py#L50-110)

### Card 27: Point Cloud RANSAC
*   **Core Principle**: Large point clouds are decimated using Voxel Grid downsampling. RANSAC fits geometric primitives (like ground planes).
*   **Technical Details**: RANSAC fits planes ($Ax+By+Cz+D=0$) to random points, scoring inlier counts to isolate floor planes from obstacles over iterations.
*   **Source Location**: [voxel_grid.cpp:L100-140](file:///pcl/filters/voxel_grid.cpp#L100-140)

### Card 28: Mobile Robot Kinematics
*   **Core Principle**: Differentiates mobile chassis constraints (differential, Ackermann, and Mecanum omnidirectional forces).
*   **Technical Details**: Mecanum kinematics mapping: $\omega_i = \frac{1}{r}(v_x \pm v_y \pm (a+b)\omega_z)$, running at high frequency in wheel controllers.
*   **Source Location**: [kinematics.cpp:L10-40](file:///navigation2/nav2_controller/src/kinematics.cpp#L10-40)

---

## 📂 Robotics Motion Planning & SLAM Trade-off Matrix

| Design Dimension | Strong Kinematics Pattern | Exploration Optimized Pattern | Trade-off Matrix Analysis |
| :--- | :--- | :--- | :--- |
| **Arm Control** | Computed Torque Control (CTC) | Decoupled Joint PID Control | CTC fully decouples multi-joint dynamics but demands highly accurate link models; PID is model-free and robust but suffers from tracking distortion under high speed. |
| **Path Planning** | Asymptotically Optimal (RRT*) | Quick Random Tree (RRT) | RRT* converges to the shortest path but consumes high CPU in rewiring; RRT is fast (~ms) but produces highly redundant, winding paths. |
| **SLAM Fusion** | Factor Graph SLAM (g2o/Ceres) | Filter-based Odometry (EKF) | Factor graphs provide loop closure corrections but occupy heavy memory; EKF is fast and light but suffers from unbound drift over time. |
| **3D Navigation** | Voxel Grid 3D Costmap | 2D Grid Costmap | 3D costmaps detect hanging obstacles and slopes but consume heavy CPU/RAM; 2D costmaps are light but blind to complex 3D topology. |

