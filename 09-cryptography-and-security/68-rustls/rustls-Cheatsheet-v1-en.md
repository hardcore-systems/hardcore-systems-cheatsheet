# Rustls Memory-Safe TLS 1.3 Protocol Stack Cheatsheet
## J-Ladder Hierarchical Model

### L0 One-Line Essence
Rustls is a modern, lightweight TLS 1.3/1.2 implementation written in Safe Rust, discarding insecure legacy components to enforce protocol safety via compile-time memory checks and modular crypto engines.

### L1 Four-Sentence Logic
1. **Memory Safety by Default**: Avoids raw pointer operations, running entirely within Rust's type and lifetime systems to eliminate buffer overflows and Use-After-Free.
2. **Strict Handshake State Machine**: Encodes protocol transitions into algebraic data types to guarantee step sequence, aborting immediately on out-of-order payloads.
3. **Pluggable Crypto Engines**: Decouples TLS core from math via `CryptoProvider` adapters, facilitating interchangeable backends like *ring*, AWS-LC, or HSMs.
4. **Resilience to Timing Attacks**: Employs constant-time binary comparison primitives to block information leaks during MAC and digital signature verifications.

### L2 Core Data Flow
`ClientConfig/ServerConfig setup` ➜ `CryptoProvider injection` ➜ `emit ClientHello` ➜ `ephemeral ECDHE key generation` ➜ `send KeyShare` ➜ `receive ServerHello & DH resolve` ➜ `HKDF derive Handshake Secrets` ➜ `Transcript Hash freeze & Finished check` ➜ `HKDF derive Application Secrets` ➜ `app write` ➜ `AEAD packet wrap` ➜ `transit` ➜ `AEAD decrypt check` ➜ `app read`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: Rustls Memory-Safe Architectural Design
*   **Theory**: Eliminates memory safety bugs at compile-time using Rust's strong type system and ownership tracking.
*   **Details**: Discards raw pointers, writing the entire stack in Safe Rust to prevent buffer overflows and dangling references. Limits support to TLS 1.2/1.3.
*   **Trade-off**: Lacks support for legacy protocols (SSLv3, TLS 1.0), meaning it cannot interact with legacy systems requiring outdated versions.

### Card 2: TLS 1.3 Handshake Protocol & State Machine
*   **Theory**: Uses algebraic data types to enforce handshake sequencing, establishing a secure channel in a single round trip (1-RTT).
*   **Details**: Models handshake states using strong enum types in `rustls::conn::HandshakeState`, consuming old states via ownership to prevent protocol transitions out of order.
*   **Trade-off**: Strict state transitions trigger a fatal UnexpectedMessage alert if any out-of-order message is received, disallowing loose OpenSSL-style fallback.

### Card 3: ClientHello & ServerHello Structures
*   **Theory**: Exchanges supported options between client and server to establish security consensus via cipher suites and key shares.
*   **Details**: ClientHello broadcasts supported CipherSuites, Supported Groups, and KeyShares. ServerHello confirms chosen parameters.
*   **Trade-off**: SNI is sent in plaintext, exposing target domains to observers. Integrating Encrypted Client Hello (ECH) is recommended for privacy.

### Card 4: Cryptographic Engine & CryptoProvider
*   **Theory**: Abstracts underlying cryptographic math to support pluggable software or hardware acceleration backends.
*   **Details**: Introduces `rustls::crypto::CryptoProvider` trait mapping signatures, AEADs, and key exchanges, defaulting to ring or AWS-LC.
*   **Trade-off**: Pluggable interfaces add modular abstraction overhead, and security is compromised if the third-party crypto engine has flaws.

### Card 5: Key Derivation & HKDF Mechanism
*   **Theory**: HMAC-based Extract-and-Expand Key Derivation Function (HKDF) derives multiple keys from negotiated shared secrets.
*   **Details**: Follows RFC 5869 to perform HKDF-Extract and HKDF-Expand using transcript hashes and custom labels to generate read/write keys.
*   **Trade-off**: Key derivation is CPU-intensive, requiring robust hash functions (SHA256/SHA384), adding noticeable CPU load in high-concurrency ephemeral connections.

### Card 6: Session Resumption & Pre-Shared Keys (PSK)
*   **Theory**: Reuses established security contexts to reduce cryptographic overhead during subsequent connection establishments.
*   **Details**: Associates derived PSKs with unique Resumption IDs after successful handshakes, allowing clients to send PSK extensions to bypass cert validation.
*   **Trade-off**: Long-lived PSKs without ephemeral ECDHE key exchanges lose forward secrecy if compromised, requiring combination with DHE.

### Card 7: Session Tickets & Distribution
*   **Theory**: Distributes encrypted session states to clients via tickets, enabling stateless session resumption across distributed servers.
*   **Details**: Server sends a NewSessionTicket encrypted with the server's master key, which the client presents in psk_identity on reconnection.
*   **Trade-off**: Master ticket encryption keys must be rotated regularly; a compromised master key exposes all associated historical tickets.

