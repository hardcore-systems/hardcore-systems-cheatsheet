# Step 1: 题材判定与卡片划分 - munificent / craftinginterpreters (手写编译器与虚拟机)

## 1. 题材判定
*   **核心内幕**: 自研解释器与字节码虚拟机的核心底层。第一阶段聚焦于词法分析（Scanner）、递归下降语法分析（Parser）与 AST Tree-Walk 解释器执行（jlox）；第二阶段探讨字节码 Chunk 组织、Pratt Parsing 普拉特优先级表达式编译、栈式 VM 译码循环指令集、开放与闭合上值（Open vs Closed Upvalues）闭包机制、在线 OOP 类方法 BoundMethod 以及三色标记垃圾回收（Tri-color Marking GC）物理清扫。
*   **设计基调**: 运用高雅的莫兰迪灰蓝与茶红、古金调色盘（groupM1~groupM6），体现编译器系统的精巧设计，排版紧密对称。
*   **目标版面**: 双页 A4 Landscape 横向海报（extarticle + multicol + tcolorbox），底部左侧为编译与执行折衷设计矩阵，底部右侧为 Zone T OPCODE 速查表与排障调试指令。

---

## 2. 核心卡片划分 (28 张核心卡片)

### 📂 M1: 词法分析、AST 语法树与树形解释器 (jlox) (Cards 1-5)
*   **Card 1. 词法扫描器（Scanner）与正则表达式转换规则**：Scanner 逐字符读取源码，利用条件分支判断操作符，根据保留字哈希表匹配标识符，将源字符流转换为 Token 列表。
*   **Card 2. 递归下降解析（Recursive Descent Parsing）与 LL(1) 冲突消除**：自顶向下递归解析表达式。通过对算符结合律和优先级的硬编码转换，解决左递归，规避回溯以满足 LL(1) 语法要求。
*   **Card 3. 抽象语法树（AST）节点设计与 Visitor 访问者模式**：定义统一表达式基类，自动生成 Binary, Unary, Grouping 等 AST 子类；植入 Visitor 接口的 `accept()`，实现算法与语法树结构的解耦。
*   **Card 4. 动态作用域环境（Environment）与变量查找机制**：Environment 内部以哈希表映射变量名与值。嵌套环境内部持有外层环境的 enclosing 指针，查找变量时沿着指针链向上追溯。
*   **Card 5. Tree-Walk Interpreter 树形解释的执行限制与性能瓶颈**：运行时遍历 AST 节点递归执行 Visitor 的 `visit()` 方法。因为每次求值都涉及动态分发及频繁的宿主语言内存分配，导致极高的运行时开销。

### 📂 M2: 字节码虚拟机 (clox) 与栈计算模型 (Cards 6-9)
*   **Card 6. 字节码指令集（OPCODE）设计与 Chunk 存储结构**：定义单字节操作码（Opcode）表示不同基本操作（如 OP_ADD）。将字节码流、常量池（ValueArray）与行号数组封装为 Chunk 进行紧凑存储。
*   **Card 7. 栈式虚拟机（Stack-based VM）计算模型与指令译码循环**：VM 运行时包含计算栈（stack）与指令指针（ip）。译码循环（Interpreter Loop）利用 `READ_BYTE()` 读取指令，通过巨大 switch 分发执行栈 Push/Pop 操作。
*   **Card 8. 虚拟机运行时值表示：C 语言 union 与 NaN Tagging 优化**：运行时基本值封装为 union 结构。高级优化采用 NaN Tagging，利用 IEEE 754 浮点数 NaN 状态下的未用空闲位包装指针和布尔值，将大小压缩至 8 字节。
*   **Card 9. 虚拟机底层调试：反汇编器（Disassembler）与打印栈回溯**：编译生成 Chunk 后，使用 `disassembleChunk()` 输出线性的汇编助记符；运行期每执行一步指令皆打印运行时计算栈状态以供排障。

