---
Title | Tools Vtune AT SystemOverview
-- | --
Created @ | `2026-10-14T07:19:26Z`
Updated @| `2026-10-14T07:19:26Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/70)

---
# Vtune System Overview

- **系统级入口分析**：无需驱动（driverless），用来快速判断代码对 CPU/GPU/DRAM/I/O/PCIe 的整体利用情况，以及定位系统级中断、延迟问题。属于 Platform Analysis Group，详见 [Analysis Type 索引](/0062_Tools_Vtune_AT)
- 两种采集模式侧重点不同：
  - **Hardware Event-Based Sampling**：作为入口分析，快速了解 CPU/GPU/DRAM/I/O/PCIe 的整体利用率
  - **Hardware Tracing**（仅 Linux/Android 目标）：微秒级粒度分析，专门用来定位延迟问题的成因

## 核心指标与视图

### 中断分析（需要 IRQ Ftrace 事件）
- 需要开启对 IRQ Ftrace 事件的监控（System Overview 或 Custom Analysis 均可），且需要访问 `/sys/kernel/debug/tracing`（通常仅 root 可访问）
- **Top Interrupt Handlers**：按 Interrupt Time 排序列出最活跃的中断处理程序，点击可展开按 Interrupt/Interrupt Duration Type/Function/Call Stack 分组的网格视图
- **Interrupt Duration Histogram**：按时长分布展示中断处理实例数，慢速实例数量偏多通常意味着性能瓶颈

### 延迟分析（Hardware Tracing 模式）
- 提供系统级、微秒粒度的时间线数据：模块边界、模块入口点函数名、线程 active/idle 时间、时间线上的中断、模块的用户态/内核态执行时间
- 适合定位"为什么这段时间卡顿"之类的具体延迟成因，而不仅是整体统计

### 相关指标
- **I/O Wait Time**（Linux 目标）：线程处于 I/O 等待状态、同时系统有空闲核心时的时间占比。若该指标在 Elapsed Time 里占比显著，应切换到 Platform 窗口查看时间线上的全部指标，排查具体诱因
- **LLC Miss**：末级缓存（LLC）未命中、需要向本地/远程 DRAM 取数的周期占比，是高延迟的常见来源之一

### Linux Ftrace 事件（HWE 模式采集）
- `sched`、`freq`、`idle`、`workq`、`irq`、`softirq`

## 典型工作流

1. 用 Hardware Event-Based Sampling 模式做入口分析，快速看 CPU/GPU/DRAM/I/O/PCIe 整体利用率是否有明显短板
2. 若怀疑中断是瓶颈，检查 Top Interrupt Handlers 和 Interrupt Duration Histogram，定位是哪类中断、哪个处理函数耗时异常
3. 若需要定位具体的延迟尖峰成因，切到 Hardware Tracing 模式（Linux/Android），在微秒级时间线上回放当时模块/线程/中断的状态
4. I/O Wait Time 偏高时，去 Platform 窗口核对当时的 I/O/中断/线程活动，确认是否和中断、I/O 子系统相关

## Reference
- [System Overview Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2025-0/system-overview-analysis.html)
- [Analyze Interrupts](https://www.intel.com/content/www/us/en/develop/documentation/vtune-help/top/analyze-performance/platform-analysis-group/system-overview-analysis/analyzing-interrupts.html)
- [Analyze Latency Issues](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-2/analyze-latency-issues.html)

