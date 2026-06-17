# OpenSSL Cryptography Protocol & Secure Communication High-Density Cheatsheet

---

## 🪜 J-Ladder Knowledge Progression Hierarchy

*   **L0 Essence**: The essence of cryptographic secure communication is to establish a secure channel over an insecure network using symmetric/asymmetric algorithms ensuring confidentiality, integrity, authenticity, and non-repudiation. Its implementation centers on the TLS handshake state machine, hardware accelerator engines, and secure entropy source maintenance.
*   **L1 Four-Sentence Logic**:
    1. **Negotiation & Authentication**: Client and Server negotiate ciphers and verify identity using X.509 certificate chains within a single round-trip (1-RTT) in TLS 1.3.
    2. **Anti-Eavesdropping Key Exchange**: Generate ephemeral share keys using ECDHE (Elliptic Curve Diffie-Hellman Ephemeral) to derive shared symmetric session keys providing forward secrecy.
    3. **Authenticated Encryption (AEAD)**: Employ dual-purpose AEAD ciphers (e.g., AES-GCM) in the record protocol to secure payload data with high-strength symmetric encryption and anti-tamper MAC integrity verification.
    4. **Hardware Acceleration**: Automatically route cryptographic tasks via the EVP abstract layer to CPU AES-NI instructions or dedicated QAT accelerators, eliminating computation bottlenecks.
*   **L2 Core Data Flow Topology**:
    `ClientHello` ➜ (Negotiate suites and share keys) ➜ `ServerHello & Cert` ➜ (ECDHE secret extract) ➜ `HKDF key derivation` ➜ `Symmetric key established` ➜ `AEAD Record Encryption` ➜ `Encrypted Application Data`

---

## 📂 Cryptographic Communication Security & Strength Trade-off Matrix

| Design Dimension | Mainstream Mode A (AES-GCM / TLS 1.3) | Alternative Mode B (AES-CBC-HMAC / TLS 1.2) | Extreme Mode C (ChaCha20-Poly1305 / Mobile) | Physical Constraint & Bottom Trade-off |
| :--- | :--- | :--- | :--- | :--- |
| **Decryption Throughput** | Extremely High (Hardware-optimized via CPU AES-NI) | Moderate (Requires two steps: encrypt then authenticate) | Extremely High (Software logic speedup, optimized for CPUs without hardware acceleration) | Symmetric algorithms are heavily dependent on CPU multipliers. Mode A falls behind Mode C on embedded platforms without AES-NI. |
| **Handshake Latency (RTT)** | 1-RTT (Single round-trip negotiation) | 2-RTT (Traditional TLS 1.2 double round-trip negotiation) | 0-RTT (Session resumption data flight) | Network round-trip time is bounded by distance and the speed of light. 0-RTT mode eliminates latency but sacrifices replay attack resistance. |
| **Side-channel Exposure** | Extremely Low (No table-lookup, algorithm is strictly constant-time) | High (CBC padding leaks timing information) | Extremely Low (Arithmetic-only, resistant to cache side-channels) | Memory table lookups expose keys via CPU cache timing differences. Mode B splits plaintext padding, exposing it to Padding Oracle attacks. |
| **CPU Resource Overhead** | Low (Register-level hardware pipeline) | High (Double pass: first AES encrypt, then HMAC digest) | Low (Optimized for mobile platforms without dedicated cryptographic ALUs) | Software-level encryption blocks CPU execution. Mode B requires copying data twice in memory, creating CPU processing bottlenecks. |

---

## 🗂️ 28 Core Knowledge Cards Dictionary

### M1: Cryptography Basics, Randomness & EVP Interface (M1.1 - M1.5)

#### Card 1. Entropy Sources & DRBG Security Boundaries
*   **Core Definition**: Cryptographic strength relies on a cryptographically secure Deterministic Random Bit Generator (DRBG) fed by high-entropy sources.
*   **Bottom Mechanism**: OpenSSL gathers physical entropy from the OS (such as CPU RdRand instructions and kernel `/dev/urandom` interrupt noise). It feeds this into a NIST SP800-90A compliant DRBG, preventing state backtracking based on computational complexity, ensuring secure key and IV generation.

#### Card 2. EVP High-Level Unified Cryptographic API
*   **Core Definition**: The EVP API provides a unified, abstract interface for symmetric and asymmetric cryptographic operations.
*   **Bottom Mechanism**: EVP decouples application logic from specific algorithm implementations through object-oriented context binding. For instance, passing `EVP_aes_256_gcm()` to `EVP_EncryptInit_ex` allows the code to execute AES-GCM without exposing low-level structural details, enabling seamless Provider swaps.