### 📂 M3: 普拉特解析法 (Pratt Parsing) 表达式编译 (Cards 10-13)
*   **Card 10. 普拉特解析法（Pratt Parsing）核心逻辑与结合律解析**：Pratt 解析法是处理表达式的高效自顶向下算符优先分析法。规则表为每种 Token 赋予绑定优先级（Binding Power），循环消费高于当前优先级的后缀/缀中算符。
*   **Card 11. 缀中与缀前解析规则函数（Prefix & Infix Parse Functions）**：解析表为 Token 关联 prefix（处理前缀与字面量，如 `OP_NEGATE`）和 infix（处理缀中与后缀，如 `OP_ADD`）函数，指针映射实现流式递归解析。
*   **Card 12. 普拉特解析器优先级表与单运算符分支映射**：优先级分为赋值、逻辑、比较、加减、乘除、一元等。调用 `parsePrecedence(PREC_TERM)` 会自动推进解析，并编译出匹配优先级的最小子树字节码。
*   **Card 13. Pratt 优先级编译示例流：`1 + 2 * 3` 的语法重组过程**：解析 `1` 时调用前缀函数编译常量；遇到 `+` 触发缀中函数，因为 `PREC_FACTOR` 比 `PREC_TERM` 高，`2` 与 `* 3` 优先结合，编译产出 `1 2 3 * +` 逆波兰字节码。

### 📂 M4: 动态类型、哈希表与字符串 Interning (Cards 14-17)
*   **Card 14. 运行时堆内存管理与动态类型 Tag 编解码**：所有复杂对象（Obj）继承基类首部结构，内含垃圾回收标记与类型标签（如 ObjString, ObjClosure），通过基类指针强转类型实现安全编解码。
*   **Card 15. 自研哈希表设计：开放寻址与线性探测（Linear Probing）**：不采用拉链法，通过 `(hash + i) % capacity` 连续探测分配。当负载因子（Load Factor）超过 75% 时触发翻倍扩容并重新计算哈希映射。
*   **Card 16. 哈希表软删除：墓碑标记（Tombstone）的作用与判定**：删除哈希表键值对时，不直接清空，而是写入 Tombstone 标记。查找时跳过 Tombstone 继续探测，插入时则直接复用，避免探测断链。
*   **Card 17. 字符串驻留（String Interning）机制与快速等值判定**：VM 在全局哈希表中缓存在线分配的所有 ObjString。新创建字符串首先在此查表，确保全局唯一，由此将字符串等值比对缩减为 $O(1)$ 指针比对。

### 📂 M5: 控制流分支、闭包与 Upvalues 机制 (Cards 18-22)
*   **Card 18. 分支与循环字节码生成：回填技术（Backpatching）**：编译控制流如 `if` 时，先发射跳转指令（OP_JUMP）并预留两字节跳转偏移；编译完对应主体后计算跨越的字节偏移量，回填到预留位置。
*   **Card 19. 闭包设计：Upvalue 上值概念与静态作用域捕捉**：闭包引用外层嵌套函数的局部变量。编译器编译该变量时生成 Upvalue 索引并作为 ObjClosure 附件打包，以便后续读取。
*   **Card 20. 开放与闭合上值（Open vs Closed Upvalues）在 VM 栈中的状态转换**：被捕捉局部变量在 VM 栈上时称为 Open 状态，Upvalue 结构体指针直接指向栈地址；对应栈帧弹出时，VM 执行闭合（Close）操作，将栈中的值拷贝至 Upvalue 结构体自身内存中。
*   **Card 21. 面向对象编程：类定义、实例属性与方法分发机制**：`ObjClass` 包含方法哈希表。实例化时创建 `ObjInstance` 并关联类结构，内部维护实例字段哈希表；方法调用通过 BoundMethod 组装接收者。
*   **Card 22. 超类继承（Superclass Inheritance）与 Method Bindings**：派生类编译时拷贝超类方法表；编译 `super.method()` 时，在编译期静态解析超类位置，在运行期动态将超类的方法与当前 `this` 实例绑定。