### Card 8: Record Layer Protocol & Encrypted Formatting
*   **Theory**: Structures the transport channel, slicing and encrypting all higher-layer protocol messages.
*   **Details**: Prepends a header defined in `rustls::record_layer::Record` containing 1-byte type, 2-byte version, and 2-byte length before encrypted payloads.
*   **Trade-off**: Record headers and length fields can expose payload sizes; zero-padding must be configured to obfuscate traffic analysis.

### Card 9: AEAD Symmetric Encryption
*   **Theory**: Authenticated Encryption with Associated Data (AEAD) ensures both confidentiality and integrity of messages.
*   **Details**: Mandates AES-128/256-GCM or ChaCha20-Poly1305, avoiding legacy mac-then-encrypt flows to encrypt and authenticate in one pass.
*   **Trade-off**: Demands strict uniqueness of Nonces; repeating a nonce leaks the key stream and completely breaks AEAD security.

### Card 10: Transcript Hash & Handshake Verification
*   **Theory**: Computes a running hash of all handshake messages to prevent active attackers from downgrading negotiation parameters.
*   **Details**: Pipes all raw sent and received handshake bytes into `rustls::hash_handshake::HandshakeHash` to derive final Finished verify data.
*   **Trade-off**: Requires buffer space to accumulate the transcript; though minimal, it adds slight memory footprint per concurrent connection.

### Card 11: Certificate Verification & webpki Integration
*   **Theory**: Validates server identity based on WebPKI standards, validating signature path legality and trust anchors.
*   **Details**: Delegates validation to the `rustls-webpki` crate, parsing DER-encoded X.509 certificates to check hostnames, dates, and sign paths.
*   **Trade-off**: Does not use OS OpenSSL trust stores directly; requires configuring `rustls-native-certs` or embedding `webpki-roots`.

### Card 12: Client Authentication & Mutual TLS (mTLS)
*   **Theory**: Allows servers to request and verify client certificates, establishing strong mutual peer-to-peer authentication.
*   **Details**: Server sends CertificateRequest. Client responds with Certificate and CertificateVerify containing a signature over the handshake transcript.
*   **Trade-off**: Double validation increases handshake latency and network overhead, demanding complex client certificate lifecycle management.

### Card 13: Signature Schemes Negotiation
*   **Theory**: Negotiates asymmetric signature algorithms and hash pairings for cryptographic signature generation and verification.
*   **Details**: Negotiates `SignatureScheme` enums like `ECDSA_NISTP256_SHA256` or `RSA_PSS_RSAE_SHA256` via client advertisements and server selections.
*   **Trade-off**: Deprecates insecure algorithms (MD5-RSA, SHA1-RSA), blocking access to older equipment that lacks modern signature scheme support.

### Card 14: Key Exchange Algorithms Execution
*   **Theory**: Ephemeral Diffie-Hellman (ECDHE/DHE) establishes symmetric keys over unsecure networks.
*   **Details**: Generates ephemeral key pairs on both sides, trading public keys in KeyShare extensions to compute identical Premaster Secrets.
*   **Trade-off**: Demands heavy modular exponentiation or elliptic curve math, creating a primary CPU bottleneck vulnerable to exhaustion DoS.

### Card 15: Zero-RTT (0-RTT) Early Data
*   **Theory**: Allows clients to bundle encrypted application data inside the first flight of the handshake, achieving zero-latency startup.
*   **Details**: Encrypts early data using keys derived from a previous session's PSK, transmitting it along with the ClientHello.
*   **Trade-off**: 0-RTT is vulnerable to replay attacks; servers must deploy anti-replay filters and limit 0-RTT to idempotent (GET) requests.

### Card 16: Encrypted Extensions Transmission
*   **Theory**: Moves non-cryptographic server extensions into the encrypted channel to reduce plaintext metadata leaks.
*   **Details**: Emits EncryptedExtensions immediately after ServerHello establishes handshake encryption, wrapping extensions like ALPN.
*   **Trade-off**: Adds decoding overhead; middleboxes cannot inspect ALPN fields for protocol analytics due to handshake encryption.

### Card 17: Dynamic Key Updates (KeyUpdate)
*   **Theory**: Rotates symmetric encryption keys within a single long-lived TCP connection to limit data volume encrypted per key.
*   **Details**: Signals a `KeyUpdate` handshake packet, executing HKDF Expand to roll over Application Traffic Secrets dynamically.
*   **Trade-off**: Frequent updates introduce minor communication latency, and flawed implementations can desynchronize cipher states, closing connections.

### Card 18: Alert Protocol & Error Handling
*   **Theory**: Provides a standardized exception signaling mechanism to notify peers of protocol conflicts or shutdowns.
*   **Details**: Alerts consist of a 1-byte level (Warning/Fatal) and 1-byte description (e.g., DecryptError), where Fatal alerts close the state machine.
*   **Trade-off**: Detailed alert codes can leak internal configuration states to adversaries; error reporting granularity must be carefully balanced.