#### Card 3. Symmetric Block Ciphers & Padding Schemes
*   **Core Definition**: Block ciphers (e.g., AES) require padding schemes to encrypt data that is not a multiple of the block size.
*   **Bottom Mechanism**: Traditional CBC mode chains cipherblocks by XORing the previous ciphertext block with the current plaintext. PKCS#7 pads the remainder with byte values equal to the padding length (e.g., three missing bytes are padded as `0x03 0x03 0x03`). If bad padding checks throw distinct errors, it leaks information via Padding Oracle timing.

#### Card 4. AEAD Authenticated Encryption & Integrity
*   **Core Definition**: AEAD ciphers combine encryption and integrity checking into a single operation.
*   **Bottom Mechanism**: AES-GCM merges Galois Field multiplication (GMAC) with CTR mode encryption. It outputs the ciphertext alongside a 128-bit authentication Tag. Decryption forces the hardware to verify the Tag against the AAD first. Any bit manipulation triggers authentication failure, preventing ciphertext modification attacks.

#### Card 5. Message Digest & HMAC Verification
*   **Core Definition**: HMAC is a key-hashed message authentication code that guarantees message integrity and authenticity.
*   **Bottom Mechanism**: HMAC uses a nested hashing structure: $HMAC_K(M) = H((K^+ \oplus opad) \parallel H((K^+ \oplus ipad) \parallel M))$. This design mathematically eliminates length extension attacks common in standard hashing functions, preventing signature forge attempts.

---

### M2: Symmetric Encryption & MAC (M2.1 - M2.5)

#### Card 6. Diffie-Hellman & Ephemeral Key Exchange (ECDHE)
*   **Core Definition**: Diffie-Hellman allows two parties to agree on a shared secret over an insecure channel.
*   **Bottom Mechanism**: Ephemeral Elliptic Curve Diffie-Hellman (ECDHE) generates a private scalar $d$ and public point $Q = d \cdot G$ for each session. The shared secret is derived as $S = d_{client} \cdot Q_{server} = d_{client} d_{server} G$. Discarding the private key after the session ensures Forward Secrecy.

#### Card 7. RSA Asymmetric Cipher & OAEP Padding
*   **Core Definition**: RSA relies on the mathematical complexity of prime factorization, requiring secure padding for confidentiality.
*   **Bottom Mechanism**: Legacy RSA PKCS#1 v1.5 padding is vulnerable to Bleichenbacher's timing attack. OpenSSL enforces RSA-OAEP (Optimal Asymmetric Encryption Padding), which uses a Mask Generation Function (MGF1) to randomize plaintext before performing modular exponentiation: $C = M^e \pmod N$, ensuring IND-CCA2 security.

#### Card 8. ECC Curves & ECDSA Signature Verification
*   **Core Definition**: ECC provides asymmetric security based on elliptic curve discrete logarithm problems using shorter keys.
*   **Bottom Mechanism**: ECDSA signature generation ($r, s$) requires a unique random nonce $k$. If $k$ is reused, the private key can be mathematically extracted. OpenSSL implements RFC 6979 to derive $k$ deterministically from the private key and message hash, eliminating nonce reuse vulnerabilities.

#### Card 9. ASN.1 Standards & DER/PEM Serialization
*   **Core Definition**: ASN.1 is a standard language for describing structured cryptographic objects such as keys and certificates.
*   **Bottom Mechanism**: DER is a binary, space-efficient Distinguished Encoding Rules format utilizing Tag-Length-Value (TLV) encoding. PEM is a Base64 encoded DER file wrapped in headers like `-----BEGIN CERTIFICATE-----` to facilitate transport over text-based networks.

#### Card 10. X.509 Certificate Chain Trust Validation
*   **Core Definition**: X.509 defines the standard format for public-key certificates linking identities to public keys.
*   **Bottom Mechanism**: Validation traverses a tree to a trusted Root CA. The client checks the CA's signature on the certificate: $S = Sign_{CA}(Hash(CertificateBody))$. A valid cryptographic chain validates the server's identity, preventing man-in-the-middle attacks.

---

### M3: Asymmetric Encryption & TLS Protocols (M3.1 - M3.5)

