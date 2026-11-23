---
Title | Tools Vtune AT AD
-- | --
Created @ | `2024-04-20T20:10:49Z`
Updated @| `2026-10-14T06:22:14Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/61)

---
# Vtune Anomaly Detection

- **预览特性（preview）**：目前仍是预览阶段，采集结果的数据格式不保证向后兼容，未来版本可能变化甚至被移除
- 适用场景：排查**循环体/高频重复代码区域**里偶发的、难以用常规 Hotspots 定位的性能异常——例如同一段循环，大多数迭代很快，少数迭代异常慢
- 核心方法：对比同一代码区域里的"正常实例"和"异常实例"，找出两者在指令路径/耗时上的差异，而不是像 Hotspots 那样看总体耗时排名
- 技术基础：依赖 Intel Processor Trace（Intel PT）做纳秒级粒度的分支与时间追踪，硬件要求 Skylake 及更新的 Intel Xeon 处理器

## 四类异常

- **Context Switch Anomaly（上下文切换异常）**：线程因同步等待（锁/条件变量）被切出，导致该实例耗时显著高于其他实例。排查方法：看 Wait Time 是否占 Elapsed Time 的较大比例
- **Kernel-Induced Anomaly（内核引发异常）**：耗时主要花在内核态（中断、缺页等）。排查方法：看 Kernel Time 占比是否偏高；内核态代码本身不可插桩，所以 Call Count/Iteration Count 在内核节点上为 0，只有 Instructions Retired 有意义
- **Frequency Drops（频率下降）**：CPU 睿频因低利用率、散热、AVX 降频等原因在某个区间内下降，即使指令数相近也会变慢
- **Control Flow Deviation Anomaly（控制流偏移异常）**：同一任务的不同实例实际走了不同的代码路径/分支（例如触发了异常处理分支、罕见的内存/存储重新分配逻辑），导致耗时差异

## 典型工作流

1. 对怀疑有异常的循环/高频代码区域跑 Anomaly Detection
2. 在结果视图里找到耗时明显偏离正常值的"异常实例"
3. 对比异常实例与正常实例的 Intel PT Details，定位是上面四类中的哪一种
4. 针对性修复后重新采集，确认异常实例消失或耗时回归正常

## 命令行

```bash
vtune -collect anomaly-detection -result-dir r002ad ./app
```

## Reference
- [Anomaly Detection Analysis (preview)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/anomaly-detection-analysis.html)
- [Anomaly Detection View](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-1/anomaly-detection-view.html)
- [anomaly-detection Command Line Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2025-0/running-anomaly-detection-from-the-command-line.html)

