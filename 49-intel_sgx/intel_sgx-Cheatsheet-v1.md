# Intel SGX 飞地安全计算与硬件隔离速查海报

## J-Ladder 阶梯逻辑模型

### L0 一句话本质
Intel SGX 的本质是通过 CPU 硬件在物理内存中划分加密区域（EPC），通过硬件存储加密引擎（MEE）和硬件访问控制器（EPCM）抵御来自操作系统、虚拟化层甚至物理探测的机密性与完整性威胁。

### L1 四句话逻辑
1. **内存硬加密**：CPU 在 PRM 中分配 EPC 页面，所有进出 CPU 缓存的 EPC 数据均由 MEE 进行实时 AES 硬件加解密和完整性校验。
2. **边界强隔离**：飞地内外交互通过特权指令进行上下文切换，异常发生时触发 AEX 强制清空 CPU 寄存器状态并保存到 SSA 中。
3. **可信度量链**：通过 ECREATE 和 EADD 指令逐步构建飞地，其代码和初始数据由 MRENCLAVE 寄存器进行不可篡改的加密哈希度量。
4. **远程链式认证**：本地飞地通过 QE（Quoting Enclave）将其物理报告转换为可被外部第三方（如 Intel PCS）通过 ECDSA 验证的 Quote 证书。

### L2 核心数据流转拓扑
`ECREATE` ➜ `EADD` ➜ `EEXTEND` ➜ `EINIT` ➜ `EENTER(ECALL)` ➜ `MEE加密读写` ➜ `EREPORT` ➜ `QE Quote签名` ➜ `Intel PCS远程校验`

---

## 📂 核心知识卡片 (Cards 1-28)

