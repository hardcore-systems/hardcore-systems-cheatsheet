# Intel SGX Enclave Secure Computing & Hardware Isolation Cheatsheet

## J-Ladder Architecture Framework

### L0 Essence
The essence of Intel SGX is to partition cryptographically isolated physical memory regions (EPC) via CPU hardware, protecting confidentiality and integrity against malicious OS, hypervisor, or hardware physical probing using the Memory Encryption Engine (MEE) and Enclave Page Cache Map (EPCM).

### L1 Logical Framework
1. **Hardware Memory Encryption**: CPU allocates EPC pages within the PRM, where all EPC data flowing in and out of the CPU cache is encrypted/decrypted in real-time by the MEE.
2. **Strict Boundary Isolation**: Inter-enclave execution transitions through hardware-specific instructions. Any interrupts or exceptions trigger an AEX to clear CPU registers and store context in the SSA.
3. **Chain of Trust Measurement**: Enclaves are constructed via ECREATE and EADD instructions, with code and initial data measured into the immutable MRENCLAVE cryptographic hash register.
4. **Remote Attestation Link**: Local enclaves generate reports via EREPORT, which are signed by the Quoting Enclave (QE) into ECDSA-verifiable Quote certificates validated by Intel PCS.

### L2 Core Data Flow Topology
`ECREATE` ➜ `EADD` ➜ `EEXTEND` ➜ `EINIT` ➜ `EENTER(ECALL)` ➜ `MEE Encrypted Read/Write` ➜ `EREPORT` ➜ `QE Quote Signing` ➜ `Intel PCS Validation`

---

## 📂 Core Knowledge Cards (Cards 1-28)