### Card 19: Preventing Heartbleed-like Vulnerabilities
*   **Theory**: Avoids unsafe data echoing logics by cutting unnecessary historical protocol features from the code base.
*   **Details**: Excludes the TLS Heartbeat extension (RFC 6520) completely. Combined with Rust's slice bounds checking, out-of-bounds reads are impossible.
*   **Trade-off**: Omitting Heartbeats shifts active connection checking to the application layer or relies on native TCP keep-alives.

### Card 20: Middlebox Compatibility Mode
*   **Theory**: Obfuscates the handshake sequence to prevent legacy firewall middleboxes from dropping TLS 1.3 traffic.
*   **Details**: Injects a legacy 32-byte session ID in ClientHello and transmits dummy ChangeCipherSpec (CCS) messages after ServerHello.
*   **Trade-off**: Introduces redundant bytes in the network stream, consuming minor bandwidth and decoder parsing time to secure transit.

### Card 21: Finished Message Computation & Verification
*   **Theory**: Verifies symmetric keys match and ensures no unauthorized alterations occurred in transit during handshakes.
*   **Details**: Uses HMAC with a key derived from HKDF over the total Transcript Hash up to the Finished message; failure to match aborts connections.
*   **Trade-off**: Serves as the ultimate firewall for handshake success; any prior error in key exchange or certificate signing surfaces here as decrypt errors.

### Card 22: Side-Channel Timing Attack Prevention
*   **Theory**: Eliminates correlation between input values and processing time in crypto code to block timing attacks.
*   **Details**: Executes comparisons (MAC, signature checks) via constant-time routines (using subtle crate), utilizing bitwise operations instead of short-circuiting loops.
*   **Trade-off**: Disables fast-fail execution paths, demanding scan of the entire block even if the first byte mismatches, slightly increasing bad-request processing latency.

### Card 23: Rustls Configuration API Design Model
*   **Theory**: Implements type safety and immutability principles in config structures, preventing configuration misuse.
*   **Details**: Uses the Builder pattern to create `ClientConfig` and `ServerConfig`, wrapped in `Arc` for thread-safe sharing. Prevents mutations post `.build()`.
*   **Trade-off**: Completed config objects are immutable; runtime dynamic changes (e.g., cert rotation) require replacing the Arc reference or custom resolvers.

### Card 24: Asynchronous I/O (tokio-rustls) Integration
*   **Theory**: Wraps synchronous connection operations into non-blocking Future models suitable for high-concurrency services.
*   **Details**: `tokio-rustls` implements `AsyncRead`/`AsyncWrite` over internal sync connections, driving handshakes and stream transfers on socket readiness.
*   **Trade-off**: Async scheduling adds context switching and Pin allocations, leading to slightly lower peak throughput than highly optimized sync threads.

### Card 25: Certificate Pinning & Trust Anchors
*   **Theory**: Restricts accepted trust anchors to explicit public key hashes, bypassing global root CAs to mitigate rogue CA certificates.
*   **Details**: Implements custom `ServerCertVerifier` traits to hardcode target certificate SHA256 hashes, matching keys during handshake path checks.
*   **Trade-off**: If the remote certificate expires or changes, clients will lose connection immediately unless they perform active firmware upgrades.

### Card 26: Fuzzing & Security Audit Implementations
*   **Theory**: Validates parsing resilience against highly corrupted packet inputs via differential and mutation fuzzing to guarantee zero crashes.
*   **Details**: Integrates `cargo-fuzz` (libFuzzer) to test record decoders and ClientHello parsers with billions of mutant packets, ensuring zero panics.
*   **Trade-off**: Demands significant compute resources and seed corpuses; hard to achieve high code coverage in complex asynchronous handshake paths.

### Card 27: Benchmarking & Performance Optimization
*   **Theory**: Uses microbenchmarks to evaluate handshake latency and data throughput, locating allocation and crypto performance bottlenecks.
*   **Details**: Runs Criterion benchmarks, optimizing memory allocations via capacity pre-sizing (`Vec::with_capacity`) and zero-copy slice reads from transport.
*   **Trade-off**: Obsessive performance optimization decreases readability, such as replacing simple Arc wrappers with tight lifetime references.

### Card 28: Post-Quantum Cryptography (PQC) Key Exchange
*   **Theory**: Integrates quantum-resistant asymmetric mathematical systems to secure channels against future quantum decryptions.
*   **Details**: Integrates hybrid schemes like x25519+ML-KEM-768 (Kyber), bundling classical and quantum keys in KeyShare extensions for double exchange.
*   **Trade-off**: PQC public keys and ciphertexts are massive, forcing ClientHello packets to span multiple packets, causing IP fragmentation.
