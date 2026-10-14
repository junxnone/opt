---
Title | Tools Vtune AT GPUHotspots
-- | --
Created @ | `2026-10-14T07:19:04Z`
Updated @| `2026-10-14T07:19:04Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/69)

---
# Vtune GPU Hotspots

- **原名 GPU Compute/Media Hotspots**（曾为预览特性），现已转正为正式分析类型，不再是 preview
- **XPU Offload Analysis 之后的深入分析**：定位最耗时的 GPU kernel，基于 GPU 硬件指标刻画 GPU 使用情况，识别内存延迟或低效算法导致的性能问题，分析 GPU 指令频率。属于 Accelerators Analysis Group，详见 [Analysis Type 索引](/0062_Tools_Vtune_AT)
- 适合 SYCL / OpenCL / Intel Media SDK 等在 GPU 上做大量计算的场景

## 核心指标

- **Summary 窗口**：展示整体及分 engine 的 GPU 使用率、EU（Execution Unit）空闲/停滞/活跃时间占比，以及内存访问等功能单元的统计
- **EU Occupancy**：kernel 耗时、实例数、SIMD 宽度、EU Array 的 active/stalled/idle 比例、GPU barrier/atomic 等
- **EU Threads Occupancy**：检测线程在 EU 间分布是否不均；occupancy 低是明显信号。EU 长期没有任务可执行时会进入 idle，对应 **EU Array Idle** 指标
- **Compute Basic 预设**：区分访问不同类型数据的 occupancy 情况，在 Occupancy 小节里标出低占用的 GPU 任务
- **Media 支持**：可分析 Intel Media SDK 任务的执行情况（仅限 Linux 目标）

## 典型工作流

1. 从 Summary 入手，看整体 GPU 使用率和 EU active/stalled/idle 占比，判断是计算瓶颈还是内存瓶颈
2. 列出耗时最长的 Top Computing Tasks，结合 EU Efficiency 和 Memory Stalls 指标初步分类问题
3. 若怀疑是内存瓶颈，用 Memory Hierarchy Diagram 查看数据传输/带宽情况，定位具体是哪一级存储或哪类数据访问导致瓶颈
4. 若 EU Threads Occupancy 低，检查 kernel 的 work-group/work-item 划分是否合理，是否存在任务分布不均

## CLI 示例

```bash
vtune -collect gpu-hotspots -- ./your_app
```

## Reference
- [GPU Compute/Media Hotspots Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-2/gpu-compute-media-hotspots-analysis.html)
- [GPU Metrics Reference](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-0/gpu-metrics-reference.html)
- [Software Optimization for Intel® GPUs](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2025-4/software-optimization-for-intel-gpus.html)

