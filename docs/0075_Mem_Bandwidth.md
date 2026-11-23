---
Title | Mem Bandwidth
-- | --
Created @ | `2026-10-16T08:35:40Z`
Updated @| `2026-10-16T08:35:40Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/75)

---
# Mem Bandwidth

> 本文是 [内存优化](0037_Mem) 的子篇，讲内存带宽作为性能瓶颈的那一面：带宽和延迟的区别、怎么测可达带宽、怎么通过 Non-temporal store 省掉一半写带宽，以及带宽打满之后唯一有效的优化维度是什么。

## 一、带宽 vs 延迟，与 Little's Law

- **延迟** = 单次访问完成所需的时间（DDR5 本地约 75-85 ns）；**带宽** = 单位时间的总吞吐量。
- 两者相对独立：延迟不变的情况下，带宽只能靠**提高并发**来提升。

**Little's Law 应用到内存**：

```
在途字节数（并发量） = 带宽 × 延迟
```

例：50 GB/s × 70 ns = 3500 B ≈ 55 条 cache line，必须**始终**保持这么多请求在途，才能打满 50 GB/s。

而硬件上限是 **MSHR / Line Fill Buffer**：Intel 自 Haswell 起每核约 10 个 L1 LFB（L2 约 16 个 MSHR），而且 L1 硬件预取器也共用这 10 个槽。单核上限大致是：

```
10 × 64 B / 70 ns ≈ 9 GB/s
```

而整机 8 通道可达 300 GB/s——**所以单核在结构上就不可能打满服务器带宽，必须靠多核/多线程**。这不是代码写得不好，是硬件决定的。

两个推论：
- L2/LLC 预取器之所以有效，是因为它降低了公式里的"延迟"项（命中 L3 时约 12 ns → 单核上限涨到约 53 GB/s）；大页和 NUMA 本地化同理；
- 反过来说，很多"看起来是 latency bound"的程序，其实是应用自身的访存并行度（MLP）太低，MSHR 根本没占满——这种情况加线程或改访问模式比换内存更有效。

## 二、怎么测可达带宽

### STREAM

事实标准，四个 kernel：

| Kernel | 操作 | 计入字节/iter |
|---|---|---|
| Copy | `a[i]=b[i]` | 16 |
| Scale | `a[i]=q*b[i]` | 16 |
| Add | `a[i]=b[i]+c[i]` | 24 |
| **Triad** | `a[i]=b[i]+q*c[i]` | 24 |

Triad 最常被引用（2 读 1 写 + FMA，最接近真实 HPC 循环）。运行规则：
- **每个数组 ≥ 所有参与 LLC 容量之和的 4 倍**，或 ≥ 1M 元素，取大者；
- 必须绑核（`OMP_PROC_BIND=spread`，每物理核 1 线程）；
- **重要**：STREAM **不计 write-allocate 流量**。如果没用 NT store，真实的总线流量要乘 1.5（Copy/Scale）或 1.33（Add/Triad）——也就是说 STREAM 报出来的数字默认**低估**了实际搬运量 33%-50%。

### Intel MLC（Memory Latency Checker）

```bash
./mlc                              # 无参数跑全套
./mlc --bandwidth_matrix           # NUMA 跨 socket 带宽矩阵
./mlc --peak_injection_bandwidth   # 按不同读写比的峰值
./mlc --loaded_latency             # 延迟-带宽曲线，找"拐点"，最有价值
./mlc -e -l128 -r                  # 测纯延迟（-e 不改预取器 MSR）
```

`--loaded_latency` 给出的那条曲线最有参考价值：它展示了随着负载上升延迟如何恶化，拐点就是这台机器实际可用的带宽上限。

### 其他

- **likwid-bench**：汇编级 kernel（`copy`/`triad`/`store` 等，带 `_mem` 后缀的版本用 NT store），`-w S0:2GB:8` 直接指定 NUMA 域和线程数；
- **VTune** 的 Memory Access 分析可以直接报告实测的 DRAM/UPI 带宽。

### 理论峰值 vs 实际可达

```
理论峰值 = 通道数 × MT/s × 8 B
8ch × DDR5-4800 × 8 B = 307.2 GB/s per socket
```

实测通常只有理论值的 **70-85%**（SPR 上偏低，约 60-70%），损耗来自 refresh、读写换向、row conflict、并发不足。**标定时请用 STREAM 实测值，而不是 datasheet 上的理论值。**

## 三、Non-temporal / Streaming Store：省掉一半写带宽

普通 store 如果目标 cache line 不在 cache 里，会先触发 **RFO（Read-For-Ownership）**——把这条 64 B 的 line 从内存读进来，再改写它。对于"只写不读"的大数组（比如输出缓冲区），这次读完全是浪费：**写 1 份数据实际搬了 2 份，约 50% 带宽白费**。

```c
#include <immintrin.h>

