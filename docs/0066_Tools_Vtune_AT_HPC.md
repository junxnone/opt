---
Title | Tools Vtune AT HPC
-- | --
Created @ | `2026-10-14T07:17:58Z`
Updated @| `2026-10-14T07:17:58Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/66)

---
# Vtune HPC Performance Characterization

- **HPC Performance Characterization 分析**：面向计算密集型/吞吐型应用（HPC、科学计算等）的综合体检，评估 CPU/内存/FPU 利用率，目标是提升 GFLOPS、缩短运行时间。属于 Parallelism Analysis Group，详见 [Analysis Type 索引](/0062_Tools_Vtune_AT)
- 常作为 HPC 应用优化的起点分析：先跑一遍，建立 Elapsed Time 和 GFLOPS 的基线，再决定后续该深入看向量化、OpenMP 还是 MPI

## 核心指标

### GFLOPS / 向量化
- **DP/SP GFLOPS**：每秒双精度/单精度浮点运算数，仅在特定处理器世代上可用（详见官方文档的处理器支持列表）
- **FP Vector**：CPU 执行的浮点向量 uop 占比，用来确认实际使用的向量指令集宽度是否符合预期
- Summary 窗口有独立的 **Vectorization** 小节：若 128-bit Packed FLOPs 等指标被标红，说明相当一部分浮点指令只用了部分向量宽度（partial vector load），是常见的向量化不充分信号

### CPU/核心利用率
- **Effective Physical Core Utilization**：展示实际使用的逻辑/物理核数量，估算核心利用率，核心利用率低会被直接标记为问题项

### OpenMP 专属指标
- **CPU Utilization (OpenMP)**：所有逻辑 CPU 的平均利用率，100% 表示全部逻辑核都在计算
- **OpenMP Potential Gain Imbalance**：若消除 OpenMP 构造里的负载不均衡，理论上能节省的最大 Elapsed Time；高值通常意味着需要改用动态调度（dynamic scheduling）
- **OpenMP Potential Gain Lock Contention**：OpenMP 锁和有序同步带来的耗时；高值提示并行化方式里同步对象竞争激烈

### 混合 MPI+OpenMP
- Critical Path 上的 "MPI Rank on Critical Path" 小节会展示该 Rank 的 Serial Time、OpenMP Potential Gain 等指标；若 MPI Busy Wait Time 显著，通常是通信模式不佳或其他节点负载不均衡导致，需要结合 Intel Trace Analyzer and Collector 做更深入分析

## 典型工作流

1. 先看 Elapsed Time + GFLOPS，作为优化前后的基线
2. 看 Effective Physical Core Utilization，确认核心是否跑满
3. 若怀疑向量化不充分，看 Vectorization 小节里的 Packed FLOPs 分布
4. 若是 OpenMP 应用，重点看 Imbalance 和 Lock Contention 两项 Potential Gain 指标，分别对应负载不均和锁竞争两类常见问题
5. 若是混合 MPI+OpenMP 应用，用 MPI Rank on Critical Path 定位瓶颈具体落在哪个 Rank

## CLI 示例

```bash
vtune -collect hpc-performance -- ./your_app
```

## Reference
- [HPC Performance Characterization View](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-2/hpc-performance-characterization-view.html)
- [OpenMP* Code Analysis Method](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-0/openmp-code-analysis-method.html)
- [CPU Metrics Reference](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-1/cpu-metrics-reference.html)

