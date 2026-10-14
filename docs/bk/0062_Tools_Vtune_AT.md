-----

| Title     | Tools Vtune AT                                    |
| --------- | ------------------------------------------------- |
| Created @ | `2024-04-21T03:01:03Z`                            |
| Updated @ | `2026-10-14T03:01:11Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/62) |

-----

# Vtune Analysis Type

VTune Profiler 把众多分析能力组织成若干 **Analysis Group**，每个 Group 下有具体的 **Analysis
Type**。本页作为索引，简述每类分析的用途、对应入口文档，并给出"遇到什么问题该选哪个"的参考表。

## 分析类型一览

### Algorithm（算法/热点类）

  - [Hotspots](/0057_Tools_Vtune_AT_Hotspots)：定位最耗时的函数/模块，是最常用的入门分析
  - Anomaly Detection（预览特性）：基于硬件采样自动发现异常的性能模式，无需预设基线 —
    [文档](/0061_Tools_Vtune_AT_AD)
  - [Memory
    Consumption](/0060_Tools_Vtune_AT_MC)：按时间线跟踪进程的内存占用与分配调用栈，定位内存增长/泄漏

### Microarchitecture（微架构类）

  - [Microarchitecture
    Exploration](/0058_Tools_Vtune_AT_Microarchitecture)：基于
    [TMA（Top-Down Microarchitecture Analysis）](/0047_OPT_TMA) 方法论，把
    CPU 瓶颈归类到 Retiring / Bad Speculation / Front-End Bound / Back-End
    Bound
  - [Memory Access](/0059_Tools_Vtune_AT_MA)：细化到 Cache(L1/L2/L3)/DRAM
    带宽、NUMA、False Sharing 等，适合 Memory Bound 的应用

### Parallelism（并行类）

  - Threading：分析多线程同步开销、线程利用率与扩展性瓶颈（暂无独立文档，待补充）
  - HPC Characterization：面向计算密集型应用，综合评估 CPU/GPU 利用率、OpenMP
    效率与向量化情况（暂无独立文档，待补充）

### Input and Output（I/O 类）

  - I/O：分析存储/网络 I/O 子系统与 CPU、总线的交互，含 DPDK/SPDK/Linux Kernel I/O
    等细分场景（暂无独立文档，待补充）

### Accelerators（加速器类）

  - GPU Offload：判断应用是 CPU-bound 还是 GPU-bound，评估 SYCL/OpenMP/OpenCL
    kernel 的 offload 效果（暂无独立文档，待补充；参考 [iGPU
    Analysis](/0009_Tools_Vtune_iGPU)）
  - GPU Compute/Media Hotspots（预览特性）：GPU Offload 之后的深入分析，定位 GPU
    端具体热点（暂无独立文档，待补充）
  - CPU/FPGA Interaction：分析 CPU 与 FPGA 协同工作时的交互效率（暂无独立文档，待补充）

### Platform（平台类）

  - System Overview：系统级中断与延迟问题分析（暂无独立文档，待补充）
  - Platform Profiler：跨进程/跨节点的平台级性能画像，含 Hybrid CPU 与 Source Code
    分析（暂无独立文档，待补充）

### 其他入口

  - [Performance
    Snapshot](/0051_Tools_Vtune_AT_PS)：不确定问题类型时的综合体检，会给出下一步该用哪个具体
    Analysis Type 的建议
  - Custom Analysis：自定义硬件事件采集，用于以上预设类型无法覆盖的场景

## 怎么选：问题类型 → 推荐分析类型

| 遇到的问题/现象                                      | 推荐分析类型                                                                  |
| --------------------------------------------- | ----------------------------------------------------------------------- |
| 不确定问题出在哪，想先有个全局判断                             | [Performance Snapshot](/0051_Tools_Vtune_AT_PS)                         |
| 想知道时间都花在哪些函数/调用栈上                             | [Hotspots](/0057_Tools_Vtune_AT_Hotspots)                               |
| 怀疑是 CPU 微架构层面的瓶颈（流水线停顿、分支预测错误等）               | [Microarchitecture Exploration](/0058_Tools_Vtune_AT_Microarchitecture) |
| 怀疑是内存访问模式问题（Cache Miss、带宽、NUMA、False Sharing） | [Memory Access](/0059_Tools_Vtune_AT_MA)                                |
| 内存占用持续增长，怀疑泄漏或分配不合理                           | [Memory Consumption](/0060_Tools_Vtune_AT_MC)                           |
| 没有明确假设，想让工具自动找异常模式                            | [Anomaly Detection](/0061_Tools_Vtune_AT_AD)                            |
| 多线程程序扩展性差、锁竞争严重                               | Threading                                                               |
| HPC/科学计算场景，想综合看 CPU/GPU 利用率和向量化效果             | HPC Characterization                                                    |
| 怀疑瓶颈在磁盘/网络 I/O                                | I/O                                                                     |
| 用了 SYCL/OpenMP/OpenCL，想确认 GPU offload 是否有效    | GPU Offload（过渡期可先看 [iGPU Analysis](/0009_Tools_Vtune_iGPU)）             |
| 已确认 GPU-bound，需要定位 GPU 端具体热点                  | GPU Compute/Media Hotspots                                              |
| 关注系统级中断、延迟或跨节点平台画像                            | System Overview / Platform Profiler                                     |

> 标注"暂无独立文档，待补充"的分析类型是当前文档体系里的空白，后续会逐步补齐对应子页。

## Reference

  - [Intel® VTune™ Profiler User Guide - Analyze
    Performance](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/overview.html)
