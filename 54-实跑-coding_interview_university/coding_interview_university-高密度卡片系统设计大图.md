# coding_interview_university-高密度卡片系统设计大图.md

本文件定义了 **coding-interview-university (计算机科学体系与底层工程素养)** 28张核心知识卡片之间的依赖拓扑结构，以及物理代码映射锚点。

---

## 🗺️ 28 张卡片依赖拓扑图 (Mermaid)

```mermaid
graph TD
    classDef default fill:#151d30,stroke:#24324f,color:#e2e8f0;
    classDef M1 fill:#4B5F7A,stroke:#2f3d52,color:white;
    classDef M2 fill:#6B8272,stroke:#4b5c50,color:white;
    classDef M3 fill:#9C6666,stroke:#704949,color:white;
    classDef M4 fill:#7A7A7A,stroke:#595959,color:white;
    classDef M5 fill:#9A825A,stroke:#6e5c40,color:white;
    classDef M6 fill:#755B77,stroke:#534054,color:white;

    Card1["Card 1: Dynamic Array"]:::M1
    Card2["Card 2: Linked Lists"]:::M1
    Card3["Card 3: Stack & Queue"]:::M1
    Card4["Card 4: Hash Table Collide"]:::M1
    Card5["Card 5: Rehash Expansion"]:::M1
    
    Card6["Card 6: BST Degradation"]:::M2
    Card7["Card 7: AVL & RBT Rotations"]:::M2
    Card8["Card 8: Heaps & Priority"]:::M2
    Card9["Card 9: Prefix Trie"]:::M2
    Card10["Card 10: Graph Matrix/List"]:::M2

    Card11["Card 11: BFS & DFS Traversal"]:::M3
    Card12["Card 12: Merge & Quick Sort"]:::M3
    Card13["Card 13: Heap Sort In-place"]:::M3
    Card14["Card 14: Binary Search Mid"]:::M3
    Card15["Card 15: Bit Manipulation"]:::M3

    Card16["Card 16: Recursion & Stack"]:::M4
    Card17["Card 17: DP Memory & Table"]:::M4
    Card18["Card 18: Greedy Local Opt"]:::M4
    Card19["Card 19: Divide & Conquer"]:::M4

    Card20["Card 20: CPU Cache Lines"]:::M5
    Card21["Card 21: VM & TLB Page Walk"]:::M5
    Card22["Card 22: Context Switch OS"]:::M5
    Card23["Card 23: TCP Socket State"]:::M5
    Card24["Card 24: Big-O Master Theorem"]:::M5

    Card25["Card 25: Design Patterns Core"]:::M6
    Card26["Card 26: TDD Red-Green-Refactor"]:::M6
    Card27["Card 27: Clean Code & SOLID"]:::M6
    Card28["Card 28: System Design Scales"]:::M6

    Card1 --> Card3
    Card1 --> Card4
    Card4 --> Card5
    Card2 --> Card3
    Card6 --> Card7
    Card6 --> Card8
    Card6 --> Card9
    Card10 --> Card11
    Card11 --> Card12
    Card12 --> Card13
    Card12 --> Card14
    Card14 --> Card24
    Card15 --> Card16
    Card16 --> Card17
    Card17 --> Card18
    Card19 --> Card12
    Card20 --> Card21
    Card21 --> Card22
    Card22 --> Card23
    Card24 --> Card28
    Card25 --> Card27
    Card26 --> Card27
    Card27 --> Card28
```

---

## 📍 Coding Interview University 物理映射锚点

本设计大图的知识节点与计算机系统级、软件工程规范进行底层物理映射：
1. **CPU Cache Line**: x86-64 及 ARM 架构中一致的 64 字节数据块对齐访存（如 C++ 中的 `alignas(64)`）。
2. **VM & TLB Page Walk**: 处理器 MMU 底层页表翻译逻辑（如 Linux 系统中对 `/proc/sys/vm/nr_hugepages` 的调优）。
3. **TCP Socket State**: 操作系统 TCP/IP 协议栈状态转移，与 Linux 的 `netstat` / `ss` 状态直接关联。
4. **Master Theorem**: 算法渐进复杂度中递归分治表达式 $T(N) = aT(N/b) + f(N)$ 的数学闭式解。
