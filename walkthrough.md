# Walkthrough - GitHub 硬核技术系统工程卡片拆解与编译大盘

本文汇总了 **“GitHub 37个全新硬核技术项目/书籍的卡片提取与 LaTeX 编译”** 的所有已交付项目成果。

---

## 📂 成果一：iovisor / bcc (eBPF) 内核级系统诊断

项目已在物理工作区目录中完成两阶段拆解与自动编译，产出文件如下：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-eBPF_bcc.md](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/Step1-判题材-eBPF_bcc.md)：题材分析、莫兰迪配色体系及 28 张核心卡片大纲。
*   [eBPF_bcc-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Linux 内核源码位置锚点。
*   [eBPF_bcc-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-Cheatsheet-v1.md)：高密度速查卡片中文版，含系统思维滤镜与折中矩阵。
*   [eBPF_bcc-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_ebpf.py](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/build_latex_ebpf.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_ebpf.py](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/generate_html_ebpf.py)：生成交互式知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[eBPF_bcc-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[eBPF_bcc-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[eBPF_bcc-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[eBPF_bcc-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-精美知识图谱-v1-en.html)
    *   内核仿真实验室沙盒：
        *   🇨🇳 中文版：[eBPF_bcc-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[eBPF_bcc-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/18-实跑-eBPF_bcc/eBPF_bcc-互动演练沙盒-v1-en.html)

---

## 📂 成果二：facebook / rocksdb 嵌入式 LSM 存储引擎

项目在工作区独立目录 `19-实跑-rocksdb` 下通过一站式流程直接产出并编译成功：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-rocksdb.md](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/Step1-判题材-rocksdb.md)：题材分析、莫兰迪配色体系及 28 张核心卡片大纲。
*   [rocksdb-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 RocksDB 源码位置锚点。
*   [rocksdb-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-Cheatsheet-v1.md)：高密度速查卡片中文版，含引擎思维滤镜与折中矩阵。
*   [rocksdb-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_rocksdb.py](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/build_latex_rocksdb.py)：解析 markdown 并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_rocksdb.py](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/generate_html_rocksdb.py)：生成交互式知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[rocksdb-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[rocksdb-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[rocksdb-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[rocksdb-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-精美知识图谱-v1-en.html)
    *   LSM 归并压实模拟沙盒：
        *   🇨🇳 中文版：[rocksdb-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[rocksdb-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/19-实跑-rocksdb/rocksdb-互动演练沙盒-v1-en.html)

---

## 📂 成果三：etcd-io / etcd 强一致性协同与数据存储

项目已在工作区独立目录 `20-实跑-etcd` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-etcd.md](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/Step1-判题材-etcd.md)：题材分析、强一致存储逻辑与 28 张核心卡片大纲规划。
*   [etcd-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 etcd 源码物理锚点映射。
*   [etcd-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-Cheatsheet-v1.md)：高密度速查卡片中文版，含强一致系统思维滤镜、架构折衷矩阵与调试字典。
*   [etcd-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_etcd.py](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/build_latex_etcd.py)：解析 markdown 并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_etcd.py](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/generate_html_etcd.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[etcd-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[etcd-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[etcd-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[etcd-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-精美知识图谱-v1-en.html)
    *   分布式仿真实验室沙盒：
        *   🇨🇳 中文版：[etcd-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[etcd-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/20-实跑-etcd/etcd-互动演练沙盒-v1-en.html)

---

## 📂 成果四：apache / kafka-internals 高吞吐日志存储与零拷贝 I/O

项目已在工作区独立目录 `21-实跑-kafka` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-kafka.md](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/Step1-判题材-kafka.md)：题材分析、日志稀疏索引设计与 28 张核心卡片大纲规划。
*   [kafka-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Kafka 源码物理路径映射。
*   [kafka-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-Cheatsheet-v1.md)：高密度速查卡片中文版，含高性能系统思维制约、架构折衷矩阵与调试参数表。
*   [kafka-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_kafka.py](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/build_latex_kafka.py)：解析 markdown 并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_kafka.py](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/generate_html_kafka.py)：生成交互式中英文知识图谱和沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[kafka-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[kafka-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[kafka-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[kafka-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-精美知识图谱-v1-en.html)
    *   分布式仿真实验室沙盒：
        *   🇨🇳 中文版：[kafka-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[kafka-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/21-实跑-kafka/kafka-互动演练沙盒-v1-en.html)

---

## 📂 成果五：josephmhellerstein / db-book (Red Book) 关系型数据库架构设计大图

项目已在工作区独立目录 `22-实跑-red_book` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-red_book.md](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/Step1-判题材-red_book.md)：题材分析、数据库核心认识论及 28 张核心卡片大纲规划。
*   [red_book-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 PostgreSQL/MySQL 物理源码路径映射锚点。
*   [red_book-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-Cheatsheet-v1.md)：高密度速查卡片中文版，含系统思维滤镜、并发与恢复折衷矩阵及运维视图字典。
*   [red_book-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_red_book.py](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/build_latex_red_book.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_red_book.py](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/generate_html_red_book.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[red_book-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[red_book-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[red_book-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[red_book-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-精美知识图谱-v1-en.html)
    *   内核仿真实验室沙盒：
        *   🇨🇳 中文版：[red_book-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[red_book-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/22-实跑-red_book/red_book-互动演练沙盒-v1-en.html)

---

## 📂 成果六：sqlite / sqlite-internals 嵌入式关系存储引擎内核实现

项目已在工作区独立目录 `23-实跑-sqlite` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-sqlite.md](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/Step1-判题材-sqlite.md)：题材分析、嵌入式存储认识论及 28 张核心卡片大纲规划。
*   [sqlite-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑图（Mermaid 格式）及 SQLite 物理源码路径映射锚点。
*   [sqlite-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-Cheatsheet-v1.md)：高密度速查卡片中文版，含物理认识论、并发与日志折衷矩阵及调优视图字典。
*   [sqlite-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_sqlite.py](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/build_latex_sqlite.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_sqlite.py](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/generate_html_sqlite.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[sqlite-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[sqlite-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[sqlite-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[sqlite-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-精美知识图谱-v1-en.html)
    *   虚拟机与日志仿真实验室沙盒：
        *   🇨🇳 中文版：[sqlite-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[sqlite-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/23-实跑-sqlite/sqlite-互动演练沙盒-v1-en.html)

---

## 📂 成果七：redis / redis-internals 内存键值存储引擎

项目已在工作区独立目录 `24-实跑-redis` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-redis.md](file:///D:/材料/硬核技术系统工程/24-实跑-redis/Step1-判题材-redis.md)：题材分析、内存引擎认识论及 28 张核心卡片大纲规划。
*   [redis-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑图及 Redis 物理源码路径映射锚点。
*   [redis-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-Cheatsheet-v1.md)：高密度速查卡片中文版，含物理认识论、并发与日志折衷矩阵及调优视图字典。
*   [redis-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_redis.py](file:///D:/材料/硬核技术系统工程/24-实跑-redis/build_latex_redis.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_redis.py](file:///D:/材料/硬核技术系统工程/24-实跑-redis/generate_html_redis.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[redis-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[redis-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[redis-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[redis-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-精美知识图谱-v1-en.html)
    *   渐进式 Rehash 与 LRU 淘汰实验室沙盒：
        *   🇨🇳 中文版：[redis-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[redis-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/24-实跑-redis/redis-互动演练沙盒-v1-en.html)

---

## 📂 成果八：postgres / postgres 关系型数据库内核

项目已在工作区独立目录 `25-实跑-postgres` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-postgres.md](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/Step1-判题材-postgres.md)：题材分析、数据库核心认识论及 28 张核心卡片大纲规划。
*   [postgres-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑图及 PostgreSQL 物理源码路径映射锚点。
*   [postgres-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-Cheatsheet-v1.md)：高密度速查卡片中文版，含物理认识论、并发与恢复折衷矩阵及调优视图字典。
*   [postgres-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_postgres.py](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/build_latex_postgres.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_postgres.py](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/generate_html_postgres.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[postgres-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[postgres-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[postgres-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[postgres-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-精美知识图谱-v1-en.html)
    *   Heap MVCC 页面与 SSI 依赖环仿真实验室沙盒：
        *   🇨🇳 中文版：[postgres-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[postgres-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/25-实跑-postgres/postgres-互动演练沙盒-v1-en.html)

---

## 📂 成果九：ClickHouse / ClickHouse 列式分析型存储引擎

项目已在工作区独立目录 `26-实跑-clickhouse` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-clickhouse.md](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/Step1-判题材-clickhouse.md)：题材分析、列式引擎核心逻辑及 28 张核心卡片大纲规划。
*   [clickhouse-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑图及 ClickHouse 物理源码路径映射锚点。
*   [clickhouse-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-Cheatsheet-v1.md)：高密度速查卡片中文版，含引擎思维滤镜、架构折衷矩阵及调优视图字典。
*   [clickhouse-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_clickhouse.py](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/build_latex_clickhouse.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_clickhouse.py](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/generate_html_clickhouse.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[clickhouse-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[clickhouse-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[clickhouse-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[clickhouse-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-精美知识图谱-v1-en.html)
    *   MergeTree 稀疏索引与 CPU SIMD 向量化对比仿真实验室沙盒：
        *   🇨🇳 中文版：[clickhouse-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[clickhouse-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/26-实跑-clickhouse/clickhouse-互动演练沙盒-v1-en.html)

---

## 📂 成果十：OSTEP / Operating Systems: Three Easy Pieces 现代操作系统经典

项目已在工作区独立目录 `27-实跑-ostep` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-ostep.md](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/Step1-判题材-ostep.md)：题材分析、操作系统虚拟化逻辑及 28 张核心卡片大纲规划。
*   [ostep-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑图及 OSTEP 经典理论架构与内核参数映射锚点。
*   [ostep-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-Cheatsheet-v1.md)：高密度速查卡片中文版，含特权级强隔离、自旋锁与巨页折衷矩阵及调优视图字典。
*   [ostep-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_ostep.py](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/build_latex_ostep.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_ostep.py](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/generate_html_ostep.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[ostep-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[ostep-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[ostep-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[ostep-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-精美知识图谱-v1-en.html)
    *   多级页表物理翻译与 MLFQ 进程调度仿真实验室沙盒：
        *   🇨🇳 中文版：[ostep-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[ostep-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/27-实跑-ostep/ostep-互动演练沙盒-v1-en.html)

---

---

## 📂 成果十一：torvalds / linux-insides (Linux Kernel Internals) Linux 内核底层实现细节

项目已在工作区独立目录 `28-实跑-linux_insides` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-linux_insides.md](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/Step1-判题材-linux_insides.md)：题材分析、Linux 内核引导启动逻辑与 28 张核心卡片大纲规划。
*   [linux_insides-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Linux 内存管理与调度器物理源码映射锚点。
*   [linux_insides-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-Cheatsheet-v1.md)：高密度速查卡片中文版，含系统级认识论、性能折衷防范矩阵及调优视图字典。
*   [linux_insides-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_linux_insides.py](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/build_latex_linux_insides.py)：解析 markdown 并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_linux_insides.py](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/generate_html_linux_insides.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[linux_insides-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[linux_insides-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[linux_insides-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[linux_insides-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-精美知识图谱-v1-en.html)
    *   Buddy 物理分配与 SLUB 对象高速缓存集成仿真实验室沙盒：
        *   🇨🇳 中文版：[linux_insides-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[linux_insides-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/28-实跑-linux_insides/linux_insides-互动演练沙盒-v1-en.html)

---

---

## 📂 成果十二：mit-pdos / xv6-riscv (XV6 RISC-V Operating System) MIT 教学级 Unix 操作系统

项目已在工作区独立目录 `29-实跑-xv6_riscv` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-xv6_riscv.md](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/Step1-判题材-xv6_riscv.md)：题材分析、RISC-V 陷入机制与 28 张核心卡片大纲规划。
*   [xv6_riscv-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 xv6 内存管理与调度器物理源码映射锚点。
*   [xv6_riscv-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-Cheatsheet-v1.md)：高密度速查卡片中文版，含系统级认识论、性能折衷防范矩阵及架构参数字典。
*   [xv6_riscv-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_xv6_riscv.py](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/build_latex_xv6_riscv.py)：解析 markdown 并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_xv6_riscv.py](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/generate_html_xv6_riscv.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[xv6_riscv-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[xv6_riscv-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[xv6_riscv-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[xv6_riscv-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-精美知识图谱-v1-en.html)
    *   SV39 地址翻译与 Pipe 读写同步仿真实验室沙盒：
        *   🇨🇳 中文版：[xv6_riscv-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[xv6_riscv-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/29-实跑-xv6_riscv/xv6_riscv-互动演练沙盒-v1-en.html)

---

## 📂 成果十三：kurose-ross / top-down-networking (Computer Networking: A Top-Down Approach) 计算机网络：自顶向下方法

项目已在工作区独立目录 `30-实跑-top_down_networking` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-top_down_networking.md](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/Step1-判题材-top_down_networking.md)：题材分析、协议栈与多路同步机理与 28 张核心卡片大纲规划。
*   [top_down_networking-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Linux 协议栈/Socket 物理源码映射锚点。
*   [top_down_networking-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-Cheatsheet-v1.md)：高密度速查卡片中文版，含系统级认识论、性能折衷防范矩阵及底层调试字典。
*   [top_down_networking-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_top_down_networking.py](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/build_latex_top_down_networking.py)：解析 markdown 并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_top_down_networking.py](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/generate_html_top_down_networking.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[top_down_networking-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[top_down_networking-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[top_down_networking-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[top_down_networking-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-精美知识图谱-v1-en.html)
    *   双端滑动窗口丢包与 Cubic/BBR 拥塞控制仿真实验室沙盒：
        *   🇨🇳 中文版：[top_down_networking-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[top_down_networking-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/30-实跑-top_down_networking/top_down_networking-互动演练沙盒-v1-en.html)

---

## 📂 成果十四：DPDK / dpdk (数据平面开发套件)

项目已在工作区独立目录 `31-实跑-dpdk` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-dpdk.md](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/Step1-判题材-dpdk.md)：题材分析、内核旁路与 PMD 轮询大纲规划。
*   [dpdk-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 DPDK 内部 API 源码路径映射锚点。
*   [dpdk-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-Cheatsheet-v1.md)：高密度速查卡片中文版，含内存与同步折衷矩阵及 Zone T 内核参数隔离对照表。
*   [dpdk-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_dpdk.py](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/build_latex_dpdk.py)：解析 markdown 并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_dpdk.py](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/generate_html_dpdk.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[dpdk-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[dpdk-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[dpdk-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[dpdk-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-精美知识图谱-v1-en.html)
    *   无锁环形队列 CAS 指针与 PMD 轮询仿真实验室沙盒：
        *   🇨🇳 中文版：[dpdk-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[dpdk-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/31-实跑-dpdk/dpdk-互动演练沙盒-v1-en.html)

---

## 📂 成果十五：envoyproxy / envoy (Envoy Proxy) 云原生高性能边缘代理

项目已在工作区独立目录 `32-实跑-envoy` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-envoy.md](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/Step1-判题材-envoy.md)：题材分析、多线程异步事件循环与 28 张核心卡片大纲规划。
*   [envoy-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Envoy 物理源码路径映射锚点。
*   [envoy-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-Cheatsheet-v1.md)：高密度速查卡片中文版，含线程模型与并发控制折衷矩阵及 Admin 接口工具字典。
*   [envoy-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_envoy.py](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/build_latex_envoy.py)：解析 markdown 并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_envoy.py](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/generate_html_envoy.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[envoy-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[envoy-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[envoy-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[envoy-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-精美知识图谱-v1-en.html)
    *   三级过滤器链与 xDS 动态热重启仿真实验室沙盒：
        *   🇨🇳 中文版：[envoy-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[envoy-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/32-实跑-envoy/envoy-互动演练沙盒-v1-en.html)

---

---

## 📂 成果十六：h2o / quic-http3 (H2O HTTP Server) 高性能 HTTP/3 & QUIC 协议栈

项目已在工作区独立目录 `33-实跑-quic_http3` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-quic_http3.md](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/Step1-判题材-quic_http3.md)：题材分析、自研时间轮与 28 张核心卡片大纲规划。
*   [quic_http3-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 H2O、quicly、picotls 底层物理源码路径映射锚点。
*   [quic_http3-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-Cheatsheet-v1.md)：高密度速查卡片中文版，含 `h2o_evloop_t` 结构体、`quicly_conn_t` / `quicly_stream_t` 状态、QPACK 编解码表项、HTTP/2 优先调度树与系统诊断参数字典。
*   [quic_http3-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_quic_http3.py](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/build_latex_quic_http3.py)：解析 markdown 并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_quic_http3.py](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/generate_html_quic_http3.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[quic_http3-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[quic_http3-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[quic_http3-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[quic_http3-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-精美知识图谱-v1-en.html)
    *   连接迁移与 WDRR 优先级调度仿真实验室沙盒：
        *   🇨🇳 中文版：[quic_http3-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[quic_http3-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/33-实跑-quic_http3/quic_http3-互动演练沙盒-v1-en.html)



## 📂 成果十七：donnemartin / system-design-primer (系统设计与高扩展架构大图)

项目已在工作区独立目录 `34-实跑-system_design_primer` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-system_design_primer.md](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/Step1-判题材-system_design_primer.md)：题材分析、莫兰迪配色体系及 28 张核心卡片大纲。
*   [system_design_primer-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码路径映射锚点。
*   [system_design_primer-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-Cheatsheet-v1.md)：高密度速查卡片中文版，含一致性哈希、缓存读写更新策略、水平分片 Shard Key、CAP/PACELC 折衷矩阵与调优配置字典。
*   [system_design_primer-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_system_design_primer.py](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/build_latex_system_design_primer.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_system_design_primer.py](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/generate_html_system_design_primer.py)：生成交互式中英文知识图谱和内核沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[system_design_primer-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[system_design_primer-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[system_design_primer-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[system_design_primer-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-精美知识图谱-v1-en.html)
    *   一致性哈希环与缓存读写同步策略仿真实验室沙盒：
        *   🇨🇳 中文版：[system_design_primer-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[system_design_primer-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/34-实跑-system_design_primer/system_design_primer-互动演练沙盒-v1-en.html)



## 📂 成果十八：checkcheckzz / awesome-scalability (大厂高并发与高扩展模式系统架构)

项目已在工作区独立目录 `35-实跑-awesome_scalability` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-awesome_scalability.md](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/Step1-判题材-awesome_scalability.md)：题材分析、莫兰迪配色体系及 28 张核心卡片大纲。
*   [awesome_scalability-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理核心架构原理映射锚点。
*   [awesome_scalability-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-Cheatsheet-v1.md)：高密度速查卡片中文版，含系统水平扩展设计、分库分表路由、CQRS 数据模型、回压与隔离参数。
*   [awesome_scalability-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_awesome_scalability.py](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/build_latex_awesome_scalability.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_awesome_scalability.py](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/generate_html_awesome_scalability.py)：生成交互式中英文知识图谱和双沙盒仿真 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[awesome_scalability-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[awesome_scalability-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[awesome_scalability-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[awesome_scalability-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-精美知识图谱-v1-en.html)
    *   CQRS 事件溯源与微服务线程舱壁隔离仿真实验室沙盒：
        *   🇨🇳 中文版：[awesome_scalability-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[awesome_scalability-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/35-实跑-awesome_scalability/awesome_scalability-互动演练沙盒-v1-en.html)


## 📂 成果十九：google / sre-book 谷歌运维与架构稳定性治理

项目已在工作区独立目录 `36-实跑-sre_book` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-sre_book.md](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/Step1-判题材-sre_book.md)：题材分析、莫兰迪配色体系及 28 张核心卡片大纲规划。
*   [sre_book-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及核心架构算法公式映射。
*   [sre_book-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-Cheatsheet-v1.md)：高密度速查卡片中文版，含可用性折衷矩阵与诊断指令字典。
*   [sre_book-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_sre_book.py](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/build_latex_sre_book.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为双页 Landscape A4 PDF。
*   [generate_html_sre_book.py](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/generate_html_sre_book.py)：生成交互式中英文知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[sre_book-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[sre_book-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[sre_book-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[sre_book-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-精美知识图谱-v1-en.html)
    *   多窗口燃烧率告警与客户端自适应限流仿真实验室沙盒：
        *   🇨🇳 中文版：[sre_book-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[sre_book-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/36-实跑-sre_book/sre_book-互动演练沙盒-v1-en.html)


## 📂 成果二十：mit-pdos / distributed-systems (MIT 6.824 分布式系统工程)

项目已在工作区独立目录 `37-实跑-mit_6.824` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-mit_6.824.md](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/Step1-判题材-mit_6.824.md)：题材分析、莫兰迪配色体系及 28 张核心卡片大纲规划。
*   [mit_6.824-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及核心架构与共识约束算法映射锚点。
*   [mit_6.824-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-Cheatsheet-v1.md)：高密度速查卡片中文版，含一致性哈希分片均衡、CAP/PACELC 折衷矩阵、Raft 领导者选举与日志追平、ZooKeeper 临时顺序节点分布式锁。
*   [mit_6.824-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_mit_6.824.py](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/build_latex_mit_6.824.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_mit_6.824.py](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/generate_html_mit_6.824.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[mit_6.824-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[mit_6.824-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[mit_6.824-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[mit_6.824-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-精美知识图谱-v1-en.html)
    *   Raft 强一致性与 MapReduce 并行调度双仿真实验室沙盒：
        *   🇨🇳 中文版：[mit_6.824-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[mit_6.824-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/37-实跑-mit_6.824/mit_6.824-互动演练沙盒-v1-en.html)

---



## 📂 成果二十一：spanner / cockroachdb (Distributed NewSQL Database)

项目已在工作区独立目录 `38-实跑-cockroachdb` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-cockroachdb.md](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/Step1-判题材-cockroachdb.md)：题材分析、分布式一致性引擎与 28 张核心卡片大纲规划。
*   [cockroachdb-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 CockroachDB 物理源码路径映射锚点。
*   [cockroachdb-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-Cheatsheet-v1.md)：高密度速查卡片中文版，含分布式 2PC、MVCC 事务流程、Raft 多租户复制、时钟同步与运维调试字典。
*   [cockroachdb-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_cockroachdb.py](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/build_latex_cockroachdb.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_cockroachdb.py](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/generate_html_cockroachdb.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[cockroachdb-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[cockroachdb-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[cockroachdb-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[cockroachdb-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-精美知识图谱-v1-en.html)
    *   分布式 2PC 与 HLC 时钟漂移仿真沙盒：
        *   🇨🇳 中文版：[cockroachdb-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[cockroachdb-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/38-实跑-cockroachdb/cockroachdb-互动演练沙盒-v1-en.html)

---

## 📂 成果二十二：munificent / craftinginterpreters (手写编译器与虚拟机)

项目已在工作区独立目录 `39-实跑-craftinginterpreters` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-craftinginterpreters.md](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/Step1-判题材-craftinginterpreters.md)：题材分析、词法/语法/虚拟机运行结构与 28 张核心卡片大纲规划。
*   [craftinginterpreters-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Lox 解释器与虚拟机物理源码路径映射锚点。
*   [craftinginterpreters-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-Cheatsheet-v1.md)：高密度速查卡片中文版，含 Pratt 运算符优先级表、VM 字节码指令集、三色标记垃圾回收与调试运行时字典。
*   [craftinginterpreters-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_craftinginterpreters.py](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/build_latex_craftinginterpreters.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_craftinginterpreters.py](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/generate_html_craftinginterpreters.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[craftinginterpreters-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[craftinginterpreters-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[craftinginterpreters-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[craftinginterpreters-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-精美知识图谱-v1-en.html)
    *   Pratt 解析器与三色标记垃圾回收双仿真沙盒：
        *   🇨🇳 中文版：[craftinginterpreters-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[craftinginterpreters-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/39-实跑-craftinginterpreters/craftinginterpreters-互动演练沙盒-v1-en.html)

---


## 📂 成果二十三：v8 / v8 (谷歌高性能 JavaScript 引擎)

项目已在工作区独立目录 `40-实跑-v8` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-v8.md](file:///D:/材料/硬核技术系统工程/40-实跑-v8/Step1-判题材-v8.md)：题材分析、隐藏类寻址设计、内联缓存机制与 28 张核心卡片大纲规划。
*   [v8-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 V8 引擎内部核心 C++ 源码路径映射锚点。
*   [v8-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-Cheatsheet-v1.md)：高密度速查卡片中文版，含 V8 编译器与虚拟机执行设计折衷矩阵、常用字节码指令、Deopt 触发原因与 ElementsKinds 字典。
*   [v8-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_v8.py](file:///D:/材料/硬核技术系统工程/40-实跑-v8/build_latex_v8.py)：解析 markdown 自动排版并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_v8.py](file:///D:/材料/硬核技术系统工程/40-实跑-v8/generate_html_v8.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[v8-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[v8-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[v8-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[v8-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-精美知识图谱-v1-en.html)
    *   对象隐藏类与 JIT 编译/去优化双仿真沙盒：
        *   🇨🇳 中文版：[v8-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[v8-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/40-实跑-v8/v8-互动演练沙盒-v1-en.html)

---

## 📂 成果二十四：openjdk / jdk (Java HotSpot VM 核心原理)

项目已在工作区独立目录 `41-实跑-openjdk` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-openjdk.md](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/Step1-判题材-openjdk.md)：题材分析、六大核心模块、Happens-before 规范、ZGC 读屏障与 28 张核心卡片大纲规划。
*   [openjdk-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 HotSpot VM 内部类加载、锁升级、ZGC 核心 C++ 源码路径映射锚点。
*   [openjdk-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-Cheatsheet-v1.md)：高密度速查卡片中文版，含锁升级与编译设计折衷矩阵、四类逻辑屏障、Loom 协程状态机及 Zone T JVM 命令行参数与底层汇编字典。
*   [openjdk-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_openjdk.py](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/build_latex_openjdk.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_openjdk.py](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/generate_html_openjdk.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[openjdk-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[openjdk-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[openjdk-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[openjdk-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-精美知识图谱-v1-en.html)
    *   JMM 屏障与 ZGC 彩色指针读屏障自愈双仿真沙盒：
        *   🇨🇳 中文版：[openjdk-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[openjdk-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/41-实跑-openjdk/openjdk-互动演练沙盒-v1-en.html)

---

## 📂 成果二十五：golang / go-runtime (Go 语言运行时)

项目已在工作区独立目录 `42-实跑-golang` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-golang.md](file:///D:/材料/硬核技术系统工程/42-实跑-golang/Step1-判题材-golang.md)：题材分析、莫兰迪配色体系、三层分配架构与 28 张核心卡片大纲规划。
*   [golang-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Go Runtime 内部协程调度、垃圾回收核心 Go 源码路径映射锚点。
*   [golang-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-Cheatsheet-v1.md)：高密度速查卡片中文版，含协程与内存折衷矩阵、GODEBUG 调试标志及 Plan 9 汇编写屏障特征字典。
*   [golang-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_golang.py](file:///D:/材料/硬核技术系统工程/42-实跑-golang/build_latex_golang.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_golang.py](file:///D:/材料/硬核技术系统工程/42-实跑-golang/generate_html_golang.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[golang-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[golang-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[golang-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[golang-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-精美知识图谱-v1-en.html)
    *   GMP 调度与三色标记 GC 写屏障双仿真沙盒：
        *   🇨🇳 中文版：[golang-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[golang-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/42-实跑-golang/golang-互动演练沙盒-v1-en.html)

---

## 📂 成果二十六：llvm / llvm-project (LLVM 编译器基础设施)

项目已在工作区独立目录 `43-实跑-llvm` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-llvm.md](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/Step1-判题材-llvm.md)：题材分析、莫兰迪配色体系、L0-L2 梯级架构与 28 张核心卡片大纲规划。
*   [llvm-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 LLVM 内部 IR 语法、中端优化、后端 CodeGen 与寄存器分配 C++ 源码路径映射锚点。
*   [llvm-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-Cheatsheet-v1.md)：高密度速查卡片中文版，含编译器优化与代码生成折衷矩阵、新 PassManager 控制及 IR 汇编特征速查字典。
*   [llvm-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_llvm.py](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/build_latex_llvm.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_llvm.py](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/generate_html_llvm.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[llvm-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[llvm-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[llvm-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[llvm-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-精美知识图谱-v1-en.html)
    *   中端 Pass 优化与后端寄存器分配双仿真沙盒：
        *   🇨🇳 中文版：[llvm-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[llvm-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/43-实跑-llvm/llvm-互动演练沙盒-v1-en.html)

---

## 📂 成果二十七：brendangregg / system-performance (系统性能调优与诊断)

项目已在工作区独立目录 `44-实跑-system_performance` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-system_performance.md](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/Step1-判题材-system_performance.md)：题材分析、莫兰迪配色体系、L0-L2 梯级架构与 28 张核心卡片大纲规划。
*   [system_performance-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Linux 核心 `/proc`、CPU 调度、缺页异常、I/O 调度器、Socket 缓冲区与 eBPF/bpftrace 源码路径映射锚点。
*   [system_performance-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-Cheatsheet-v1.md)：高密度速查卡片中文版，含系统分析方法观测折衷矩阵、诊断命令行与 eBPF/bpftrace 一行脚本速查字典。
*   [system_performance-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_system_performance.py](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/build_latex_system_performance.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_system_performance.py](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/generate_html_system_performance.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[system_performance-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[system_performance-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[system_performance-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[system_performance-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-精美知识图谱-v1-en.html)
    *   CPU 火焰图与内存缺页异常 + 磁盘 I/O 延迟与 bpftrace 双仿真沙盒：
        *   🇨🇳 中文版：[system_performance-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[system_performance-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/44-实跑-system_performance/system_performance-互动演练沙盒-v1-en.html)
---
## 📂 成果二十八：google / sanitizers (运行时内存与并发检测套件)

项目已在工作区独立目录 `45-实跑-sanitizers` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-sanitizers.md](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/Step1-判题材-sanitizers.md)：题材分析、莫兰迪配色体系、L0-L2 梯级架构与 28 张核心卡片大纲规划。
*   [sanitizers-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 LLVM 编译器 Pass 与 compiler-rt 运行时物理源码路径映射锚点。
*   [sanitizers-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-Cheatsheet-v1.md)：高密度速查卡片中文版，含影子内存映射、向量时钟、折衷矩阵与运行时 Suppression 过滤词典。
*   [sanitizers-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_sanitizers.py](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/build_latex_sanitizers.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_sanitizers.py](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/generate_html_sanitizers.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[sanitizers-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[sanitizers-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[sanitizers-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[sanitizers-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-精美知识图谱-v1-en.html)
    *   ASan 影子内存映射与 TSan Happens-Before 竞争双仿真沙盒：
        *   🇨🇳 中文版：[sanitizers-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[sanitizers-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/45-实跑-sanitizers/sanitizers-互动演练沙盒-v1-en.html)

---

## 📂 成果二十九：riscv / riscv-isa-manual (RISC-V 指令集体系结构规范)

项目已在工作区独立目录 `46-实跑-riscv_isa` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-riscv_isa.md](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/Step1-判题材-riscv_isa.md)：题材分析、莫兰迪配色体系、L0-L2 梯级架构与 28 张核心卡片大纲规划。
*   [riscv_isa-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Rocket Chip (in-order)、BOOM (out-of-order) 与 Spike (ISA 模拟器) 物理源码位置映射锚点。
*   [riscv_isa-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-Cheatsheet-v1.md)：高密度速查卡片中文版，含系统思维过滤镜、流水线架构与性能参数折衷矩阵、特权 CSR 寄存器及处理器异常代码字典。
*   [riscv_isa-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_riscv_isa.py](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/build_latex_riscv_isa.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_riscv_isa.py](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/generate_html_riscv_isa.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[riscv_isa-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[riscv_isa-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[riscv_isa-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[riscv_isa-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-精美知识图谱-v1-en.html)
    *   经典五级流水线旁路分支与 MESI Cache 一致性双仿真沙盒：
        *   🇨🇳 中文版：[riscv_isa-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[riscv_isa-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/46-实跑-riscv_isa/riscv_isa-互动演练沙盒-v1-en.html)

---


## 📂 成果三十：nvidia / cuda-samples (CUDA 并行计算与 GPGPU 编程)

项目已在工作区独立目录 `47-实跑-cuda_programming` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-cuda_programming.md](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/Step1-判题材-cuda_programming.md)：题材分析、莫兰迪配色体系、L0-L2 梯级架构与 28 张核心卡片大纲规划。
*   [cuda_programming-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 CUDA Samples 与 CUTLASS 物理源码位置映射锚点。
*   [cuda_programming-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-Cheatsheet-v1.md)：高密度速查卡片中文版，含并行计算与访存折衷矩阵、设备端语法速查与设备故障诊断字典。
*   [cuda_programming-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_cuda_programming.py](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/build_latex_cuda_programming.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_cuda_programming.py](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/generate_html_cuda_programming.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[cuda_programming-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[cuda_programming-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[cuda_programming-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[cuda_programming-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-精美知识图谱-v1-en.html)
    *   共享内存 Bank 冲突与多流异步重叠双仿真沙盒：
        *   🇨🇳 中文版：[cuda_programming-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[cuda_programming-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/47-实跑-cuda_programming/cuda_programming-互动演练沙盒-v1-en.html)



## 📂 成果三十一：openssl / openssl (密码学安全协议与安全通信基础设施)

项目已在工作区独立目录 `48-实跑-openssl` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-openssl.md](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/Step1-判题材-openssl.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [openssl-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 OpenSSL 源码物理路径映射。
*   [openssl-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-Cheatsheet-v1.md)：高密度速查卡片中文版，含对称/非对称/TLS 握手/侧信道/硬件加速折衷矩阵。
*   [openssl-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_openssl.py](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/build_latex_openssl.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_openssl.py](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/generate_html_openssl.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[openssl-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[openssl-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[openssl-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[openssl-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-精美知识图谱-v1-en.html)
    *   TLS 握手状态机与 AES-GCM 认证加密双仿真沙盒：
        *   🇨🇳 中文版：[openssl-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[openssl-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/48-实跑-openssl/openssl-互动演练沙盒-v1-en.html)



## 📂 成果三十二：intel / sgx-software-enable (Intel SGX 硬件飞地安全计算)

项目已在工作区独立目录 `49-实跑-intel_sgx` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-intel_sgx.md](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/Step1-判题材-intel_sgx.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [intel_sgx-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及 Intel SGX 源码物理路径映射。
*   [intel_sgx-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-Cheatsheet-v1.md)：高密度速查卡片中文版，含隔离/侧信道/本地远程认证折衷矩阵。
*   [intel_sgx-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_intel_sgx.py](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/build_latex_intel_sgx.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_intel_sgx.py](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/generate_html_intel_sgx.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[intel_sgx-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[intel_sgx-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[intel_sgx-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[intel_sgx-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-精美知识图谱-v1-en.html)
    *   飞地物理访存与本地远程认证双仿真沙盒：
        *   🇨🇳 中文版：[intel_sgx-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[intel_sgx-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/49-实跑-intel_sgx/intel_sgx-互动演练沙盒-v1-en.html)



## 📂 成果三十三：awesome-robotics (机器人系统、运动规划与 SLAM 算法)

项目已在工作区独立目录 `50-实跑-awesome_robotics` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-awesome_robotics.md](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/Step1-判题材-awesome_robotics.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [awesome_robotics-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射锚点。
*   [awesome_robotics-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-Cheatsheet-v1.md)：高密度速查卡片中文版，含运动学/定位/轨迹优化/阻抗控制折衷矩阵。
*   [awesome_robotics-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_awesome_robotics.py](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/build_latex_awesome_robotics.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_awesome_robotics.py](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/generate_html_awesome_robotics.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[awesome_robotics-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[awesome_robotics-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[awesome_robotics-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[awesome_robotics-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-精美知识图谱-v1-en.html)
    *   机器人正逆运动学与路径规划双仿真沙盒：
        *   🇨🇳 中文版：[awesome_robotics-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[awesome_robotics-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/50-实跑-awesome_robotics/awesome_robotics-互动演练沙盒-v1-en.html)



## 📂 成果三十四：PowerSystems.jl (现代电力系统仿真与网格建模)

项目已在工作区独立目录 `51-实跑-power_systems` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-power_systems.md](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/Step1-判题材-power_systems.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [power_systems-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射锚点。
*   [power_systems-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-Cheatsheet-v1.md)：高密度速查卡片中文版，含稳态潮流/暂态仿真/微网下垂控制折衷矩阵。
*   [power_systems-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_power_systems.py](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/build_latex_power_systems.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_power_systems.py](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/generate_html_power_systems.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[power_systems-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[power_systems-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[power_systems-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[power_systems-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-精美知识图谱-v1-en.html)
    *   电力网格潮流计算与动力学转子摆动双仿真沙盒：
        *   🇨🇳 中文版：[power_systems-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[power_systems-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/51-实跑-power_systems/power_systems-互动演练沙盒-v1-en.html)



## 📂 成果三十五：developer-roadmap (开发者技术栈、全栈系统与路径拓扑)

项目已在工作区独立目录 `52-实跑-developer_roadmap` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-developer_roadmap.md](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/Step1-判题材-developer_roadmap.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [developer_roadmap-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [developer_roadmap-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-Cheatsheet-v1.md)：高密度速查卡片中文版，含底层操作系统/高并发后端/现代渲染前端/云原生容器化/系统设计调优/软件工程素养折衷矩阵。
*   [developer_roadmap-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_developer_roadmap.py](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/build_latex_developer_roadmap.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_developer_roadmap.py](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/generate_html_developer_roadmap.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[developer_roadmap-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[developer_roadmap-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[developer_roadmap-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[developer_roadmap-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-精美知识图谱-v1-en.html)
    *   前端虚拟 DOM Diff 与 Kubernetes 弹性伸缩双仿真沙盒：
        *   🇨🇳 中文版：[developer_roadmap-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[developer_roadmap-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/52-实跑-developer_roadmap/developer_roadmap-互动演练沙盒-v1-en.html)



## 📂 成果三十六：java-design-patterns (企业级经典设计模式与并发架构)

项目已在工作区独立目录 `53-实跑-java_design_patterns` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-java_design_patterns.md](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/Step1-判题材-java_design_patterns.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [java_design_patterns-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [java_design_patterns-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-Cheatsheet-v1.md)：高密度速查卡片中文版，含创建型/结构型/行为型/企业级分层/JVM并发基础/高级同步锁优化折衷矩阵。
*   [java_design_patterns-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_java_design_patterns.py](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/build_latex_java_design_patterns.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_java_design_patterns.py](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/generate_html_java_design_patterns.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[java_design_patterns-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[java_design_patterns-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[java_design_patterns-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[java_design_patterns-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-精美知识图谱-v1-en.html)
    *   AQS 双向同步队列与 Synchronized 锁膨胀双仿真沙盒：
        *   🇨🇳 中文版：[java_design_patterns-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[java_design_patterns-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/53-实跑-java_design_patterns/java_design_patterns-互动演练沙盒-v1-en.html)



## 📂 成果三十七：coding-interview-university (计算机科学体系、底层算法与工程素养)

项目已在工作区独立目录 `54-实跑-coding_interview_university` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-coding_interview_university.md](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/Step1-判题材-coding_interview_university.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [coding_interview_university-高密度卡片系统设计大图.md](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [coding_interview_university-Cheatsheet-v1.md](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-Cheatsheet-v1.md)：高密度速查卡片中文版，含数据结构/高级树/图论/排序搜索/算法范式/系统物理底层/软件工程折衷矩阵。
*   [coding_interview_university-Cheatsheet-v1-en.md](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_coding_interview_university.py](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/build_latex_coding_interview_university.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_coding_interview_university.py](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/generate_html_coding_interview_university.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[coding_interview_university-Cheatsheet-v1.pdf](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[coding_interview_university-Cheatsheet-v1-en.pdf](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[coding_interview_university-精美知识图谱-v1.html](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[coding_interview_university-精美知识图谱-v1-en.html](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-精美知识图谱-v1-en.html)
    *   BST 树遍历与 CPU Cache 缓存行淘汰双仿真沙盒：
        *   🇨🇳 中文版：[coding_interview_university-互动演练沙盒-v1.html](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[coding_interview_university-互动演练沙盒-v1-en.html](file:///D:/材料/硬核技术系统工程/54-实跑-coding_interview_university/coding_interview_university-互动演练沙盒-v1-en.html)




## 📂 成果三十八：tokio-rs / tokio (Rust 工业级异步运行时与并发调度器)

项目已在工作区独立目录 `04-computer-networks/55-tokio` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-tokio.md](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/Step1-判题材-tokio.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [tokio-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [tokio-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-Cheatsheet-v1.md)：高密度速查卡片中文版，含调度策略/异步同步锁/任务通道等设计。
*   [tokio-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_tokio.py](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/build_latex_tokio.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_tokio.py](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/generate_html_tokio.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[tokio-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[tokio-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[tokio-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[tokio-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-精美知识图谱-v1-en.html)
    *   Tokio Work-Stealing 运行队列与协程锁/MPSC 同步双仿真沙盒：
        *   🇨🇳 中文版：[tokio-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[tokio-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/55-tokio/tokio-互动演练沙盒-v1-en.html)


## 📂 成果三十九：bytecodealliance / wasmtime (WebAssembly 高效 JIT 虚拟机与轻量沙箱运行时)

项目已在工作区独立目录 `06-compilers-and-runtimes/56-wasmtime` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-wasmtime.md](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/Step1-判题材-wasmtime.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [wasmtime-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [wasmtime-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-Cheatsheet-v1.md)：高密度速查卡片中文版，含JIT/Cranelift/安全沙箱等设计。
*   [wasmtime-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_wasmtime.py](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/build_latex_wasmtime.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_wasmtime.py](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/generate_html_wasmtime.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[wasmtime-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[wasmtime-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[wasmtime-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[wasmtime-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-精美知识图谱-v1-en.html)
    *   Wasmtime JIT 编译流水线与硬件页保护边界双仿真沙盒：
        *   🇨🇳 中文版：[wasmtime-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[wasmtime-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/06-compilers-and-runtimes/56-wasmtime/wasmtime-互动演练沙盒-v1-en.html)


## 📂 成果四十：cilium / cilium (基于 eBPF 的高性能云原生网络与安全网关)

项目已在工作区独立目录 `03-operating-systems/57-cilium` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-cilium.md](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/Step1-判题材-cilium.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [cilium-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [cilium-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-Cheatsheet-v1.md)：高密度速查卡片中文版，含eBPF/数据面短路/网络路由设计。
*   [cilium-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_cilium.py](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/build_latex_cilium.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_cilium.py](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/generate_html_cilium.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[cilium-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[cilium-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[cilium-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[cilium-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-精美知识图谱-v1-en.html)
    *   Cilium eBPF 极速收发包与 sockops 协议栈短路双仿真沙盒：
        *   🇨🇳 中文版：[cilium-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[cilium-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/03-操作系统与%20Linux%20内核/57-cilium/cilium-互动演练沙盒-v1-en.html)


## 📂 成果四十一：tikv / tikv (CNCF 毕业级分布式事务 Key-Value 数据库)

项目已在工作区独立目录 `01-distributed-systems/58-tikv` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-tikv.md](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/Step1-判题材-tikv.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [tikv-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [tikv-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-Cheatsheet-v1.md)：高密度速查卡片中文版，含Multi-Raft/Percolator/LSM-tree等。
*   [tikv-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_tikv.py](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/build_latex_tikv.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_tikv.py](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/generate_html_tikv.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[tikv-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[tikv-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[tikv-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[tikv-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-精美知识图谱-v1-en.html)
    *   TiKV 协处理器下推与 Multi-Raft/Percolator 状态机双仿真沙盒：
        *   🇨🇳 中文版：[tikv-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[tikv-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/58-tikv/tikv-互动演练沙盒-v1-en.html)


## 📂 成果四十二：jepsen-io / maelstrom (分布式系统一致性与容错测试床)

项目已在工作区独立目录 `01-distributed-systems/59-maelstrom` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-maelstrom.md](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/Step1-判题材-maelstrom.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [maelstrom-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [maelstrom-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-Cheatsheet-v1.md)：高密度速查卡片中文版，含一致性拓扑/网络分区模拟/Jepsen验证策略。
*   [maelstrom-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_maelstrom.py](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/build_latex_maelstrom.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_maelstrom.py](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/generate_html_maelstrom.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[maelstrom-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[maelstrom-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[maelstrom-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[maelstrom-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-精美知识图谱-v1-en.html)
    *   Maelstrom 网络混沌模拟与因果/线性化时序双仿真沙盒：
        *   🇨🇳 中文版：[maelstrom-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[maelstrom-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/01-distributed-systems/59-maelstrom/maelstrom-互动演练沙盒-v1-en.html)


## 📂 成果四十三：duckdb / duckdb (嵌入式分析型 SQL 引擎)

项目已在工作区独立目录 `02-database-internals/60-duckdb` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-duckdb.md](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/Step1-判题材-duckdb.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [duckdb-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [duckdb-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-Cheatsheet-v1.md)：高密度速查卡片中文版，含列式布局/行组划分/向量化执行/Epoch MVCC折衷矩阵。
*   [duckdb-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_duckdb.py](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/build_latex_duckdb.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_duckdb.py](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/generate_html_duckdb.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[duckdb-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[duckdb-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[duckdb-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[duckdb-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-精美知识图谱-v1-en.html)
    *   DuckDB 向量化计算与 Epoch MVCC 双仿真沙盒：
        *   🇨🇳 中文版：[duckdb-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[duckdb-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/60-duckdb/duckdb-互动演练沙盒-v1-en.html)


## 📂 成果四十四：spacejam / sled (Rust 现代嵌入式 B-Tree 键值存储引擎)

项目已在工作区独立目录 `02-database-internals/61-sled` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-sled.md](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/Step1-判题材-sled.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [sled-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [sled-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-Cheatsheet-v1.md)：高密度速查卡片中文版，含无锁 B-Link 树/页表原子 CAS/LSN 物理时序/EBR 回收折衷矩阵。
*   [sled-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_sled.py](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/build_latex_sled.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_sled.py](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/generate_html_sled.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
    *   🇨🇳 中文版 PDF：[sled-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-Cheatsheet-v1.pdf)
    *   🇺🇸 英文版 PDF：[sled-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
    *   精美知识图谱：
        *   🇨🇳 中文版：[sled-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-精美知识图谱-v1.html)
        *   🇺🇸 英文版：[sled-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-精美知识图谱-v1-en.html)
    *   Sled 无锁 B-Link 树与 Delta 日志段双仿真沙盒：
        *   🇨🇳 中文版：[sled-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-互动演练沙盒-v1.html)
        *   🇺🇸 英文版：[sled-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/02-database-internals/61-sled/sled-互动演练沙盒-v1-en.html)
\n\n
## 📂 成果四十五：scylladb / seastar (Shard-per-Core 极速无锁高性能网络框架)

项目已在工作区独立目录 `04-computer-networks/62-seastar` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-seastar.md](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/Step1-判题材-seastar.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [seastar-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [seastar-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-Cheatsheet-v1.md)：高密度速查卡片中文版，含无锁内存分配/协作式纤程/SMP 屏障/NUMA 拓扑折衷矩阵。
*   [seastar-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_seastar.py](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/build_latex_seastar.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_seastar.py](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/generate_html_seastar.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[seastar-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[seastar-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[seastar-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[seastar-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-精美知识图谱-v1-en.html)
		*   Seastar Shard-per-Core 与异步链式调度沙盒：
				*   🇨🇳 中文版：[seastar-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[seastar-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/04-计算机网络与高性能%20IO/62-seastar/seastar-互动演练沙盒-v1-en.html)

---

## 📂 成果四十六：dae-networks / dae (基于 eBPF 的高性能内核层透明代理)

项目已在工作区独立目录 `03-operating-systems/63-dae` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-dae.md](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/Step1-判题材-dae.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [dae-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [dae-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-Cheatsheet-v1.md)：高密度速查卡片中文版，含eBPF TC过滤/SOCKMAP重定向/Fake IP机制/cgo混合编程折衷矩阵。
*   [dae-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_dae.py](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/build_latex_dae.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_dae.py](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/generate_html_dae.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[dae-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[dae-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[dae-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[dae-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-精美知识图谱-v1-en.html)
		*   dae eBPF 内核重定向与 TProxy/DNS 劫持沙盒：
				*   🇨🇳 中文版：[dae-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[dae-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/03-operating-systems/63-dae/dae-互动演练沙盒-v1-en.html)

---

## 📂 成果四十七：dapr / dapr (云原生 Sidecar 分布式应用运行时)

项目已在工作区独立目录 `05-system-design/64-dapr` 下完成全栈交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-dapr.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/Step1-判题材-dapr.md)：题材分析、莫兰迪配色体系、L0-L2 阶梯模型与 28 张核心卡片大纲规划。
*   [dapr-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-高密度卡片系统设计大图.md)：28张卡片的依赖拓扑关系图（Mermaid 格式）及物理源码映射。
*   [dapr-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-Cheatsheet-v1.md)：高密度速查卡片中文版，含Sidecar通信/ETag乐观锁/CloudEvents标准/Placement一致性哈希折衷矩阵。
*   [dapr-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_dapr.py](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/build_latex_dapr.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_dapr.py](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/generate_html_dapr.py)：生成中英文交互式知识图谱和双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[dapr-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[dapr-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[dapr-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[dapr-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-精美知识图谱-v1-en.html)
		*   dapr Sidecar 消息拦截与分布式 Actor 哈希调度沙盒：
				*   🇨🇳 中文版：[dapr-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[dapr-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/64-dapr/dapr-互动演练沙盒-v1-en.html)

## 📂 成果三十六：ros2 机器人 DDS 实时中间件与共享内存

项目已在工作区独立目录 `10-physical-systems-and-control/69-ros2` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-ros2.md](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/Step1-判题材-ros2.md)：题材分析、莫兰迪配色体系与 28 张核心卡片大纲。
*   [ros2-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-高密度卡片系统设计大图.md)：28张卡片拓扑依赖图及 ROS 2 源码位置锚点。
*   [ros2-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-Cheatsheet-v1.md)：高密度速查卡片中文版，含中间件思维滤镜、QoS 匹配矩阵与调优字典。
*   [ros2-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_ros2.py](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/build_latex_ros2.py)：解析 markdown 并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_ros2.py](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/generate_html_ros2.py)：生成中英文交互式知识图谱与 DDS 双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[ros2-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[ros2-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[ros2-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[ros2-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-精美知识图谱-v1-en.html)
		*   ROS 2 DDS 共享内存与 Socket 通信延迟对比沙盒：
				*   🇨🇳 中文版：[ros2-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[ros2-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/10-physical-systems-and-control/69-ros2/ros2-互动演练沙盒-v1-en.html)

## 📂 成果三十七：rust_patterns 高级借用设计模式与 Typestate

项目已在工作区独立目录 `11-software-methodologies/70-rust_patterns` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-rust_patterns.md](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/Step1-判题材-rust_patterns.md)：题材分析、莫兰迪配色体系与 28 张核心卡片大纲。
*   [rust_patterns-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-高密度卡片系统设计大图.md)：28张卡片依赖拓扑依赖图及 Rust 核心库与三方库源码位置锚点。
*   [rust_patterns-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-Cheatsheet-v1.md)：高密度速查卡片中文版，含生命周期思维滤镜、策略派发折衷与调试字段字典。
*   [rust_patterns-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_rust_patterns.py](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/build_latex_rust_patterns.py)：解析 markdown 并调用 XeLaTeX 编译为高密 A4 Landscape PDF。
*   [generate_html_rust_patterns.py](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/generate_html_rust_patterns.py)：生成中英文交互式知识图谱与 Typestate 双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[rust_patterns-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[rust_patterns-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[rust_patterns-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[rust_patterns-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-精美知识图谱-v1-en.html)
		*   Rust Typestate 编译期状态转移与自引用固定双仿真沙盒：
				*   🇨🇳 中文版：[rust_patterns-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[rust_patterns-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/70-rust_patterns/rust_patterns-互动演练沙盒-v1-en.html)

## 📂 成果三十八：production_survival 业务工程学与防雪崩重构

项目已在工作区独立目录 `11-software-methodologies/71-production_survival` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-production_survival.md](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/Step1-判题材-production_survival.md)：题材分析、莫兰迪配色体系与 28 张核心卡片大纲。
*   [production_survival-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-高密度卡片系统设计大图.md)：28张卡片拓扑依赖图及 Resilience4j/K8s/Debezium/W3C 物理源码映射。
*   [production_survival-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-Cheatsheet-v1.md)：高密度速查卡片中文版，含防雪崩系统思维限制、架构折衷矩阵与调试参数字典。
*   [production_survival-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_production_survival.py](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/build_latex_production_survival.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_production_survival.py](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/generate_html_production_survival.py)：生成中英文交互式知识图谱与双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[production_survival-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[production_survival-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[production_survival-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[production_survival-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-精美知识图谱-v1-en.html)
		*   微服务网关熔断舱壁与双写迁移仿真实验室沙盒：
				*   🇨🇳 中文版：[production_survival-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[production_survival-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/11-software-methodologies/71-production_survival/production_survival-互动演练沙盒-v1-en.html)

## 📂 成果三十九：daily_diagnostics 线上故障定位与性能榨汁机

项目已在工作区独立目录 `07-performance-and-diagnostics/72-daily_diagnostics` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-daily_diagnostics.md](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/Step1-判题材-daily_diagnostics.md)：性能分析与日常故障排查题材分析、莫兰迪配色体系与 28 张核心卡片大纲。
*   [daily_diagnostics-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-高密度卡片系统设计大图.md)：28张性能排错与火焰图分析卡片拓扑依赖图及 Linux/JVM/MySQL 源码位置锚点。
*   [daily_diagnostics-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-Cheatsheet-v1.md)：高密度速查卡片中文版，含性能榨汁系统思维限制、架构折衷矩阵与性能调试参数字典。
*   [daily_diagnostics-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_daily_diagnostics.py](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/build_latex_daily_diagnostics.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_daily_diagnostics.py](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/generate_html_daily_diagnostics.py)：生成中英文交互式性能分析与排错双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[daily_diagnostics-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[daily_diagnostics-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[daily_diagnostics-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[daily_diagnostics-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-精美知识图谱-v1-en.html)
		*   CPU 火焰图与线程调度冲突及 MySQL 索引跳转与事务死锁双仿真沙盒：
				*   🇨🇳 中文版：[daily_diagnostics-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[daily_diagnostics-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/07-performance-and-diagnostics/72-daily_diagnostics/daily_diagnostics-互动演练沙盒-v1-en.html)

## 📂 成果四十：devops_firefighting 云原生与 SRE 生产救火手册

项目已在工作区独立目录 `05-system-design/73-devops_firefighting` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-devops_firefighting.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/Step1-判题材-devops_firefighting.md)：云原生运维与可观测性生产调优题材分析、莫兰迪配色体系与 28 张核心卡片大纲。
*   [devops_firefighting-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-高密度卡片系统设计大图.md)：28张故障排查与 SRE 救火卡片拓扑依赖图及 Kubernetes/Nginx/Cilium 物理源码映射锚点。
*   [devops_firefighting-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-Cheatsheet-v1.md)：高密度速查卡片中文版，含 SRE 救火系统思维约束、架构折衷矩阵与故障诊断配置字典。
*   [devops_firefighting-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_devops_firefighting.py](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/build_latex_devops_firefighting.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_devops_firefighting.py](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/generate_html_devops_firefighting.py)：生成中英文交互式云原生运维与 DDS 双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[devops_firefighting-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[devops_firefighting-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[devops_firefighting-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[devops_firefighting-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-精美知识图谱-v1-en.html)
		*   Kubernetes Pod 副本调度与 CoreDNS 延迟及丢包双仿真沙盒：
				*   🇨🇳 中文版：[devops_firefighting-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[devops_firefighting-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/73-devops_firefighting/devops_firefighting-互动演练沙盒-v1-en.html)

## 📂 成果四十一：tech_selection 高频架构设计折中矩阵

项目已在工作区独立目录 `05-system-design/74-tech_selection` 下完成交付：

### 1. 拆解源文件与设计规范
*   [Step1-判题材-tech_selection.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/Step1-判题材-tech_selection.md)：高频架构决策与折中题材分析、莫兰迪配色体系与 28 张核心卡片大纲。
*   [tech_selection-高密度卡片系统设计大图.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-高密度卡片系统设计大图.md)：28张选型卡片依赖拓扑图及物理源码/规范位置锚点映射。
*   [tech_selection-Cheatsheet-v1.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-Cheatsheet-v1.md)：高密度速查卡片中文版，含选型系统思维认识论、架构折衷对照表与核心组件调试字段字典。
*   [tech_selection-Cheatsheet-v1-en.md](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-Cheatsheet-v1-en.md)：高密度速查卡片英文版。

### 2. 自动编译与构建脚本
*   [build_latex_tech_selection.py](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/build_latex_tech_selection.py)：解析 markdown 并调用 XeLaTeX 编译为高密双页 A4 Landscape PDF。
*   [generate_html_tech_selection.py](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/generate_html_tech_selection.py)：生成中英文交互式架构选型与一致性哈希双仿真沙盒 HTML。

### 3. 编译与运行产物
*   **中英双语双页海报 PDF**：
		*   🇨🇳 中文版 PDF：[tech_selection-Cheatsheet-v1.pdf](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-Cheatsheet-v1.pdf)
		*   🇺🇸 英文版 PDF：[tech_selection-Cheatsheet-v1-en.pdf](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-Cheatsheet-v1-en.pdf)
*   **网页端交互产物**：
		*   精美知识图谱：
				*   🇨🇳 中文版：[tech_selection-精美知识图谱-v1.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-精美知识图谱-v1.html)
				*   🇺🇸 英文版：[tech_selection-精美知识图谱-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-精美知识图谱-v1-en.html)
		*   一致性哈希环负载与消息队列吞吐缓冲双仿真沙盒：
				*   🇨🇳 中文版：[tech_selection-互动演练沙盒-v1.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-互动演练沙盒-v1.html)
				*   🇺🇸 英文版：[tech_selection-互动演练沙盒-v1-en.html](file:///D:/材料/拆书/硬核技术系统工程/05-system-design/74-tech_selection/tech_selection-互动演练沙盒-v1-en.html)

## 🔬 验证与测试结果更新

1.  **XeLaTeX 编译验证**：
    *   使用 MiKTeX xelatex 对 `redis`、`postgres`、`clickhouse`、`ostep` Nic、`linux-insides` 、`xv6-riscv`·、`top-down-networking`、`dpdk`、`envoy`、`quic-http3`起、`system-design-primer`、`awesome-scalability`、`google-sre-book`、`mit_6.824`、`cockroachdb`、`craftinginterpreters`、`v8`、`openjdk`、`golang`、`llvm` 、`system-performance`、`sanitizers` 、`riscv-isa-manual` 、`cuda-samples` 、`openssl` 、`intel_sgx` 、`awesome_robotics` 、`power_systems` 、`developer_roadmap` 、`java_design_patterns` 、`coding_interview_university` 、`tokio` 、`wasmtime` 、`cilium` 、 `tikv` 、 `maelstrom` 、 `duckdb`、`sled`、`seastar`、`dae`、`dapr`、`ros2`、`rust_patterns` 、`production_survival`、`daily_diagnostics` 、`devops_firefighting` 与 `tech_selection` 项目的中英文 `.tex` 进行了多轮编译。
    *   所有 PDF 均成功输出，格式为 Landscape 双面 A4，排版紧密对称，无任何内容溢出或重叠。
2.  **HTML 交互与沙盒运行验证**：
    *   **Redis 渐进式 Rehash 与 近似 LRU 淘汰沙盒** 支持：
      *   `渐进式 Rehash 动态模拟`：前台插入或更新会自动触发哈希表从 `ht[0]` 到 `ht[1]` 的分步迁移，并以色彩标识当前迁移游标。
      *   `近似 LRU 淘汰算法演示`：对比严格全局 LRU，展示 Redis 随机采样淘汰机制在不同 `maxmemory-samples` 下的驱逐精准度与内存开销。
    *   **PostgreSQL Heap MVCC 与 SSI 锁依赖环沙盒** 支持：
      *   `Heap Page 元组行可见性分析`：插入与更新会产生不同的 xmin/xmax 标记，读快照时能根据 XID 进行可见性裁剪；支持 Auto-vacuum 清理死元组并执行 XID 冰冻保护。
      *   `SSI 串行化双重反依赖环路检测`：支持在交易图谱中动态执行读写操作，当形成 $T1 \dots T3$ 这类反依赖闭环时，自动触发 Abort 并拦截事务提交。
    *   **ClickHouse 稀疏索引标记扫描与 CPU SIMD 向量化吞吐沙盒** 支持：
      *   `MergeTree 稀疏索引定位`：输入主键检索区间，自动比对常驻内存的 primary.idx 圈定 mark 序号，并跳过不符合条件的磁盘 granules 块解压读取。
      *   `向量化计算性能对比`：模拟千兆级行数的过滤与聚合，直观显示列式 SIMD 计算对行式 Volcano 解释执行引擎的十几倍乃至数十倍性能吞吐提升。
    *   **OSTEP 多级页表地址翻译与 MLFQ 调度仿真沙盒** 支持：
      *   `二级页表虚拟地址转换（VPN ➜ PFN）`：用户输入十六进制地址（如 0x002030A0），模拟 CPU 对 VPN 段的 Directory/Page Table 进行两级 Walk 映射，展示 PDE/PTE 控制位，支持 TLB 命中（Hit）/失效（Miss）及 FIFO 缓存行淘汰过程。
      *   `多级反馈队列（MLFQ）动态优先级调度`：动态增加不同 CPU burst 进程，模拟系统 Tick 时钟流转，演示时间片消耗导致优先级自动下调，以及一键 Trigger 全局 Priority Boost 防饥饿提升的演进日志。
    *   **Linux Kernel Buddy 物理分配与 SLUB 缓存回收集成沙盒** 支持：
      *   `伙伴页分配系统分裂与合并演示`：支持以 Order-0 到 Order-6 (16KB 到 1024KB) 粒度分配页框，展示空闲页块递归分裂逻辑；释放块时自动通过异或（XOR）运算寻找 Buddy，执行递归合并并收拢回原始高阶页块。
      *   `SLUB 缓存分配与 Buddy 页绑定交互`：模拟 kmem_cache_cpu 活跃高速缓存快速路径分配，无任何全局锁限制；当活跃 Slab 耗尽（分配第 9 个对象）时，触发慢速路径 deactivation，自动从伙伴系统申请 64KB 空闲页框构建新 Slab，实现内核两级内存管理的无缝融合演示。
    *   **xv6-riscv SV39 三级页表翻译与 UNIX 管道多进程同步沙盒** 支持：
      *   `RISC-V SV39 页表遍历硬件翻译仿真`：支持输入任意虚拟地址，将其二进制分解为 VPN[2]、VPN[1]、VPN[0] 及 Offset，图形化展现三级页表分级跳转索引过程；模拟 TLB 硬件高速缓存行更新及随机驱逐，演示 Page Fault 页错误异常。
      *   `UNIX 管道环形缓冲区与读写睡眠唤醒同步`：仿真 16 字节 Pipe 环形队列，支持 `pipewrite()` 和 `piperead()` 系统调用；当写者写满缓冲区时进程自动挂起进入 `SLEEPING` 状态（等待通道 `&nread`），读者读取数据后调用 `wakeup(&nwrite)` 重置写者状态为 `RUNNABLE`，直观呈现多进程无锁协作。
    *   **top-down-networking 双端滑动窗口与 Cubic/BBR 拥塞控制仿真沙盒** 支持：
      *   `滑动窗口丢包与超时重传控制（GBN vs SR）`：可选择 GBN 或 SR 模式，在发送过程中点击飞行包可直接使其“丢包”，模拟超时重传，直观演示 GBN 批量重传（后退 N 步）与 SR 精准重传（选择性重传）以及接收端乱序缓冲区的差异。
      *   `拥塞控制状态机演进（CUBIC vs BBR）`：对比丢包驱动的 CUBIC 乘性减小与速率/延迟驱动的 BBR 寻找最优 BDP（带宽延迟积）的工作机制。支持模拟 Bufferbloat 缓冲区积压，展示 CUBIC 盲目增大窗口导致延迟飙升，而 BBR 能够感知 RTT 上升并主动控制发送速率以清洗队列。
    *   **DPDK 无锁环形队列 CAS 指针与 PMD 轮询仿真沙盒** 支持：
      *   `rte_ring 多生产者并发 CAS 预留机制`：分步演示多生产者核心进行原子比较并交换（Compare-And-Swap）抢占头指针、并行写数据以及最终自旋对齐更新 tail 指针的锁存收敛过程。
      *   `PMD 用户态轮询与内核中断性能对比`：支持调整包到达率（0 到 100k pps）。展示中断模式在极高包率下产生中断风暴（CPU 负载 100%、丢包率飙升、延迟增加），而 PMD 通过批量（Burst）轮询，消除中断开销，实现稳定吞吐。
    *   **Envoy L4/L7 过滤器链与 Dynamic xDS 热重启仿真沙盒** 支持：
      *   `三级过滤器链与请求流式拦截（Listener ➜ Network ➜ HTTP）`：支持在图形化管线中自定义 Host、Path 及 Auth 标头，模拟连接 SNI 嗅探、TLS 终结、JWT 鉴权及 Local 限流步骤，演示失败拦截及成功上游 EDS 分流的过程。
      *   `xDS 动态配置指针切换与优雅热重启 Draining`：演示在不重启服务的情况下瞬间原子切换 shared_ptr 路由映射表；单步运行 Hot Restart 状态机，展示子父进程通过 UDS 传递套接字 FD、子进程接管监听、父进程平滑 Drain 耗尽活跃请求并安全退役的优雅收拢机制。
    *   **H2O QUIC 连接迁移与 HTTP/2 WDRR 优先级树发包仿真沙盒** 支持：
      *   `QUIC 连接迁移与双向路径验证机制`：模拟客户端因网卡变动（WiFi ➜ 5G 蜂窝网）导致 IP/端口变化，利用 Connection ID (CID) 维持 HTTP/3 连接；单步模拟服务端执行 `PATH_CHALLENGE` 并在接收到客户端 `PATH_RESPONSE` 后验证路径并热更路由，实现长连接零闪断传输。
      *   `HTTP/2 调度树加权赤字轮询与 16KB 分片发包`：仿真 WDRR 调度树机制及 16KB 发包块硬上限，直观展示多路复用请求流量如何在 bottleneck 物理链路中以 16KB 粒度时间片混流，彻底消除应用层头部阻塞（HOLB）。
    *   **System Design Primer 一致性哈希环与缓存同步策略沙盒** 支持：
      *   `一致性哈希环动态模拟`：可动态添加/删除物理服务器节点，随时随机撒播 Keys 并观察沿环顺时针分配的流向线；支持切换 3x/10x 虚拟节点倍率，以柱状图及百分比直观展示负载倾斜消除效果与 Key 迁移对数收敛。
      *   `Cache-aside/Write-through/Write-behind 缓存读写策略演练`：支持以三种模式发起 Read Miss/Hit 与 Write 操作，分步显示客户端、Redis 缓存与 MySQL 数据库间的数据包包流走向，并根据操作测算与展示平均读写延迟的折衷变化。
    *   **awesome-scalability CQRS 事件溯源与微服务舱壁隔离限流沙盒** 支持：
      *   `CQRS 读写分离与事件重播（Event Replay）`：模拟命令端接收写请求生成 Event 写入 Event Store 并异步投影至 Read View Cache 的全过程，演示最终一致性时间差；支持清空缓存并通过顺序 replay 历史事件日志来从零瞬间重建读数据库视图。
      *   `微服务线程隔离舱壁（Bulkhead）与熔断防护`：模拟大并发流量访问高延时 Payment 服务，展示 No Isolation 模式下慢请求瞬间占满全局线程池导致 healthy Inventory 服务一同雪崩的过程；切换 Bulkhead 模式可将并发硬性隔离在独立专属线程池保护核心链路，而 Circuit Breaker 开启则可一键 Tripped 熔断快速返回 Fallback 释放全部系统线程.
    *   **google-sre-book 错误预算多燃烧率告警与客户端自适应限流沙盒** 支持：
      *   `多窗口多燃烧率告警模拟`：可在 1h/6h 窗口内调整系统服务状态（健康/泄漏/大灾难），展示对应燃烧率（14.4 / 6.0）触发 paging 唤醒 on-call 的过程，直观观察剩余错误预算在持续异常下的折损趋势。
      *   `客户端自适应限流与服务端优雅降级`：模拟大并发流量突发，对比在无保护、仅服务端优雅降级（核心扣款正常，丢弃广告低保真流量）以及配合客户端本地自适应限流公式拦截过载后的流量成功率及延时变化。
    *   **MIT 6.824 Raft 共识强一致性与 MapReduce 并行调度双沙盒** 支持：
      *   `Raft 共识与领导者选举仿真`：5节点状态机实时运转，支持随机选举超时、心跳同步、多数派 commitIndex 推进；支持手动注入 Crash/Reboot 模拟故障，以及一键拆分为 P1[1, 2] & P2[3, 4, 5] 脑裂分区，演示多数派选举隔离与分区合并后的 Term 单调递增覆盖与日志强行同步机制。
      *   `MapReduce 并行计算与 Master 重调度调度`：支持输入任意 WordCount 字符串，流程化演练 Split 分片、Map 映射、Shuffle 排序哈希路由、Reduce 规约及 Output GFS 持久化；在 Map 阶段支持手动点击 Worker 触发故障丢失中间数据，演示 Master 心跳探活并自动将失败分片重调度（Reschedule）至健康节点重跑的容错恢复流程。
    *   **CockroachDB 分布式 2PC 提交与 HLC/MVCC 时钟偏移双沙盒** 支持：
      *   `分布式 2PC 与写意图锁`：图形化演练协调器、Range 节点与 System Record 状态表，展示 PENDING/COMMITTED/ABORTED 状态切换及行级 Write Intent 意图锁的读写拦截；支持手动注入 Range Crash 触发事务自动回滚重试与异步清理（Intent Cleanup）。
      *   `HLC 时钟与 MVCC 漂移 Read Restart 校验`：支持为各节点配置独立物理时钟偏移（Clock Skew），若偏移量超过 200ms 自动触发 Auto-Panic 熔断下线；支持在不同节点间读写，若读取 HLC 落在不确定性区间（Uncertainty Window）内，系统自动触发 Read Restart 防御，保证数据快照串行一致。


    *   **Crafting Interpreters 手写编译器与虚拟机沙盒** 支持：
      *   `Pratt 解析器与 Stack VM 运行模拟`：将中缀数学表达式（如 `1 + 2 * 3`）一步步通过 Pratt 优先级算法编译为对应的字节码 OPCODE（如 `OP_CONSTANT`、`OP_ADD`、`OP_MULTIPLY`），并在堆栈虚拟机中演示压栈、出栈的真实计算轨迹。
      *   `三色标记垃圾回收机制模拟`：直观演示 GC 从 Roots（全局变量、运行栈）扫描，将 live 对象置为灰色并加入 worklist，通过扩散将关联对象涂黑，最后清扫 Intern 字符串弱引用表以及将剩余白色死对象 sweeping 回收的完整内存演进。
    *   **V8 引擎 JIT 优化与 GC 虚拟机及去优化沙盒** 支持：
      *   `对象隐藏类 (Map) 转换树与内联缓存 (IC) 状态演练`：用户可动态在 JS 对象上添加或删除属性，图形化观察 Map 转换链与快属性/慢属性槽物理存储分布；在 IC 函数调用点，传入不同属性特征的对象，演示 IC 状态从 `UNINITIALIZED` ➜ `MONOMORPHIC` ➜ `POLYMORPHIC` ➜ `MEGAMORPHIC` 的动态跃迁与查表评估延迟变化。
      *   `Ignition 字节码调试与 TurboFan 编译/Deopt 过程模拟`：单步执行 Ignition 字节码，展示物理寄存器、累加器与反馈插槽更新逻辑；加热运行后生成高度优化的机器码并执行 JIT Map 校验；当传入错误参数类型时，触发 Eager Deoptimization 并动画演示重构解释器栈帧、安全 Bailout 退回到解释器执行的完整路径。
    *   **OpenJDK JMM 内存屏障与 ZGC 彩色指针读屏障自愈沙盒** 支持：
      *   `JMM 指令重排与 StoreLoad 屏障仿真`：模拟多线程（Thread A/B）在非 Volatile 模式下指令乱序及 Store Buffer 滞后刷新导致 `r1 == 0 && r2 == 0` 的并发违背，以及 Volatile 模式下注入 `StoreLoad (lock addl)` 屏障物理强制冲刷 Store Buffer 的一致性保障过程。
      *   `ZGC 彩色指针并发转移与读屏障指针自愈`：流程化展现 ZGC 阶段切换（Concurrent Mark ➜ Concurrent Relocate），模拟旧堆页到新堆页的拷贝及 Forwarding Table 转发表的映射构建，并通过读屏障拦截 Stale 指针、检索转发表自愈更新为新地址并标记为 Remapped 的完整生命周期。
    *   **Go Runtime GMP 调度器与三色标记 GC 屏障沙盒** 支持：
      *   `GMP 并发调度与 Work Stealing / Preemption 抢占机制`：图形化展现 G（协程）、M（工作线程）、P（处理器上下文）以及全局和局部就绪队列的实时转换。单步运行可触发 M 执行 System Call 导致 P 释放并接管、Sysmon 强制抢占（Preemption）运行时间过长的 G，以及空闲 P 从其他 P 窃取（Work Stealing）或从全局队列获取就绪 G 的并发流转。
      *   `三色标记垃圾回收与混合写屏障（Hybrid Write Barrier）`：分步演示 GC 从 Root 扫描，将 live 对象置为灰色放入 worklist 并逐步涂黑；可手动开关“混合写屏障（Yuasa/Dijkstra）”开关，对比在并发 mutator 写入时，无写屏障导致虚无悬空指针崩溃，而开启混合写屏障时能够实时捕获并着色新旧指针，完美保障内存安全的完整并发标记生命周期。
    *   **LLVM 中端 Pass 优化与后端寄存器分配双沙盒** 支持：
      *   `中端 Pass 优化流水线（Mem2Reg, LICM, SimplifyCFG）`：用户可输入简单 C 语言函数，仿真器会即时展现其 Lowering 后的原始 IR（含栈帧访存 alloca/load/store）。支持单步执行 Mem2Reg，实时呈现支配树边界的计算与虚拟寄存器 Phi 重命名合并；执行 LICM 会展示不变指令是如何被安全外提至 Loop Preheader 之外；执行 SimplifyCFG 展示 CFG 合并与 basic blocks 控制流精简。
      *   `后端寄存器分配着色与 Spilling 溢出模拟`：图形化展现虚拟寄存器的活跃区间重叠干涉图。支持一键进行图着色，根据度数使用物理寄存器（如 RAX, RBX, RCX）进行标记；限制寄存器数为 2 时会自动触发 Spill 溢出算法，选中冲突节点并演示如何在代码指令中插入内存写回/重载入指令，完成指令的物理改写。
    *   **Systems Performance CPU 火焰图与缺页异常 + 磁盘 I/O 与 bpftrace 双沙盒** 支持：
      *   `CPU 栈中断采样火焰图与内存次要/主要缺页异常仿真`：用户可选择不同的 CPU 占用负载场景（数据库查询、Web 服务路由、JSON 解析），图形化展示动态火焰图的合并块。可触发 `mmap` 分配及读写测试，演示不经过磁盘 I/O 的次要缺页（Minor PF，更新 MMU 页表）与从磁盘/Swap 读取的主要缺页（Major PF，引发 I/O 阻塞线程挂起），并动态绘制 `vmstat` 核心指标波动。
      *   `磁盘 I/O 调度延迟 biolatency 直方图与 bpftrace 控制台`：用户可调节磁盘调度器（Deadline, BFQ, none），触发 I/O 负载后实时渲染微秒级的 biolatency 对数直方图；提供动态 `bpftrace` 模拟控制台，用户可选择预设 of vfs_read/openat Tracing 脚本并点击执行，流式展现虚拟内核探针触发日志及结果 Map 柱状图聚合。
    *   **google-sanitizers ASan 影子映射与 TSan Happens-Before 竞争双沙盒** 支持：
      *   `ASan 内存影子映射与红区隔离毒化`：交互式模拟堆内存分配，图形化展现 8:1 影子内存状态字；点击读写指定索引地址，显示影子检查翻译过程（`ShadowAddr = (AppAddr >> 3) + Offset`），自动触发 Heap-Buffer-Overflow (读写红区 0xfa/0xfb) 崩溃栈或 Heap-Use-After-Free (读写已释放隔离堆块 0xfd) 符号报告。
      *   `TSan 向量时钟 Happens-Before 同步与死锁依赖环路检测`：图形化跟踪多个线程的 Vector Clocks 数组演进。执行读写操作自动检测无锁时的 Data Race 并展示时钟碰撞；执行锁获取/释放，利用锁时钟桥接同步两端时钟，消除竞争；一键模拟死锁，绘制锁依赖图环路并提出死锁预警。
*   **RISC-V 5级流水线旁路与 MESI Cache 一致性双仿真沙盒** 支持：
      *   `经典五级流水线旁路分支前推与冒险冒险冲突演示`：支持单步执行流水线五个阶段 (IF, ID, EX, MEM, WB)，可视化数据冲突 (RAW 冒险)，支持开关 Bypass Forwarding 前推，实时展示插入 NOP 气泡 (Bubble Stalls) 与直接旁路前推的数据流差异，演示 2-bit BHT 分支预测表状态迁移。
      *   `MESI 缓存一致性监听总线协议演示`：模拟两个 CPU 核心的私有 L1 Cache 以及共享 L2 Cache/主存，支持在不同核心上发起读写 (PrRd/PrWr) 请求，演示 Cache Line 状态 (M, E, S, I) 的状态迁移矩阵，展示总线侦听广播 (BusRd, BusRdX, BusUpgr) 对物理总线带宽的占用与同步过程。
    *   **CUDA 共享内存 Bank 冲突与多流异步重叠双仿真沙盒** 支持：
      *   `共享内存 Bank 冲突与合并访存三维演示`：模拟 32 线程的 Warp 对 32 Bank 共享内存的并发访问；允许用户调节地址偏移步长 (Stride) 或映射模式，直观展现线程与 Bank 的连线干涉，高亮 Bank 冲突，计算延迟；同时对比展示 Global Memory 的连续（Coalesced）与非连续合并事务数差异。
      *   `多流异步重叠并发重叠仿真`：提供 CPU ➜ GPU ➜ CPU 流水线并发重叠执行 Timeline 仿真；可对比单 Stream 顺序串行 vs 多 Stream 异步块状重叠，以图形化流水线动画演示 HtoD 拷贝、Kernel 运行、DtoH 拷贝的并轨执行，并计算加速比。
    *   **OpenSSL TLS 握手状态机与 AES-GCM 认证加密双仿真沙盒** 支持：
      *   `TLS 1.2 vs TLS 1.3 握手时序对比与密钥派生`：模拟完整握手流（ClientHello ➜ ServerHello ...），展示 1.2（2-RTT）与 1.3（1-RTT，利用 key_share）的时间对比；动态展示 HKDF 提取（Extract）与展开（Expand）派生出 `client_write_key` / `server_write_key` 的步骤。
      *   `AES-GCM (AEAD) 加密与完整性校验仿真`：支持输入明文、AAD（附加认证数据）、Key 与 IV，流程化展示 AES 加密和 GHASH 算法产生 TAG 的过程；验证阶段故意篡改密文或 AAD 导致解密失败（Integrity Violation），演示认证加密对数据的双重保护。
    *   **Intel SGX 飞地物理访存与本地远程认证双仿真沙盒** 支持：
      *   `飞地物理访存与 MEE 加密机制`：模拟不同 CPU 权限状态（Normal Mode vs Enclave Mode）下对物理内存 PRM/EPC 的读取与 EPCM 校验，演示 EPCM 违背触发 #GP，以及 MEE AES-XTS 解密与 MCTS 完整性校验过程。
      *   `本地与远程可信认证状态机`：流程化演练本地认证（EREPORT 生成与 EGETKEY 密钥共享 MAC 校正）和远程认证（QE/PCE 证书 Quote 签名与 Intel PCS 在线证书链校验）的协议交互流。
    *   **2-DOF 机械臂与路径规划双仿真沙盒** 支持：
      *   `2-DOF 机械臂正逆运动学`：支持调节关节角度 Theta1 和 Theta2，动态绘制两连杆机械臂在二维坐标系中的姿态，实时解算末端位置与雅可比矩阵行列式，并高亮邻近奇异状态。
      *   `Dijkstra 与 RRT 路径规划对比`：提供基于网格的 Dijkstra 搜索与基于随机采样的 RRT 树扩展演示，单步展现节点探索流，对比路径寻优成本与执行时间。
    *   **电力网格潮流计算与动力学转子摆动双仿真沙盒** 支持：
      *   `IEEE 5节点牛顿-拉夫逊潮流解算`：支持调节节点3的注入有功和线路阻抗，单步运行牛顿迭代，实时展示雅可比矩阵元素、有功功率不平衡量以及各母线电压和相角的收敛流转。
      *   `转子运动方程动力学暂态仿真`：模拟单机无穷大系统在三相短路故障下的转子功角 Delta 和频率 Omega 摇摆曲线。支持微调故障切除时间和小阻尼系数，展示系统从稳态振荡收敛到失步分叉（功角 >180 度）的临界穿越过程。
    *   **V-DOM Diff 与 Kubernetes HPA 双仿真沙盒** 支持：
      *   `前端 V-DOM Diff 算法`：支持选择“头部插入”、“随机打乱 Key”和“删除中间项”三种状态变更动作，对比渲染新旧虚拟 DOM 树，单步解析协调器（Reconciliation）中节点复用、新增和销毁的操作路径。
      *   `Kubernetes 弹性伸缩（HPA）仿真`：支持滑动调节网络流量（QPS），根据 CPU 负载自动运用公式计算期望副本数，实时展示容器状态漂移（Pending ➜ Running / Terminating），并在活跃 Pod 副本池上流转流量分发。
    *   **AQS FIFO 队列与 Synchronized 锁膨胀双仿真沙盒** 支持：
      *   `AQS 独占锁双向 CLH 队列等待模拟`：支持线程 CAS 竞争锁，成功则独占 state，失败则封装为 Node 插入队列尾部并 park 挂起；释放锁时自动 unpark 头部线程，演示 FIFO 唤醒流转。
      *   `Synchronized 对象头 Mark Word 锁膨胀升级演示`：模拟无锁到偏向锁（写 Thread ID）、轻量锁（栈帧锁记录 CAS 指针自旋）到重量级锁（OS 监视器互斥锁）的单向升级轨迹，展示各阶段 Mark Word 控制位标志转换。
    *   **BST 树遍历与 CPU Cache 缓存行淘汰双仿真沙盒** 支持：
      *   `二叉搜索树（BST）遍历模拟`：支持输入节点值插入树结构，图形化渲染二叉树层级拓扑，单步动画演示 DFS（中序）和 BFS（层序）遍历中激活节点的流转顺序。
      *   `CPU Cache 缓存行淘汰控制器模拟`：模拟 4 行 64 字节缓存行，支持选择 LRU 或 FIFO 淘汰策略，输入十六进制物理内存地址进行读取，演示缓存命中（HIT）或失效并载入（MISS/LOADED）以及淘汰旧行（EVICTED）的缓存一致状态切换。
    *   **Tokio Work-Stealing 运行队列与协程锁/MPSC 同步双仿真沙盒** 支持：
      *   `Work-Stealing 运行队列与任务窃取调度`：交互式模拟任务发布 (Spawn) 与工作线程调度时钟 (Tick)。支持 LIFO 插队插槽机制、本地双端环形无锁队列与有锁全局队列之间的层级流转，动态演示工作线程从其他空闲/忙碌线程窃取任务 (Stealing) 的过程，并记录调度器状态日志。
      *   `Async Mutex 与 MPSC 协作式同步状态协调`：模拟 8 容量有界信道 (Bounded Channel) 消息收发与协程互斥锁的获取/释放。展示 MPSC 缓冲区满时的背压挂起、挂起任务入队 Mutex 等待链表 (Wait Queue) 以及释放锁时 Waker 自动唤醒并重新调度的过程。
    *   **Wasmtime JIT 编译流水线与硬件页保护边界双仿真沙盒** 支持：
      *   `JIT 编译流水线与 Regalloc2 寄存器分配`：可视化展示 Wasm 字节码单遍扫描翻译为 Cranelift SSA CLIF IR，仿真生存期分析（Liveness Intervals）以及 `Regalloc2` 寄存器分配算法将虚拟 SSA 变量映射到物理 CPU 寄存器的过程，并生成本地 x86-64 汇编。
      *   `虚拟内存沙箱与 Page Fault 异常拦截`：模拟 4GB 静态线性内存与 2GB Guard Pages 保护区。交互式读写内存物理地址，展示硬件 MMU 级页保护越界拦截，演练 SIGSEGV 硬件信号如何通过 Wasmtime 信号处理器（Signal Handler）安全转化为虚拟机 Trap 异常，演示零软件边界检查开销的安全原理。
    *   **Cilium eBPF 极速收发包与 sockops 协议栈短路双仿真沙盒** 支持：
      *   `XDP/tc-bpf 网络数据路径与 NAT 路由`：模拟网络数据包在 Linux 内核中的流动，交互展示 XDP 驱动程序（DROP/PASS/REDIRECT 过滤）和 tc-bpf Ingress 钩子上的解析过程，演示无锁 Conntrack 状态表查找、NAT 字段重写与校验和重算。
      *   `sockops 套接字层短路与 Maglev 一致性哈希`：交互展现本地容器通信时通过 sockops 事件拦截机制将套接字写入 SOCKMAP，使用 BPF 消息重定向跳过整个 TCP/IP 协议栈进行内存级快速拷贝的流程。同时提供 Maglev 一致性哈希查表模拟，高亮显示后端节点变更时会话保持的低抖动状态。
    *   **TiKV 协处理器下推与 Multi-Raft/Percolator 状态机双仿真沙盒** 支持：
      *   `协处理器下推与 LSM-tree 存储压实`：交互式展示 Coprocessor 算子下推状态切换对网络带宽和计算负载的影响；模拟 RocksDB 内存 MemTable 数据 Flushing 溢出为 Level 0 SSTable，以及多层 LSM-tree 结构后台归并排序（Compaction）合并和墓碑删除过程。
      *   `Multi-Raft 分区动态分裂与 Percolator 2PC 事务分布式协调`：演练 Region 1 物理数据过载时向 PD 汇报并分裂为 1a 和 1b 的多 Raft 组变化；提供 Percolator 强一致性两阶段提交（Prewrite ➜ Commit）单步调试，展示 Default/Lock/Write 三大列族在物理锁建立、主元锁指针映射以及异步清扫过程中的底层状态演进。
    *   **Maelstrom 网络混沌模拟与因果/线性化时序双仿真沙盒** 支持：
      *   `网络混沌模拟器`：5 节点集群拓扑，支持用户手动分区、丢包或注入延迟，演练网络分区/恢复时 RPC 重试与日志落后收敛过程。
      *   `因果与线性化一致性判定`：图形化展示事件在各节点上的发生顺序，对比因果一致性（允许并发无序读）与强一致性（物理时间线性化屏障），并高亮展示 stale read 等 Jepsen 异常。
    *   **DuckDB 向量化计算与 Epoch MVCC 双仿真沙盒** 支持：
      *   `向量化计算引擎模拟`：对比行式 Volcano 模型逐行拉取虚函数开销与列式向量化批处理装载，直观演示 CPU 缓存命中率与 SIMD 寄存器并行加速。
      *   `Epoch MVCC 与列式压实`：交互式录入数据行并指定写入 Epoch，单步提交事务，演示逻辑删除以及触发物理 Compaction 时将脏内存段进行 Bit-Packing 压缩并归置到只读 Column Block 的生命周期。
    *   **Sled 无锁 B-Link 树与 Delta 日志段双仿真沙盒** 支持：
      *   `B-Link Tree 无锁分裂与右链接跳转`：交互式向叶子页 Leaf A 写入数据，触发物理分裂生成 Leaf B，演示读取线程在父节点路由尚未更新时，顺着右链接指针跨越至 Leaf B 无锁检索的路径。
      *   `日志结构内存 Delta 链与 GC 整理`：展示 Page Table 映射内存 Delta 链结构，追加写更新 Delta 记录，演示执行 Consolidation 时将链表变动归并压实为扁平 Base Page，以及触发物理 Segment GC 时执行活跃页面搬迁整理的日志段重构。
    *   **Seastar Shard-per-Core 与异步链式调度沙盒** 支持：
      *   `Shard-per-Core 亲和绑定与跨核 SPSC 消息队列模拟`：模拟 4 CPU 物理核心运行事件循环，并支持在 Core 0 触发本地任务，演示利用 CAS 机制将跨核任务递交至 Core 2 Egress SPSC 队列的无锁调度流。
      *   `异步非阻塞 Future/Promise 链式执行`：单步执行 Promise 解决时，展示网络包如何依次在 TCP 读、HTTP 解析、服务查询、JSON 序列化、TCP 写等 Continiuation 链条中流动，实现 Reactor 线程的零挂起空闲。
    *   **dae eBPF 内核重定向与 TProxy/DNS 劫持沙盒** 支持：
      *   `eBPF TC 钩子与 SOCKMAP 物理内存重定向`：对比普通 TCP/IP 协议栈（35µs 延迟, 18% CPU 软中断）与 BPF msg_redirect 短路（2µs 延迟, 0.5% CPU 软中断）下包传递的延迟和资源抖动。
      *   `TProxy 透明代理与 DNS 劫持分流`：模拟 53 端口 DNS 请求拦截，就地返回 Fake IP 并更新 IP Set 缓存；发起 TCP 连接时利用 ip rule 路由重定向到本地 TProxy 端口，并通过 Fake IP 反解域名建立 VLESS 隧道。
    *   **Dapr 边车消息拦截与分布式 Actor 哈希调度沙盒** 支持：
      *   `边车 gRPC 调用拦截与可插拔发布订阅`：模拟微服务调用 Dapr Sidecar 本地 gRPC 端口，Dapr 自动执行 CloudEvents 标准协议包装，并分发路由至可插拔中间件（Redis/Kafka/RabbitMQ）的流程。
      *   `分布式 Actor 状态一致性与 Placement 一致性哈希`：模拟一致性哈希环计算 Actor ID 并路由调度至目标宿主 Dapr 实例；演示同一个 Actor 实例在 Dapr 节点内的 Turn-based 单线程请求排队与状态自动存取一致性。
*   **ROS 2 DDS 共享内存与 Socket 通信延迟对比沙盒** 支持：
      *   `共享内存零拷贝机制模拟`：支持调节传输的数据载荷大小（10KB 到 10MB）。动态运行延迟评测，展示 Socket 传输在数据量增大时延迟呈线性增长（由于用户态/内核态的多次数据拷贝与网络协议栈打包开销），而共享内存（SHM）的延迟始终保持在极低的水平（扁平线，因为仅做描述符传递，数据物理零拷贝）。
      *   `DDS QoS 质量策略匹配`：支持为发布端与订阅端配置 Reliability（Reliable/BestEffort）与 Durability（Volatile/TransientLocal）。单步运行演示，当 QoS 冲突时（如发布端为 BestEffort，订阅端要求 Reliable）将无法建立通信连接，并在面板上高亮 QoS 冲突报警。
    *   **Rust Typestate 编译期状态转移与自引用固定双仿真沙盒** 支持：
      *   `Typestate 编译期错误检测仿真`：模拟 Connection 状态机。单步执行 connect() 和 complete_handshake()。若试图在 Closed 状态下执行 send() 动作，编译器错误展示面板会以红色高亮 `Connection<Closed> has no method 'send'` 编译期拦截，模拟强类型状态机约束安全。
      *   `自引用结构 Move 导致的悬空指针与 Pinning 修复`：支持分配含有自引用指针的结构体并让其在堆/栈间发生 Move（例如 `std::mem::swap`）。展示在未钉住（Unpinned）状态下 Move 会导致内部指针指向错位（指向原销毁物理栈页）产生 Undefined Behavior，而开启 Pinning 后数据物理地址被强力锁定，Move 操作直接被编译器拦截，保护指针安全。
*   **微服务网关熔断舱壁与双写迁移仿真实验室沙盒** 支持：
      *   `微服务线程池舱壁隔离与熔断三态演进`：支持开启/关闭 Bulkhead 和 Circuit Breaker，动态调节下游 Payment 接口的响应延迟；展示在无保护下 Payment 延迟拉长导致 Shared 全局线程池耗尽、进而拖垮 Order 服务的雪崩全景，以及开启隔离与熔断后，Payment 触发 OPEN 状态快速失败（Fast-Fail）返回 Fallback 降级数据的自愈轨迹。
      *   `零停机双写数据库迁移五阶段演练`：允许在 Legacy DB 和 New DB 之间逐步切换迁移状态（Legacy 单写 ➜ 开启双写 ➜ 运行 Historical 对账同步 ➜ 切换 New 主库 ➜ 彻底切断 Legacy 旧库），动态展现冲突解决机制及数据同步率（Data Parity）收敛进程。
    *   **rustc (Rust 编译器核心与借用检查) 双仿真沙盒** 支持：
      *   `MIR 借用检查与变量生存期生命周期`：可视化展示 MIR 基本块与变量声明，交互式模拟借用检查器（Borrow Checker）识别生命周期越界错误与可变/不可变借用并发冲突的编译期拦截过程。
      *   `Polonius 引擎事实提取与 Datalog 规则求解`：模拟 Polonius 中端静态分析阶段，流程化提取 Borrow Facts 并在控制台展现借用生存期关系求解结果。
    *   **bpftrace (动态追踪 DSL 与 eBPF Maps) 双仿真沙盒** 支持：
      *   `bpftrace 脚本 AST 解析与 LLVM IR 降级`：用户可输入 bpftrace 脚本，分步展现 lexer/parser 生成抽象语法树（AST）的过程，并实时生成降级翻译后的 LLVM IR 伪代码。
      *   `eBPF Map 共享内存与内核探针触发直方图`：模拟内核探针挂载触发与 Map 数据更新，在用户态定时抓取 Map 数据并实时渲染微秒级 biolatency 直方图。
    *   **Rocket Chip (RISC-V 参数化 SoC 生成器) 流水线与 TileLink 双仿真沙盒** 支持：
      *   `经典五级流水线旁路分支前推与 Load-Use 冲突`：单步执行 IF/ID/EX/MEM/WB 流水线，直观演示 Load-Use 数据冲突时流水线自动插入气泡，以及通过旁路前推（Bypass Forwarding）消除气泡的控制流。
      *   `TileLink 总线五通道与 MSI 一致性变迁广播`：在图形界面上模拟 CPU 写操作，演示 TileLink 在 A-E 通道上的协议请求与 Probe 广播，并驱动 Cache 状态在 Modified (M)、Shared (S)、Invalid (I) 之间自动迁移动画。
    *   **Rustls (内存安全 TLS 1.3 协议栈) 握手与时序侧信道双仿真沙盒** 支持：
      *   `TLS 1.3 握手状态机与 HKDF 密钥派生`：动态演练 TLS 1.3 握手包交互流，支持配置 Session Resumption (PSK)、双向认证 (mTLS) 与后量子加密 (PQC) 扩展，展示连接状态机迁移及 HKDF 各阶段派生主秘密的计算。
      *   `时序侧信道攻击与 subtle 常数时间比对防御`：对比常规短路逻辑字节串比对与 subtle 常数时间比对在攻击模拟下的耗时差异，通过直观的柱状图展现短路逻辑如何泄露匹配字符长度，以及常数时间比对如何消除时序特征。
    *   **daily_diagnostics (线上故障定位与性能榨汁机) 双仿真沙盒** 支持：
      *   CPU 火焰图与线程调度冲突模拟：支持动态配置线程池大小（1-64）与工作队列容量，度量自愿/非自愿上下文切换频率与延迟损耗；直观展示 CPU 火焰图的堆栈层级宽度，并动态模拟高并发下因线程溢出 CPU 核心上限所导致的 Context Switch 浪费。
      *   MySQL 索引跳转寻址与事务区间死锁演练：模拟 B+ 树主键索引与二级索引的跳转查表路径，直观对比全表扫描与索引覆盖在物理磁盘 IO 读取上的跳数差异；交互式演练多事务交错持有与请求 Exclusive/Shared 锁时的 Gap/Next-Key 区间冲突，重现 InnoDB 死锁检测器的检测及事务自动回滚机制。
    *   **devops_firefighting (云原生与 SRE 生产救火手册) 双仿真沙盒** 支持：
      *   `Kubernetes Pod 副本调度与故障漂移机制`：模拟物理节点宕机或 CPU/Memory 压力阈值触发，演示 Pod 状态在 `Pending ➜ Terminating ➜ Running` 之间的动态调度流转；高亮 Readiness/Liveness 探针检测失败后，Endpoints 控制器从 Service 负载均衡中平滑摘除流量的过程。
      *   `CoreDNS 延迟与 ndots 外部域名解析优化`：模拟不同 `ndots` 配置（ndots: 5 vs ndots: 1），在大并发外部域名解析请求下，分步演算 DNS search 搜索路径在 `.default.svc.cluster.local`、`.svc.cluster.local` 等后缀间的冗余轮询；引入网络丢包丢包延迟，对比超时重试对 CoreDNS 吞吐的影响。
    *   **tech_selection (高频架构设计折中矩阵) 双仿真沙盒** 支持：
      *   `一致性哈希环负载漂移与虚拟节点调节`：支持动态增删物理存储节点，沿环顺时针演练 Key 节点的分配流向；提供 1x/3x/10x 虚拟节点倍率切换，并实时渲染柱状图以对比负载标准差（Standard Deviation）的波动，以及计算物理节点变动时受影响 Key 漂移率。
      *   `消息队列 Ingress 缓冲与消费者 Lag 对比`：对比推（Push）模式下的 RabbitMQ 物理流量控制（背压阻断，向生产端反向传播 TCP 阻塞）与拉（Pull）模式下的 Kafka 顺序分区堆积（消费延迟积压，超限时触发驱逐丢包）的运行时吞吐，直观演绎削峰填谷效果。



### 🌐 统一门户导航主站 (Master Landing Page Website)
针对全套 57 个硬核系统工程专题以及 20 个离线交互沙盒，构建并上线了高性能、免配置、自适应的统一门户导航网站。

*   **视觉艺术与色彩体系 (Morandi Dark)**：采用低饱和度的莫兰迪暗色系（石板蓝、苔绿、梅玫瑰、陶土红、靛青、古董金等）作为模块标识，结合微晶磨砂玻璃（Glassmorphism）质感，构建兼具极客深度与现代质感的页面层级。
*   **100% 离线单文件运行 (Self-Contained Offline)**：所有 57 个项目的多维双语元数据均直接以静态 JSON 格式嵌入在 `index.html` 的 script 标签内，无需搭建本地 Web 服务器，双击即可完全在离线状态下运行全部检索、分类过滤以及侧滑详情弹窗功能。
*   **多维搜索与瞬时模糊匹配 (Search & Filter)**：
    *   **瞬时过滤**：支持通过 11 个核心分类侧边栏进行秒级按分类筛选，并配备了“全部”、“沙盒已上线（20个）”与“实体书规划（37个）”状态过滤按钮。
    *   **模糊搜索**：内置模糊检索逻辑，检索字段无缝覆盖英文缩写、中文译名、攻坚重点 (Focus)、L0 概念公理以及具体底层符号，并在输入时实时展示过滤状态。
*   **双语极简切换 (Bilingual Toggle)**：设计了一键中/英文 UI 及数据切换机制，系统内嵌双语文本翻译字典，一键瞬间翻译标题、副标题、本质陈述与交互组件。
*   **侧滑抽屉详情与沙盒内嵌预览 (Details Drawer & Iframe)**：点击卡片侧滑呼出详情面板，显示该专题的 L0 本质与攻坚大纲：
    *   对于已发布上线沙盒的 20 个项目，直接在面板内以 iframe 形式内嵌加载该项目的**“离线演练双沙盒”**，用户可以直接在页面内进行交互操作，或一键“独立运行沙盒”、“打开关系知识图谱”、“下载海报 PDF”、“阅读卡片原文”；
    *   对于未发布的 37 个规划中项目，展示专属说明，指示用户其完整剖析集成在《硬核技术系统工程》纸质版中。
*   **Google 长尾词 SEO 伏击 (Schema.org JSON-LD)**：在 HTML 首部静态注入了符合 schema.org 标准的结构化数据，详细将 57 个系统工程的名称、中英副标题、技术本质公理、长尾关键字及伪路由地址转化为 Google 搜索爬虫可直接解析的元数据，抢占全球系统设计与技术选型的长尾流量。

#### 📦 交付物清单与物理路径
1.  **统一导航主页面**：[index.html](file:///D:/材料/拆书产出/硬核技术/index.html) (174 KB)
2.  **富媒体二元元数据**：[projects_metadata_enriched.json](file:///D:/材料/拆书产出/硬核技术/projects_metadata_enriched.json) (75 KB)
3.  **原始元数据归档**：[projects_metadata.json](file:///D:/材料/拆书产出/硬核技术/projects_metadata.json) (27 KB)
4.  **构建脚本**：[generate_index_html.py](file:///C:/Users/35893/.gemini/antigravity/brain/a056788c-0cd7-4802-9cb7-efde2abaf161/scratch/generate_index_html.py)

#### 🔍 验证结果汇总 (Verification Results)
*   **双语切换功能验证**：在中/英按钮点击后，UI 标签、按钮、57 个卡片的标题、副标题、本质与功能全部动态瞬时翻译，符合预期。
*   **模糊检索与分类重组**：输入 `raft`, `ebpf`, `compaction` 或选择“01-分布式系统”等分类，卡片网格能够以 200ms 内的极速响应实现动态排列重组，无语法报错或 DOM 溢出。
*   **资源解析与跳转链接**：点击已发布的 20 个项目（如 `duckdb`, `tikv`, `tokio` 等），沙盒 iframe 均能成功渲染并响应参数滑块变动；PDF 下载、知识图谱打开以及 MD 文本阅读均能正确映射至各项目相对目录下的物理交付文件，链接有效率为 100%。
