-----

| Title     | Tools Vtune AT MA                                 |
| --------- | ------------------------------------------------- |
| Created @ | `2024-04-20T19:48:00Z`                            |
| Updated @ | `2026-10-14T05:51:02Z`                            |
| Labels    | `documentation`                                   |
| Edit @    | [here](https://github.com/junxnone/opt/issues/59) |

-----

# Vtune Memory Access

  - 适用于怀疑是**内存访问模式**导致性能瓶颈的场景，尤其是 [Performance
    Snapshot](/0051_Tools_Vtune_AT_PS) 或 [Microarchitecture
    Exploration](/0058_Tools_Vtune_AT_Microarchitecture) 已经显示 Memory
    Bound 较高时，用 Memory Access 进一步细化到具体是哪一级存储、哪个数据结构的问题
  - 基于内存分配的插桩，可以把性能事件归因到具体的内存对象（数据结构）/分配调用栈，而不仅是函数

## 核心指标

  - **Memory Bound**: 遵循 [TMA](/0047_OPT_TMA) 方法论中 Back-End Bound
    下的细分，指流水线因访存而停顿的 slot 占比
      - **L1/L2/L3 Bound**: 命中各级 Cache 所付出的延迟占比，数值越高代表该级 Cache 命中率/延迟是瓶颈
      - **DRAM Bound**: 请求穿透到主存（DRAM）的延迟占比，通常意味着数据局部性差或工作集超出 LLC 容量
  - **Bandwidth Utilization Histogram**:
    采集期间内存带宽利用率的分布，若长时间接近峰值带宽，说明已达到硬件带宽上限，需要减少数据搬运量而非只优化访问顺序
  - **NUMA 相关指标**: 在多路（NUMA）平台上，统计本地 DRAM 访问 vs 远端 DRAM
    访问的比例；远端访问延迟显著高于本地，应尽量让线程访问的数据绑定在本地
    NUMA 节点
  - **Memory Bandwidth per Function / per Memory Object**:
    按函数或数据结构维度拆分内存流量，定位具体是哪段代码/哪个对象在产生大量内存访问

## 常见问题模式

  - **Cache 命中率低 / 访问模式不连续**：L1/L2/L3 Bound
    偏高，考虑调整数据结构布局、提升访问局部性（如循环分块、数组替代链表）
  - **带宽瓶颈**：DRAM Bound 高且 Bandwidth Utilization
    长期接近峰值，需要减少总内存流量（如算法级优化、数据压缩/复用）而不是单纯调整访问顺序
  - **False Sharing（伪共享）**：多个线程各自访问不同变量，但这些变量恰好落在同一条 Cache Line 上，导致
    Cache Line 在核间频繁失效/回写。排查方法：开启 "Analyze Memory Objects" 并将 Memory
    Object Size Threshold 设为 1，以捕获所有内存分配，再观察是否有多线程集中访问同一小块内存区域
  - **NUMA 远端访问占比高**：考虑用 `numactl` 绑核绑内存，或在代码里显式做 NUMA-aware 的内存分配

## 命令行

``` bash
vtune -collect memory-access -knob analyze-mem-objects=true -result-dir r001ma ./app
```

## Reference

  - [Memory Access Analysis for Cache Misses and High Bandwidth
    Issues](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/memory-access-analysis.html)
  - [Finding Your Memory Access Performance
    Bottlenecks](https://www.intel.com/content/www/us/en/developer/articles/technical/finding-your-memory-access-performance-bottlenecks.html)
  - [Profiling High Bandwidth Memory Performance on Intel® Xeon® CPU Max
    Series](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2024-1/profiling-hbm-performance-on-intel-max.html)