for (size_t i = 0; i < n; i += 8) {
    __m256 v = _mm256_load_ps(&src[i]);
    _mm256_stream_ps(&dst[i], v);     // NT store，不发 RFO，不污染 LLC
}
_mm_sfence();                          // 必须！刷 write-combining buffer
```

`_mm_stream_ps` / `_mm256_stream_si256` 对应 `MOVNTPS`/`MOVNTDQ` 指令，按 **WC（Write-Combining）** 语义执行：绕过 cache、不做 write-allocate、不发 RFO，整条 line 合并成一次总线写事务，同时不污染 LLC。

**使用要点**：
- **必须写满整条 cache line**。WC buffer 数量有限（典型约 10 个，和 fill buffer 复用），部分写会导致 buffer 提前被逐出，产生低效的亚 line 事务，**反而比普通 store 更慢**；
- NT store 是**弱序**的，循环结束必须 `_mm_sfence()`（跨核可见性场景用 `MFENCE` 更稳妥）；
- **适用**：大块顺序写、大尺寸 memset/memcpy、不会很快回读的输出缓冲；
- **不适用**：数据马上还要再读（NT store 把它踢出了 cache，回读要再跑一趟 DRAM）、小块/非对齐/部分 line 写；
- 补充：`MOVNTDQA`（NT load）只对 WC 类型内存有意义，对普通 WB 内存基本无效，不要指望它。

> 关键认知：NT store 的收益来自**消除 RFO**，而不是"绕过 cache"本身。

## 四、读写不对称与 DRAM 效率

- DRAM 数据总线是**半双工**的，读写切换要付 **bus turnaround** 代价（tWTR/tRTW，约 7.5-12 ns），期间总线闲置。所以 MLC 测出来的"全读"带宽总是最高，1:1 读写最低；
- 内存控制器用**写队列水位线**来缓解：优先服务读请求，把写攒到高水位（如 80%）再批量 drain——代价是 drain 期间所有读被阻塞，形成尾延迟尖峰；
- **Row buffer（DRAM page）**命中情况直接影响延迟：命中已打开的 row 约 20 ns；row 为空约 40 ns；row conflict（需要 precharge + activate）约 60 ns；
- **Bank 并行**：每个 bank 有独立的 row buffer，可以流水化。地址交织策略要在"row hit 率"和"bank 并行度"之间权衡。

实践含义：即使是"顺序访问"，多个数据流同时跑时如果映射到同一 bank 的不同 row，会互相踢掉 row buffer，带宽显著下降——这是 STREAM 用太多并发流反而掉速的原因之一。

## 五、高带宽内存

### Intel Xeon CPU Max（SPR + HBM）

4 个 HBM2e stack，共 **64 GB/socket**，约 1 TB/s 级带宽；32-56 核，保证每核 > 1 GB。BIOS 启动时三选一：

| 模式 | 说明 | 代价 |
|---|---|---|
| **HBM-only** | 不插 DDR，最高带宽、零代码改动 | 容量锁死 64 GB/socket，OS 和应用共享 |
| **HBM-flat (1LM)** | HBM 与 DDR 同为主存但属于不同 NUMA node | 需要 `numactl`/memkind 显式绑定，可能要改代码 |
| **HBM-caching (2LM)** | HBM 作为 DDR 的 memory-side cache（类似 L4），软件完全透明 | 插了 DDR 时的默认模式；cache 模式写路径有额外读，NT store 收益不如 flat 模式 |

### MCDRAM / Knights Landing（已停产，仅作历史参考）

16 GB、峰值 > 450 GB/s（实测 STREAM 约 350 GB/s，需要跑满 256 线程），同样是 flat / cache / hybrid 三模式——这正是今天 HBM 模式设计的前身。

### CXL

PCIe 5.0 之上的 Type-3 内存扩展。定位是**容量 + 带宽扩展，而不是替代 DDR**：延迟 130-200 ns（比 DDR 多约 70 ns），但它是全双工的，读写混合场景的效率反而优于 DDR。实测（Micron CZ122 + Xeon 6 6900P）：DDR5:CXL 按 **3:1 加权交织**（Linux 6.9+ 的 weighted interleave，见 [Mem NUMA](0072_Mem_NUMA)），只读带宽 +24%，读写混合 +39%，整体负载几何平均提速 24%。

## 六、代码层优化 + 怎么确认自己真的是带宽瓶颈

### 带宽打满时，唯一真正有效的是减少搬运的总字节数

优化计算毫无意义——这时候 CPU 本来就在等数据。有效手段：

- **数据类型收窄/量化**：fp32 → bf16/fp16 → int8，直接按比例减少流量（AI 推理最有效的一招，参见 aiwiki 的 Model Precision 篇）；
- **压缩**：稀疏格式、位打包、delta 编码——用空闲的算力换带宽；
- **Cache blocking / tiling**：提高算术强度（Arithmetic Intensity），让数据在 L2/LLC 里被复用多次而不是反复回 DRAM（见 [内存优化 - Data Use and Reuse](0037_Mem)）；
- **[Loop fusion](0063_Loop_Fusion)**：把多个遍历同一数组的循环合并，避免数组被重复读入；
- **AoS → SoA**：消除无用字段占用的带宽；
- **消除冗余拷贝**：in-place 操作、move 语义、零拷贝/`mmap`、算子融合；
- **NT store** 去掉写侧的 RFO（见第三节）；**大页**减少 TLB miss 带来的额外访存（见 [Mem TLB](0073_Mem_TLB)）。

### 确认是否 bandwidth-bound（三者要互相印证）

1. **[VTune Memory Access](0059_Tools_Vtune_AT_MA)**：`DRAM Bound` 高，且 **Bandwidth Utilization Histogram** 有大量时间落在 > 70% 平台峰值的 High 桶里；
2. **[Roofline](0003_OPT)**：实测的算术强度位于 **ridge point 左侧**（ridge point = 峰值算力 ÷ 峰值带宽），且实测 GFLOP/s 贴着那条斜屋顶；
3. **反例判断**：如果 `DRAM Bound` 高但带宽利用率**低**——那就不是带宽瓶颈，而是**延迟/MLP 问题**（指针追逐、预取失效、NUMA 放置错误），回头看第一节。

> Roofline 的口径差异需要注意：[Intel Advisor](0008_Tools_Advisor) 默认是 **cache-aware roofline**（算术强度固定于算法本身），而 VTune 用 DRAM 计数器算出来的是 **traffic-based AI**（会随 blocking 改善而右移）。

## Reference
- [STREAM Benchmark 官方参考](https://www.cs.virginia.edu/stream/ref.html)
- [Intel Memory Latency Checker (MLC)](https://www.intel.com/content/www/us/en/developer/articles/tool/intelr-memory-latency-checker.html)
- [Intel 优化手册 - Cacheability Control](https://zzqcn.github.io/perf/intel_opt_manual/7.html)
- [Non-temporal store 实测](https://vgatherps.github.io/2018-09-02-nontemporal/)
- [Intel: Optimizing Memory Bandwidth on STREAM Triad](https://www.intel.com/content/www/us/en/developer/articles/technical/optimizing-memory-bandwidth-on-stream-triad.html)
- [Intel Xeon CPU Max Series 配置调优指南](https://cdrdv2-public.intel.com/769060/354227-intel-xeon-cpu-max-series-configuration-and-tuning-guide.pdf)
- [VTune Cookbook: Profiling HBM Performance](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-2/profiling-hbm-performance-on-intel-max.html)
- [NERSC Roofline 文档](https://docs.nersc.gov/tools/performance/roofline/)
- [Cimple: Instruction and Memory Level Parallelism](https://arxiv.org/pdf/1807.01624)

