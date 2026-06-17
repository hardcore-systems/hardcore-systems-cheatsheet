# power_systems-高密度卡片系统设计大图.md

本文件定义了 **PowerSystems.jl (现代电力系统仿真)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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

    Card1["Card 1: Bus Modeling"]:::M1
    Card2["Card 2: Pi Line Model"]:::M1
    Card3["Card 3: Ybus Matrix"]:::M1
    Card4["Card 4: Generator Limit"]:::M1
    Card5["Card 5: ZIP Load Model"]:::M1
    
    Card6["Card 6: Power Flow Eq"]:::M2
    Card7["Card 7: Newton-Raphson"]:::M2
    Card8["Card 8: Fast Decoupled"]:::M2
    Card9["Card 9: DC Power Flow"]:::M2
    Card10["Card 10: Optimal OPF"]:::M2

    Card11["Card 11: Swing Equation"]:::M3
    Card12["Card 12: Detailed Generator"]:::M3
    Card13["Card 13: AVR & Governor"]:::M3
    Card14["Card 14: DAE Integration"]:::M3
    Card15["Card 15: Network Interface"]:::M3

    Card16["Card 16: Small-Signal eigenvalues"]:::M4
    Card17["Card 17: Equal Area Criterion"]:::M4
    Card18["Card 18: CCT Calculation"]:::M4
    Card19["Card 19: Droop & AGC Control"]:::M4

    Card20["Card 20: BESS SoC Modeling"]:::M5
    Card21["Card 21: Inverter Droop"]:::M5
    Card22["Card 22:ADN state Estimate"]:::M5
    Card23["Card 23: Virtual Sync Machine"]:::M5
    Card24["Card 24: Islanding & Black Start"]:::M5

    Card25["Card 25: Wind DFIG LVRT"]:::M6
    Card26["Card 26: Solar MPPT P&O"]:::M6
    Card27["Card 27: LMP Pricing"]:::M6
    Card28["Card 28: Inertia Dilution"]:::M6

    Card1 --> Card3
    Card2 --> Card3
    Card3 --> Card6
    Card4 --> Card6
    Card5 --> Card6
    Card6 --> Card7
    Card7 --> Card8
    Card7 --> Card9
    Card7 --> Card10
    Card10 --> Card27
    Card11 --> Card12
    Card12 --> Card13
    Card13 --> Card14
    Card14 --> Card15
    Card15 --> Card16
    Card15 --> Card17
    Card17 --> Card18
    Card19 --> Card28
    Card21 --> Card23
    Card23 --> Card24
    Card25 --> Card28
    Card26 --> Card21
```

---

## 📍 PowerSystems.jl 物理源码位置映射

本设计大图的知识节点与 Julia 电力系统核心仿真软件物理源码模块强关联：
1. **Grid Modeling**: `PowerSystems.jl` 中的 `src/models/` 目录，涵盖 `Bus.jl`、`Branch.jl`、`ThermalGen.jl` 及 `ZIPLoad.jl`。
2. **Power Flow Solvers**: `PowerFlows.jl` 仓库下的 `src/power_flow.jl`，利用稀疏矩阵求解非线性雅可比。
3. **Dynamic Simulator**: `PowerSimulationsDynamics.jl` 仓库下的 `src/models/` (Swing, AVR, Governor 模型) 与 `src/simulation/` (DAE 求解模块)。
4. **Market & OPF**: `PowerModels.jl` 中的数学规划接口 `src/prob/opf.jl`，基于 JuMP 库。
