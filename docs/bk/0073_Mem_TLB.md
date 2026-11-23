-----

| Title     | Mem TLB                                           |
| --------- | ------------------------------------------------- |
| Created @ | `2026-10-16T08:35:37Z`                            |
| Updated @ | `2026-10-16T08:42:20Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/73) |

-----

# Mem TLB

> 本文是 [内存优化](0037_Mem) 的子篇，讲 TLB（Translation Lookaside
> Buffer，地址翻译缓存）和大页。
> 
> 注意和 [Memory Cacheline](0036_Mem_CacheLine) 区分：Cache 缓存的是**数据本身**，TLB
> 缓存的是**虚拟地址→物理地址的翻译结果**，是完全不同的两层缓存，各自有各自的命中率问题。

## 一、地址翻译与 Page Walk

x86-64 的 4 级页表：`CR3 → PML4E → PDPTE → PDE → PTE`，每级用虚拟地址的 9 个 bit
索引（9+9+9+9+12 = 48 位虚拟地址，256 TiB 空间）。

\*\*5 级分页（LA57）\*\*增加了 PML5 层，支持 57 位虚拟地址（128 PiB）/ 4 PiB 物理地址，由
`CR4.LA57` 开启。Intel 从 **Ice Lake**（服务器端为第三代 Xeon Scalable）开始支持；Linux 在
5.5 起默认开启，`lscpu | grep la57` 可查，内核参数 `no5lvl` 可关。

**TLB miss 为什么贵**：一次 miss 最多需要 4（或
5）次**串行依赖**的内存访问——每一级的地址都依赖上一级的结果，无法并行。

**Paging-structure caches**（Intel 官方术语，俗称 page walk
cache）用来缓解这个问题：PML5/PML4/PDPTE/PDE
各有自己的缓存，保存部分走表结果，命中后可以直接跳到最后一级去取
PTE。另外 PTE 本身也会作为普通数据缓存在 L1D/L2
里。注意这些缓存条目**不会因为之后对页表的写入而自动失效**，所以修改页表后必须显式
`INVLPG`/`INVPCID`。

## 二、TLB 结构与 "Reach"

现代 Intel 核心的 TLB 层级（条目数，不同代际有差异）：

|             | Skylake/-SP | Ice Lake | Sapphire Rapids |
| ----------- | ----------- | -------- | --------------- |
| L1 iTLB（4K） | 128         | 128      | **256**         |
| L1 iTLB（2M） | 8/线程        | 16       | 32              |
| L1 dTLB（4K） | 64          | 64       | **96**          |
| L1 dTLB（2M） | 32          | 32       | 32              |
| L1 dTLB（1G） | 4           | 8        | 8               |
| L2 STLB（统一） | **1536**    | **2048** | **2048**        |

> 置信度说明：STLB 的 1536/2048 和 Golden Cove 的 iTLB 256 / dTLB 96 来自 Intel
> 优化手册，较可靠；1G 页的条目数以及 1G 翻译是否占用 STLB 在公开文档里语焉不详，仅供参考。Skylake 有个
> **SKL148** 勘误——CPUID leaf 2 会把 STLB 的相联度误报为 6 路，不要采信。

**TLB Reach（覆盖范围）= 条目数 × 页大小**，这是理解大页价值的关键：

| 页大小  | SPR STLB（2048 条）能覆盖            |
| ---- | ------------------------------ |
| 4 KB | 8 MB                           |
| 2 MB | 4 GB                           |
| 1 GB | 8 GB（受限于 L1 dTLB 只有 8 条 1G 条目） |

也就是说，一个 64 GB 的内存索引在 4KB 页下，TLB 只能覆盖其约 0.01%
的footprint——随机访问时几乎每次都要走表。实测 SPR 上
dTLB 命中时 load-to-use 是 5 个周期，miss 后是 12 个周期，即约 7 周期的额外惩罚。

## 三、Linux 大页机制

### Transparent Huge Pages（THP，透明大页）

``` bash
cat /sys/kernel/mm/transparent_hugepage/enabled   # always | madvise | never
cat /sys/kernel/mm/transparent_hugepage/defrag    # always | defer | defer+madvise | madvise | never
```

  - `enabled` 控制是否自动使用大页，`defrag` 控制**分配不到大页时要不要同步做内存整理**——后者才是延迟尖峰的真凶；
  - **上游默认是 `madvise`**，而 `defer`/`defer+madvise` 选项在内核 4.11 才引入。很多"THP
    必须关掉"的老经验是基于 `defrag=always` 时代的行为（文档原文：会"在分配失败时阻塞、直接回收内存并整理"），现在
    `defer` 只唤醒 kswapd/kcompactd、把实际安装交给后台的 khugepaged；
  - khugepaged 相关调节项在 `khugepaged/`
    目录下（`pages_to_scan`、`scan_sleep_millisecs`、`max_ptes_none`
    等）。其中 `max_ptes_none` 是"内存膨胀"的调节旋钮——调大意味着更激进地用大页，但浪费更多内存。

THP 最本质的缺点，内核文档自己说得最清楚：

> 应用可能 mmap 了一大片区域但只碰了其中 1 个字节，这种情况下却分配了一个 2MB 的大页，完全是浪费。

另外 COW（写时复制）场景下，对 PMD 映射的 THP 发生一次 COW 缺页会复制 **2MB 而不是 4KB**。

较新的缓解机制：**mTHP**（约 6.8
起，支持按大小粒度开关，`hugepages-<size>kB/enabled`）、`shrink_underused=1`（内存压力下拆分使用率低的
THP）、`prctl(PR_SET_THP_DISABLE)`（让单个进程退出 THP，可跨 fork/exec 继承）。

### 显式 hugetlbfs

``` bash
# 预留大页池
echo 1024 > /proc/sys/vm/nr_hugepages
# 或按大小分别预留
echo 1024 > /sys/kernel/mm/hugepages/hugepages-2048kB/nr_hugepages

