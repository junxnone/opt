---
Title | Mem Allocator
-- | --
Created @ | `2026-10-16T08:35:39Z`
Updated @| `2026-10-16T08:43:35Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/74)

---
# Mem Allocator

> 本文是 [内存优化](0037_Mem) 的子篇，讲内存分配器本身的性能问题——"为什么多线程下 malloc 成了瓶颈"、几种主流分配器的差异、内存池怎么用，以及分配器引发的伪共享/碎片/RSS 膨胀怎么诊断。

## 一、通用 malloc 是怎么工作的

- **Size class / bin**：请求被圆整到对齐边界（glibc 是 16B 粒度），按大小分到不同的 bin。glibc 里有 fastbins（小块、LIFO、不合并）、unsorted bin、small bins、large bins、top chunk；
- **brk vs mmap**：`M_MMAP_THRESHOLD` 默认 128 KB。超过阈值且 free list 满足不了的请求走 `mmap`，否则用 `sbrk` 抬高 program break。mmap 的块能独立归还 OS，但必须页对齐、内核要清零，释放后也不进 free list 复用；
- **free 时会做什么**：与相邻空闲 chunk 合并（靠 prev/next size 字段 + PREV_INUSE 位），只有堆顶空闲量超过 trim 阈值（默认 128 KB）才**可能** `sbrk` 收缩归还 OS。所以 **free ≠ RSS 下降**，这是很多"内存泄漏"误报的根源；
- **多线程为什么会成瓶颈**：堆的 bin/free list 是共享结构，访问要加锁；高并发小对象分配会在这把锁上串行化。

> **一个很反直觉的坑**：现代 glibc 的 mmap 阈值是**动态**的——释放一个大于当前阈值且 ≤ `DEFAULT_MMAP_THRESHOLD_MAX`（64 位是 32MB）的块时，阈值会自动上调到该块大小。但是，一旦你显式设置了 `M_TRIM_THRESHOLD`/`M_TOP_PAD`/`M_MMAP_THRESHOLD`/`M_MMAP_MAX` 里的**任何一个**，这套动态调整机制就被**永久关闭**了。

## 二、glibc malloc（ptmalloc2）

- **Arena**：main arena（用 brk）+ 每线程 arena（用 mmap，每个 heap 占 64MB 虚拟空间），**每个 arena 一把锁**；
- **数量上限**：64 位是 `8 × 核数`，32 位是 `2 × 核数`。线程首次 malloc 时绑定到某个 arena；
- **虚拟内存膨胀问题**：容器里如果 CPU 数探测错误（比如探到宿主机的 96 核 → 768 个 arena），每个 arena 实际占用 10-30 MB，RSS 可以膨胀到 GB 级；而且线程退出时 arena 只标记为空闲、并不归还。典型修复：

```bash
export MALLOC_ARENA_MAX=2          # 或在代码里 mallopt(M_ARENA_MAX, 2)
```

必须在**第一次 malloc 之前**生效。有实测案例：峰值 RSS 从 3126 MB 降到约 550 MB，而延迟基本不变。

- **tcache**（glibc 2.26 引入）：thread-local 的无锁单链表，覆盖 24-1032 B 的 chunk（`TCACHE_MAX_BINS=64`），每个 bin 最多缓存 7 个。命中时完全不碰 arena 锁，这是 2.26 之后小对象吞吐大幅提升的主因。做对比实验时可以用 `GLIBC_TUNABLES=glibc.malloc.tcache_count=0` 关掉它。

## 三、替代分配器

| | 设计要点 | 擅长场景 |
|---|---|---|
| **tcmalloc**（Google） | 三层结构：front-end（per-thread 或 per-CPU cache，无锁）→ middle-end（TransferCache + CentralFreeList，各有 mutex）→ back-end page heap。free list 长度用 slow-start 自适应 | 小对象高频分配、线程数中等的 CPU 密集型服务 |
| **jemalloc**（FB/FreeBSD） | 多 arena + tcache，分小/大/huge 三档；核心目标是**低碎片**。`MALLOC_CONF` 调优和 profiling 能力极强，`background_thread` + `dirty_decay_ms`/`muzzy_decay_ms`（半衰期式，不是硬超时）控制回吐节奏 | 长跑服务、碎片/RSS 敏感场景（Redis、Rust 生态、ClickHouse）。有团队实测 RSS 873MB → 458MB |
| **mimalloc**（Microsoft） | **free list sharding**：不是每个 size class 一条大链表，而是每个约 64KiB 的 "mimalloc page" 各自持有三条 page-local 链表（local free / thread free / free），全原子操作无锁。代码量约 3500 行（tcmalloc 约 2 万、jemalloc 约 2.5 万），元数据开销约 0.2% | 跨线程 alloc/free 的场景优势明显；论文称在 redis 上比 tcmalloc/jemalloc 快 7%/14% |

> **诚实提示**：mimalloc 的 benchmark 由其作者维护；`cfrac`（单线程小对象）这类场景几乎没差别，而 `sh6bench` 能拉开 2.5× 以上。**结论强依赖具体负载，必须在自己的硬件和负载上实测**（可以用 [mimalloc-bench](https://github.com/daanx/mimalloc-bench)）。

**不重新编译就换分配器**：

```bash
LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2 ./app
# 验证是否生效
pldd $(pidof app) | grep jemalloc
```

注意必须用全路径；静态链接的程序无效；不要全局 `export`（tcmalloc 不适合做系统级分配器）；生产环境建议灰度验证（有过 alpine + jemalloc 段错误的报告）。

## 四、内存池与自定义分配器

- **Object pool（对象池）**：固定大小对象的 free list，分配 = pop，释放 = push；
- **Arena / bump allocator**：指针自增分配，释放靠整体 reset——最快，但只适合"一批对象同生共死"的场景（一帧、一个请求、一次解析）；
- **Slab**：按对象类型预切好，复用已构造的对象（Linux 内核 `kmem_cache` 的思路）。

### C++17 `std::pmr`

`memory_resource` 只有 `allocate`/`deallocate`/`is_equal` 三个虚函数，把"容器行为"和"分配策略"解耦，运行时可替换：

```cpp
#include <memory_resource>