#### Card 11. TLS 1.2 Handshake & RTT Delay
*   **Core Definition**: The TLS 1.2 handshake negotiates security parameters and establishes keys in two network round-trips (2-RTT).
*   **Bottom Mechanism**: The steps involve: 1. `ClientHello`/`ServerHello` (cipher suite negotiation); 2. `Certificate` / `ServerKeyExchange` (CA validation and DH parameters); 3. `ClientKeyExchange` (client sends key exchange material); 4. `Finished`. This results in a 2-RTT latency penalty before data transmission.

#### Card 12. TLS 1.3 1-RTT Handshake Optimization
*   **Core Definition**: TLS 1.3 simplifies the handshake to a single round-trip (1-RTT) to reduce latency.
*   **Bottom Mechanism**: The client speculatively includes a Key Share (e.g., X25519 public key) in the `ClientHello`. The server calculates the shared secret immediately, returning its Key Share and encrypted certificate in `ServerHello`. Symmetrical session keys are established in 1-RTT.

#### Card 13. HKDF Key Derivation Tree
*   **Core Definition**: HKDF is the core key derivation function in TLS 1.3 used to extract and expand keys.
*   **Bottom Mechanism**: Steps include: 1. **Extract**: $PRK = HMAC\_Hash(salt, IKM)$ concentrates entropy into a pseudorandom key; 2. **Expand**: $OKM = HKDF\_Expand(PRK, info, L)$ generates independent session keys (Client/Server write keys and IVs) iteratively, avoiding cross-key leakage.

#### Card 14. Session Tickets & 0-RTT Replay Risks
*   **Core Definition**: Session resumption allows returning clients to skip handshake validation, while 0-RTT enables sending payload in the first packet.
*   **Bottom Mechanism**: The server encrypts session state into a Ticket session bundle given to the client. Upon reconnecting, the client sends this Ticket to restore session state. However, 0-RTT data lack replay protection; attackers can intercept and re-send these packets (Replay Attack). It must be restricted to idempotent GET requests.

#### Card 15. TLS Record Protocol Frame Encapsulation
*   **Core Definition**: The Record Protocol is responsible for fragmenting, compressing, and encrypting transport payload.
*   **Bottom Mechanism**: Plaintext is partitioned into frames up to 16KB. Each frame gets a header (type, version, length), and is encrypted using the session AEAD cipher keyed with a monotonic Sequence Number. The receiver checks the sequence number, neutralizing packet injection and out-of-order replay attacks.

---

### M4: Memory Security & Side-Channel Protections (M4.1 - M4.4)

#### Card 16. OPENSSL_malloc Secure Memory Management
*   **Core Definition**: OpenSSL manages secure memory pools to prevent sensitive key material from leaking to disk or memory space.
*   **Bottom Mechanism**: The allocator locks memory pages (using OS calls like `mlock`), preventing the OS from swapping the pages to swap partitions on disk. It intercepts out-of-memory states and halts immediately on corruption to avoid exploitation.

#### Card 17. Timing Attacks & Constant-Time Operations
*   **Core Definition**: Timing attacks extract keys by measuring differences in execution times during cryptographic calculations.
*   **Bottom Mechanism**: Code branch logic like `if (key_bit == 1)` alters CPU execution timing. OpenSSL uses constant-time algorithms for critical calculations, utilizing bitwise masks (`val & mask`) instead of conditional branches to ensure execution time is independent of key values.

#### Card 18. Zeroization & Memory Cleansing
*   **Core Definition**: Zeroization ensures that sensitive cryptographic variables are erased immediately after use.
*   **Bottom Mechanism**: Standard `memset` calls can be optimized out by compilers if the memory is not accessed again (Dead-code elimination). OpenSSL implements `OPENSSL_cleanse`, using memory barrier primitives and non-inlined pointers to force zeroing in physical memory.

#### Card 19. Heartbleed Vulnerability Root Cause (CVE-2014-0160)
*   **Core Definition**: Heartbleed was a critical vulnerability in OpenSSL's implementation of the TLS Heartbeat extension.
*   **Bottom Mechanism**: The server read a client-reported payload length field without verifying the actual packet size. When allocating the response buffer, it executed `memcpy` based on the fake, larger length, reading adjacent process memory containing session keys and returning it to the attacker.

---

### M5: BIO I/O Pipeline & Provider Slots (M5.1 - M5.5)

