# Go Runtime 高密度卡片系统设计大图

## 1. 28张卡片依赖拓扑关系图

```mermaid
graph TD
    %% M1: GMP Scheduler & Netpoller
    Card1["Card 1: G, M, P 核心结构体物理定义与分工"] --> Card2["Card 2: 调度主循环核心决策链 schedule"]
    Card2 --> Card3["Card 3: 任务工作窃取机制 Work Stealing"]
    Card1 --> Card4["Card 4: 网络轮询器 Netpoller epoll 汇流"]
    Card2 --> Card5["Card 5: Sysmon 监控线程与 SIGURG 信号抢占"]

    %% M2: Memory Allocator & Stack
    Card6["Card 6: 内存分配三层物理架构 mcache/mcentral/mheap"] --> Card7["Card 7: mspan 划分与 Size Classes 分配"]
    Card7 --> Card8["Card 8: 微小对象 Tiny Allocator 压缩分配"]
    Card6 --> Card9["Card 9: 逃逸分析 静态编译器逃逸场景判定"]
    Card9 --> Card10["Card 10: 协程物理栈伸缩 copystack 连续栈拷贝"]

    %% M3: Tricolor Concurrent GC
    Card11["Card 11: 三色标记状态机与 GC 四阶段状态转换"] --> Card12["Card 12: Dijkstra/Yuasa Go 混合写屏障"]
    Card6 --> Card13["Card 13: 用户辅助标记机制 GC Assist"]
    Card11 --> Card14["Card 14: 并发清理阶段 Sweep 惰性扫描"]
    Card11 --> Card15["Card 15: GC Pacer 自反馈水位线动态调整"]

    %% M4: Channels & Sync Primitives
    Card16["Card 16: Channel 底层 hchan 环形缓冲区控制"] --> Card17["Card 17: Channel 锁与 sudog 队列管理"]
    Card17 --> Card18["Card 18: Select 多路复用非阻塞随机洗牌 selectgo"]
    Card16 --> Card19["Card 19: Mutex 正常/自旋与饥饿模式判定模型"]
    Card19 --> Card20["Card 20: WaitGroup 与 Once 原子同步实现"]

    %% M5: Maps & Interfaces
    Card21["Card 21: Map 底层 hmap/bmap 内存排布"] --> Card22["Card 22: Map 渐进式翻倍与等量扩容槽位迁移"]
    Card23["Card 23: 空接口 eface 与非空接口 iface 区别 itab"] --> Card24["Card 24: 虚函数调用与去虚拟化 Devirtualization"]

    %% M6: Defer, Panic, Syscall & Cgo
    Card25["Card 25: Defer 三种编译路径 堆/栈/开放编码"] --> Card26["Card 26: Panic/Recover 嵌套链式处理 _panic"]
    Card27["Card 27: 系统调用 Syscall M/P 解绑与进入退出"] --> Card28["Card 28: Cgo 调用双向栈切换与运行时开销"]

    %% Cross-module relationships
    Card2 --> Card6
    Card10 --> Card25
    Card8 --> Card11
    Card12 --> Card21
    Card17 --> Card2
```

---

## 2. Go Runtime 物理源码位置映射锚点

为便于硬核技术速查，以下是 28 张核心卡片对应在 Go 语言官方源码仓 `golang/go` 中的核心源码文件及函数位置：

*   **Goroutine GMP 调度与网络轮询器 (M1)**:
    *   G, M, P 核心结构体定义：`src/runtime/runtime2.go` -> `type g struct`、`type m struct`、`type p struct`
    *   调度循环入口决策链：`src/runtime/proc.go` -> `schedule()`、`findrunnable()`
    *   工作窃取实现：`src/runtime/proc.go` -> `stealWork()`
    *   网络轮询器物理底层：`src/runtime/netpoll.go` & `src/runtime/netpoll_epoll.go` -> `netpoll()`
    *   系统监控与非协作抢占：`src/runtime/proc.go` -> `sysmon()`、`preemptone()` & `src/runtime/preempt.go`
*   **内存分配与协程栈 (M2)**:
    *   分配器总架构：`src/runtime/malloc.go` -> `mallocgc()`
    *   本地缓存与全局堆结构：`src/runtime/mcache.go`、`src/runtime/mcentral.go`、`src/runtime/mheap.go`
    *   内存跨度管理：`src/runtime/mspan.go` -> `mspan`
    *   逃逸分析静态检查入口：`src/cmd/compile/internal/escape/escape.go`
    *   连续栈拷贝与扩容：`src/runtime/stack.go` -> `copystack()`、`newstack()`
*   **三色 GC 与写屏障 (M3)**:
    *   GC 主控状态机：`src/runtime/mgc.go` -> `gcStart()`
    *   三色标记与工作队列：`src/runtime/mgcmark.go` -> `gcDrain()`
    *   混合写屏障底层：`src/runtime/mbarrier.go` -> `writebarrierptr()`
    *   GC 协程辅助机制：`src/runtime/mgcassist.go` -> `gcAssistAlloc()`
    *   GC Pacer 调优器：`src/runtime/mgcpacer.go`
*   **Channels 与同步原语 (M4)**:
    *   管道核心实现：`src/runtime/chan.go` -> `hchan` & `makechan()`、`chansend()`、`chanrecv()`
    *   Select 展开运行期：`src/runtime/select.go` -> `selectgo()`
    *   互斥锁核心实现：`src/sync/mutex.go` -> `Mutex.Lock()`、`Mutex.lockSlow()`
    *   Once 与同步机制：`src/sync/once.go` & `src/sync/waitgroup.go`
*   **Maps 与接口 (M5)**:
    *   字典底层排布与冲突：`src/runtime/map.go` -> `hmap` & `makemap()`、`mapaccess1()`
    *   字典扩容槽位迁移：`src/runtime/map.go` -> `hashGrow()`、`evacuate()`
    *   接口底层表示与反射元数据：`src/runtime/iface.go` -> `iface`、`eface` & `itab`
    *   去虚拟化静态Pass：`src/cmd/compile/internal/devirtualize/devirtualize.go`
*   **Defer、Panic、Syscall 与 Cgo (M6)**:
    *   Defer 堆栈优化与开放编码：`src/runtime/panic.go` -> `deferproc()`、`deferreturn()`
    *   Panic 与恢复控制：`src/runtime/panic.go` -> `gopanic()`、`gorecover()`
    *   系统调用包装与 GMP 剥离：`src/runtime/proc.go` -> `entersyscall()`、`exitsyscall()`
    *   Cgo 运行时切换与屏障：`src/runtime/cgocall.go` -> `cgocall()` & `src/runtime/cgo/`