### 📂 M6: 三色标记垃圾回收 (GC) 运行期算法 (Cards 23-28)
*   **Card 23. 根对象扫描（Marking Roots）与 GC 触发阈值公式**：GC 发生时，首先将活动虚拟机栈（Stack）、框架调用栈（CallFrames）、全局变量哈希表及当前编译器等标记为灰色根对象。触发公式为：当分配字节数超过 `nextGC = allocated * 2` 时。
*   **Card 24. 三色标记机制：白色、灰色与黑色对象的含义**：白色代表待扫描或未被引用的死对象；灰色代表已由根部推导引用、但尚未遍历其子引用的对象；黑色代表对象自身及其子引用均已扫描标记完毕的活对象。
*   **Card 25. 灰色对象工作队列（Gray Stack）维护与溢出防范**：GC 维护一个灰色指针栈。遍历过程中不断压入推导出的灰色对象。若因系统内存极度匮乏导致 Gray Stack 溢出，则将所有对象退回灰色并以堆链表全盘慢速补救。
*   **Card 26. 引用网推导扫描（Tracing References）与黑化过程**：当 Gray Stack 不为空时，GC 持续弹出灰色对象，将其染成黑色，并访问其所持有的子对象。发现白色子对象则立刻染灰并压入 Gray Stack，不断拓扑扩散。
*   **Card 27. 弱引用消除：字符串驻留哈希表（Intern Table）的 GC 回收**：全局 String Intern 表存储的指针不视为根对象引用以避免泄露。在 Mark 阶段结束后、Sweep 开始前，GC 遍历字符串表，主动清空残留的白色（未标记）字符串键值对。
*   **Card 28. 清扫阶段（Sweeping Phase）物理内存回收与堆链表整理**：GC 物理遍历整个堆对象链表。若对象标记为白色，则从链表摘除并调用 `freeObject()` 回收物理内存；若为黑色，则重置为白色并清除 GC 标记以备下一次 GC 轮询。

---

## 3. L0 ~ L2 知识阶梯

*   **L0 一句话本质**：语言编译器与虚拟机的本质是通过词法/语法分析（Pratt 解析）将高层代码转化为低级字节码序列（OPCODE），利用栈式计算模型与 HLC 类似的运行时帧栈驱动逻辑译码，辅以三色标记 GC 自动化控制周期内堆内存的周转。
*   **L1 四句话逻辑**：
    1.  **词法解析与树形建模**：利用 Scanner 自动状态转换生成 Token 流，基于 Visitor 模式构建抽象语法树（AST），提供面向行为解耦的动态树求值基底。
    2.  **字节码Chunk与指令译码**：通过 Pratt 解析算符优先级并将表达式重组编译为线性 Chunk 字节码，使用 IP 寄存器在 switch-loop 中高速求值。
    3.  **栈帧闭包与上值转换**：在 VM 计算栈上实现 CallFrame 函数调用，利用 Open 转向 Closed 的 Upvalue 结构体，打通静态闭包作用域与栈式生命周期的壁垒。
    4.  **三色拓扑扫描与白消清理**：由分配翻倍公式触发三色 GC，借助 Gray Stack 拓扑网罗并黑化活跃堆引用，弱引用消除后，通过 Sweep 阶段彻底清退白色死亡对象。
*   **L2 核心数据流转拓扑**：
    *   `Source String` ➜ `Scanner Token Stream` ➜ `Pratt Compile` ➜ `Bytecode Chunk` ➜ `VM Run ip++` ➜ `READ_INSTRUCTION` ➜ `Switch OPCODE` ➜ `Push/Pop Operand Stack` ➜ `Heap Allocation` ➜ `Triggers GC (allocated > nextGC)` ➜ `Mark Roots to Gray` ➜ `Pop Gray & Push Children` ➜ `Blacken active objects` ➜ `Sweep remaining white objects` ➜ `Free Memory`.
