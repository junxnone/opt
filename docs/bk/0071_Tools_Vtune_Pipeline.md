-----

| Title     | Tools Vtune Pipeline                              |
| --------- | ------------------------------------------------- |
| Created @ | `2026-10-15T06:04:20Z`                            |
| Updated @ | `2026-10-15T06:50:35Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/71) |

-----

# Tools Vtune Pipeline

本页用一张决策流程图，把 [Analysis Type 索引](0062_Tools_Vtune_AT)
里分散的各个分析类型串成一条可以实际走下去的排查路径：先怎么做、途中会碰到哪些现象、该往哪个分支走、最终定位到什么粒度的问题。图里只画流程分支，每类分析的指标原理不重复展开，均链接到对应详情页。

## 决策流程图

[全屏查看](https://junxnone.github.io/jstools/mermaid/?md=https://raw.githubusercontent.com/junxnone/opt/refs/heads/main/docs/0071_Tools_Vtune_Pipeline.md)
↓ mermaid code

``` mermaid
flowchart TD
    Start(["开始: 发现/怀疑性能问题"]) --> Q1{"是否已有明确怀疑方向?"}

    Q1 -->|"不确定, 想先全局体检"| PS["Performance Snapshot"]
    Q1 -->|"怀疑内存持续增长/泄漏"| MC["Memory Consumption"]
    Q1 -->|"多线程扩展性差/锁竞争"| TH["Threading"]
    Q1 -->|"HPC/科学计算, 关注GFLOPS"| HPC["HPC Performance<br/>Characterization"]
    Q1 -->|"怀疑磁盘/网络I-O瓶颈"| IO["I-O Analysis"]
    Q1 -->|"用了SYCL/OpenMP/OpenCL异构计算"| XPU["XPU Offload Analysis"]
    Q1 -->|"关注系统级中断/延迟"| SO["System Overview"]
    Q1 -->|"没有明确假设, 想自动找异常"| AD["Anomaly Detection"]
    Q1 -->|"预设类型覆盖不了, 需自定义硬件事件"| Custom["Custom Analysis"]

    Custom --> CustomResult[["定位: 自行选择PMU事件<br/>组合排查特定硬件事件"]]

    PS --> PSQ{"Analysis Tree标红<br/>提示是什么?"}
    PSQ -->|"CPU Utilization低 且 多线程"| TH
    PSQ -->|"Memory Bound-Memory Stalls高"| HS["Hotspots"]
    PSQ -->|"Vectorization低"| HPC
    PSQ -->|"检测到GPU Utilization"| XPU
    PSQ -->|"无明显标红, 仍需定位热点"| HS

    HS --> HSMode{"采集模式"}
    HSMode -->|"默认, HWE可用"| HSView["Bottom-up / Caller-Callee /<br/>Top-down Tree / Flame Graph"]
    HSMode -->|"权限/平台不支持HWE"| UMS["降级为UMS采样"]
    UMS --> HSView

    HSView --> MX["Microarchitecture Exploration<br/>对热点函数细化TMA分类"]
    MX --> MXQ{"哪个TMA一级分类<br/>占比最大?"}
    MXQ -->|"Memory Bound高"| MA["Memory Access"]
    MXQ -->|"Core Bound高"| CoreResult[["定位: 执行单元资源争用-<br/>依赖链过长-向量化不足"]]
    MXQ -->|"Front-End Bound高"| FEResult[["定位: I-Cache-DSB miss,<br/>代码体积过大"]]
    MXQ -->|"Bad Speculation高"| BSResult[["定位: 分支预测失败,<br/>分支密集代码路径"]]
    MXQ -->|"Retiring高但仍然慢"| AlgoResult[["定位: 硬件已高效利用,<br/>需回到算法层(0003 Roofline)"]]

    MA --> MAQ{"具体是哪种访存模式?"}
    MAQ -->|"L1-L2-L3 Bound高"| CacheResult[["定位: Cache命中率低-<br/>访问模式不连续"]]
    MAQ -->|"DRAM Bound高 且 带宽利用率接近峰值"| BWResult[["定位: 内存带宽瓶颈,<br/>需减少总访存量"]]
    MAQ -->|"多线程集中访问同一小块内存"| FSResult[["定位: False Sharing"]]
    MAQ -->|"NUMA远端DRAM访问占比高"| NUMAResult[["定位: NUMA远端访问延迟高"]]

    TH --> THQ{"核心指标里<br/>哪项异常?"}
    THQ -->|"Effective CPU Utilization低<br/>且Thread Count固定"| OversubResult[["定位: 线程数固定限制扩展-<br/>过度订阅"]]
    THQ -->|"Wait Time高"| LockResult[["定位: 具体锁-Barrier竞争<br/>的调用栈"]]
    THQ -->|"Spin Time-Overhead Time高"| SpinResult[["定位: 忙等或任务调度开销"]]
    THQ -->|"Inactive Wait Count高"| CtxResult[["定位: 同步竞争频繁或<br/>线程过度订阅引发上下文切换"]]

    HPC --> HPCQ{"哪项指标异常?"}
    HPCQ -->|"Effective Physical Core<br/>Utilization低"| TH
    HPCQ -->|"Packed FLOPs偏低<br/>(部分向量宽度)"| VecResult[["定位: 向量化不充分"]]
    HPCQ -->|"OpenMP Potential Gain<br/>Imbalance高"| ImbalanceResult[["定位: OpenMP负载不均,<br/>需改用动态调度"]]
    HPCQ -->|"OpenMP Potential Gain<br/>Lock Contention高"| OMPLockResult[["定位: OpenMP锁-<br/>有序同步竞争"]]
    HPCQ -->|"MPI Busy Wait Time高"| MPIResult[["定位: MPI通信模式不佳-<br/>节点负载不均"]]

    IO --> IOQ{"怀疑哪类I-O?"}
    IOQ -->|"平台层异常<br/>(PCIe-DDIO-内存带宽-UPI)"| PlatformIOResult[["定位: 平台层I-O带宽瓶颈"]]
    IOQ -->|"应用基于DPDK"| DPDKQ{"Rx-Event Dequeue<br/>Spin Time高?"}
    DPDKQ -->|"是"| DPDKResult[["定位: 轮询线程大量空转,<br/>event pipeline负载不均"]]
    IOQ -->|"应用基于SPDK"| SPDKQ{"SPDK Latency高<br/>Effective Time低?"}
    SPDKQ -->|"是"| SPDKResult[["定位: SPDK设备吞吐未打满"]]
    IOQ -->|"怀疑内核I-O路径"| KernelIOResult[["定位: 开启kernel-stack<br/>细查内核I-O栈"]]

    XPU --> XPUQ{"Host端耗时高<br/>还是Device端耗时高?"}
    XPUQ -->|"Host端高, 怀疑offload<br/>调用开销大"| APIResult[["定位: 开启Trace computing<br/>programming APIs细查调用耗时"]]
    XPUQ -->|"需定位发起offload<br/>的代码位置"| HostStackResult[["定位: 开启Collect host stacks"]]
    XPUQ -->|"确认是XPU-bound"| GH["GPU Hotspots"]

    GH --> GHQ{"EU指标异常?"}
    GHQ -->|"EU Active低-Stalled高"| GPUMemResult[["定位: GPU内存延迟瓶颈,<br/>查Memory Hierarchy Diagram"]]
    GHQ -->|"EU Threads Occupancy低"| WorkGroupResult[["定位: work-group-work-item<br/>划分不合理-任务分布不均"]]

    SO --> SOMode{"采集模式"}
    SOMode -->|"Hardware Event-Based<br/>Sampling入口"| SOQ{"哪项利用率-<br/>指标异常?"}
    SOQ -->|"Top Interrupt Handlers耗时高"| IRQResult[["定位: 具体中断处理<br/>函数耗时异常"]]
    SOQ -->|"I-O Wait Time高<br/>且有空闲核心"| IOWaitResult[["定位: 需结合Platform窗口<br/>核对I-O-中断-线程活动"]]
    SOQ -->|"LLC Miss高"| LLCResult[["定位: 末级缓存未命中"]]
    LLCResult --> MA
    SOMode -->|"需微秒级延迟成因<br/>(仅Linux-Android)"| SOTrace["Hardware Tracing模式"]
    SOTrace --> LatencyResult[["定位: 时间线回放<br/>具体延迟尖峰成因"]]

    AD --> ADQ{"对比正常-异常实例的<br/>Intel PT Details"}
    ADQ -->|"Wait Time占比高"| CtxAnomalyResult[["定位: Context Switch Anomaly,<br/>同步等待引发"]]
    ADQ -->|"Kernel Time占比高"| KernelAnomalyResult[["定位: Kernel-Induced Anomaly,<br/>中断-缺页等内核开销"]]
    ADQ -->|"指令数相近但变慢"| FreqAnomalyResult[["定位: Frequency Drops,<br/>睿频下降-AVX降频"]]
    ADQ -->|"实际走了不同代码路径"| CFAnomalyResult[["定位: Control Flow Deviation,<br/>触发异常-罕见分支"]]

    MC --> MCQ{"Allocation-Deallocation<br/>Delta持续增长?"}
    MCQ -->|"是"| LeakResult[["定位: 内存泄漏,<br/>定位具体分配源码行"]]
    MCQ -->|"否, 只是占用高峰"| PeakResult[["定位: 某时间段内存峰值,<br/>Filter In by Selection圈定函数"]]

    CustomResult --> Fix["针对性优化<br/>(0003 优化方法论 步骤3)"]
    CoreResult --> Fix
    FEResult --> Fix
    BSResult --> Fix
    AlgoResult --> Fix
    CacheResult --> Fix
    BWResult --> Fix
    FSResult --> Fix
    NUMAResult --> Fix
    OversubResult --> Fix
    LockResult --> Fix
    SpinResult --> Fix
    CtxResult --> Fix
    VecResult --> Fix
    ImbalanceResult --> Fix
    OMPLockResult --> Fix
    MPIResult --> Fix
    PlatformIOResult --> Fix
    DPDKResult --> Fix
    SPDKResult --> Fix
    KernelIOResult --> Fix
    APIResult --> Fix
    HostStackResult --> Fix
    GPUMemResult --> Fix
    WorkGroupResult --> Fix
    IRQResult --> Fix
    IOWaitResult --> Fix
    LatencyResult --> Fix
    CtxAnomalyResult --> Fix
    KernelAnomalyResult --> Fix
    FreqAnomalyResult --> Fix
    CFAnomalyResult --> Fix
    LeakResult --> Fix
    PeakResult --> Fix

    Fix --> Verify["验证回归<br/>(0003 优化方法论 步骤4)"]
    Verify --> Start
