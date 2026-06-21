# Crafting Interpreters 高密度卡片系统设计大图

## 1. 28张卡片依赖拓扑关系图

```mermaid
graph TD
    %% M1: jlox Front-End & Interpreter
    Card1["Card 1: Scanner 词法状态转换"] --> Card2["Card 2: Parser 递归下降解析"]
    Card2 --> Card3["Card 3: AST 节点与 Visitor 模式"]
    Card3 --> Card4["Card 4: Environment 作用域链"]
    Card4 --> Card5["Card 5: Tree-Walk 解释器性能瓶颈"]

    %% M2: clox VM Core
    Card6["Card 6: Opcode 设计与 Chunk 存储"] --> Card7["Card 7: Stack-based VM 译码循环"]
    Card7 --> Card8["Card 8: NaN Tagging 内存表示"]
    Card7 --> Card9["Card 9: Disassembler 打印栈回溯"]

    %% M3: Pratt Parsing Compiler
    Card10["Card 10: Pratt Parsing 绑定优先级"] --> Card11["Card 11: 缀前与缀中规则函数指针"]
    Card10 --> Card12["Card 12: parsePrecedence 优先级控制"]
    Card11 --> Card13["Card 13: 1 + 2 * 3 优先级编译流"]
    Card12 --> Card13
    Card13 --> Card6

    %% M4: Hash Tables & Interning
    Card15["Card 15: 开放寻址与线性探测哈希表"] --> Card16["Card 16: Tombstone 墓碑软删除防断链"]
    Card15 --> Card17["Card 17: String Interning 字符串单例驻留"]
    Card14["Card 14: Obj 堆头结构与类型标签"] --> Card17

    %% M5: Control Flow & Closures
    Card18["Card 18: OP_JUMP 分支回填技术"] --> Card7
    Card19["Card 19: Upvalue 闭包静态作用域捕捉"] --> Card20["Card 20: Open/Closed Upvalue 栈到堆迁移"]
    Card20 --> Card7
    Card21["Card 21: OOP 类与 BoundMethod 绑定"] --> Card22["Card 22: super 继承与超类方法静态绑定"]

    %% M6: Garbage Collection
    Card23["Card 23: allocated 翻倍公式触发 GC"] --> Card24["Card 24: 三色标记活死对象划分"]
    Card24 --> Card25["Card 25: Gray Stack 灰色栈维护与溢出逃逸"]
    Card25 --> Card26["Card 26: Tracing References 遍历黑化"]
    Card26 --> Card27["Card 27: tableRemoveWhite 驻留表弱引用过滤"]
    Card27 --> Card28["Card 28: Sweep 清扫白色释放物理内存"]
```

---

## 2. Crafting Interpreters 物理源码位置映射锚点

为便于硬核技术速查，以下是 28 张核心卡片对应在《Crafting Interpreters》官方开源仓库 `munificent/craftinginterpreters` 中的核心源码文件及函数位置：

*   **jlox 解释器前端与运行期 (M1)**:
    *   词法扫描状态机：`lox/Scanner.java` -> `scanToken()`
    *   递归下降解析：`lox/Parser.java` -> `expression()`, `equality()`
    *   AST 动态定义与 Visitor：`lox/Expr.java` & `tool/GenerateAst.java`
    *   变量环境作用域：`lox/Environment.java` -> `define()`, `get()`
    *   Tree-Walk 树遍历求值：`lox/Interpreter.java` -> `visitBinaryExpr()`, `visitLiteralExpr()`
*   **clox 字节码虚拟机基础 (M2)**:
    *   OPCODE 设计与 Chunk 组装：`src/chunk.c` -> `initChunk()`, `writeChunk()`
    *   Stack VM 核心译码循环：`src/vm.c` -> `run()` -> `switch (instruction = READ_BYTE())`
    *   Value 运行时值及 NaN Tagging 优化：`src/value.h` -> `Value` (带 `NAN_MASK` 的位操作)
    *   反汇编器与栈打印：`src/debug.c` -> `disassembleInstruction()`
*   **Pratt Parsing 普拉特优先级编译器 (M3)**:
    *   Pratt 解析绑定优先级表：`src/compiler.c` -> `rules` 数组 (包含前缀/缀中函数指针及优先级)
    *   缀前与缀中编译流分发：`src/compiler.c` -> `number()`, `binary()`, `unary()`
    *   优先级解析自愈驱动：`src/compiler.c` -> `parsePrecedence()`
*   **哈希表与字符串驻留 (M4)**:
    *   Obj 堆头基类定义：`src/object.h` -> `struct Obj` (带 `type` 标签与 `next` 堆链表指针)
    *   开放寻址线性探测实现：`src/table.c` -> `findEntry()`
    *   Tombstone 墓碑软删除逻辑：`src/table.c` -> `tableDelete()` (标记为 `val=BOOL_VAL(true)` 表示墓碑)
    *   字符串驻留去重 Interning：`src/object.c` -> `tableFindString()`, `allocateString()`
*   **控制流与闭包机制 (M5)**:
    *   Backpatching 分支回填代码：`src/compiler.c` -> `emitJump()`, `patchJump()`
    *   Upvalue 闭包上值编译结构：`src/object.h` -> `ObjClosure`, `ObjUpvalue`
    *   Open 转向 Closed Upvalue 栈迁移：`src/vm.c` -> `captureUpvalue()`, `closeUpvalues()`
    *   面向对象实例字段与 BoundMethod：`src/vm.c` -> `bindMethod()`, `callValue()`
    *   Inheritance 继承超类绑定：`src/compiler.c` -> `super_()`
*   **三色标记垃圾回收 GC (M6)**:
    *   根扫描与 GC 触发阈值计算：`src/memory.c` -> `collectGarbage()`, `markRoots()`
    *   三色标记与着色逻辑：`src/memory.c` -> `markObject()`, `markValue()`
    *   工作队列灰色黑化推导：`src/memory.c` -> `blackenObject()`
    *   Intern 驻留表白色清除：`src/table.c` -> `tableRemoveWhite()`
    *   Sweep 物理清扫回收堆内存：`src/memory.c` -> `sweep()`
