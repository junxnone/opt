-----

| Title     | OPT TMA                                           |
| --------- | ------------------------------------------------- |
| Created @ | `2022-04-27T01:48:41Z`                            |
| Updated @ | `2024-04-07T03:17:39Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/47) |

-----

# TMA

  - TMA - `Top-Down Microarchitecture Analysis`
  - **Retiring**: 正常执行的 μOps 比例
  - **Bad Speculation**: Cancelled ( `Mispredicted branches`/`Incorrect
    data speculation`)
  - **Front-End Bound**: front-end 导致的 pipeline slots 不能被充分使用
  - **Back-End Bound**: back-end 导致的 pipeline slots 不能被充分使用

## 层次结构/Hierarchy

![image](media/1d4cc5b87c1cac822fd2edcddc8ebc30ed3260ea.png)

## 相关名词

  - **μOps**: `micro-ops/micro-operations` 微指令
  - **DSB**: `Decoded-μop Stream Buffer`

### Pipeline Slots

  - **Pipeline Slots**: 处理一个 μOps 需要的硬件资源
      - 对于每个 `CPU Core`, 在每个时钟周期, 有多个可用的 `Pipeline Slots`, 数量称为
        `Pipeline Width`

| 40 Pipeline Slots = 4-wide CPU X 10 Clock cycles             | Retiring(50%) = SlotsRetired/TotalSlots                      |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](media/ddc2eb7105c6b291d3a342ab3f15b0c370f4fb39.png) | ![image](media/f4264fa46f4f31e5abf7a76b840866ca714d4184.png) |

## Reference

  - [TMA\_自顶向下的CPU架构性能瓶颈分析方法](https://zhuanlan.zhihu.com/p/60569271)
  - [《A Top-Down Method for Performance Analysis and Counters
    Architecture》阅读笔记](https://andrewei1316.github.io/2020/12/20/top-down-performance-analysis/)
  - [自顶向下的微架构分析方法 -
    Intel](https://www.intel.com/content/www/us/en/develop/documentation/vtune-cookbook-zh-cn/top/methodologies/top-down-microarchitecture-analysis-method.html)
    2014 **TMA** [A Top-Down method for performance analysis and
    counters
    architecture.pdf](https://github.com/junxnone/linuxwiki/files/8559242/A.Top-Down.method.for.performance.analysis.and.counters.architecture.pdf)
  - [Pipeline
    Slots](https://www.intel.com/content/www/us/en/develop/documentation/vtune-help/top/reference/cpu-metrics-reference/pipeline-slots.html)
  - [Instructions Retired
    Event](https://www.intel.com/content/www/us/en/develop/documentation/vtune-help/top/analyze-performance/custom-analysis/custom-analysis-options/hardware-event-list/instructions-retired-event.html)
