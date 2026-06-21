# rocket_chip-高密度卡片系统设计大图.md

本文件定义了 **rocket_chip (RISC-V 参数化芯片生成器与 TileLink 一致性总线)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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

    Card1["Card 1: Chisel Language"]:::M1
    Card2["Card 2: Cake Config"]:::M1
    Card3["Card 3: Rocket Core Pipeline"]:::M1
    Card4["Card 4: FPU & MMU"]:::M1
    Card5["Card 5: Instruction Cache"]:::M1
    Card6["Card 6: Data Cache MSHR"]:::M1
    
    Card7["Card 7: TileLink Five Channels"]:::M2
    Card8["Card 8: MSI Coherency"]:::M2
    Card9["Card 9: SystemBus Interconnect"]:::M2
    Card10["Card 10: Bus Adapters"]:::M2
    Card11["Card 11: Crossbar Arbitration"]:::M2
    Card12["Card 12: MMIO Controllers"]:::M2

    Card13["Card 13: Debug JTAG Module"]:::M3
    Card14["Card 14: BootROM Reset"]:::M3
    Card15["Card 15: Interrupt PLIC CLINT"]:::M3
    Card16["Card 16: FIRRTL Compiler"]:::M3
    Card17["Card 17: Verilog Simulation"]:::M3
    Card18["Card 18: Clock Gating Power"]:::M3
    Card19["Card 19: Physical SRAM Macros"]:::M3

    Card20["Card 20: RoCC Interface"]:::M4
    Card21["Card 21: Performance HPM"]:::M4
    Card22["Card 22: PMP Protections"]:::M4
    Card23["Card 23: Coherency Hub"]:::M4
    Card24["Card 24: DMA Agent"]:::M4

    Card25["Card 25: Main Entry.scala"]:::M5
    Card26["Card 26: Cache Configuration Specs"]:::M5
    Card27["Card 27: C++ Sim Harness"]:::M5
    Card28["Card 28: Multitile Subsystems"]:::M5

    Card1 --> Card2
    Card2 --> Card3
    Card3 --> Card4
    Card3 --> Card5
    Card3 --> Card6
    
    Card6 --> Card7
    Card7 --> Card8
    Card8 --> Card9
    Card9 --> Card10
    Card9 --> Card11
    Card10 --> Card12

    Card3 --> Card13
    Card14 --> Card3
    Card15 --> Card3
    Card1 --> Card16
    Card16 --> Card17
    Card18 --> Card9
    Card19 --> Card6

    Card3 --> Card20
    Card3 --> Card21
    Card4 --> Card22
    Card8 --> Card23
    Card9 --> Card24

    Card25 --> Card2
    Card26 --> Card6
    Card27 --> Card17
    Card28 --> Card9
```

---

## 📍 Rocket Chip 物理源码位置映射

本设计大图的知识节点与 Rocket Chip 核心 Scala/Chisel 物理源码强关联：
1. **Diplomacy & Subsystem**: `src/main/scala/diplomacy/`, `src/main/scala/subsystem/`。
2. **Rocket Core & Cache**: `src/main/scala/rocket/` 下的 `RocketCore.scala`, `IBuf.scala`, `HellaCache.scala` (D$)。
3. **TileLink Bus**: `src/main/scala/tilelink/`。
4. **Devices & MMIO**: `src/main/scala/devices/tilelink/` (PLIC, CLINT)。
5. **Generator Entry**: `src/main/scala/system/Generator.scala`。
