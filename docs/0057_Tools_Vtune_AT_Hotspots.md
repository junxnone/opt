---
Title | Tools Vtune AT Hotspots
-- | --
Created @ | `2024-04-20T19:17:10Z`
Updated @| `2026-10-14T05:30:35Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/57)

---
# Vtune Hotspots

- **热点分析**: 定位最耗时的函数/模块，是最常用、开销也最低的入门分析类型，详见 [Analysis Type 索引](/0062_Tools_Vtune_AT)
- 了解应用流程：通过调用栈还原 `谁调用了谁`，建立对程序执行路径的整体认知
- 定位 `code` 问题：把 CPU 时间精确归因到具体函数/源码行

## 采集方式
- 默认走 Hardware Event-based Sampling（HWE，详见 [HWE 文档](/0053_Tools_Vtune_HWE)），基于硬件 PMU 采样、开销低
- 若当前权限/平台不支持 HWE，会自动降级为 User-Mode Sampling（UMS，详见 [UMS 文档](/0052_Tools_Vtune_UMS)），通过代码插桩采样实现

## Tab

- **Summary**: 本次采集结果的整体统计信息，是分析的起点
- **Bottom-up**: 按 Grouping（如 Function/Module/Source Line）聚合后展示热点函数，可展开查看调用栈，适合"哪个函数最热"类问题
- **Caller/Callee**: 以调用关系展示某函数的调用者和被调用者，适合沿调用链向上/向下追踪耗时来源
- **Top-down Tree**: 按调用树逐层展开，天然保留调用层级关系，适合理解耗时在调用树里是如何分布/传递的
- **Flame Graph**: 调用栈的可视化火焰图，宽度代表耗时占比，适合快速扫视整体热点分布
- **Platform**: 按时间线展示 CPU/线程/任务的活动情况，适合关联"什么时间点在做什么"

### Summary
- **Elapsed Time**: 本次采集的总耗时
- **Top Hotspots**: 花费 CPU 时间最长的函数列表，点击可直接跳转到 Bottom-up 视图定位源码
- **Top Tasks**: 耗时最长的 Task 列表（Task 指用 Task API 标记的代码区域，或系统层面的 Ftrace/Atrace 事件、OpenCL kernel 等），点击某个 Task 类型会按 Task Type 粒度切换到 Bottom-up/Event Count 等网格视图
- **Effective CPU Utilization Histogram**: 统计采集期间"同时有 N 个逻辑核在工作"的时间占比分布，用来判断并行度是否充分（例如单核跑满但其余核空闲，说明并行度不足）
- **Collection and Platform Info**: 本次采集的上下文信息，包含应用启动命令行、可执行文件路径、操作系统/平台等，用于在非本机查看结果时核对采集环境

## Reference
- [Hotspots View](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2025-1/hotspots-view.html)
- [Window: Summary - Hotspots](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-2/window-summary-hotspots.html)
- [Task Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/task-analysis.html)

