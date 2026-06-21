# Step1-判题材-openssl.md: 密码学安全协议与安全通信基础设施设计大纲

本审计项目聚焦于全球安全通信与密码学的基石——OpenSSL。我们将通过 28 张核心知识卡片，深度剖析 TLS 握手状态机、对称与非对称硬件加速、侧信道防御及内存管理，设计 L0-L2 的 J-Ladder 阶梯逻辑模型，并规划双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

本速查海报分为六大核心主题模块，采用精选的莫兰迪配色体系进行区域划分与视觉区分：

*   **M1: 密码学基础、随机数与 EVP 接口** (Cards 1–5) - `#4B5F7A` (Slate Blue)
    - 聚焦加密原语设计、高熵安全随机数发生器（DRBG）、对称与非对称 EVP 统一高层接口及安全初始化。
*   **M2: 对称加密与消息认证码** (Cards 6–10) - `#6B8272` (Muted Sage)
    - 剖析流式与块式加密、AEAD 认证加密模式（AES-GCM/ChaCha20-Poly1305）、HMAC 消息完整性及物理碰撞防御。
*   **M3: 非对称加密、公钥基础设施 (PKI) 与证书** (Cards 11–15) - `#9C6666` (Tea Red)
    - 详解 RSA/ECC/ECDSA 非对称算法、Diffie-Hellman 密钥交换、X.509 证书链验证拓扑及 ASN.1 序列化编解码。
*   **M4: TLS 握手状态机与密钥派生机制** (Cards 16–19) - `#7A7A7A` (Iron Grey)
    - 深入剖析 TLS 1.2/1.3 握手时序、会话恢复（Session Resumption）、HKDF 密钥派生树及加密记录协议（Record Protocol）。
*   **M5: 内存安全、Side-channel 侧信道防御与零化** (Cards 20–24) - `#9A825A` (Dusty Gold)
    - 覆盖 OpenSSL 专属内存分配器（OPENSSL_malloc）、常数时间（Constant-time）执行规避时序攻击、内存零化（Zeroization）防信息泄漏。
*   **M6: 硬件密码加速、Engine 引擎与硬件抽象层** (Cards 25–28) - `#755B77` (Muted Grape)
    - 涉及 Intel QAT 硬件密码加速卡对接、CPU AES-NI/AVX 向量级算法指令优化、OpenSSL Engine/Provider 模块化动态路由设计。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    密码学安全通信的本质是通过对称/非对称算法在不安全信道上建立机密性、完整性、真实性与不可否认性保证的安全信道，其工程实现核心为 TLS 握手状态机、密码算法硬加速与随机数熵源维护。
*   **L1 四句话逻辑**：
    1. **协商与身份鉴权**：Client 与 Server 在不安全网络中通过 TLS 1.3 握手单往返（1-RTT）协商密码套件，利用 X.509 证书链验证实体真实身份。
    2. **抗监听密钥交换**：借助 ECDHE 椭圆曲线 Diffie-Hellman 算法动态交换密钥份额，派生出具备前向安全（Forward Secrecy）的共享对称密钥。
    3. **认证加密保护**：在记录协议中采用 AEAD 双效算法（如 AES-GCM），为传输的明文数据提供高强度对称加密与抗篡改的完整性 MAC 校验。
    4. **算法硬件加速**：通过 EVP 抽象层自动重定向，利用 CPU 的 AES-NI 指令或独立 QAT 加速卡，消除海量吞吐下的对称加解密和非对称签名计算瓶颈。
*   **L2 核心数据流转拓扑**：
    `ClientHello` ➜ (协商套件与共享密钥份额) ➜ `ServerHello & Cert` ➜ (ECDHE 密钥提取) ➜ `HKDF 密钥派生` ➜ `对称密钥建立` ➜ `AEAD 记录协议对称流加密` ➜ `Encrypted Application Data`

---

