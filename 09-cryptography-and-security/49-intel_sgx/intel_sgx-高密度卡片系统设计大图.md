# intel_sgx-高密度卡片系统设计大图.md

本文件定义了 **intel / sgx-software-enable (Intel SGX 安全计算)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

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

    Card1["Card 1: PRM & EPC Allocation"]:::M1
    Card2["Card 2: EPCM Hard Access Check"]:::M1
    Card3["Card 3: Memory Encryption Engine"]:::M1
    Card4["Card 4: SGX Lifecycle Instructions"]:::M1
    Card5["Card 5: Enclave Execution Limit"]:::M1
    
    Card6["Card 6: AEX State Cleanup"]:::M2
    Card7["Card 7: Page Fault Side-Channel"]:::M2
    Card8["Card 8: L1TF / Foreshadow Defence"]:::M2
    Card9["Card 9: TCS & SSA Struct"]:::M2
    Card10["Card 10: EPC Page Swapping"]:::M2

    Card11["Card 11: Local Attestation"]:::M3
    Card12["Card 12: Remote Attestation Quote"]:::M3
    Card13["Card 13: Intel PCS Certificate"]:::M3
    Card14["Card 14: EPID vs ECDSA Signatures"]:::M3
    Card15["Card 15: MRENCLAVE & MRSIGNER"]:::M3

    Card16["Card 16: ECALL/OCALL Context Switch"]:::M4
    Card17["Card 17: Parameter Marshalling"]:::M4
    Card18["Card 18: SGX2 EDMM Dynamic Memory"]:::M4
    Card19["Card 19: Runtime LibOS Systems"]:::M4

    Card20["Card 20: Debug vs Prod Mode"]:::M5
    Card21["Card 21: XML Configurations"]:::M5
    Card22["Card 22: sgx_sign & SIGSTRUCT"]:::M5
    Card23["Card 23: Simulation Mode Boundaries"]:::M5
    Card24["Card 24: Architectural Enclaves"]:::M5

    Card25["Card 25: EPC Paging Tuning"]:::M6
    Card26["Card 26: Multi-Enclave IPC"]:::M6
    Card27["Card 27: KVM vEPC Virtualization"]:::M6
    Card28["Card 28: SVN & TCB Recovery"]:::M6

    Card1 --> Card2
    Card2 --> Card3
    Card3 --> Card4
    Card4 --> Card5
    Card5 --> Card9
    Card9 --> Card6
    Card6 --> Card7
    Card7 --> Card8
    Card10 --> Card25
    Card4 --> Card15
    Card15 --> Card11
    Card11 --> Card12
    Card12 --> Card13
    Card13 --> Card14
    Card5 --> Card16
    Card16 --> Card17
    Card18 --> Card19
    Card20 --> Card21
    Card21 --> Card22
    Card22 --> Card24
    Card24 --> Card23
    Card23 --> Card26
    Card25 --> Card27
    Card26 --> Card28
```

---

## 📍 SGX-Software-Enable 物理源码位置映射

本设计大图的知识节点与 Intel SGX 驱动及 SDK 物理源码模块强关联：
1. **EPCM & Memory Management**: `arch/x86/kernel/cpu/sgx/main.c` (EPC 页面分配与管理)。
2. **SGX Core Driver**: `arch/x86/kernel/cpu/sgx/encl.c` (Enclave 飞地内存结构与页表控制)。
3. **Attestation & Cryptography**: `sdk/attestation/` (本地与远程认证实现库)。
4. **Context Switching & SSA**: `sdk/simulation/assembly/` (AEX 汇编与飞地边界跳转)。
5. **Architectural Enclaves**: `psw/ae/` (QE, LE, PCE 等架构飞地模块源码)。
6. **Virtualization vEPC**: `arch/x86/kvm/vmx/sgx.c` (KVM SGX 虚拟化驱动)。
