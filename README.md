# 🚀 Hardcore Systems Cheatsheet (硬核系统工程可视化速查海报)

> **37 个标杆级开源项目与底层巨著的高密可视化卡片系统。一图速通大厂系统架构、操作系统内核与分布式共识算法。**
> 
> **A high-density visual cheatsheet system compressing 37 benchmark open-source codebases and system engineer classics. Speedrun OS kernels, distributed databases, and high-performance networks in 3 hours.**

[![GitHub Stars](https://img.shields.io/github/stars/hardcore-systems/hardcore-systems-cheatsheet?style=for-the-badge&color=blue)](https://github.com/hardcore-systems/hardcore-systems-cheatsheet)
[![Sponsor on GitHub](https://img.shields.io/badge/Sponsor-GitHub-pink?style=for-the-badge&logo=github)](https://github.com/sponsors/bsjzmj)
[![Get Bundle on Gumroad](https://img.shields.io/badge/Gumroad-Get%20Bundle-orange?style=for-the-badge&logo=gumroad)](https://gumroad.com)

---

## 🖼️ Landscape Preview (效果预览)

本项目中的每一份可视化海报都由专业的 **XeLaTeX + PGF/TikZ** 绘图系统配合定制的 **莫兰迪（Morandi）美学配色** 精心渲染而成，旨在通过最直观的物理拓扑图，揭示巨型系统底层的真相。

* 📥 **[RocksDB 视觉卡片预览 (高清矢量 PDF)](https://raw.githubusercontent.com/hardcore-systems/hardcore-systems-cheatsheet/main/docs/rocksdb/cheatsheet.pdf)**
* 💻 **[RocksDB 在线知识图谱](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/)** | **[RocksDB 交互式运行沙盒](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/sandbox.html)**

---

## 💡 Why This? / 核心设计哲学

现代软件工程师、系统设计师和架构师在提升自身技术深度时，常常面临两大窒息痛点：
1. **源码浩如烟海，缺乏“第一性原理”骨架**：像 Linux 内核、Kubernetes、V8 引擎等巨型项目，动辄数百万行代码。在没有全局宏观“物理拓扑大图”的情况下，直接通读源码就像在没有地图的原始森林里散步，极易迷失于细枝末节。
2. **官方文档过于抽象，丢失核心信号**：常规的技术博客和架构图总是习惯于画一些抽象的“方块与箭头”，隐藏了最关键的低级细节（如页缓存 Page Cache、锁竞争 Lock Contention、内存屏障 Memory Barrier、并发冲突控制机制等）。

为了打破这种“知识黑盒”，我们提出了一套 **“SOP 级题材解构法”**。对于每一个复杂的底层技术栈，我们均将其精确提炼并解构成以下物理层级：
* **L0 单句本质 (One-Sentence Essence)**：以最纯粹的第一性原解释该项目在计算机世界里的根本立足点。
* **L1 四行核心逻辑 (4-Line Core Logic)**：用 4 行代码或极致伪代码，提炼系统最核心的工作循环（Work Loop）或状态扭转过程。
* **L2 物理数据流拓扑图 (L2 Architecture & Data Flow)**：拒绝抽象，还原真正的物理内存布局、磁盘文件组织、或网络数据包交互时序图。
* **M1 ~ M6 核心速查卡片 (28 Concept Cards Matrix)**：每张速查卡片均覆盖 28 个核心工业概念（如协议细节、关键算法、性能优化与生产排雷技巧），信息密度极高，绝无凑字数的多余文案。

---

## 🏭 Special Traffic Whitepaper (特别策划引流白皮书)

针对当前热门的 **互联网程序员转型智能制造/工业 IT/工业自动化** 趋势，本项目特别整理并发布了一份深度市场与技术路径分析报告。这份报告已经开源并放在仓库中，它可以帮助您快速梳理并理解制造业 IT 的全貌：
👉 **[制造业 IT 与工业自动化 (PLC/SCADA/MES/WMS) 市场可行性与打法分析报告](./工业IT与自动化市场可行性分析报告.md)**
*(包含：ISA-95 五层工控模型、OPC UA 信息模型、Modbus 规约、PLC 循环扫描与中断、MES 制造执行核心流、WMS 状态机、以及 28 张工业 IT 核心卡片的设计蓝图。)*

---

## 📂 Catalog & Interactive Maps (37个硬核项目速查目录)

| 编号 | 项目名称 | 源码目录 | PDF 预览 (低清) | 在线知识图谱 | 交互仿真沙盒 |
|:---:|:---|:---:|:---:|:---:|:---:|
| **18** | `EBPF-BCC` | [Markdown](./18-实跑-eBPF_bcc/) | [PDF Preview](./docs/eBPF_bcc/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/eBPF_bcc/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/eBPF_bcc/sandbox.html) |
| **19** | `ROCKSDB` | [Markdown](./19-实跑-rocksdb/) | [PDF Preview](./docs/rocksdb/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/rocksdb/sandbox.html) |
| **20** | `ETCD` | [Markdown](./20-实跑-etcd/) | [PDF Preview](./docs/etcd/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/etcd/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/etcd/sandbox.html) |
| **21** | `KAFKA` | [Markdown](./21-实跑-kafka/) | [PDF Preview](./docs/kafka/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/kafka/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/kafka/sandbox.html) |
| **22** | `RED-BOOK` | [Markdown](./22-实跑-red_book/) | [PDF Preview](./docs/red_book/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/red_book/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/red_book/sandbox.html) |
| **23** | `SQLITE` | [Markdown](./23-实跑-sqlite/) | [PDF Preview](./docs/sqlite/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sqlite/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sqlite/sandbox.html) |
| **24** | `REDIS` | [Markdown](./24-实跑-redis/) | [PDF Preview](./docs/redis/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/redis/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/redis/sandbox.html) |
| **25** | `POSTGRES` | [Markdown](./25-实跑-postgres/) | [PDF Preview](./docs/postgres/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/postgres/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/postgres/sandbox.html) |
| **26** | `CLICKHOUSE` | [Markdown](./26-实跑-clickhouse/) | [PDF Preview](./docs/clickhouse/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/clickhouse/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/clickhouse/sandbox.html) |
| **27** | `OSTEP` | [Markdown](./27-实跑-ostep/) | [PDF Preview](./docs/ostep/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/ostep/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/ostep/sandbox.html) |
| **28** | `LINUX-INSIDES` | [Markdown](./28-实跑-linux_insides/) | [PDF Preview](./docs/linux_insides/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/linux_insides/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/linux_insides/sandbox.html) |
| **29** | `XV6-RISCV` | [Markdown](./29-实跑-xv6_riscv/) | [PDF Preview](./docs/xv6_riscv/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/xv6_riscv/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/xv6_riscv/sandbox.html) |
| **30** | `TOP-DOWN-NETWORKING` | [Markdown](./30-实跑-top_down_networking/) | [PDF Preview](./docs/top_down_networking/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/top_down_networking/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/top_down_networking/sandbox.html) |
| **31** | `DPDK` | [Markdown](./31-实跑-dpdk/) | [PDF Preview](./docs/dpdk/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/dpdk/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/dpdk/sandbox.html) |
| **32** | `ENVOY` | [Markdown](./32-实跑-envoy/) | [PDF Preview](./docs/envoy/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/envoy/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/envoy/sandbox.html) |
| **33** | `QUIC-HTTP3` | [Markdown](./33-实跑-quic_http3/) | [PDF Preview](./docs/quic_http3/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/quic_http3/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/quic_http3/sandbox.html) |
| **34** | `SYSTEM-DESIGN-PRIMER` | [Markdown](./34-实跑-system_design_primer/) | [PDF Preview](./docs/system_design_primer/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_design_primer/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_design_primer/sandbox.html) |
| **35** | `AWESOME-SCALABILITY` | [Markdown](./35-实跑-awesome_scalability/) | [PDF Preview](./docs/awesome_scalability/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/awesome_scalability/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/awesome_scalability/sandbox.html) |
| **36** | `SRE-BOOK` | [Markdown](./36-实跑-sre_book/) | [PDF Preview](./docs/sre_book/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sre_book/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sre_book/sandbox.html) |
| **37** | `MIT-6.824` | [Markdown](./37-实跑-mit_6.824/) | [PDF Preview](./docs/mit_6.824/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/mit_6.824/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/mit_6.824/sandbox.html) |
| **38** | `COCKROACHDB` | [Markdown](./38-实跑-cockroachdb/) | [PDF Preview](./docs/cockroachdb/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/cockroachdb/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/cockroachdb/sandbox.html) |
| **39** | `CRAFTINGINTERPRETERS` | [Markdown](./39-实跑-craftinginterpreters/) | [PDF Preview](./docs/craftinginterpreters/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/craftinginterpreters/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/craftinginterpreters/sandbox.html) |
| **40** | `V8` | [Markdown](./40-实跑-v8/) | [PDF Preview](./docs/v8/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/v8/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/v8/sandbox.html) |
| **41** | `OPENJDK` | [Markdown](./41-实跑-openjdk/) | [PDF Preview](./docs/openjdk/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/openjdk/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/openjdk/sandbox.html) |
| **42** | `GOLANG` | [Markdown](./42-实跑-golang/) | [PDF Preview](./docs/golang/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/golang/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/golang/sandbox.html) |
| **43** | `LLVM` | [Markdown](./43-实跑-llvm/) | [PDF Preview](./docs/llvm/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/llvm/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/llvm/sandbox.html) |
| **44** | `SYSTEM-PERFORMANCE` | [Markdown](./44-实跑-system_performance/) | [PDF Preview](./docs/system_performance/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_performance/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/system_performance/sandbox.html) |
| **45** | `SANITIZERS` | [Markdown](./45-实跑-sanitizers/) | [PDF Preview](./docs/sanitizers/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sanitizers/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/sanitizers/sandbox.html) |
| **46** | `RISCV-ISA` | [Markdown](./46-实跑-riscv_isa/) | [PDF Preview](./docs/riscv_isa/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/riscv_isa/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/riscv_isa/sandbox.html) |
| **47** | `CUDA-PROGRAMMING` | [Markdown](./47-实跑-cuda_programming/) | [PDF Preview](./docs/cuda_programming/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/cuda_programming/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/cuda_programming/sandbox.html) |
| **48** | `OPENSSL` | [Markdown](./48-实跑-openssl/) | [PDF Preview](./docs/openssl/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/openssl/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/openssl/sandbox.html) |
| **49** | `INTEL-SGX` | [Markdown](./49-实跑-intel_sgx/) | [PDF Preview](./docs/intel_sgx/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/intel_sgx/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/intel_sgx/sandbox.html) |
| **50** | `AWESOME-ROBOTICS` | [Markdown](./50-实跑-awesome_robotics/) | [PDF Preview](./docs/awesome_robotics/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/awesome_robotics/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/awesome_robotics/sandbox.html) |
| **51** | `POWER-SYSTEMS` | [Markdown](./51-实跑-power_systems/) | [PDF Preview](./docs/power_systems/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/power_systems/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/power_systems/sandbox.html) |
| **52** | `DEVELOPER-ROADMAP` | [Markdown](./52-实跑-developer_roadmap/) | [PDF Preview](./docs/developer_roadmap/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/developer_roadmap/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/developer_roadmap/sandbox.html) |
| **53** | `JAVA-DESIGN-PATTERNS` | [Markdown](./53-实跑-java_design_patterns/) | [PDF Preview](./docs/java_design_patterns/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/java_design_patterns/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/java_design_patterns/sandbox.html) |
| **54** | `CODING-INTERVIEW-UNIVERSITY` | [Markdown](./54-实跑-coding_interview_university/) | [PDF Preview](./docs/coding_interview_university/cheatsheet.pdf) | [Map](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/coding_interview_university/) | [Sandbox](https://hardcore-systems.github.io/hardcore-systems-cheatsheet/coding_interview_university/sandbox.html) |

---

## 💎 Unlock Paid Bundle (解锁商业高级包)

如果您需要以下极客资产，可以通过 [Gumroad 购买链接](https://gumroad.com) 解密完整增值包（支持双币信用卡、支付宝）：
- **[ ] 100% 矢量级高解析度 PDF**：无水印，支持超大画幅打印（A4/A3/A2）贴在办公室墙上作为极客勋章。
- **[ ] 离线单文件 HTML 互动演练沙盒**：断网可用，内置 LocalStorage 自动保存的“心智审计日志”与“算法运行状态机”。
- **[ ] 完整编译脚本源码 (XeLaTeX & Python Compiler)**：自主扩展并定制您自己的系统设计速查卡片。

👉 **[立刻获取 37 本系统工程全量大礼包](https://gumroad.com)**

---

## 🔧 How to Use & Print Guide (打印与使用指南)

为了让这套可视化卡片系统发挥最大的价值，我们建议您按照以下方式进行使用和打印：
1. **墙面极客壁挂 (Poster Decor)**：我们强烈建议将高清晰度矢量 PDF（付费包提供）以 **A3 或 A2 画幅** 打印出来，使用哑光铜版纸（Matte Paper, 300+ DPI），双页拼接挂在书房或办公室的墙上。这不仅是一张随手可查的技术地图，更是一张硬核极客的身份勋章。
2. **面试速通复习 (Interview Speedrun)**：在面试或系统架构设计前，花 15-30 分钟阅读对应卡片的 `L2 物理拓扑图` 与 `M1~M6 速查矩阵`，能够瞬间激活您的底层记忆，在讨论架构方案时轻松聊出真实的内存模型、I/O 通道与协议细节。
3. **本地断网仿真 (Offline Sandbox)**：付费包中提供了单文件 HTML 版本的交互仿真沙盒。您可以将这些网页下载到本地，断网状态下（如飞机上、咖啡馆或强隔离内网环境）依然可以顺畅运行，直观查看算法状态机。

---

## 💬 Frequently Asked Questions (常见问题解答)

#### Q1: 为什么每张速查卡片只限制在双页 (2 Pages)？
**A**: 信息压缩（Compression）本身就是极高的价值。很多技术资料动辄上百页，包含大量冗长的文字和无用的修饰。我们认为：**只有被极度压缩、直击本质的知识，才最容易在大脑中形成长期的直觉性记忆**。我们将内容死死限制在 2 个 Page 里，强迫自己只留下纯粹的信号，过滤掉杂质和噪音。

#### Q2: 本项目是完全免费的吗？
**A**: 本项目采用 **Freemium（免费+付费增值）** 模式：
* **免费内容**：GitHub 仓库中公开的所有速查 Markdown 详细文本、低分辨率预览 PDF、以及托管在 GitHub Pages 上的在线知识图谱和交互式沙盒网页，全部对社区 **100% 免费**。
* **商业付费包**：如果您需要用于打印的高清无水印矢量 PDF（适合 A4/A3/A2 打印）、断网可用的离线版 HTML 互动沙盒、以及用于自定义编译修改的 **XeLaTeX 源码 + Python Compiler 编译器脚本**，可以前往 [Gumroad 商业包链接](https://gumroad.com) 解锁。您的支持将帮助我们持续拆解并产出更多高质量的技术海报。

#### Q3: 为什么我在公开的 Git 仓库里找不到 XeLaTeX 源码和 Python 编译脚本？
**A**: 为了防止有不良商家直接复制我们的源码进行二次售卖，并保护我们自主研发的可视化 XeLaTeX 编译器套件，我们对编译器及源码进行了商业闭源保护。所有生成工具链均只在付费 Gumroad Bundle 中提供。

#### Q4: 是否可以为本项目贡献内容或提交报错？
**A**: 非常欢迎！由于这套速查卡片包含大量深度的操作系统内核、汇编指令、以及复杂的分布式系统协议细节，难免会出现文字拼写或局部概念偏差。如果您在阅读过程中发现任何错误，欢迎直接提交 **Issue** 或 **Pull Request** 修改 Markdown 文本。我们会在第一时间校验并在下一次 PDF 编译中予以修正，您的名字将被记录在贡献者名单中。

---

## ⚖️ License (版权声明)

本项目文本及低清 PDF 遵循 [Creative Commons Attribution-NonCommercial-ShareAlike 4.0 International (CC BY-NC-SA 4.0)](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议。
* 允许自由共享和演绎。
* **严禁用于任何商业目的**（如转售、收费社群分发）。商业授权使用请联系作者。