#### Card 20. Side-Channel Cache Timing & RSA Blinding
*   **Core Definition**: Cache timing attacks observe CPU cache misses to infer exponent patterns in modular exponentiation.
*   **Bottom Mechanism**: OpenSSL applies RSA Blinding: it multiplies the plaintext $M$ by a random factor $r^e \pmod N$ before modular exponentiation: $M' = M \cdot r^e \pmod N$. Decryption outputs are multiplied by $r^{-1}$ to remove the bias. This randomizes CPU cache access patterns, nullifying cache analysis.

#### Card 21. EVP_PKEY Asymmetric Interface Abstraction
*   **Core Definition**: EVP_PKEY wraps diverse asymmetric algorithms (RSA, ECC, DH) under a unified key abstraction structure.
*   **Bottom Mechanism**: It wraps concrete key structures (`RSA*`, `EC_KEY*`) into a generic `EVP_PKEY` handler. Operations like signature generation (`EVP_PKEY_sign`) use unified wrappers, isolating application logic from changes in the underlying asymmetric curves or math libraries.

#### Card 22. SSL_CTX Configuration Context Management
*   **Core Definition**: SSL_CTX is a shared context that configures certificates, trusted CA roots, and protocol parameters for TLS connections.
*   **Bottom Mechanism**: Designed as a singleton wrapper, SSL_CTX loads CA certificates, CRLs, and enforce cipher suites (e.g., disabling SSLv3 and legacy cipher algorithms). Connection-specific `SSL*` structures inherit these settings, optimizing memory.

#### Card 23. BIO I/O Stream Pipeline Abstraction
*   **Core Definition**: BIO (Basic Input/Output) is an abstraction layer that wraps files, sockets, memory buffers, and SSL layers into stream pipelines.
*   **Bottom Mechanism**: BIO blocks can be chained together: `BIO_new_ssl` ➜ `BIO_push` ➜ `BIO_new_socket`. Writing to the top BIO automatically filters data down, encrypting it before writing to the underlying network socket.

#### Card 24. OpenSSL Engine Hardware Integration Interface (v1.1.1)
*   **Core Definition**: The Engine interface allowed OpenSSL v1.1.1 to offload cryptographic calculations to external ASIC hardware.
*   **Bottom Mechanism**: It exposes a callback vtable for cryptographic algorithms. Hardware vendors provided dynamic libraries registering implementations (e.g., `ENGINE_set_RSA`), letting OpenSSL dynamically route calls to accelerator cards.

---

### M6: Hardware Accelerators & Assembly Tuning (M6.1 - M6.4)

#### Card 25. OpenSSL 3.0 Provider Slot Architecture
*   **Core Definition**: Providers replace Engines in OpenSSL 3.0, introducing a modular architecture.
*   **Bottom Mechanism**: Primitives are categorized into namespaces: default, fips, and legacy. Algorithms are queried by name strings (e.g., `"AES-256-GCM"`). This removes compile-time dependencies, letting developers load, unload, and prioritize custom cryptographic providers at runtime.

#### Card 26. Intel QAT Cryptographic Core Offloading
*   **Core Definition**: Intel QuickAssist Technology (QAT) offloads heavy cryptographic workloads to external acceleration chips.
*   **Bottom Mechanism**: The QAT Provider routes asymmetric and symmetric operations to physical PCIe accelerators. Tasks are queued to hardware ring buffers asynchronously, allowing the CPU to execute network routing while the chip handles encryption.

#### Card 27. AES-NI & CPU Vector Cryptographic Instructions
*   **Core Definition**: Hardware-accelerated instructions built into the CPU (e.g., Intel AES-NI) speed up block encryption.
*   **Bottom Mechanism**: EVP automatically queries CPU capability flags. If supported, it executes assembly code leveraging instructions like `AESENC` / `AESDEC` and `PCLMULQDQ` (carry-less multiplication for GCM). This accelerates GCM throughput by orders of magnitude while reducing CPU load.

#### Card 28. Cipher Suite Auditing & Policy Enforcement
*   **Core Definition**: Compliance auditing requires disabling legacy and compromised algorithms in corporate servers.
*   **Bottom Mechanism**: Weak algorithms like RC4 (vulnerable to bias attacks), 3DES (short keys and slow execution), and MD5 (hash collision risks) are disabled. The SSL_CTX blocks these suites via list filters (e.g., `DEFAULT:!RC4:!3DES:!MD5`), preventing downgrade attacks.
