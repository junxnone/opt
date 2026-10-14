-----

| Title     | Tools Vtune AT Microarchitecture                  |
| --------- | ------------------------------------------------- |
| Created @ | `2024-04-20T19:32:56Z`                            |
| Updated @ | `2026-10-14T06:10:52Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/58) |

-----

# Vtune Microarchitecture

  - 全称 Microarchitecture Exploration（原名 General Exploration），用于排查 CPU
    微架构层面的硬件瓶颈
  - 典型使用顺序：先用 [Hotspots](/0057_Tools_Vtune_AT_Hotspots) 定位到热点函数，再对该函数跑
    Microarchitecture Exploration，确认具体是哪类硬件瓶颈
  - 底层方法论是 [TMA（Top-Down Microarchitecture
    Analysis）](/0047_OPT_TMA)，本页不重复其四分类（Retiring / Bad
    Speculation / Front-End Bound / Back-End Bound）的原理解释，只说明 VTune
    这个分析类型特有的呈现方式
  - 硬件要求：基于 Ivy Bridge 及之后的 Intel 微架构预置的 Top-Down 事件采集，更早的 CPU 不支持该分析类型

## 这个分析类型特有的视图

  - **Summary 窗口**: 用一张"µPipe"管道图展示整机在各高层分类上的 Pipeline Slots
    占比——管道变窄的位置就是被该类别"卡住"的部分；Retiring
    在中间（绿色，占满代表效率高），Memory Bound/Core Bound 分别在上下
  - **Bottom-up 分层列**: 结果按 TMA 层级组织成多级列（第一层 Retiring/Bad
    Speculation/Front-End/Back-End，往下可展开到更细的子类别），鼠标悬停列头可以看到具体计算公式
  - 最常见的排查方式是按函数（Function）粒度查看这些指标，定位具体是哪个函数主导了某一类瓶颈

## 如何判断某类 Bound 是否偏高

  - **VTune 自带的阈值高亮**: 每个 TMA 指标都有 Intel
    架构师预设的阈值（因微架构而异）。当某函数该指标超过阈值、且该函数占总
    CPU 时间
    \>5%（即确实是热点）时，对应单元格会标红/标粉——鼠标悬停可看到具体阈值和建议。不是热点的冷代码即使超阈值也不会被标红，避免噪音干扰
  - **没有 GUI 时的相对判断法**：按 TMA 层级比较同级分支，哪个占比大就优先深挖——`Back-End Bound ≈
    Memory Bound + Core Bound`，`Memory Bound ≈ L1 Bound + L2 Bound + L3
    Bound + DRAM Bound + Store Bound`。这是"跟着占比最大的分支往下追"的方法，而非看某个绝对百分比数字
  - **层级越深越要当作权重而非精确占比**：只有第一、二层级（以 Pipeline Slots
    计）能严格加总到父级；更深层级的指标之间可能重叠、基于启发式估算，数值仅用于比较相对大小

## 常见下一步

  - **Memory Bound 高**: 转到 [Memory Access](/0059_Tools_Vtune_AT_MA)
    分析类型，细化到具体是哪一级 Cache/DRAM/NUMA 的问题
  - **Core Bound 高**: 通常与指令级并行不足、执行单元资源争用有关，关注向量化率、依赖链长度
  - **Front-End Bound 高**: 常见于指令 Cache 命中率低或分支预测失败率高、代码体积过大导致的
    I-Cache/DSB miss
  - **Bad Speculation 高**: 分支预测错误或数据推测失败，关注分支密集的代码路径

## Reference

  - [Microarchitecture Exploration Analysis for Hardware
    Issues](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-0/general-exploration-analysis.html)
  - [Microarchitecture Exploration
    View](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-2/general-exploration-view.html)
  - [Top-down Microarchitecture Analysis
    Method](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-1/top-down-microarchitecture-analysis-method.html)
