# 《sqlite-internals》高密度卡片系统设计大图

本设计大图为《sqlite-internals》（SQLite 存储引擎内核实现）的嵌入式数据库核心架构与系统设计高密度拆解卡片设计指南。我们将 28 张核心速查卡片划分为六大核心模块，每个模块采用低饱和度的莫兰迪（Morandi）色彩进行视觉归类，并设计了其拓扑交互图与物理源头锚点。

---

## 🎨 莫兰迪内核诊断视觉配色方案 (Morandi Color System)

为保证排版的高级感与学术硬核感，采用低饱和度、高质感的莫兰迪色彩体系：

| 模块编码 | 模块名称 | 莫兰迪色系 | 浅色底色 (Light Mode) | 深色边框 / 文字 (Dark Mode) | 对应设计领域 |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **M1** | SQL 编译与分析树 | 石板蓝 (Slate Blue) | `#F0F3F5` / `#D2DBE0` | `#4E5D6C` / `#2F3C47` | Lemon 词法语法解析、查询打平与重写、Prepared 语句缓存复用 |
| **M2** | VDBE 虚拟机与字节码 | 苔绿 (Moss Green) | `#F2F4F0` / `#D5DDD1` | `#5F6C5B` / `#3A4438` | VDBE 寄存器虚拟机、巨型解释循环、游标寻址、Explain 执行计划分析 |
| **M3** | B-Tree 与 Pager 页面引擎 | 梅玫瑰 (Plum Rose) | `#F5F0F2` / `#E0D2D7` | `#6F525A` / `#4A353A` | 叶子与内部节点结构、变长单元槽页布局、PCache 页面缓存、真空整理 |
| **M4** | 回滚日志机制 | 陶土红 (Terracotta) | `#F5F1EF` / `#E0D3CD` | `#793C2C` / `#522114` | Rollback Journal 结构、SHARED $\rightarrow$ EXCLUSIVE 锁升级、崩坏事务回滚恢复 |
| **M5** | WAL 模式与并发控制 | 靛青 (Indigo) | `#F0F2F5` / `#D1D8E0` | `#3E4C5B` / `#232F3C` | WAL 日志格式、`-shm` 共享内存哈希索引、读写非阻塞并发、脏页回源 |
| **M6** | OS 抽象层与 VFS 锁 | 古董金 (Antique Gold) | `#F6F4EE` / `#E3DEC8` | `#8C7344` / `#5C4A28` | VFS 操作系统抽象、POSIX/Win32 文件锁映射、mmap 零拷贝、页面加密 SEE |

---

## 🗺️ 28张高密速查卡片大图拓扑 (Card Topology)

```mermaid
graph TD
    subgraph M1_Compile ["M1: SQL 编译与分析树 (Slate Blue)"]
        C01["Card 01: Tokenizer 与 Lemon 语法解析器"]
        C02["Card 02: 代码生成器与查询重写优化"]
        C03["Card 03: 索引选择与代价评估模型"]
        C04["Card 04: Prepared Statement 缓存机制"]
    end

    subgraph M2_VM ["M2: VDBE 虚拟机与字节码 (Moss Green)"]
        C05["Card 05: VDBE 核心架构与操作码循环"]
        C06["Card 06: VDBE 寄存器类型与动态存储"]
        C07["Card 07: 游标寻址与 B-Tree 交互操作"]
        C08["Card 08: 虚拟机子程序与协程调度"]
        C09["Card 09: VDBE 字节码诊断与计划分析"]
    end

    subgraph M3_BTree ["M3: B-Tree 与 Pager 页面引擎 (Plum Rose)"]
        C10["Card 10: B-Tree 物理页面布局"]
        C11["Card 11: 溢出页链表组织"]
        C12["Card 12: Pager 逻辑层与页面缓存管理"]
        C13["Card 13: Free-list 空闲页链表与 VACUUM 整理"]
    end

    subgraph M4_Journal ["M4: 回滚日志机制 (Terracotta)"]
        C14["Card 14: -journal 回滚日志物理结构"]
        C15["Card 15: 回滚模式事务提交与崩溃恢复"]
        C16["Card 16: 回滚日志状态的锁变迁"]
        C17["Card 17: 独占写冲突与多进程锁死"]
    end

    subgraph M5_WAL ["M5: WAL 模式与并发控制 (Indigo)"]
        C18["Card 18: -wal 日志文件物理结构"]
        C19["Card 19: -shm 共享内存索引文件"]
        C20["Card 20: WAL 读写并发非阻塞控制"]
        C21["Card 21: Checkpoint 定期脏页回源机制"]
        C22["Card 22: WAL 物理写屏障与 fsync 规则"]
        C23["Card 23: 追尾避免与 WAL 重写重置"]
    end

    subgraph M6_VFS ["M6: OS 抽象层与 VFS 锁 (Antique Gold)"]
        C24["Card 24: VFS 虚拟文件系统抽象层"]
        C25["Card 25: Windows 与 Unix 锁定原语物理映射"]
        C26["Card 26: 多线程与多进程锁隔离屏障"]
        C27["Card 27: mmap 内存映射物理 I/O"]
        C28["Card 28: 加密扩展与物理页安全性加密"]
    end

    M1_Compile -->|生成 Prepared 字节码| M2_VM
    M2_VM -->|通过游标执行底层寻址操作| M3_BTree
    M3_BTree -->|向 Pager 请求物理页映射| M4_Journal
    M4_Journal -->|旧版单写排他锁机制保护| M6_VFS
    M3_BTree -->|WAL 模式追加页记录与缓存| M5_WAL
    M5_WAL -->|使用共享内存索引加速| M2_VM
    M5_WAL -->|Checkpoint 回源主库| M3_BTree
    M6_VFS -->|封装 OS 文件锁与 mmap I/O| M3_BTree
```

---

## ⚡ 物理代码与规范源头锚点 (Physical Source Anchors)

本设计大图与 SQLite 官方源码仓库的物理代码路径映射如下：
1. **Lemon 语法解析与编译**：映射 `src/tokenize.c` 和 `src/parse.y`。Lemon 解析器不包含全局变量，保证了线程安全性；生成的语法分析树在 `src/select.c` 中进行重写拍平优化（Query Flattening）。
2. **VDBE 字节码分发解释循环**：映射 `src/vdbe.c` 中的 `sqlite3VdbeExec` 函数。内部由一个数万行的巨大 switch-case 分发器组成，利用 `Opcode` 对寄存器及内存进行指针计算与游标控制。
3. **B-Tree 页面分裂与变长单元布局**：映射 `src/btree.c` 和 `src/btreeInt.h` 中的 `MemPage` / `CellInfo` 结构体。观察溢出页在 payload 超过页容量时的分配逻辑 `sqlite3BtreeInsert`。
4. **Pager 缓存分配与 Rollback Journal**：映射 `src/pager.c`。跟踪 Pager 状态转换（SHARED $\rightarrow$ RESERVED $\rightarrow$ EXCLUSIVE）以及日志物理写入、回滚恢复逻辑。
5. **WAL 日志记录与共享内存哈希索引**：映射 `src/wal.c`。关注共享内存映射 `walIndexPage` 以及基于哈希的 WAL 快速页号定位，以及脏页 Checkpoint 回写逻辑。
6. **VFS 物理锁定与 OS API 转化**：映射 `src/os_unix.c` (利用 POSIX `fcntl` 字节范围锁映射 SHARED/PENDING/EXCLUSIVE 锁状态) 与 `src/os_win.c` (使用 Windows API `LockFileEx` 实现同等锁机制)。
