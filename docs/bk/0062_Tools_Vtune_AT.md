-----

| Title     | Tools Vtune AT                                    |
| --------- | ------------------------------------------------- |
| Created @ | `2024-04-21T03:01:03Z`                            |
| Updated @ | `2026-10-14T07:30:19Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/62) |

-----

# Vtune Analysis Type

VTune Profiler 把众多分析能力组织成若干 **Analysis Group**，每个 Group 下有具体的 **Analysis
Type**。本页作为索引，简述每类分析的用途、对应入口文档，并给出"遇到什么问题该选哪个"的参考表。

> 以下以 2026.1 文档为准；VTune 的分析类型命名会随版本演进（合并/改名/废弃），本页会随官方文档更新同步调整。

## 分析类型一览

### Algorithm（算法/热点类）

  - [Hotspots](/0057_Tools_Vtune_AT_Hotspots)：定位最耗时的函数/模块，是最常用的入门分析
  - [Anomaly
    Detection](/0061_Tools_Vtune_AT_AD)（预览特性）：基于硬件采样自动发现异常的性能模式，无需预设基线
  - [Memory
    Consumption](/0060_Tools_Vtune_AT_MC)：按时间线跟踪进程的内存占用与分配调用栈，定位内存增长/泄漏（**仅支持
    Linux**，不支持 Windows/Android）

### Microarchitecture（微架构类）

  - [Microarchitecture
    Exploration](/0058_Tools_Vtune_AT_Microarchitecture)：基于
    [TMA（Top-Down Microarchitecture Analysis）](/0047_OPT_TMA) 方法论，把
    CPU 瓶颈归类到 Retiring / Bad Speculation / Front-End Bound / Back-End
    Bound
  - [Memory Access](/0059_Tools_Vtune_AT_MA)：细化到 Cache(L1/L2/L3)/DRAM
    带宽、NUMA、False Sharing 等，适合 Memory Bound 的应用

### Parallelism（并行类）

  - [Threading](/0065_Tools_Vtune_AT_Threading)：分析多线程同步开销、线程利用率与扩展性瓶颈
  - [HPC Performance
    Characterization](/0066_Tools_Vtune_AT_HPC)：面向计算密集型/吞吐型应用，综合评估
    CPU/FPU/内存利用率、OpenMP 效率与向量化情况，目标是提升 GFLOPS

### Input and Output（I/O 类）

  - [I/O](/0067_Tools_Vtune_AT_IO)：分析存储/网络 I/O 子系统与 CPU、总线的交互，含
    DPDK/SPDK/Linux Kernel I/O 等细分场景（**仅支持 Linux/FreeBSD**，完整指标集仅限 Intel
    Xeon 处理器）

### Accelerators（加速器类）

  - [XPU Offload Analysis](/0068_Tools_Vtune_AT_XPU)（预览特性）：统一视角评估
    CPU/GPU/NPU 的 offload 效果，判断应用是 CPU-bound 还是 XPU-bound，评估
    SYCL/OpenMP/OpenCL kernel 的 offload 效果。自 2026.1 起取代并合并了已废弃的 GPU
    Offload Analysis 和 NPU Exploration Analysis（过渡期可先看 [iGPU
    Analysis](/0009_Tools_Vtune_iGPU)）
  - [GPU Hotspots](/0069_Tools_Vtune_AT_GPUHotspots)（原名 GPU
    Compute/Media Hotspots，现已转正、不再是预览特性）：XPU Offload 之后的深入分析，定位 GPU
    端具体热点
  - ~~CPU/FPGA Interaction~~：已于 2025.1 标记为
    **deprecated**，计划在未来版本移除，不建议新项目使用；FPGA
    profiling 需求请联系 Altera oneAPI 团队（<oneapi@altera.com>）

### Platform（平台类）

  - [System Overview](/0070_Tools_Vtune_AT_SystemOverview)：系统级中断与延迟问题分析
  - ~~Platform Profiler~~：已于 2023.2 从 VTune 的 GUI/CLI
    中移除，此后仅作为独立下载工具存在，2025
    发布最终版本后能力已全部转移到 **EMON** 命令行工具，不再是 VTune 的
    Analysis Type

### 其他入口

  - [Performance
    Snapshot](/0051_Tools_Vtune_AT_PS)：不确定问题类型时的综合体检，会给出下一步该用哪个具体
    Analysis Type 的建议
  - Custom Analysis：自定义硬件事件采集，用于以上预设类型无法覆盖的场景

## 怎么选：问题类型 → 推荐分析类型

| 遇到的问题/现象                                           | 推荐分析类型                                                                                           |
| -------------------------------------------------- | ------------------------------------------------------------------------------------------------ |
| 不确定问题出在哪，想先有个全局判断                                  | [Performance Snapshot](/0051_Tools_Vtune_AT_PS)                                                  |
| 想知道时间都花在哪些函数/调用栈上                                  | [Hotspots](/0057_Tools_Vtune_AT_Hotspots)                                                        |
| 怀疑是 CPU 微架构层面的瓶颈（流水线停顿、分支预测错误等）                    | [Microarchitecture Exploration](/0058_Tools_Vtune_AT_Microarchitecture)                          |
| 怀疑是内存访问模式问题（Cache Miss、带宽、NUMA、False Sharing）      | [Memory Access](/0059_Tools_Vtune_AT_MA)                                                         |
| 内存占用持续增长，怀疑泄漏或分配不合理                                | [Memory Consumption](/0060_Tools_Vtune_AT_MC)                                                    |
| 没有明确假设，想让工具自动找异常模式                                 | [Anomaly Detection](/0061_Tools_Vtune_AT_AD)                                                     |
| 多线程程序扩展性差、锁竞争严重                                    | [Threading](/0065_Tools_Vtune_AT_Threading)                                                      |
| HPC/科学计算场景，想综合看 CPU/内存利用率和向量化效果                    | [HPC Performance Characterization](/0066_Tools_Vtune_AT_HPC)                                     |
| 怀疑瓶颈在磁盘/网络 I/O                                     | [I/O](/0067_Tools_Vtune_AT_IO)                                                                   |
| 用了 SYCL/OpenMP/OpenCL，想确认 CPU/GPU/NPU offload 是否有效 | [XPU Offload Analysis](/0068_Tools_Vtune_AT_XPU)（过渡期可先看 [iGPU Analysis](/0009_Tools_Vtune_iGPU)） |
| 已确认 XPU-bound，需要定位 GPU 端具体热点                       | [GPU Hotspots](/0069_Tools_Vtune_AT_GPUHotspots)                                                 |
| 关注系统级中断、延迟                                         | [System Overview](/0070_Tools_Vtune_AT_SystemOverview)                                           |
| 想做跨节点/跨进程的平台级性能画像                                  | 改用独立工具 **EMON**（Platform Profiler 已停止集成在 VTune 中）                                                |

## 近期命名/废弃变更（截至 2026.1）

  - **GPU Offload Analysis**、**NPU Exploration Analysis** → 已废弃，合并为统一的
    **XPU Offload Analysis**
  - **GPU Compute/Media Hotspots**（曾为预览特性）→ 转正为正式分析类型 **GPU Hotspots**
  - **CPU/FPGA Interaction** → 2025.1 起标记为 deprecated，计划移除
  - **Platform Profiler** → 2023.2 起移出 VTune（GUI/CLI
    均不再支持），转为独立下载工具，2025 发布最终版本，能力转移到
    **EMON** 命令行工具

## Reference

  - [Intel® VTune™ Profiler User Guide - Analyze
    Performance](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/overview.html)
  - [Intel® VTune™ Profiler Release Notes and New
    Features](https://www.intel.com/content/www/us/en/developer/articles/release-notes/vtune-profiler/current.html)
