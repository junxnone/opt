---
Title | Tools Vtune UMS
-- | --
Created @ | `2024-04-20T17:25:12Z`
Updated @| `2024-04-20T17:25:12Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/52)

---
# User-Mode Sampling and Tracing Colllection

- 该模式运行过程中，中断进程，收集活动的指令和调用堆栈，统计堆栈中的调用消耗的时间，以此来分析程序性能
- 当使用采样间隔为 `10ms` 时，采样跟踪系统约占用 `5%` 系统资源
- 支持的分析类型
  - Hotspots
  - Threading
  - Memory Consumption
- 可能会存在堆栈分析错误
  - 采样间隔影响堆栈的细粒度
