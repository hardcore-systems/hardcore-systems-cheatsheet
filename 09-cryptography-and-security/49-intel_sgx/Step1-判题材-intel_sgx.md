# Step1-判题材-intel_sgx.md: Intel SGX 飞地安全计算与硬件隔离审计大纲

本审计项目聚焦于硬件级可信执行环境（TEE）的基石——Intel SGX。我们将通过 28 张核心卡片剖析飞地内存管理、硬件加密引擎、远程认证拓扑以及微架构侧信道防御，设计 J-Ladder 梯级架构，并部署双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: TEE 与硬件飞地基础** (Cards 1–5) - `#4B5F7A` (Slate Blue)
    - 飞地页面缓冲（EPC）、处理器预留内存（PRM）、硬件级 EPCM 访问检查。
*   **M2: 飞地隔离安全与侧信道防御** (Cards 6–10) - `#6B8272` (Muted Sage)
    - 异步飞地退出（AEX）、页错误通道攻击防御、L1TF (Foreshadow) 瞬态执行漏洞缓解。
*   **M3: 密码学证明与远程认证** (Cards 11–15) - `#9C6666` (Tea Red)
    - 本地认证、远程认证、QE 与 Launch Enclave、测量寄存器（MRENCLAVE/MRSIGNER）。
*   **M4: 飞地生命周期与边界交互** (Cards 16–19) - `#7A7A7A` (Iron Grey)
    - ECALL/OCALL 参数封送与上下文切换开销、动态内存管理（SGX2 EDMM）。
*   **M5: SDK 工具、签名与调试规范** (Cards 20–24) - `#9A825A` (Dusty Gold)
    - 调试/生产模式差异、飞地签名工具（sgx_sign）与 XML 配置。
*   **M6: 虚拟化与高级部署优化** (Cards 25–28) - `#755B77` (Muted Grape)
    - KVM 虚拟 EPC、LibOS（Gramine/Occlum）多进程隔离、TCB 恢复与微码更新。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    Intel SGX 的本质是通过 CPU 硬件在物理内存中划分加密区域（EPC），通过硬件存储加密引擎（MEE）和硬件访问控制器（EPCM）抵御来自操作系统、虚拟化层甚至物理探测的机密性与完整性威胁。
*   **L1 四句话逻辑**：
    1. **内存硬加密**：CPU 在 PRM 中分配 EPC 页面，所有进出 CPU 缓存的 EPC 数据均由 MEE 进行实时 AES 硬件加解密和完整性校验。
    2. **边界强隔离**：飞地内外交互通过特权指令进行上下文切换，异常发生时触发 AEX 强制清空 CPU 寄存器状态并保存到 SSA 中。
    3. **可信度量链**：通过 ECREATE 和 EADD 指令逐步构建飞地，其代码 and 初始数据由 MRENCLAVE 寄存器进行不可篡改的加密哈希度量。
    4. **远程链式认证**：本地飞地通过 QE（Quoting Enclave）将其物理报告转换为可被外部第三方（如 Intel PCS）通过 ECDSA 验证的 Quote 证书。
*   **L2 核心数据流转拓扑**：
    `ECREATE` ➜ `EADD` ➜ `EEXTEND` ➜ `EINIT` ➜ `EENTER(ECALL)` ➜ `MEE加密读写` ➜ `EREPORT` ➜ `QE Quote签名` ➜ `Intel PCS远程校验`

---

## 🗂️ 28 张核心知识卡片大纲
1. 处理器预留内存（PRM）与飞地页面缓存（EPC）分配。
2. 飞地页面缓存表（EPCM）硬件成员访问控制规则。
3. 内存加密引擎（MEE）的实时 AES-MCTS 加密与重放保护。
4. 飞地硬件生命周期指令（ECREATE/EADD/EINIT/EEXTEND）。
5. Enclave 模式下 CPU 环 3 执行限制与内存边界。
6. 异步飞地退出（AEX）时寄存器状态清空与恢复机制。
7. 飞地页错误（SGX #PF）与控制通道时序侧信道防御。
8. 瞬态执行漏洞（L1TF / Foreshadow）在 TEE 中的防御与缓解。
9. 线程控制结构（TCS）与状态保存区（SSA）帧设计。
10. SGX 飞地页面换出（EBLOCK/ETRACK/EWB）与硬件重加密。
11. 本地认证硬件指令（EREPORT/EGETKEY）及 symmetric 共享密钥。
12. 远程认证 Quote 生成架构（QE / PCE 协同）。
13. Intel Provisioning Certification Service (PCS) 证书分发机制。
14. EPID 群组签名与 ECDSA 认证技术路线演进。
15. 测量寄存器（MRENCLAVE 与 MRSIGNER）定义与飞地身份核验。
16. ECALL 与 OCALL 指令控制流切换的硬件级上下文开销。
17. 飞地内外数据参数封送（Marshalling）与安全指针检查。
18. SGX2 EDMM 动态内存管理及 EMA 动态分配机制。
19. 运行时库支持与 LibOS 兼容层设计。
20. Debug Mode 调试特性与 `EDGRD`/`EDWR` 指令功能限制。
21. 飞地配置文件（XML）及核心安全属性设定。
22. 飞地签名工具（sgx_sign）双密钥签名方案与 SIGSTRUCT 构成。
23. SGX 模拟模式（Simulation Mode）与硬件真跑的对比边界。
24. 架构飞地（Architectural Enclave: LE/QE/PCE/AE）设计作用。
25. EPC 物理内存耗尽时页面换出抖动与调优。
26. 多飞地间 IPC 通道与安全物理内存共享机制。
27. SGX 虚拟化与 KVM vEPC 设备驱动映射。
28. 安全版本号（ISVSVN）控制与 TCB 恢复及微码更新审计。
