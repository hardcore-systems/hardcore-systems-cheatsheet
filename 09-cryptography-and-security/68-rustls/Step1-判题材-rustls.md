# Step1-判题材-rustls.md: Rustls 现代密码安全 TLS 1.3 协议栈审计大纲

本审计项目聚焦于现代内存安全 TLS 协议栈 `Rustls`。我们将通过 28 张高密度核心知识卡片，深度解构其基于 Rust 安全所有权特性的内存安全设计架构、TLS 1.3 单往返（1-RTT）与零往返（0-RTT）握手协议状态机、可插拔密码学适配接口（CryptoProvider）、基于 HMAC 的 HKDF 密钥派生计算、无状态会话恢复票据分发、AEAD 对称加密（AES-GCM 与 ChaCha20-Poly1305）、基于 Transcript Hash 的握手防篡改校验、双向 TLS（mTLS）身份验证、防御侧信道时间攻击的常数时间比对算法、以及高并发异步集成（tokio-rustls）与后量子密码学（PQC）扩展。建立 L0-L2 阶梯模型，并设计双仿真沙盒。

---

## 🎨 莫兰迪色系设计系统

*   **M1: 内存安全与配置接口** (Cards 1–6) - `#4B5F7A` (Slate Blue)
    - 内存安全设计架构、代数类型握手状态机、ClientHello 与 ServerHello 结构、CryptoProvider 适配接口、HKDF 密钥派生机制、会话恢复与 PSK。
*   **M2: 密钥协商与 1-RTT 握手** (Cards 7–12) - `#6B8272` (Muted Sage)
    - 会话票据分发、记录协议包装、AEAD 对称加密、Transcript Hash 握手哈希校验、webpki 证书验证、双向 TLS (mTLS) 验证。
*   **M3: 记录协议与 AEAD 加密** (Cards 13–18) - `#9C6666` (Tea Red)
    - 签名方案协商、密钥交换算法执行、0-RTT 早期数据、加密扩展传输、动态密钥更新 (KeyUpdate)、警报协议错误处理。
*   **M4: 证书验证与双向认证** (Cards 19–23) - `#7A7A7A` (Iron Grey)
    - 规避 Heartbleed 设计、中间盒兼容模式、Finished 消息计算、常数时间比对防时序攻击、API 配置模式与 Arc 共享。
*   **M5: 侧道防御与前沿加密** (Cards 24–28) - `#9A825A` (Dusty Gold)
    - 异步 I/O (tokio-rustls) 集成、证书锁 (Certificate Pinning) 机制、模糊测试 (Fuzzing) 与审计、Criterion 性能基准测试、后量子密码学 (PQC) 密钥协商。

---

## 🪜 J-Ladder 架构分层体系

*   **L0 一句话本质**：
    Rustls 的本质是一个完全基于 Rust 内存安全特性构建的现代、轻量、无历史包袱的 TLS 1.3/1.2 安全传输协议栈，它抛弃了不安全密码组件，通过严格的状态机和第三方密码提供者接口保证数据通道的绝对安全。
*   **L1 四句话逻辑**：
    1. **内存安全无漏洞**：抛弃 C/C++ 指针算术，完全运行于 Rust 强类型与生命周期检查之上，从根本上杜绝了缓冲区溢出与 Use-After-Free 漏洞。
    2. **严格单向状态机**：利用 Rust 的 Enum 强类型编码握手状态，销毁旧状态并产生新状态，避免了握手流程被乱序执行或恶意旁路。
    3. **可插拔密码学引擎**：通过 `CryptoProvider` 接口，解耦协议核心与底层数学运算，支持无缝替换为 ring、AWS-LC 或硬件 HSM。
    4. **防御时序侧道攻击**：证书解析与底层校验算法采用常数时间（Constant-Time）操作，消除依据执行时长推算密钥特征的侧信道风险。
*   **L2 核心数据流转拓扑**：
    `ClientConfig/ServerConfig 建立` ➜ `CryptoProvider 注入` ➜ `ClientHello 发起` ➜ `ECDHE 临时私钥生成` ➜ `KeyShare 发送` ➜ `ServerHello 响应并完成 DH 协商` ➜ `HKDF 派生握手主密钥` ➜ `握手哈希 (Transcript Hash) 冻结并校验 Finished` ➜ `HKDF 派生应用数据密钥` ➜ `应用层 Write 写入` ➜ `AEAD 加密封包` ➜ `网络传输` ➜ `AEAD 解密验证` ➜ `应用层 Read 读取`

---

## 🗂️ 28 张核心知识卡片大纲
1. 内存安全设计架构：利用所有权与边界检查屏蔽 OpenSSL 类越界漏洞。
2. 1-RTT 握手协议与状态机：基于类型系统和 State 消费设计无旁路握手转换。
3. ClientHello 与 ServerHello 结构：套件选择、随机数交互与 KeyShare 密钥扩展协商。
4. 密码学适配接口与 CryptoProvider：定义底层加密引擎的可替换适配器。
5. 密钥派生与 HKDF 机制：基于 HMAC 的 Extract & Expand 逐级派生体系。
6. 会话恢复与预共享密钥 (PSK)：基于票据复用密钥降低握手握手延迟。
7. 会话票据 (Session Tickets) 分发：服务端无状态恢复票据加解密交互。
8. 记录协议与记录加密格式：帧格式头部定义及应用密文封装。
9. AEAD 对称加密：AES-GCM 与 ChaCha20-Poly1305 关联数据认证加密。
10. Transcript Hash 握手哈希校验：记录所有握手交互防中间人降级攻击。
11. 证书链验证与 webpki 集成：X.509 DER 解析与严格根证书路径校检。
12. 客户端证书验证与双向 TLS：服务端主动请求及客户端 Transcript 签名回复。
13. 签名方案协商：SignatureScheme 的明示列表协商机制。
14. 密钥交换算法执行：ECDHE 密钥协商计算和临时共享秘密导出。
15. 零往返延迟 (0-RTT) 早期数据：客户端第一飞发送加密数据及其重放风险防御。
16. 加密扩展传输：ServerHello 后的 EncryptedExtensions 加密参数协商。
17. 动态密钥更新 (KeyUpdate)：在单条 TCP 链接中轮转对称密钥防密钥爆破。
18. 警报协议错误处理：标准 Alert 帧传输与 Fatal/Warning 连接控制。
19. 规避 Heartbleed 心跳设计：剔除不必要的心跳协议并以内存检查杜绝信息泄漏。
20. 中间盒兼容模式：ChangeCipherSpec 假报文及 Legacy Session ID 注入。
21. Finished 结束校验消息计算：基于 Transcript Hash 的握手正确性终检。
22. 侧道时间攻击防范 (Constant-Time)：subtle 常数时间比对机制防时序泄露。
23. Rustls Config API 设计模型：Builder 模式构建不可变 API 防并发误修改。
24. 异步 I/O 异步集成：tokio-rustls 包装同步 Connection 驱动握手。
25. 证书锁 (Certificate Pinning)：自定义 Verifier 提取 SHA256 强验证证书。
26. 模糊测试与安全审计机制：cargo-fuzz 解码测试和内存泄漏漏洞审计。
27. 基准测试与性能调优对比：Criterion 基准测试及零拷贝切片优化。
28. 后量子密码学密钥协商：Kyber 算法混合密钥协商机制集成。
