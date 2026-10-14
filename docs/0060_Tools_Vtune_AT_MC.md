---
Title | Tools Vtune AT MC
-- | --
Created @ | `2024-04-20T19:52:30Z`
Updated @| `2026-10-14T07:41:37Z`
Labels | `documentation`
Edit @| [here](https://github.com/junxnone/opt/issues/60)

---
# Vtune Memory Consumption

- **用途**：按时间线跟踪进程的内存占用（RAM）变化，识别分析期间分配/释放的内存对象及其分配调用栈，定位内存增长或泄漏。属于 Algorithm Analysis Group，详见 [Analysis Type 索引](/0062_Tools_Vtune_AT)
- **平台支持**：原生（native）目标**仅支持 Linux**；若分析对象是 **Python** 程序，则 Windows/Linux 均可
- 典型开销：采样方式采集，一般只带来约 5% 的运行时开销

## 工作原理

采集时，VTune 的数据采集器会拦截内存分配/释放事件，并为每次分配事件捕获一次调用栈（释放事件只记录释放发生的函数）。采集到的调用指令地址（IP）与调用栈会一起存入结果文件，供后续在结果视图中展示。

## 核心指标与概念

- **Dynamic 内存对象**：通过 `malloc`/`new` 等方式在堆上分配，按分配发生的源码行定位
- **Global 内存对象**：全局/静态变量，按模块名 + 变量名定位（如 `libiomp5.so!_kmp_avail_proc (4B)`）
- **Memory Object / Memory Object Allocation Source 分组**：前者按单次分配（调用点+大小）分组，后者按分配发生的位置分组
- **Minimal dynamic memory object size to track**：可配置的最小追踪对象大小，默认 32 字节，调大可降低插桩开销
- **Allocation/Deallocation Delta**：同一函数/调用树下分配总量与释放总量的差值，持续增长的 delta 是内存泄漏的典型信号
- 注意：VTune 报告的是按函数/调用树统计的内存分配总量，和 `top` 等系统工具按进程某一时刻的驻留内存（RES）是不同口径，两者数值不能直接对比

## 典型工作流

1. 默认打开 **Memory Consumption** viewpoint，先看 Summary 里耗内存最多的函数列表
2. 在 Timeline 面板上找到内存占用的峰值点，用 Filter In by Selection 圈定该时间段内执行的函数/模块/进程
3. 切到 Bottom-up 标签，选择 Memory Object 或 Memory Object Allocation Source 分组，核对具体的 Allocation/Deallocation Delta
4. 双击可疑函数跳转到 Source 视图，定位到具体分配内存的源码行

## CLI 示例

```bash
vtune -collect memory-consumption -- ./your_app
```

## Reference
- [Memory Consumption Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/memory-consumption-analysis.html)
- [Memory Consumption and Allocations View](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-1/memory-consumption-and-allocations-view.html)

