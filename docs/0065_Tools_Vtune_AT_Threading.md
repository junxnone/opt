---
Title | Tools Vtune AT Threading
-- | --
Created @ | `2026-10-14T07:17:43Z`
Updated @| `2026-10-14T07:17:43Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/65)

---
# Vtune Threading

- **Threading 分析**：定位多线程应用里"核心没跑满"的原因——同步开销、线程饥饿、过度订阅（oversubscription）等，帮助判断线程扩展性瓶颈出在哪。属于 Parallelism Analysis Group，详见 [Analysis Type 索引](/0062_Tools_Vtune_AT)
- 自 VTune 2021 起，Threading 分析合并了此前独立的 Concurrency 和 Locks and Waits 两种分析类型，现在是并行瓶颈分析的统一入口

## 核心指标

- **Effective CPU Utilization**：衡量线程利用率的核心指标，反映应用实际用了多少逻辑核。吞吐计算场景下，通常期望每个物理核对应加载一个逻辑核
- **Thread Count**：线程数是否固定、是否随负载伸缩，固定线程数可能导致应用无法扩展到更多核，或造成线程过度订阅
- **Wait Time**：线程因等待同步对象（Mutex/Semaphore 等）或 I/O 而被挂起的时间。分两类同步对象：线程间同步对象、I/O 相关等待对象
- **Spin Time / Overhead Time**：线程运行时花在忙等（busy wait）或任务调度开销上的时间。目标是尽量减少这部分 CPU 周期的消耗
- **Inactive Sync Wait Time / Inactive Wait Count**（HWE/Context Switch 模式下）：线程因同步或被抢占而被 OS 调度器排除在执行之外的时间与次数，过多的上下文切换会拖累性能

## 采集方式

- **User-Mode Sampling and Tracing**：可识别具体的同步对象，按对象统计等待时间，便于理解线程交互语义
- **Hardware Event-based Sampling（HWE/Context Switches）**：通过 Caller/Callee 定位调用高等待时间等待函数的具体代码位置

## 典型工作流

1. 以 Elapsed Time 作为优化前后对比的基线
2. 先看 Effective CPU Utilization、Thread Count，判断核心是否跑满、线程数是否合理
3. 看 Wait Time：在 Bottom-up 视图里按 Wait Time/Wait Count 排序，定位等待时间最长、等待次数最多的同步对象，分析其调用栈
4. 看 Spin Time/Overhead Time：定位花费在忙等或任务调度上的调用路径，减少不必要的忙等
5. 若上下文切换次数（Inactive Wait Count）偏高，优先排查同步竞争是否过于频繁，或是否存在线程过度订阅

## CLI 示例

```bash
vtune -collect threading -- ./your_app
```

## Reference
- [Threading Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/threading-analysis.html)
- [Threading Efficiency View](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-0/threading-efficiency-view.html)
- [CPU Metrics Reference](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-1/cpu-metrics-reference.html)