// bump allocator，deallocate 是空操作，靠 release()/析构整体回收
std::array<std::byte, 4096> buf;
std::pmr::monotonic_buffer_resource mbr{buf.data(), buf.size()};
std::pmr::vector<int> v{&mbr};          // 分配全部来自栈上的 buf

// 按 size class 分池，降低碎片；适合 node-based 容器
std::pmr::unsynchronized_pool_resource pool;   // 单线程/外部同步
std::pmr::list<MyType> lst{&pool};
```

`monotonic_buffer_resource` 在 benchmark 中最快（Windows 上相比 `new_delete_resource` 可差约 10×）。pool resource 的 upstream 还可以串联指向 monotonic resource。

**什么时候值得自己写**：有**实测证据**表明分配是热点、分配模式单一（固定大小/固定生命周期）、或者需要确定性延迟时。否则属于过早优化——现代通用分配器的小对象快路径已经很难手工超越，而自定义池会引入容量上限、生命周期和线程约束。

## 五、分配器引发的性能问题

### 分配器导致的伪共享

两个线程各自的小对象落在同一条 cache line 上，一方写就让另一方的行失效（原理见 [Memory Cacheline](0036_Mem_CacheLine)）。Hoard 论文把它分成两类：
- **Active false sharing**：不同线程分配的对象落到同一行；
- **Passive false sharing**：线程 A 远程释放的块立刻被 A 自己复用。

注意**分配器自身的元数据**也会污染其他核的 cache line。要完全消除就得把每次请求都 pad 到整条 cache line，代价是内存和 cache 双重浪费——所以实践上是用 per-thread heap 的分配器（tcmalloc/mimalloc/jemalloc）+ 应用层显式 `alignas(64)`。

### 碎片

- **内部碎片** = size class 圆整造成的浪费（mimalloc 声称最多 16.7%）；
- **外部碎片** = 空闲总量够但没有足够大的连续块。

表现是**没有泄漏但 RSS 持续增长**。判据：`RSS >> 分配器报告的 allocated`（jemalloc 可以看 `active` 与 `allocated` 的差距）。

### 热循环内分配

把 `vector`/`string` 提到循环外复用（`clear()` 会保留 capacity），或者用 pmr 的 monotonic resource 每轮 `release()`。

### 首次触碰的代价

`malloc` 只给虚拟地址，物理页是在第一次写时通过 **minor page fault** 分配并清零的。大 buffer 的首次写成本可能远超分配本身。

在 NUMA 上这直接决定了 **first-touch 放置**：页落在**首次触碰的那个线程所在的 node**，不是分配它的线程。所以"主线程 malloc + 主线程 memset，再交给 worker 并行处理"会把所有页钉在一个 node 上。详见 [Mem NUMA](0072_Mem_NUMA)。

## 六、怎么诊断

- **[VTune Memory Consumption](0060_Tools_Vtune_AT_MC)**：拦截 alloc/dealloc 事件，**为每个分配事件抓完整调用栈**（释放只记录释放函数，是不对称的）。Timeline 看 RAM 曲线和峰值，框选峰值区间 → Filter In by Selection；在 Call Stack 面板比较 allocation 与 deallocation 总量的**差值**来定位滞留内存。"Minimal dynamic memory object size to track" 默认 32B，调大可降低开销。自己写的包装分配器需要用 Memory Allocation API 标注才能被识别；
- **heaptrack**（KDE）：相比 valgrind massif 的优势是不串行化多线程、只在分配时付开销、能 attach 到运行中的进程，因此能报告**临时分配**（分配后立刻释放）的次数——这是 massif 的快照式聚合丢失掉的信息。massif 的开销可达两个数量级，但它能 profile page 级分配（绕过 malloc 的池分配器）和栈分配，这是 heaptrack 做不到的；
- **jemalloc 内置 heap profiler**：需要 `--enable-prof` 编译，`MALLOC_CONF=prof:true,prof_leak:true,lg_prof_sample:19,lg_prof_interval:30`。长跑守护进程要用 `lg_prof_interval` 周期性 dump（因为 `prof_leak` 走 `atexit`，被 `kill` 掉就不触发）。分析要用 `jeprof`（不是 gperftools 的 `pprof`）。统计信息用 `malloc_stats_print()`；
- **perf**：`perf stat -e page-faults,minor-faults,major-faults`，`perf record -e page-faults -g` 定位首次触碰的热点。

### 判据：是"分配器慢"还是"分配太多"

| 现象 | 结论 | 该做什么 |
|---|---|---|
| 分配次数不算离谱，但 CPU 时间集中在 `_int_malloc`/`malloc_consolidate`/arena 锁上；锁等待随线程数超线性增长 | **分配器是瓶颈** | 换 jemalloc/mimalloc，或调 `MALLOC_ARENA_MAX` |
| 分配次数/秒极高，临时分配占比大 | **只是分配太多** | 换分配器收益有限；要复用缓冲区、改数据结构、用 pmr 池 |
| RSS 与 allocated 长期背离，`malloc_trim` 后 RSS 骤降 | **碎片/不归还** | 调 jemalloc 的 decay 参数，或周期性 trim |
| minor-faults 计数与运行时间强相关，或远端访存比例高 | **首次触碰 / NUMA** | 见 [Mem NUMA](0072_Mem_NUMA) |

## Reference
- [mallopt(3)](https://manpages.debian.org/testing/manpages-dev/mallopt.3.en.html)
- [glibc Malloc Tunable Parameters](https://www.sourceware.org/glibc/manual/latest/html_node/Malloc-Tunable-Parameters.html)
- [TCMalloc Design](https://google.github.io/tcmalloc/design.html)
- [jemalloc TUNING.md](https://github.com/jemalloc/jemalloc/blob/dev/TUNING.md)
- [mimalloc 技术报告](https://www.microsoft.com/en-us/research/wp-content/uploads/2019/06/mimalloc-tr-v1.pdf)
- [Hoard: A Scalable Memory Allocator（ASPLOS 2000）](https://www.cs.utexas.edu/~mckinley/papers/asplos-2000.pdf)
- [VTune Memory Consumption Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-2/memory-consumption-analysis.html)
- [heaptrack](https://github.com/KDE/heaptrack)
- [Polymorphic Allocators in C++17](https://www.modernescpp.com/index.php/polymorphic-allocators-in-c17/)


