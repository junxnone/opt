---
Title | OPT TMA
-- | --
Created @ | `2022-04-27T01:48:41Z`
Updated @| `2026-10-14T06:30:51Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/47)

---
# TMA
- TMA - `Top-Down Microarchitecture Analysis`


## 层次结构/Hierarchy
- 利用 `TMA` 结果进行分析的时候，我们只需要关注第一层级中比重最高的分支，然后逐级向下去追溯最关键的性能瓶颈便可.
  - 例如 如果 `Backend Bound` 占用较多，则向下看 `Core Bound` 和 `Memory Bound` 哪个占用的更多，如果是 `Memory Bound`, 则是因为 `CPU` 不能够及时的从 `Memory` 获取所需的数据导致的，继续查看是 Cache/DRAM/... 哪个的问题

![image](https://user-images.githubusercontent.com/2216970/165234008-d2226b5f-9fee-4eb7-8b2a-2fa14a24ddc4.png)

### 第一层: CPU 状态分类

- **Retiring**: 正常执行的 μOps 比例
- **Bad Speculation**: Cancelled ( `Mispredicted branches`/`Incorrect data speculation`)
- **Front-End Bound**: front-end 导致的 pipeline slots 不能被充分使用
- **Back-End Bound**: back-end 导致的 pipeline slots 不能被充分使用

### 第二层: 继续向下拆分

第一层四个分类每个都会再细分，帮助进一步定位根因：

| 第一层 | 第二层 | 含义 |
| --- | --- | --- |
| Front-End Bound | **Fetch Latency**: 取指延迟 | 典型如 i-cache miss、分支预测错误恢复期间的取指阻塞 |
| Front-End Bound | **Fetch Bandwidth**: 取指带宽 | 指令译码器（decoder）供给 μOps 的效率不足 |
| Bad Speculation | **Branch Mispredicts**: 分支预测错误 | 预测错误导致流水线被清空重来 |
| Bad Speculation | **Machine Clears**: 机器清空 | 非分支预测错误引起的流水线清空（如内存序冲突） |
| Back-End Bound | **Core Bound**: 执行单元/计算资源受限 | 执行端口利用不均衡，或长延迟的除法等指令导致的执行停滞；继续细分为 **Divider**（除法器占用）和 **Ports Utilization**（端口利用率） |
| Back-End Bound | **Memory Bound**: 访存受限 | 继续细分为 L1/L2/L3 Bound、DRAM Bound、Store Bound，详见 [Memory Access](/0059_Tools_Vtune_AT_MA) |

- Memory Bound 的详细拆分（L1/L2/L3/DRAM/NUMA/False Sharing 等）本页不重复，参见 [Memory Access](/0059_Tools_Vtune_AT_MA)
- 在 VTune 里查看 TMA 结果、判断某个分支是否"偏高"的具体方法，参见 [Microarchitecture Exploration](/0058_Tools_Vtune_AT_Microarchitecture)



## 相关名词
- **μOps**: `micro-ops/micro-operations` 微指令
- **DSB**: `Decoded-μop Stream Buffer`

### Pipeline Slots
- **Pipeline Slots**: 处理一个 μOps 需要的硬件资源
  - 对于每个 `CPU Core`, 在每个时钟周期, 有多个可用的 `Pipeline Slots`, 数量称为 `Pipeline Width`

40 Pipeline Slots = 4-wide CPU X 10 Clock cycles | Retiring(50%) = SlotsRetired/TotalSlots
-- | --
![image](https://user-images.githubusercontent.com/2216970/165475906-e72622c8-af6f-4f73-8982-a12051fde74e.png) | ![image](https://user-images.githubusercontent.com/2216970/165475920-9ef67a82-8bbb-4fc9-b090-8da041b85dd4.png)

## Reference
- [TMA_自顶向下的CPU架构性能瓶颈分析方法](https://zhuanlan.zhihu.com/p/60569271)
- [《A Top-Down Method for Performance Analysis and Counters Architecture》阅读笔记](https://andrewei1316.github.io/2020/12/20/top-down-performance-analysis/)
- [自顶向下的微架构分析方法 - Intel](https://www.intel.com/content/www/us/en/develop/documentation/vtune-cookbook-zh-cn/top/methodologies/top-down-microarchitecture-analysis-method.html)
2014 **TMA** [A Top-Down method for performance analysis and counters architecture.pdf](https://github.com/junxnone/linuxwiki/files/8559242/A.Top-Down.method.for.performance.analysis.and.counters.architecture.pdf)
- [Pipeline Slots](https://www.intel.com/content/www/us/en/develop/documentation/vtune-help/top/reference/cpu-metrics-reference/pipeline-slots.html)
- [Instructions Retired Event](https://www.intel.com/content/www/us/en/develop/documentation/vtune-help/top/analyze-performance/custom-analysis/custom-analysis-options/hardware-event-list/instructions-retired-event.html)
- [Top-down Microarchitecture Analysis Method (Cookbook)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-0/top-down-microarchitecture-analysis-method.html)