### Card 1: 处理器预留内存（PRM）与飞地页面缓存（EPC）分配
*   **核心原理**: BIOS 在启动阶段通过 PRMRR 寄存器划分出一块不受任何 CPU 之外设备（包括 DMA）访问的 Processor Reserved Memory (PRM)，并在其中切分出 Enclave Page Cache (EPC) 用以分配物理 Enclave 页面。
*   **技术细节**: 物理 EPC 页面由内核中的 SGX 驱动维护。在 Linux 5.11+ 中，由核心函数 `sgx_alloc_epc_page` 分配，页面大小为 4KB。PRM 最大支持几百 MB，耗尽时会触发软件分页。
*   **源码位置**: [main.c:L30-80](file:///arch/x86/kernel/cpu/sgx/main.c#L30-80)

### Card 2: 飞地页面缓存表（EPCM）硬件成员访问控制规则
*   **核心原理**: EPCM 是 CPU 内部维护的安全元数据表，包含了每个 EPC 页面的权限（R/W/X）、类型（SECS, REG, TCS, VA）及所属 Enclave (SECS ID) 等，硬件层在 MMU 地址翻译后强制比对 EPCM。
*   **技术细节**: 即使内核页表将某 EPC 页面的物理地址映射给了普通进程，CPU 执行单元检测到当前不处于该 Enclave 内部执行状态时，也会抛出 #GP 异常，完全绕过了 OS 内核。
*   **源码位置**: [encl.c:L120-160](file:///arch/x86/kernel/cpu/sgx/encl.c#L120-160)

### Card 3: 内存加密引擎（MEE）的实时 AES-MCTS 加密与重放保护
*   **核心原理**: MEE 是嵌入在 CPU 内存控制器内部的专用硬件。当数据从 CPU 缓存写入 PRM 时，MEE 采用 AES-XTS 模式对其进行加密；读入缓存时解密。
*   **技术细节**: 为防止重放攻击和剪切攻击，MEE 内部维护了一棵基于 Merkle-Tree 的 MCTS (Memory Cryptographic Tree)，根节点常驻 CPU，从而对物理总线窃听者呈现随机噪音。
*   **源码位置**: [main.c:L180-210](file:///arch/x86/kernel/cpu/sgx/main.c#L180-210)

### Card 4: 飞地硬件生命周期指令（ECREATE/EADD/EINIT/EEXTEND）
*   **核心原理**: 构建飞地包括：`ECREATE` 创建 SECS 安全上下文，`EADD` 将代码/数据页面加载进 EPC，`EEXTEND` 对页面 256 字节分块进行累加度量，`EINIT` 锁定飞地并锁定度量寄存器。
*   **技术细节**: 运行 `EINIT` 时，需要传入包含 Launch Token 的 SIGSTRUCT 签名包。如果平台启动密钥配置不合规，飞地将初始化失败。
*   **源码位置**: [ioctl.c:L200-290](file:///arch/x86/kernel/cpu/sgx/ioctl.c#L200-290)

### Card 5: Enclave 模式下 CPU 环 3 执行限制与内存边界
*   **核心原理**: 进入 Enclave 内部后，CPU 强制处于特殊的“Enclave 模式”，且仅允许在 Ring 3 执行，禁止执行大多数系统敏感特权指令（如 IN, OUT, INT, CLI, STI）。
*   **技术细节**: 在 Enclave 模式下发生任何 I/O 操作、特权级变动或外部中断，CPU 都会发生 AEX，切出飞地，以防止飞地内部越权或破坏宿主系统稳定。
*   **源码位置**: [encl.c:L300-340](file:///arch/x86/kernel/cpu/sgx/encl.c#L300-340)

### Card 6: 异步飞地退出（AEX）时寄存器状态清空与恢复机制
*   **核心原理**: 当 Enclave 内部发生中断或硬件异常时，CPU 不会将上下文直接交给普通异常处理程序（防止信息泄露），而是执行 AEX，自动用零或哑元数据覆盖全部通用寄存器。
*   **技术细节**: 真实的 CPU 状态被硬件压入飞地内部的 State Save Area (SSA) 栈帧。异常处理结束后，通过 `ERESUME` 指令恢复 SSA 寄存器状态并重新进入 Enclave。
*   **源码位置**: [assembly.S:L10-60](file:///sdk/simulation/assembly/assembly.S#L10-60)

### Card 7: 飞地页错误（SGX #PF）与控制通道时序侧信道防御
*   **核心原理**: OS 依然负责 Enclave 内部虚拟地址到物理 EPC 页的页表映射。如果发生换页，会触发页错误。恶意 OS 可以故意构造频发的页错误来分析 Enclave 的数据访问控制流。
*   **技术细节**: 防御方案包括：在 Enclave 内部检测时钟周期比率变化，或使用软件层自适应平滑路径（如 ORAM）抹平内存访问的时序差异。
*   **源码位置**: [encl.c:L410-450](file:///arch/x86/kernel/cpu/sgx/encl.c#L410-450)

### Card 8: 瞬态执行漏洞（L1TF / Foreshadow）在 TEE 中的防御与缓解
*   **核心原理**: Foreshadow 漏洞利用了 CPU 乱序执行的分支预测机制。即使页表呈现 Present = 0，CPU 预测执行依然可能将 L1 Cache 中的 EPC 数据泄露给推测代码。
*   **技术细节**: 缓解方式是关闭超线程（Hyper-Threading）、强制刷写 L1 Data Cache，或在微码层面隔离宿主与飞地的物理核绑定关系。
*   **源码位置**: [main.c:L310-350](file:///arch/x86/kernel/cpu/sgx/main.c#L310-350)

### Card 9: 线程控制结构（TCS）与状态保存区（SSA）帧设计
*   **核心原理**: TCS 是飞地内部固定的硬件页面，包含飞地线程的入口地址（OENTRY）、SSA 栈的基址及偏移，每个逻辑 CPU 在飞地中必须独占一个 TCS。
*   **技术细节**: SSA 包含了保存通用寄存器值、浮点数及 SSE 状态所需的物理缓存空间。TCS 与 SSA 物理页面必须在 `EINIT` 之前通过 `EADD` 被加载且设为不可变。
*   **源码位置**: [encl.h:L50-80](file:///arch/x86/kernel/cpu/sgx/encl.h#L50-80)

### Card 10: SGX 飞地页面换出（EBLOCK/ETRACK/EWB）与硬件重加密
*   **核心原理**: 当物理 EPC 页不够用时，操作系统必须配合 CPU 将飞地页面换出（Paging）到不安全的主存。这需要调用 `EBLOCK` 阻塞访问，`ETRACK` 清理页表的 TLB。
*   **技术细节**: 接着运行 `EWB` 将页面移出，期间 CPU 会使用临时封包密钥（Wrap Key）对页面加密并生成版本令牌，写回普通内存；换入时执行 `ELDU` 重新载入解密并比对版本令牌。
*   **源码位置**: [ioctl.c:L340-410](file:///arch/x86/kernel/cpu/sgx/ioctl.c#L340-410)

### Card 11: 本地认证硬件指令（EREPORT/EGETKEY）及 symmetric 共享密钥
*   **核心原理**: 同一平台上两个飞地之间要建立信任，可调用 `EREPORT` 生成一份包含对方 SECS 度量的 Report，并在 CPU 内部采用对称密钥（Report Key）生成 MAC。
*   **技术细节**: 对方飞地收到 Report 后，调用 `EGETKEY` 获取对应的 Report Key，并重新计算 MAC，一致则代表两者均在同一合规 SGX CPU 上运行。
*   **源码位置**: [attestation/local.c:L15-60](file:///sdk/attestation/local.c#L15-60)

### Card 12: 远程认证 Quote 生成架构（QE / PCE 协同）
*   **核心原理**: 远程验证要求将 Report 签名成可公开传输的证书（Quote）。Report 发给平台架构飞地——Quoting Enclave (QE)。
*   **技术细节**: QE 通过本地认证机制检验该 Report，确认真实后，使用 PCE (Provisioning Certification Enclave) 分发的非对称密钥对其进行签名，转化为可对外披露的 Quote。
*   **源码位置**: [attestation/quote.c:L40-95](file:///sdk/attestation/quote.c#L40-95)

### Card 13: Intel Provisioning Certification Service (PCS) 证书分发机制
*   **核心原理**: 平台验证需要确认 QE 签名的 Quote 公钥属于真实的 Intel 芯片。Intel PCS 提供了在线证书吊销与验证服务。
*   **技术细节**: 远端验证服务拉取平台的 PCK（Processor Certificate Key）证书链，Intel 作为根 CA（Root CA）对 PCK 进行签名，从而实现芯片出厂的链式回溯。
*   **源码位置**: [attestation/pcs.c:L10-45](file:///sdk/attestation/pcs.c#L10-45)

### Card 14: EPID 群组签名与 ECDSA 认证技术路线演进
*   **核心原理**: 早期 SGX 远程认证采用 EPID (Enhanced Privacy ID) 零知识证明群组签名，隐藏平台唯一物理 ID，保护隐私；SGX3 代后全面转向易于离线验证的 ECDSA 签名体系。
*   **技术细节**: ECDSA 签名由 QE3 飞地生成，配合 PCCS（平台证书缓存服务）能实现秒级的本地证书链解析，避免了频繁的远程 Intel 在线 API 调用。
*   **源码位置**: [attestation/ecdsa.c:L50-110](file:///sdk/attestation/ecdsa.c#L50-110)

### Card 15: 测量寄存器（MRENCLAVE 与 MRSIGNER）定义与飞地身份核验
*   **核心原理**: `MRENCLAVE` 是飞地代码、数据和加载时序的 SHA-256 加密度量值，代表飞地“我是什么”；`MRSIGNER` 是飞地签名公钥的哈希，代表飞地“谁开发了我”。
*   **技术细节**: 升级飞地代码时，`MRENCLAVE` 会变，但只要保持签名密钥不变，`MRSIGNER` 就能保持不变。这允许本地飞地使用 `MRSIGNER` 策略来共享升级前后的数据封包密钥。
*   **源码位置**: [sdk/tlibc/tattest.c:L20-55](file:///sdk/tlibc/tattest.c#L20-55)

### Card 16: ECALL 与 OCALL 指令控制流切换的硬件级上下文开销
*   **核心原理**: 进入飞地使用 ECALL（通过 `EENTER`/`EEXIT` 指令组合），飞地调用宿主使用 OCALL。每次 ECALL/OCALL 都涉及页表缓存重置、SSA 压栈和寄存器深度洗牌。
*   **技术细节**: 控制流切换开销约为数千个 CPU 时钟周期。因此，系统设计中应极力避免在循环中触发 ECALL/OCALL，转而使用“无跳转飞地”（Switchless Enclave）线程共享技术。
*   **源码位置**: [edger8r/codegen.ml:L40-80](file:///sdk/edger8r/codegen.ml#L40-80)

### Card 17: 飞地内外数据参数封送（Marshalling）与安全指针检查
*   **核心原理**: 飞地内部的代码不能盲目解引用传递进来的指针（这可能是 OS 的越权陷阱，即 TOCTOU 漏洞）。SGX 自动生成的边界桥接代码必须对参数执行封送拷贝。
*   **技术细节**: SDK 提供的 Edger8r 工具链会自动生成代理代码，检查外部指针所指向的地址是否完全位于飞地外部，如果是，则先将内容深拷贝至飞地受保护内存中，再行操作。
*   **源码位置**: [edger8r/parser.ml:L100-140](file:///sdk/edger8r/parser.ml#L100-140)

### Card 18: SGX2 EDMM 动态内存管理及 EMA 动态分配机制
*   **核心原理**: 第一代 SGX 要求飞地的 EPC 大小在编译时静态确定，不可动态扩容。SGX2 引入了 EDMM (Enclave Dynamic Memory Management) 硬件机制。
*   **技术细节**: EDMM 支持运行期修改 EPC 页面权限（`EMODPR`）以及动态增加页面（`EMODT`）。这允许飞地内部库使用标准的 `malloc` 从动态扩充的 Heap 中安全分配大内存。
*   **源码位置**: [sdk/trts/trts_util.cpp:L30-70](file:///sdk/trts/trts_util.cpp#L30-70)

### Card 19: 运行时库支持与 LibOS 兼容层设计
*   **核心原理**: 由于 Enclave 内严禁执行系统调用，直接运行复杂应用（如 Redis, Nginx）会导致奔溃。LibOS (Occlum/Gramine) 在 Enclave 内部实现了完整的系统调用翻译层。
*   **技术细节**: LibOS 拦截 libc 的 `syscall`，并在 Enclave 内用软件模拟（如内存虚拟文件系统、受保护的网络栈），仅将真正无法处理的底层 I/O 封送并托管给外部普通 OS。
*   **源码位置**: [sdk/tlibc/string/memset.c:L5-20](file:///sdk/tlibc/string/memset.c#L5-20)

### Card 20: Debug Mode 调试特性与 `EDGRD`/`EDWR` 指令功能限制
*   **核心原理**: SGX 支持调试模式（Debug Mode）。在 SECS 中，如果设置了 `DEBUG` 标志位，CPU 将允许使用 `EDGRD` 和 `EDWR` 特权指令来读取或修改该飞地的 EPC 物理页面。
*   **技术细节**: 生产模式飞地此标志必须设为 0。一旦关闭调试，即使内核调试器（GDB）通过硬件调试寄存器，也无法抓取到任何 EPC 明文数据，物理 JTAG 调试也会被锁止。
*   **源码位置**: [sdk/simulation/tcs.cpp:L40-75](file:///sdk/simulation/tcs.cpp#L40-75)

### Card 21: 飞地配置文件（XML）及核心安全属性设定
*   **核心原理**: 编译 Enclave 时，通过 XML 文件指定安全属性，如堆栈深度（StackSize）、最大堆大小（HeapSize）、并发线程数（TCSNum）及硬件策略配置。
*   **技术细节**: XML 中的配置属性会被 `sgx_sign` 工具解析并填充进 SIGSTRUCT 结构的属性字段中，最终由 `ECREATE` 写入 SECS 并和 `MRENCLAVE` 度量绑定。
*   **源码位置**: [sdk/sign_tool/SignTool/SignTool.cpp:L120-170](file:///sdk/sign_tool/SignTool/SignTool.cpp#L120-170)

### Card 22: 飞地签名工具（sgx_sign）双密钥签名方案与 SIGSTRUCT 构成
*   **核心原理**: 飞地签名的目的是生成 SIGSTRUCT 结构体。它包含飞地的测量哈希值、ISVSVN 版本号、开发者公钥，并使用开发者的 RSA-3072 私钥对这部分数据进行数字签名。
*   **技术细节**: 硬件 `EINIT` 指令会自动解析并验证 SIGSTRUCT 是否包含正确公钥，若被篡改则拒绝启动。这保证了即使用户拿到了物理机器并篡改了 EPC 文件，也无法强行加载。
*   **源码位置**: [sdk/sign_tool/SignTool/sgx_sign.cpp:L200-280](file:///sdk/sign_tool/SignTool/sgx_sign.cpp#L200-280)

### Card 23: SGX 模拟模式（Simulation Mode）与硬件真跑的对比边界
*   **核心原理**: 在没有真实 SGX CPU 的平台上，SGX SDK 提供了模拟模式（Simulation Mode），使用软件内存模拟 EPCM 和 AEX 上下文切换。
*   **技术细节**: 模拟模式下不存在 MEE 的物理总线加密，`EREPORT` 指令计算出的 MAC 仅依赖于软件伪造的硬密钥。模拟模式绝对不可用于生产环境，否则形同明文存储。
*   **源码位置**: [sdk/simulation/assembly/](file:///sdk/simulation/assembly/)

### Card 24: 架构飞地（Architectural Enclave: LE/QE/PCE/AE）设计作用
*   **核心原理**: 架构飞地是 Intel PSW 提供的一组核心特权 Enclave，包括 LE (Launch Enclave - 分发启动令牌)、QE (Quoting Enclave - 远程认证) 和 PCE (Provisioning Enclave)。
*   **技术细节**: 架构飞地的证书由 Intel Root Key 签发，它们能访问特殊的 CPU Hardware Key（如 Provisioning Key），普通用户飞地无法读取这些密钥，从而实现了特权隔离。
*   **源码位置**: [psw/ae/common/](file:///psw/ae/common/)

### Card 25: EPC 物理内存耗尽时页面换出抖动与调优
*   **核心原理**: 频繁的大内存分配（如对超大数组进行扫描）会导致严重的 EPC 页面换入换出（Paging Thrashing），此时 CPU 处于频繁的重加密和 TLB 刷写状态，性能急剧下降。
*   **技术细节**: 优化手段包括：使用 NUMA 亲和性，避免跨 Socket 的 EPC 访存；减少大数组的全表扫描，设计内存紧凑的数据结构，并在编译 XML 中合理增大初始堆大小。
*   **源码位置**: [arch/x86/kernel/cpu/sgx/main.c#L450-510](file:///arch/x86/kernel/cpu/sgx/main.c#L450-510)

### Card 26: 多飞地间 IPC 通道与安全物理内存共享机制
*   **核心原理**: 两个独立飞地直接的物理内存是完全隔离的（受 EPCM 控制）。为了通信，它们必须协商建立对称会话密钥（Session Key），或者向宿主申请一段非 EPC 的共享内存。
*   **技术细节**: 两个飞地在非 EPC 内存上构建无锁环形队列（Ring Buffer），所有交互报文在飞地内部加密后写入共享内存，由对方飞地读取并解密，确保了宿主 OS 无法窃取通道数据。
*   **源码位置**: [sdk/attestation/local.c#L80-120](file:///sdk/attestation/local.c#L80-120)

### Card 27: SGX 虚拟化与 KVM vEPC 设备驱动映射
*   **核心原理**: 虚拟机也需要运行 SGX Enclave。KVM 通过虚拟 EPC (vEPC) 设备驱动，将宿主机的一部分物理 EPC 内存直接物理透传给虚拟机。
*   **技术细节**: KVM 重写了 EPT 页表配置，拦截并重映射虚拟机的 `ECREATE` 和 `EINIT` 指令，使得虚拟机内的客户操作系统能够像使用物理 CPU 一样管理 EPC 页面。
*   **源码位置**: [kvm/vmx/sgx.c:L20-100](file:///arch/x86/kvm/vmx/sgx.c#L20-100)

### Card 28: 安全版本号（ISVSVN）控制与 TCB 恢复及微码更新审计
*   **核心原理**: `ISVSVN` 记录了 Enclave 代码的安全版本号。当硬件微架构暴露漏洞时，Intel 会发布微码更新升级 TCB (TCB Recovery)，并更新物理 CPU 的 CPUSVN。
*   **技术细节**: TCB Recovery 后，PCS 会电召并升级 PCK 证书。Enclave 代码必须对比 `ISVSVN` 和 `CPUSVN`，强制拒绝在未打安全补丁的旧物理平台上进行密钥派生和初始化。
*   **源码位置**: [sdk/tlibc/tattest.h:L10-40](file:///sdk/tlibc/tattest.h#L10-40)

---

## 📂 密码学通信安全设计与强度折衷矩阵

| 设计维度 | 高安全保障策略 (Strong TEE Pattern) | 性能侧重策略 (Performance Optimized Pattern) | 关键折衷分析 (Trade-off Matrix Analysis) |
| :--- | :--- | :--- | :--- |
| **飞地切换机制** | 传统同步同步 ECALL/OCALL | 线程共享的无跳转 (Switchless) ECALL/OCALL | 传统 ECALL 带来硬件级的 AEX 和上下文重载，延迟高（~8k 周期）；Switchless 通过飞地内外共享无锁队列以轮询取代中断，在大并发小负载下提升几倍吞吐，但会额外持续耗用一个物理核心（CPU 100%）。 |
| **内存加密级别** | 开启重放保护的完整 MEE 树 | 仅加密不启用防重放 (SGA 无树模式) | 开启 MCTS 重放保护可以完全防止物理总线数据替换及重放，但每次读取需遍历 Merkle 树，访存延迟翻倍；仅加密模式性能高，但无法抵御高级主动硬件嗅探攻击。 |
| **认证方案路线** | 每次会话生成完整 Remote Quote | 平台级缓存 PCK 与 Quote 复用 | 每次握手实时生成 Quote 并联机 Intel PCS 进行验证可确保绝对实时可信，但受限于公网延迟；缓存证书和 Quote 复用消除了对 PCS 的即时依赖，但存在长达数小时的吊销状态信息延迟。 |
| **EPC 溢出换页** | 使用 EDMM 硬件动态按需申请 | 编译时静态分配固定大 EPC 内存 | EDMM 支持运行期弹性扩容，极大地降低了冷启动 EPC 开销，但涉及复杂指令流与内核交互；静态 EPC 启动慢，但运行时无动态申请带来的虚页缺页中断开销。 |