### Card 1: Processor Reserved Memory (PRM) & EPC Allocation
*   **Core Principle**: During BIOS initialization, the PRMRR register sets aside a contiguous segment of RAM called Processor Reserved Memory (PRM) that is inaccessible to any off-CPU devices (including DMA). The Enclave Page Cache (EPC) resides inside the PRM to hold secure enclaves.
*   **Technical Details**: EPC pages are allocated via the SGX kernel driver. In Linux 5.11+, the core allocator function is `sgx_alloc_epc_page`, distributing 4KB physical pages. PRM size is bios-configured and limited; exceeding it causes paging.
*   **Source Location**: [main.c:L30-80](file:///arch/x86/kernel/cpu/sgx/main.c#L30-80)

### Card 2: EPCM Hard Access Check
*   **Core Principle**: EPCM (Enclave Page Cache Map) is an internal hardware table within the CPU storing safety metadata for each EPC page (permissions R/W/X, page type SECS/REG/TCS, and owner SECS ID).
*   **Technical Details**: Even if the OS maps an EPC physical page to an untrusted process, the MMU checks the EPCM and triggers a General Protection fault (#GP) if the processor is not executing inside the matching Enclave context.
*   **Source Location**: [encl.c:L120-160](file:///arch/x86/kernel/cpu/sgx/encl.c#L120-160)

### Card 3: Memory Encryption Engine (MEE)
*   **Core Principle**: The MEE is embedded in the memory controller. It encrypts cache lines written to PRM via AES-XTS and decrypts them on read, shielding data from hardware probes.
*   **Technical Details**: To prevent replay and relocation attacks, MEE builds a Memory Cryptographic Tree (MCTS) with its root hash stored in CPU registers, turning bus communications into pseudorandom noise.
*   **Source Location**: [main.c:L180-210](file:///arch/x86/kernel/cpu/sgx/main.c#L180-210)

### Card 4: SGX Lifecycle Instructions (ECREATE/EADD/EINIT/EEXTEND)
*   **Core Principle**: Enclave setup sequence: `ECREATE` initializes SECS metadata; `EADD` copies pages into the EPC; `EEXTEND` cryptographically hashes 256-byte chunks of pages; `EINIT` seals the enclave measurement registers.
*   **Technical Details**: `EINIT` requires a cryptographically signed SIGSTRUCT file containing a valid Launch Token. Enclave initialization fails if platform launch keys are mismatched.
*   **Source Location**: [ioctl.c:L200-290](file:///ioctl.c#L200-290)

### Card 5: Enclave Execution Limit
*   **Core Principle**: Once in Enclave mode, the CPU operates in a restricted User Mode (Ring 3), and most privileged system instructions (e.g., IN, OUT, INT, CLI, STI) are hardware-blocked.
*   **Technical Details**: Any attempts to change privilege rings or handle I/O within the enclave force an AEX, preserving the security integrity of the enclave and the host OS.
*   **Source Location**: [encl.c:L300-340](file:///arch/x86/kernel/cpu/sgx/encl.c#L300-340)

### Card 6: AEX State Cleanup
*   **Core Principle**: When interrupts/exceptions occur inside the enclave, CPU prevents register leakage to untrusted handlers by executing an Asynchronous Enclave Exit (AEX), clearing all general-purpose registers.
*   **Technical Details**: The hardware saves the active CPU register values onto the enclave's State Save Area (SSA) frame. Control resumes via `ERESUME`, restoring the SSA states back.
*   **Source Location**: [assembly.S:L10-60](file:///sdk/simulation/assembly/assembly.S#L10-60)

### Card 7: Page Fault Side-Channel
*   **Core Principle**: The host OS retains page table control. A malicious OS can intentionally trigger selective page faults (#PF) to trace internal control flow based on memory access patterns.
*   **Technical Details**: Mitigations involve software techniques such as Oblivious RAM (ORAM) or utilizing hardware timers to detect page fault cycle frequency anomalies.
*   **Source Location**: [encl.c:L410-450](file:///arch/x86/kernel/cpu/sgx/encl.c#L410-450)

### Card 8: L1TF / Foreshadow Defence
*   **Core Principle**: Foreshadow exploits CPU speculative execution. Even when page tables mark data as non-present, out-of-order execution loads EPC cache lines from L1D Cache.
*   **Technical Details**: Mitigations include disabling Hyper-Threading (HT), flushing L1 Data Cache on enclave transitions, and restricting core pin mappings in microcode.
*   **Source Location**: [main.c:L310-350](file:///arch/x86/kernel/cpu/sgx/main.c#L310-350)

### Card 9: TCS & SSA Struct
*   **Core Principle**: TCS (Thread Control Structure) is a hardware-reserved page specifying thread entry points (OENTRY) and SSA offsets. Each active CPU thread inside the enclave requires a distinct TCS.
*   **Technical Details**: The SSA holds space for saving general-purpose registers, floating-points, and SSE states. These pages must be added via `EADD` before calling `EINIT`.
*   **Source Location**: [encl.h:L50-80](file:///arch/x86/kernel/cpu/sgx/encl.h#L50-80)

### Card 10: EPC Page Swapping
*   **Core Principle**: If EPC memory gets exhausted, the OS can swap enclave pages to normal RAM. This requires executing `EBLOCK` to block accesses, and `ETRACK` to clear TLB cache.
*   **Technical Details**: The driver invokes `EWB` to encrypt and copy the target page into standard system memory, along with a version token. `ELDU` reloads and decrypts the page.
*   **Source Location**: [ioctl.c:L340-410](file:///arch/x86/kernel/cpu/sgx/ioctl.c#L340-410)

### Card 11: Local Attestation
*   **Core Principle**: Intra-platform enclaves establish mutual trust using `EREPORT`, which generates a report cryptographically bound to the target's SECS measurement via a platform-specific symmetric key.
*   **Technical Details**: The target enclave calls `EGETKEY` to derive the same Report Key and verifies the MAC checksum, ensuring both run on the same genuine SGX processor.
*   **Source Location**: [attestation/local.c:L15-60](file:///sdk/attestation/local.c#L15-60)

### Card 12: Remote Attestation Quote
*   **Core Principle**: Remote attestation wraps an enclave report into an exportable certificate (Quote). The report is handed over to the Quoting Enclave (QE), a local architectural enclave.
*   **Technical Details**: QE validates the local report, then signs it using a platform-specific private key issued by the PCE, transforming the report into an off-platform Quote.
*   **Source Location**: [attestation/quote.c:L40-95](file:///sdk/attestation/quote.c#L40-95)

### Card 13: Intel PCS Certificate
*   **Core Principle**: The verifying party needs to prove that the QE Quote public key originates from a genuine Intel CPU. Intel PCS publishes PCK certificate CRLs online.
*   **Technical Details**: Verifiers fetch the platform's PCK (Processor Certification Key) cert chain signed by Intel CA to verify the hardware trust path back to Intel's factory.
*   **Source Location**: [attestation/pcs.c:L10-45](file:///sdk/attestation/pcs.c#L10-45)

### Card 14: EPID vs ECDSA Signatures
*   **Core Principle**: Early SGX Remote Attestation relied on Intel EPID (Enhanced Privacy ID) zero-knowledge group signatures. Modern SGX uses standard ECDSA signatures for local verification.
*   **Technical Details**: ECDSA quotes are signed by QE3 and validated using local cached certificate servers (PCCS), eliminating internet latencies to Intel's cloud.
*   **Source Location**: [attestation/ecdsa.c:L50-110](file:///sdk/attestation/ecdsa.c#L50-110)

### Card 15: MRENCLAVE & MRSIGNER
*   **Core Principle**: `MRENCLAVE` is a SHA-256 hash measuring the enclave's code, data, and load order ("what it is"). `MRSIGNER` is the hash of the developer's public signing key ("who signed it").
*   **Technical Details**: Enclave updates change `MRENCLAVE` but preserve `MRSIGNER`. This allows developers to use `MRSIGNER` policy to share sealed database keys across versions.
*   **Source Location**: [sdk/tlibc/tattest.c:L20-55](file:///sdk/tlibc/tattest.c#L20-55)

### Card 16: ECALL/OCALL Context Switch
*   **Core Principle**: ECALL transfers control inside the enclave via `EENTER`/`EEXIT`. OCALL makes callbacks to host functions. Each transition incurs heavy registers/state flush overhead.
*   **Technical Details**: Context switching costs several thousand CPU cycles. Large-scale enclaves employ "Switchless Enclaves" utilizing shared lockless ring buffers to prevent exit.
*   **Source Location**: [edger8r/codegen.ml:L40-80](file:///sdk/edger8r/codegen.ml#L40-80)

### Card 17: Parameter Marshalling
*   **Core Principle**: Enclave code must not directly dereference memory pointers passed from the outside (vulnerable to TOCTOU attacks). Enclave bridge code must copy arguments securely.
*   **Technical Details**: Auto-generated Edger8r boundary wrappers verify whether pointers lie strictly outside the enclave boundary, copy content into EPC heap, then perform operations.
*   **Source Location**: [edger8r/parser.ml:L100-140](file:///sdk/edger8r/parser.ml#L100-140)

### Card 18: SGX2 EDMM Dynamic Memory
*   **Core Principle**: SGX1 required enclave EPC sizes to be statically defined at compilation. SGX2 introduces EDMM (Enclave Dynamic Memory Management) hardware support.
*   **Technical Details**: EDMM allows runtime page allocation (`EMODT`) and permission changes (`EMODPR`), enabling dynamic heap allocation via standard `malloc` routines.
*   **Source Location**: [sdk/trts/trts_util.cpp:L30-70](file:///sdk/trts/trts_util.cpp#L30-70)

### Card 19: Runtime LibOS Systems
*   **Core Principle**: Since enclaves cannot trigger system calls directly, running unmodified binaries (like Redis or Nginx) requires a Library OS (e.g. Gramine/Occlum) compatibility layer.
*   **Technical Details**: The LibOS intercepts standard system calls and simulates them inside the enclave (such as secure file systems or network loops), shielding the host OS from raw calls.
*   **Source Location**: [sdk/tlibc/string/memset.c:L5-20](file:///sdk/tlibc/string/memset.c#L5-20)

### Card 20: Debug vs Prod Mode
*   **Core Principle**: Debug mode enclaves configure the `DEBUG` flag inside SECS, allowing the CPU to execute `EDGRD` and `EDWR` instructions to inspect EPC pages.
*   **Technical Details**: Production enclaves set `DEBUG` to 0. Hardware debug registers and JTAG debuggers are physically locked out, returning null or causing hard crash.
*   **Source Location**: [sdk/simulation/tcs.cpp:L40-75](file:///sdk/simulation/tcs.cpp#L40-75)

### Card 21: XML Configurations
*   **Core Principle**: Enclave parameters (StackSize, HeapSize, TCSNum) are defined in a custom XML configuration file loaded during compilation.
*   **Technical Details**: Attributes are parsed by `sgx_sign` and packed into the SIGSTRUCT metadata, matching the MRENCLAVE measurement upon `ECREATE` register checks.
*   **Source Location**: [sdk/sign_tool/SignTool/SignTool.cpp:L120-170](file:///sdk/sign_tool/SignTool/SignTool.cpp#L120-170)

### Card 22: sgx_sign & SIGSTRUCT
*   **Core Principle**: `sgx_sign` generates the SIGSTRUCT file containing measurements, attributes, and public key of the developer, signed with their RSA-3072 private key.
*   **Technical Details**: The hardware instruction `EINIT` validates the signature before executing code. This prevents malicious modification of EPC code files.
*   **Source Location**: [sdk/sign_tool/SignTool/sgx_sign.cpp:L200-280](file:///sdk/sign_tool/SignTool/sgx_sign.cpp#L200-280)

### Card 23: Simulation Mode Boundaries
*   **Core Principle**: Simulation Mode runs on non-SGX hardware by mimicking EPCM structures and AEX flows in software.
*   **Technical Details**: MEE encryption is missing in simulation mode, and local attestation reports rely on simulated software keys. Must never be used in production.
*   **Source Location**: [sdk/simulation/assembly/](file:///sdk/simulation/assembly/)

### Card 24: Architectural Enclaves
*   **Core Principle**: Architectural Enclaves (AEs) are system enclaves developed by Intel (LE, QE, PCE) to manage platform-specific tasks.
*   **Technical Details**: AEs possess keys signed by Intel Root. They can access hardware-specific platform keys that normal user enclaves are restricted from reading.
*   **Source Location**: [psw/ae/common/](file:///psw/ae/common/)

### Card 25: EPC Paging Tuning
*   **Core Principle**: Paging thrashing happens when an enclave allocates memory exceeding physical EPC limits, triggering heavy encryption/decryption overhead.
*   **Technical Details**: Tuning involves leveraging NUMA-aware layouts, reducing global scans, adopting cache-friendly data structures, and optimizing XML initial allocation sizes.
*   **Source Location**: [arch/x86/kernel/cpu/sgx/main.c#L450-510](file:///arch/x86/kernel/cpu/sgx/main.c#L450-510)

### Card 26: Multi-Enclave IPC
*   **Core Principle**: Enclaves are memory-isolated from each other. They communicate by establishing shared symmetric keys or using non-EPC shared RAM pages.
*   **Technical Details**: Enclaves exchange encrypted messages via non-EPC ring buffers. The message is decrypted only inside the target enclave, avoiding OS sniffing.
*   **Source Location**: [sdk/attestation/local.c#L80-120](file:///sdk/attestation/local.c#L80-120)

### Card 27: KVM vEPC Virtualization
*   **Core Principle**: KVM supports virtual EPC (vEPC) by directly mapping physical EPC segments into guest VM memory spaces.
*   **Technical Details**: Hypervisors trap guest VM `ECREATE` and `EINIT` operations, updating EPT page tables to handle virtual EPC translations seamlessly.
*   **Source Location**: [kvm/vmx/sgx.c:L20-100](file:///arch/x86/kvm/vmx/sgx.c#L20-100)

### Card 28: SVN & TCB Recovery
*   **Core Principle**: `ISVSVN` denotes enclave versions. Microcode updates increment physical CPUSVN to recover from architectural security vulnerabilities.
*   **Technical Details**: Following TCB recovery, PCS revokes old PCK certificates. Enclaves compare SVN and CPUSVN, refusing to start if host lacks vital security patches.
*   **Source Location**: [sdk/tlibc/tattest.h:L10-40](file:///sdk/tlibc/tattest.h#L10-40)

---

## 📂 Cryptographic Communication Security & Strength Trade-off Matrix

| Design Dimension | Strong TEE Pattern | Performance Optimized Pattern | Trade-off Matrix Analysis |
| :--- | :--- | :--- | :--- |
| **Enclave Switch** | Synchronous ECALL/OCALL | Switchless ECALL/OCALL | Synchronous ECALL triggers AEX context switches (~8k cycles); Switchless uses shared ring buffer worker threads to avoid switching overhead, boosting throughput but occupying a full CPU core. |
| **Memory Encryption** | MEE with Replay Protection | SGA Encryption (No Tree) | Memory cryptographic trees (MCTS) prevent replay and splice attacks but double DRAM latency; SGA-only is faster but vulnerable to active physical hardware sniffing. |
| **Attestation Flow** | Live Remote Quote Generation | Cached PCK/Quote Verification | Live Quote Generation provides fresh attestation but relies on slow online connections; cached verification avoids PCS API latency but introduces certificate revocation lag. |
| **EPC Paging** | EDMM Dynamic Page Allocation | Static Large EPC Allocation | EDMM dynamically expands EPC but increases page fault interrupt handling; static allocation has slow boot time but offers steady execution without page faults. |