# 挂载（也可以不挂载，直接用 mmap 的 MAP_HUGETLB）
mount -t hugetlbfs -o pagesize=2M,size=4G none /mnt/huge

# 1GB 大页必须在启动时预留（内核命令行）
#   hugepagesz=1G hugepages=16
#   hugepagesz=1G hugepages=0:8,1:8     （按 NUMA node 分别预留）
```

和 THP 的核心区别：hugetlbfs
是**预先保留**的，这部分内存被内核锁定、不能作他用、**不会被换出**，也就不存在运行时整理内存导致的延迟抖动。代价是要提前规划容量、不够灵活。

代码里按区域选择性启用：`madvise(addr, len, MADV_HUGEPAGE)` / `MADV_NOHUGEPAGE` /
`MADV_COLLAPSE`（最后这个即使在 `never` 模式下也有效）。

### 数据库厂商的立场（注意有重要反转）

| 厂商                     | 建议                                                                              |
| ---------------------- | ------------------------------------------------------------------------------- |
| Redis                  | `never`（fork + COW 时 RSS 暴涨），二进制自身会调 `PR_SET_THP_DISABLE`                       |
| MongoDB ≤ 7.0          | `enabled=never`、`defrag=never`                                                  |
| **MongoDB ≥ 8.0**      | **改为启用**：`always` + `defer+madvise` + `max_ptes_none=0`（换用了新的 per-CPU TCMalloc） |
| Oracle ≤ 21c           | `transparent_hugepage=never`                                                    |
| **Oracle 23ai / 26ai** | **改为 `madvise`**，文档明确说"不要像之前版本建议的那样禁用"                                          |
| Couchbase              | `never`（THP 会通过膨胀 RSS 导致 OOM kill）                                              |
| PostgreSQL             | 上游没有 THP 立场，只文档化了显式 hugetlb（`huge_pages = on\|try\|off`，默认 `try`）               |

**所以"数据库都要关 THP"这条流传很广的经验，MongoDB 和 Oracle 都已经明确推翻了**，不要再无条件照搬。

## 四、大页什么时候有用、什么时候反而有害

**有用**：工作集大且随机访问的场景。

  - GUPS（32 GB 随机访问）：4K 页 \~0.012 → 2M 页 0.041 → 1G 页 0.066，即 3.4× /
    5.4× 提升；
  - 大哈希表探查：1.14-1.25× 提升，dTLB load miss 率从 1.5%-2.6% 降到约 0.01%；
  - 大二进制文件的**代码段**（text）用大页也能减少 iTLB 压力。

**有害或无收益**：

  - 内部碎片 / RSS 膨胀（Couchbase 的 OOM kill 就是这个机制）；
  - 直接回收/内存整理造成的停顿，实测可达"几十毫秒，有时甚至几秒"；
  - 和 NUMA 冲突：一个 2MB 页整体属于一个 node，粒度变粗会让放置更难精细，还会削弱按 4K 粒度做的 AutoNUMA
    迁移（见 [Mem NUMA](0072_Mem_NUMA)）；
  - 1G 页不是单调更好——它的 TLB 条目只有约 8 条；
  - **顺序流式访问收益很小**：走表开销会被一个页里的 512 条 cache line
    摊薄。（不过这里有个更细的机制：硬件预取器不跨页边界，大页减少了预取重启的次数，所以也不是完全没有收益。）

**方法论**：先测页走查占了多少周期，这就是大页能带来的收益上限——不要凭感觉开。

## 五、怎么测量 TLB 压力

### perf

``` bash
# 通用事件（注意：这些别名在不同代际映射到不同的原生事件，跨机器不要直接比绝对值）
perf stat -e dTLB-loads,dTLB-load-misses,dTLB-stores,dTLB-store-misses,\
iTLB-loads,iTLB-load-misses ./app

# 原生页走查事件（Skylake 及之后）
perf stat -e cycles,dtlb_load_misses.walk_active,dtlb_load_misses.walk_pending,\
dtlb_load_misses.walk_completed,dtlb_load_misses.stlb_hit,\
dtlb_store_misses.walk_active,itlb_misses.walk_active ./app

# 看大页到底有没有生效（Skylake/SKX 支持按页大小分别统计）
perf stat -e dtlb_load_misses.walk_completed_4k,\
dtlb_load_misses.walk_completed_2m_4m,dtlb_load_misses.walk_completed_1g ./app