## 🗂️ 28 张核心知识卡片大纲
1. 随机数熵源与 DRBG 安全边界：高熵源汇聚，Intel RdRand 硬件熵及 SP800-90A 伪随机标准。
2. EVP 统一密码高层抽象接口：对加解密、签名算法的高度封装，统一 `EVP_CIPHER` 与 `EVP_MD` 逻辑。
3. 对称分组密码模式与明文填充：ECB/CBC 链式模式与 PKCS#7 填充的漏洞（Padding Oracle）。
4. AEAD 认证加密与完整性保护：AES-GCM 与 ChaCha20-Poly1305，数据加密与 MAC 校验同步输出。
5. 消息摘要与 HMAC 防伪造签名：SHA-256/SHA-3 算法，HMAC 防长度扩展攻击机制。
6. Diffie-Hellman 与前向安全密钥交换：ECDHE 的临时密钥生成与离散对数计算抗监听机制。
7. RSA 非对称密码与填充安全：OAEP 填充规约，抗公钥明文恢复攻击。
8. ECC 椭圆曲线与 ECDSA 签名验证：坐标标量乘法、点加算法及 ECDSA 的零化防签名伪造。
9. ASN.1 规范与 DER/PEM 序列化：ASN.1 语法，二进制 DER 与 base64 PEM 的编解码流转换。
10. X.509 证书格式与数字证书验证：证书主体、CA 签名、吊销列表（CRL/OCSP）及信任链构建。
11. TLS 1.2 握手协商与 RTT 延迟：ClientKeyExchange 消息，DH/RSA 传统握手状态机。
12. TLS 1.3 单往返 (1-RTT) 握手机制：密钥共享（Key Share）提前发射，消减一次往返时延。
13. HKDF 密码级密钥派生算法：提取（Extract）与展开（Expand）双阶段派生会话密钥。
14. Session Ticket 会话恢复与 0-RTT 发送：预共享密钥（PSK）恢复，0-RTT 带来的重放攻击隐患。
15. TLS 记录协议 (Record Protocol) 帧分片：明文分段，AEAD 加密填充，抗重放滑动窗口。
16. OPENSSL_malloc 安全内存管理：禁止内存交换到磁盘，专属内存池防内存越界劫持。
17. 时序攻击 (Timing Attack) 与常数时间规约：避免按密钥分支跳转，保证数据处理耗时与密钥值无关。
18. 内存零化 (Zeroization) 与数据残留：调用 `OPENSSL_cleanse` 彻底清除内存中残留的密钥资产。
19. 心脏出血 (Heartbleed) 漏洞根本原因：TLS 心跳检测未进行长度边界检查，导致越界内存回传。
20. 侧信道缓存攻击 (Cache Timing Attack)：利用 CPU 缓存命中差窥探 RSA 密钥，防范掩码技术（Blinding）。
21. EVP_PKEY 统一非对称操作抽象：EVP_PKEY 统一 RSA/ECC/ECDH 接口，解耦具体物理算法。
22. SSL_CTX 会话上下文管理机制：全局参数配置，多路复用连接池与证书链装载。
23. BIO 抽象 I/O 管道机制：内存 BIO、文件 BIO、套接字 BIO 链式叠加封装。
24. OpenSSL Engine 引擎体系架构：v1.1.1 硬件适配层架构，通过回调函数动态挂载外部密码卡。
25. OpenSSL 3.0 Provider 模块化插槽：取代 Engine 的全新三层动态提供者架构。
26. Intel QAT 硬件密码卡异步驱动：QuickAssist 异步指令加速大并发 SSL/TLS 卸载。
27. 硬件指令集优化 (AES-NI / AVX-512)：CPU AES-NI/CLMUL 专有指令在向量级大幅削减加解密时钟。
28. 弱加密套件检测与安全合规性审计：禁用 RC4/DES/MD5 弱算法，合规审计 TLS 密码强度。