```

## 图中节点对应文档

| 分析类型/环节                                 | 文档                                                                   |
| --------------------------------------- | -------------------------------------------------------------------- |
| Performance Snapshot                    | [0051](0051_Tools_Vtune_AT_PS)                                       |
| Hotspots                                | [0057](0057_Tools_Vtune_AT_Hotspots)                                 |
| HWE / UMS 采集模式                          | [0053](0053_Tools_Vtune_HWE) / [0052](0052_Tools_Vtune_UMS)          |
| Microarchitecture Exploration / TMA 方法论 | [0058](0058_Tools_Vtune_AT_Microarchitecture) / [0047](0047_OPT_TMA) |
| Memory Access                           | [0059](0059_Tools_Vtune_AT_MA)                                       |
| Memory Consumption                      | [0060](0060_Tools_Vtune_AT_MC)                                       |
| Anomaly Detection                       | [0061](0061_Tools_Vtune_AT_AD)                                       |
| Threading                               | [0065](0065_Tools_Vtune_AT_Threading)                                |
| HPC Performance Characterization        | [0066](0066_Tools_Vtune_AT_HPC)                                      |
| I/O Analysis                            | [0067](0067_Tools_Vtune_AT_IO)                                       |
| XPU Offload Analysis                    | [0068](0068_Tools_Vtune_AT_XPU)                                      |
| GPU Hotspots                            | [0069](0069_Tools_Vtune_AT_GPUHotspots)                              |
| System Overview                         | [0070](0070_Tools_Vtune_AT_SystemOverview)                           |
| Analysis Type 总索引（含"问题→推荐分析类型"对照表）      | [0062](0062_Tools_Vtune_AT)                                          |
| 优化方法论（图末尾"针对性优化/验证回归"两步）                | [0003](0003_OPT)                                                     |

## 几点说明

  - 图里的分支条件均取自对应分析类型文档里已有的"核心指标"和"典型工作流"章节，这里只是把它们串成一条可以跟着走的路径，具体指标含义/阈值判断方法见各自页面。
  - 不是严格的决策树：例如 System Overview 发现 LLC Miss 高会汇入 Memory Access 继续细化，HPC
    Performance Characterization 发现核心利用率低会汇入
    Threading——这类交叉在实际排查中很常见，图里用边直接连接到对应节点，而不是重复画一份。
  - 所有"定位:"开头的叶子节点代表一次排查的终点，之后统一进入 [0003 优化方法论](0003_OPT) 的"针对性优化 →
    验证回归"两步，验证后若问题仍在或出现新瓶颈，回到最开始重新走一遍这条流程。
  - 以下场景目前在 VTune 里已废弃/转移，不在图中单独列出：**CPU/FPGA Interaction**（2025.1 起
    deprecated）、**Platform Profiler**（已移出 VTune，能力转移到独立工具 EMON），详见
    [0062](0062_Tools_Vtune_AT) 的"近期命名/废弃变更"一节。

## Reference

  - [Intel® VTune™ Profiler User Guide - Analyze
    Performance](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/overview.html)
  - [Intel VTune Profiler Performance Analysis
    Cookbook](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook)
