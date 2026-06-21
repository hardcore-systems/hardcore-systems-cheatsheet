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
| **18** | `EBPF_BCC` | [Markdown](./03-operating-systems/18-eBPF_bcc/) | [PDF Preview](./docs/eBPF_bcc/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/eBPF_bcc/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/eBPF_bcc/sandbox.html) |
| **19** | `ROCKSDB` | [Markdown](./02-database-internals/19-rocksdb/) | [PDF Preview](./docs/rocksdb/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/sandbox.html) |
| **20** | `ETCD` | [Markdown](./01-distributed-systems/20-etcd/) | [PDF Preview](./docs/etcd/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/etcd/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/etcd/sandbox.html) |
| **29** | `XV6_RISCV` | [Markdown](./03-operating-systems/29-xv6_riscv/) | [PDF Preview](./docs/xv6_riscv/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/xv6_riscv/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/xv6_riscv/sandbox.html) |
| **34** | `SYSTEM_DESIGN_PRIMER` | [Markdown](./05-system-design/34-system_design_primer/) | [PDF Preview](./docs/system_design_primer/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_design_primer/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_design_primer/sandbox.html) |
| **36** | `SRE_BOOK` | [Markdown](./05-system-design/36-sre_book/) | [PDF Preview](./docs/sre_book/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sre_book/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sre_book/sandbox.html) |
| **40** | `V8` | [Markdown](./06-compilers-and-runtimes/40-v8/) | [PDF Preview](./docs/v8/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/v8/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/v8/sandbox.html) |
| **52** | `DEVELOPER_ROADMAP` | [Markdown](./11-software-methodologies/52-developer_roadmap/) | [PDF Preview](./docs/developer_roadmap/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/developer_roadmap/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/developer_roadmap/sandbox.html) |
| **54** | `CODING_INTERVIEW_UNIVERSITY` | [Markdown](./11-software-methodologies/54-coding_interview_university/) | [PDF Preview](./docs/coding_interview_university/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/coding_interview_university/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/coding_interview_university/sandbox.html) |
| **55** | `TOKIO` | [Markdown](./04-computer-networks/55-tokio/) | [PDF Preview](./docs/tokio/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/tokio/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/tokio/sandbox.html) |
| **70** | `RUST_PATTERNS` | [Markdown](./11-software-methodologies/70-rust_patterns/) | [PDF Preview](./docs/rust_patterns/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rust_patterns/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rust_patterns/sandbox.html) |
| **73** | `DEVOPS_FIREFIGHTING` | [Markdown](./05-system-design/73-devops_firefighting/) | [PDF Preview](./docs/devops_firefighting/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/devops_firefighting/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/devops_firefighting/sandbox.html) |

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
