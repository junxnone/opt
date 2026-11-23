-----

| Title     | Tools Vtune HWE                                   |
| --------- | ------------------------------------------------- |
| Created @ | `2024-04-20T17:53:17Z`                            |
| Updated @ | `2026-10-14T09:05:16Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/53) |

-----

# Hardware Event-based Sampling Collection

  - 使用 PMU(Performance Monitoring Unit) 的 计数器溢出来采样
  - `1ms` 采样间隔时, 消耗 2% 的资源
  - 支持大部分分析模式

## 原理

  - CPU 的 PMU 可以配置若干硬件计数器（如 cycles、instructions retired、cache misses
    等），当某个计数器累计到预设阈值（溢出）时触发一次中断。
  - VTune
    在每次中断时打断目标进程，记录当时的指令指针（IP），由此统计出"哪些代码区域触发了最多的硬件事件"。这是一种**事件驱动**的采样：采样时机由硬件计数器溢出决定，而不是固定的时钟节拍。
  - 默认只采 IP，不解析调用栈；如果在分析配置里勾选 **Collect stacks**，VTune
    会在每次线程被调度上/下处理器（quantum
    边界）时额外抓取调用栈、时间戳等信息，代价是更高的开销。Linux
    下默认用无驱动的 Perf 模式采集带栈数据。
  - 单个 CPU 同时能采集的硬件计数器数量有限（通常 ≤ 4 个）。当需要的事件数超过这个上限时，VTune 用两种方式之一处理：
      - 多次运行目标程序，每次运行采集一组事件（`-allow-multiple-runs`）；
      - 只运行一次，把多组事件按时间片轮转复用（multiplexing），但并非所有 OS/硬件组合都支持。
  - 同一时刻系统上只允许一个基于 HWE 的采集任务在跑。
  - 依赖 PMU 硬件资源，因此在虚拟化环境中可能完全不可用：若 BIOS 关闭了 PMU 直通，或者在不受支持的平台上开启了
    Hyper-V，会报 "Cannot Enable Event-Based Sampling Collection" 错误。

## 与 SW（UMS，用户态采样）采样的区别

VTune 的 [`-knob sampling-mode=hw|sw`](0007_Tools_Vtune_CommandLine.md)
实际对应的就是本页（HWE）和 [0052 UMS](0052_Tools_Vtune_UMS.md) 两种底层采集器，两者的核心差异：

|         | HWE（硬件事件采样，本页）                                                                                                                                                                                                                        | SW / UMS（用户态采样）                                                           |
| ------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------- |
| 采样触发方式  | PMU 硬件计数器溢出中断                                                                                                                                                                                                                         | OS 定时器到期，给线程发 `SIGPROF`（Linux）等信号                                         |
| 典型开销    | `1ms` 间隔约 2%                                                                                                                                                                                                                          | 默认 `10ms` 间隔约 5%                                                          |
| 调用栈     | 默认不采，需显式开启 Collect stacks（额外开销）                                                                                                                                                                                                       | 默认就采集调用栈（每 10ms 展开一次，中间间隔靠历史匹配/猜测）                                        |
| 栈准确性    | 开启后在 quantum 边界精确抓栈，准确度更高                                                                                                                                                                                                             | 可能出现 `[Guessed frame(s)]`/`[Skipped frame(s)]`/`[Unknown frame(s)]`，是统计近似 |
| 平台依赖    | 依赖 CPU PMU + 驱动（或 Linux 下无驱动 Perf 模式）；虚拟化/云环境下可能因 PMU 不可用而完全无法使用                                                                                                                                                                      | 不依赖 PMU，只需能注入 agent 库 / 设置信号处理，跨平台可用性更好，是 HWE 不可用时的兜底方案                   |
| 支持的分析类型 | Hotspots（hw 模式）、Anomaly Detection、Performance Snapshot、Microarchitecture Exploration、Memory Access、System Overview、Threading、HPC Performance Characterization、GPU Compute/Media Hotspots（预览）、XPU Offload（预览）、CPU/FPGA Interaction（预览） | Hotspots、Threading、Memory Consumption                                     |
| 可精细到的指标 | 能归因到具体硬件事件（如 LLC Miss、分支预测失败等），是 Microarchitecture/Memory Access 等分析类型的数据来源                                                                                                                                                           | 只能看到"花了多少时间"，看不到具体硬件事件                                                    |

  - 经验法则：能用 HWE 就优先用 HWE（信息更丰富、能看微架构层面的瓶颈）；当目标跑在虚拟机/容器/云实例且无法拿到 PMU
    访问权限，或者 CPU 不支持所需的硬件事件时，才会退回到 SW/UMS 模式——`-collect
    hotspots -knob sampling-mode=sw` 就是手动强制走这条兜底路径。

## Reference

  - [Hardware Event-based Sampling Collection
    (2026-1)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/hw-event-based-sampling-collection.html)
  - [Hardware Event-based Sampling Collection with Stacks
    (2026-1)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/hw-event-based-sampling-collection-with-stacks.html)
  - [User-Mode Sampling and Tracing Collection
    (2026-1)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/user-mode-sampling-and-tracing-collection.html)
  - [Error Message: Cannot Enable Event-Based Sampling Collection
    (2026-1)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/error-pmu-resources-unavailable.html)