# 推荐：直接用 TMA 指标
perf stat -M tma_dtlb_load,tma_dtlb_store,tma_itlb_misses ./app
```

关键区别：Skylake 把旧的 `WALK_DURATION` 拆成了
**`WALK_ACTIVE`**（至少有一个页walk单元在忙的周期数，是真正的停顿代理指标）和
**`WALK_PENDING`**（按单元累计的占用数，可能超过总周期数）。两者相除 = 平均页走查并行度。

### VTune

分析类型用 **Microarchitecture Exploration**（不是 Memory Access），指标位置：

  - `Front-End Bound → Front-End Latency → ITLB Overhead`
  - `Back-End Bound → Memory Bound → L1 Bound → DTLB Overhead`
  - `→ Store Bound → DTLB Store Overhead`
  - `Page Walk`（同时覆盖取指侧和数据侧）

VTune 在 DTLB Overhead 下自带的建议就是："尝试对大量频繁访问的数据使用更大的页"。详见 [Vtune
Microarchitecture](0058_Tools_Vtune_AT_Microarchitecture)、[Vtune
Metrics](0056_Tools_Vtune_Metrics)。

> 坑：VTune 文档里有个标题叫 "TLB Misses" 的页面讲的是 **GPU** 指标，不是这里说的 CPU TLB。

### 查看当前大页使用情况

``` bash
grep -E 'AnonHugePages|ShmemPmdMapped|FilePmdMapped|HugePages_|Hugepagesize|Hugetlb' /proc/meminfo
grep -E 'AnonHugePages|Shared_Hugetlb|Private_Hugetlb' /proc/$PID/smaps_rollup
grep HugetlbPages /proc/$PID/status
grep -E 'thp_fault_alloc|thp_fault_fallback|thp_collapse_alloc|thp_split' /proc/vmstat  # 成功 vs 回退次数
```

注意 `/proc/meminfo` 里的 `HugePages_*` 只统计**默认大小**，`Hugetlb`
才是所有大小的总量；`*PmdMapped` 才是真正以 PMD 方式映射（即真正省下 TLB
条目）的部分。

## 六、代码层面的优化手段

大页是配置层面的旋钮，下面这些才是代码层面的：

  - **减少触碰的页数量，而不只是减少触碰的 cache line 数量**。Intel 明确指出循环分块（blocking）"也能降低
    DTLB miss 的惩罚"——机制是**翻译结果的时间复用**（每个 tile
    把页工作集约束在很小范围内），而不是空间局部性。但要注意
    tile 形状选得不好时分块**反而会增加** TLB miss：有实测显示非结构化网格代码在小 tile 下 TLB miss 增加
    16%-35%，因为每次循环跳转都落到新的页上。所以分块时要同时考虑 L2/LLC 和 TLB 两个 footprint；
  - **AoS → SoA / 结构体拆分**：只读少数字段时，能大幅减少每轮循环触碰的页数（见 [内存优化](0037_Mem)）；
  - **避免指针追逐**：链表节点在不同时刻分配，散落在各处；而且下一个地址要等当前节点返回才知道，乱序执行和预取器都没法隐藏这次页走查。改成数组化/用索引代替指针、用
    arena 分配器让一次遍历尽量落在少数几个页里（见 [Mem Allocator](0074_Mem_Allocator)）；
  - **TLB 友好 vs Cache 友好**：两者通常一致（都偏好连续和复用），但衡量粒度不同，存在"某个变换减少了 LLC miss
    却触碰了更多页、最终输给页走查"的情况。应当把它们当作两份独立的 footprint 预算来看。

## Reference

  - [Intel SDM Vol.3A §4.10 - Caching Translation
    Information](https://xem.github.io/minix86/manual/intel-x86-and-64-manual-vol3/o_fe12b1e2a880e0ce-142.html)
  - [Intel TLB and Paging-Structure Caches（App
    Note 317080）](https://kib.kiev.ua/x86docs/Intel/WhitePapers/317080-002.pdf)
  - [5-Level Paging White
    Paper](https://cdrdv2-public.intel.com/671442/5-level-paging-white-paper.pdf)
  - [Linux Kernel: Transparent Hugepage
    Support](https://docs.kernel.org/admin-guide/mm/transhuge.html)
  - [Linux Kernel: HugeTLB
    Pages](https://docs.kernel.org/admin-guide/mm/hugetlbpage.html)
  - [WikiChip: Skylake
    (server)](https://en.wikichip.org/wiki/intel/microarchitectures/skylake_\(server\))
  - [Speeding up address translation: TLB and Huge
    Pages](https://johnnysswlab.com/speeding-up-translation-of-virtual-to-physical-memory-addresses-tlb-and-huge-pages/)
  - [Utilizing Huge Pages For
    Code](https://easyperf.net/blog/2022/09/01/Utilizing-Huge-Pages-For-Code)
  - [Intel: Loop Optimizations Where Blocks are
    Required](https://www.intel.com/content/www/us/en/developer/articles/technical/loop-optimizations-where-blocks-are-required.html)
