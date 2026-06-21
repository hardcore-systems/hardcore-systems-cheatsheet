# 🚀 Hardcore Systems Cheatsheet (Visual Systems Engineering Maps)

> **High-density visual cheatsheet system compressing 25 benchmark open-source codebases and system engineering classics. Speedrun OS kernels, distributed databases, and high-performance networks in 3 hours.**

[![GitHub Stars](https://img.shields.io/github/stars/hardcore-systems/hardcore-systems-cheatsheet?style=for-the-badge&color=blue)](https://github.com/hardcore-systems/hardcore-systems-cheatsheet)
[![Sponsor on GitHub](https://img.shields.io/badge/Sponsor-GitHub-pink?style=for-the-badge&logo=github)](https://github.com/sponsors/bsjzmj)
[![Get Bundle on Gumroad](https://img.shields.io/badge/Gumroad-Get%20Bundle-orange?style=for-the-badge&logo=gumroad)](https://gumroad.com)

---

## 🖼️ Landscape Preview

Every visual poster in this project is carefully rendered using the professional **XeLaTeX + PGF/TikZ** graphics system with a customized **Morandi aesthetic color palette**, aiming to reveal the truth under giant codebases through clean physical topologies.

* 📥 **[RocksDB Visual Sheet Preview (High-Res Vector PDF)](https://raw.githubusercontent.com/hardcore-systems/hardcore-systems-cheatsheet/main/docs/rocksdb/cheatsheet.pdf)**
* 💻 **[RocksDB Online Knowledge Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/)** | **[RocksDB Interactive Simulation Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/sandbox.html)**

---

## 💡 Core Philosophy: Why This?

Modern software engineers, system designers, and architects face two suffocating bottlenecks when improving their technical depth:
1. **Endless codebase size lacking a "first principles" skeleton**: Giant projects like the Linux kernel, Kubernetes, or the V8 engine have millions of lines of code. Reading the source code directly without a macro-level "physical topology map" is like walking in a rainforest without a map—it is extremely easy to get lost in details.
2. **Abstract documentation missing the key engineering signal**: Regular technical articles and architecture diagrams tend to draw abstract boxes and arrows, hiding the most critical lower-level details (such as Page Cache, Lock Contention, Memory Barriers, Concurrency Conflict Control mechanisms, etc.).

To break this "knowledge black box," we propose a **"First-Principles Deconstruction Framework"**. For every complex underlying technical stack, we precisely extract and deconstruct it into the following physical layers:
* **L0 One-Sentence Essence**: Explains the project's absolute core purpose in the computer world under first principles.
* **L1 4-Line Core Logic**: Uses 4 lines of code or pseudo-code to capture the core work loop or state machine of the system.
* **L2 Physical Data Flow Topology**: Restores the actual memory layout, disk file organization, or network packet timeline.
* **M1~M6 Concept Cards Matrix (28 Cards)**: Every cheatsheet is packed with 28 core concept cards covering protocol specs, key algorithms, performance optimization, and production troubleshooting tips, with zero fluff.

---

## 📂 Catalog & Interactive Maps (25 Core Projects)

| ID | Project Name | Markdown Source | PDF Preview (Low-Res) | Online Map | Simulation Sandbox |
|:---:|:---|:---:|:---:|:---:|:---:|
| **18** | `EBPF_BCC` | [CN](./03-operating-systems/18-eBPF_bcc/eBPF_bcc-Cheatsheet-v1.md) / [EN](./03-operating-systems/18-eBPF_bcc/eBPF_bcc-Cheatsheet-v1-en.md) | [CN](./docs/eBPF_bcc/cheatsheet.pdf) / [EN](./docs/eBPF_bcc/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/eBPF_bcc/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/eBPF_bcc/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/eBPF_bcc/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/eBPF_bcc/sandbox-en.html) |
| **19** | `ROCKSDB` | [CN](./02-database-internals/19-rocksdb/rocksdb-Cheatsheet-v1.md) / [EN](./02-database-internals/19-rocksdb/rocksdb-Cheatsheet-v1-en.md) | [CN](./docs/rocksdb/cheatsheet.pdf) / [EN](./docs/rocksdb/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/sandbox-en.html) |
| **20** | `ETCD` | [CN](./01-distributed-systems/20-etcd/etcd-Cheatsheet-v1.md) / [EN](./01-distributed-systems/20-etcd/etcd-Cheatsheet-v1-en.md) | [CN](./docs/etcd/cheatsheet.pdf) / [EN](./docs/etcd/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/etcd/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/etcd/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/etcd/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/etcd/sandbox-en.html) |
| **29** | `XV6_RISCV` | [CN](./03-operating-systems/29-xv6_riscv/xv6_riscv-Cheatsheet-v1.md) / [EN](./03-operating-systems/29-xv6_riscv/xv6_riscv-Cheatsheet-v1-en.md) | [CN](./docs/xv6_riscv/cheatsheet.pdf) / [EN](./docs/xv6_riscv/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/xv6_riscv/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/xv6_riscv/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/xv6_riscv/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/xv6_riscv/sandbox-en.html) |
| **34** | `SYSTEM_DESIGN_PRIMER` | [CN](./05-system-design/34-system_design_primer/system_design_primer-Cheatsheet-v1.md) / [EN](./05-system-design/34-system_design_primer/system_design_primer-Cheatsheet-v1-en.md) | [CN](./docs/system_design_primer/cheatsheet.pdf) / [EN](./docs/system_design_primer/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_design_primer/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_design_primer/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_design_primer/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_design_primer/sandbox-en.html) |
| **36** | `SRE_BOOK` | [CN](./05-system-design/36-sre_book/sre_book-Cheatsheet-v1.md) / [EN](./05-system-design/36-sre_book/sre_book-Cheatsheet-v1-en.md) | [CN](./docs/sre_book/cheatsheet.pdf) / [EN](./docs/sre_book/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sre_book/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sre_book/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sre_book/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sre_book/sandbox-en.html) |
| **40** | `V8` | [CN](./06-compilers-and-runtimes/40-v8/v8-Cheatsheet-v1.md) / [EN](./06-compilers-and-runtimes/40-v8/v8-Cheatsheet-v1-en.md) | [CN](./docs/v8/cheatsheet.pdf) / [EN](./docs/v8/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/v8/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/v8/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/v8/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/v8/sandbox-en.html) |
| **52** | `DEVELOPER_ROADMAP` | [CN](./11-software-methodologies/52-developer_roadmap/developer_roadmap-Cheatsheet-v1.md) / [EN](./11-software-methodologies/52-developer_roadmap/developer_roadmap-Cheatsheet-v1-en.md) | [CN](./docs/developer_roadmap/cheatsheet.pdf) / [EN](./docs/developer_roadmap/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/developer_roadmap/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/developer_roadmap/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/developer_roadmap/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/developer_roadmap/sandbox-en.html) |
| **54** | `CODING_INTERVIEW_UNIVERSITY` | [CN](./11-software-methodologies/54-coding_interview_university/coding_interview_university-Cheatsheet-v1.md) / [EN](./11-software-methodologies/54-coding_interview_university/coding_interview_university-Cheatsheet-v1-en.md) | [CN](./docs/coding_interview_university/cheatsheet.pdf) / [EN](./docs/coding_interview_university/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/coding_interview_university/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/coding_interview_university/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/coding_interview_university/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/coding_interview_university/sandbox-en.html) |
| **55** | `TOKIO` | [CN](./04-computer-networks/55-tokio/tokio-Cheatsheet-v1.md) / [EN](./04-computer-networks/55-tokio/tokio-Cheatsheet-v1-en.md) | [CN](./docs/tokio/cheatsheet.pdf) / [EN](./docs/tokio/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/tokio/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/tokio/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/tokio/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/tokio/sandbox-en.html) |
| **70** | `RUST_PATTERNS` | [CN](./11-software-methodologies/70-rust_patterns/rust_patterns-Cheatsheet-v1.md) / [EN](./11-software-methodologies/70-rust_patterns/rust_patterns-Cheatsheet-v1-en.md) | [CN](./docs/rust_patterns/cheatsheet.pdf) / [EN](./docs/rust_patterns/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rust_patterns/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rust_patterns/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rust_patterns/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rust_patterns/sandbox-en.html) |
| **73** | `DEVOPS_FIREFIGHTING` | [CN](./05-system-design/73-devops_firefighting/devops_firefighting-Cheatsheet-v1.md) / [EN](./05-system-design/73-devops_firefighting/devops_firefighting-Cheatsheet-v1-en.md) | [CN](./docs/devops_firefighting/cheatsheet.pdf) / [EN](./docs/devops_firefighting/cheatsheet-en.pdf) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/devops_firefighting/) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/devops_firefighting/map-en.html) | [CN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/devops_firefighting/sandbox.html) / [EN](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/devops_firefighting/sandbox-en.html) |

## 💎 Unlock Paid Bundle

If you need the following premium developer assets, you can unlock the full bundle via [our Gumroad store](https://gumroad.com) (supports credit cards, Alipay, and PayPal):
- **[ ] 100% Vector High-Res PDFs**: Watermark-free, print-ready for massive sizes (A4/A3/A2) to hang on your wall.
- **[ ] Offline Interactive Sandbox**: Single-file HTML versions that run offline without internet.
- **[ ] Source Code & Tools (XeLaTeX & Python Compiler)**: Build, modify, and customize your own high-density cheatsheets.

👉 **[Get the 25 Systems Engineering Sheets Bundle Now](https://gumroad.com)**

---

## 🔧 How to Use & Print Guide

To maximize the value of this visual cheatsheet system, we recommend the following use cases:
1. **Wall Poster Decor**: We highly recommend printing the high-res vector PDFs (available in the paid bundle) on **A3 or A2 matte paper** (300+ DPI). Hanging them on your office or study wall serves not only as a quick-access technical map but also as a badge of honor for hardcore geeks.
2. **Interview Speedrun**: Before a system design interview or architectural review, spend 15–30 minutes studying the card's `L2 Physical Topology` and `M1~M6 Cheat Matrix`. It instantly activates low-level memory, enabling you to speak confidently about actual memory layouts, I/O paths, and protocol constraints.
3. **Offline Simulation**: The paid bundle contains offline single-file HTML interactive sandboxes. You can download them to run simulation state machines locally without any network connection (e.g., on flights, in coffee shops, or within secured intranets).

---

## 💬 Frequently Asked Questions

#### Q1: Why is every cheatsheet strictly limited to 2 pages?
**A**: Compression is value. Most engineering documents are hundreds of pages long, filled with fluff and wordy descriptions. We believe that **only highly compressed, first-principles knowledge can form long-term intuitive memory**. Restricting content to 2 pages forces us to filter out the noise and leave only the high-integrity signal.

#### Q2: Is this project completely free?
**A**: This project follows a **Freemium** model:
* **Free Content**: All cheatsheet markdown texts, low-resolution preview PDFs, and online interactive sandbox maps hosted on GitHub Pages are **100% free** to the community.
* **Commercial Bundle**: If you need high-resolution watermark-free vector PDFs, offline HTML sandboxes, and the **XeLaTeX source code + Python compiler scripts** for custom modifications, you can unlock them via [our Gumroad store](https://gumroad.com). Your support keeps us building more high-quality sheets.

#### Q3: Why are the XeLaTeX source code and Python compiler scripts not in the public repository?
**A**: To protect our custom-built XeLaTeX compiler suite and prevent unauthorized resellers from duplicating our source code, the compilation pipeline remains closed-source and is only available in the paid Gumroad Bundle.

#### Q4: Can I contribute or report bugs?
**A**: Absolutely! Since these cheatsheets cover deep kernel internals, assembly instructions, and complex distributed consensus protocols, typos or minor conceptual deviations may occur. If you find any errors, please submit an **Issue** or **Pull Request** to correct the markdown files. We will review and apply the fixes in the next PDF compilation, and your name will be added to our contributors list.

---

## 🧠 Career & Mind Counterpart

In addition to core tech skills, a developer's career progression depends heavily on soft skills, upward management, and cognitive psychology. Check out our soft-skills sister project:
👉 **[Developer Mind OS (Visual Soft-Skills Cheatsheets)](https://github.com/developer-mind-OS/developer-soft-skills-guide)**
*(Includes high-density sheets and sandboxes for 12 career and system-thinking classics like Upward Management, Thinking Fast and Slow, and Antifragile.)*

---

## ⚖️ License

The markdown texts and low-res PDFs in this repository are licensed under the [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/) license.
* You are free to share and adapt the material.
* **Commercial use (reselling or paid group distribution) is strictly prohibited.** For commercial licensing inquiries, please contact the author.
