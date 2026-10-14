-----

| Title     | Tools Vtune CommandLine                          |
| --------- | ------------------------------------------------ |
| Created @ | `2022-09-09T06:48:44Z`                           |
| Updated @ | `2026-10-14T08:06:15Z`                           |
| Labels    | \`\`                                             |
| Edit @    | [here](https://github.com/junxnone/opt/issues/7) |

-----

# Vtune CommandLine Mode

  - **CLI 工具名**：`vtune`（GUI 启动器是 `vtune-gui`）。CLI 下**无需先创建
    Project**，直接指定 `-collect` 和目标程序即可运行
  - 适合无 GUI 的服务器/远程环境，或需要脚本化批量采集的场景
  - 语法：`vtune <action> [action-options] [global-options] -- <target>
    [target-args]`，例如 `vtune -collect hotspots -result-dir r001hs -quiet
    /home/test/sample` 中，`-collect` 是 action，`hotspots`
    是其参数，`-result-dir` 是 action-option，`-quiet` 是
    global-option

> **以下参数列表以 VTune 2026.1 版本文档为准。** 不同大版本之间 action/knob 参数（尤其是 `-collect`
> 支持的 analysis type 及其专属 knob）可能增减或改名，具体以 `vtune -help <action>`
> 在本机实际安装版本上的输出为准。

## 常用命令

采集（以 Hotspots 为例，`--` 之后是目标程序及其参数）：

``` bash
vtune -collect hotspots -result-dir r000hs -- ./your_app arg1 arg2
```

指定高级选项（knob），例如切换采样方式为软件计时器：

``` bash
vtune -collect hotspots -knob sampling-mode=sw -- ./your_app
```

生成/查看报告，对比两次采集结果：

``` bash
vtune -report hotspots -r r000hs -r r001hs
```

MPI 程序采集（每个 rank 各自落盘，`-trace-mpi` 会自动按 host 名区分结果目录）：

``` bash
mpiexec -np 4 vtune -collect hotspots -result-dir r000hs -- ./your_mpi_app
# 或
srun -N 2 -n 16 vtune -trace-mpi -r my_result -collect hotspots -- ./your_mpi_app arg1
```

用 GUI 打开 CLI 采集的结果：

``` bash
vtune-gui r000hs
```

## 参数可选列表

### Actions（第一个位置参数）

| Action           | 说明                                            |
| ---------------- | --------------------------------------------- |
| `collect` (`-c`) | 执行指定分析类型并采集数据                                 |
| `collect-with`   | 与 `collect` 类似，但用于自定义硬件事件采样/用户态采样配置           |
| `report` (`-R`)  | 从已有结果生成指定类型的报告                                |
| `import`         | 导入一个或多个采集数据文件/目录                              |
| `archive`        | 打包归档已采集的结果                                    |
| `finalize`       | 对结果做符号解析，完成或重新 finalize                       |
| `command`        | 向正在运行的 `collect` 发送控制命令（如暂停/恢复）               |
| `version`        | 显示 `vtune` 工具版本信息                             |
| `help`           | 显示命令行参数的简要说明，如 `vtune -help collect hotspots` |

### `-collect` 常用 analysis type 参数

| Analysis Type                                                                                      | 说明                                                       | 采集方式      |
| -------------------------------------------------------------------------------------------------- | -------------------------------------------------------- | --------- |
| `hotspots`                                                                                         | 定位最耗时代码；`-knob sampling-mode=sw\|hw` 切换软件计时器/硬件采样模式      | UMS 或 HWE |
| `threading`                                                                                        | 分析多线程并行度、同步开销                                            | UMS       |
| `memory-consumption`                                                                               | 分析内存占用与分配调用栈（仅 Linux 原生目标）                               | UMS       |
| `memory-access`                                                                                    | Cache/DRAM 带宽、NUMA 相关问题                                  | HWE       |
| `hpc-performance`                                                                                  | CPU/内存/FPU 利用率、向量化情况                                     | HWE       |
| `uarch-exploration`（原 `general-exploration`）                                                       | 微架构瓶颈定位，TMA 分析入口                                         | HWE       |
| `anomaly-detection`                                                                                | 自动发现高频重复代码段（如循环）里的性能异常                                   | UMS 或 HWE |
| `io`                                                                                               | I/O 子系统与 CPU/总线交互；`-knob dpdk\|spdk\|kernel-stack` 等细分场景 | HWE       |
| `system-overview`                                                                                  | 系统级行为、IRQ 处理与功耗/性能关联                                     | HWE       |
| `gpu-hotspots`                                                                                     | GPU 任务利用率与热点                                             | HWE       |
| `xpu-offload`（预览特性）                                                                                | 统一评估 CPU/GPU/NPU 的 offload 效果                            | \-        |
| `fpga-interaction`（已废弃）                                                                            | CPU/FPGA 交互，分析 FPGA 上的 OpenCL kernel                     | HWE       |
| `sgx-hotspots` / `tsx-exploration` / `tsx-hotspots` / `cpugpu-concurrency` / `gpu-profiling`（均已废弃） | 历史分析类型，不建议新项目使用                                          | HWE       |

> 完整列表及每种类型支持的 knob，执行 `vtune -help collect` 和 `vtune -help collect
> <analysis_type>` 查看本机实际版本的输出。

### 常用 knob（`-knob <name>=<value>`，按 analysis type 区分）

**跨多个 analysis type 通用的 knob：** | Knob | 说明 | 适用的 analysis type（举例） | |
--- | --- | --- | | `sampling-interval` | 采样间隔（ms） |
threading、hpc-performance、memory-access、fpga-interaction | |
`enable-stack-collection` | 是否采集调用栈 | hotspots（hw
模式）、hpc-performance、uarch-exploration、gpu-hotspots、fpga-interaction
| | `collect-memory-bandwidth` | 是否采集内存带宽 |
hpc-performance、uarch-exploration、io | | `dram-bandwidth-limits` | 自定义
DRAM 带宽上限（用于带宽利用率计算） | hpc-performance、memory-access、io | |
`enable-user-tasks` | 是否识别用户自定义任务（`__itt_task` 标注的区域） |
uarch-exploration、cpugpu-concurrency 等 | | `mem-object-size-min-thres` |
追踪的最小内存对象大小（字节），默认 32 | memory-consumption、memory-access（仅 Linux） |

**analysis type 专属 knob：** | Analysis Type | 专属 Knob | | --- | --- | |
`hotspots` |
`sampling-mode=sw\|hw`（软件计时器/硬件采样）、`enable-characterization-insights`
| | `anomaly-detection` |
`ipt-regions-to-load`（加载的代码区域数，10-5000）、`max-region-duration`（单区域最大分析时长，0.001-1000ms）
| | `memory-access` | `analyze-openmp`、`analyze-mem-objects`（仅 Linux） |
| `io` |
`collect-pcie-bandwidth`、`mmio`、`iommu`、`dpdk`、`spdk`、`kernel-stack`
| | `system-overview` | `collection-detail` | | `gpu-hotspots` |
`gpu-sampling-interval`、`enable-gpu-usage`、`gpu-counters-mode`、`enable-gpu-runtimes`
| | `fpga-interaction`（已废弃） |
`sampling-interval`、`enable-stack-collection` |

设置方式：`-knob <name>=<value>`，可重复多次指定多个 knob，例如：

``` bash
vtune -collect hotspots -knob sampling-mode=hw -knob enable-stack-collection=true -- ./your_app
```

> knob 是 analysis type 专属的，不能跨类型混用；具体某个 analysis type 支持哪些 knob 及取值范围，用
> `vtune -help collect <analysis_type>` 查看。

### `collect` 常用 action-options / global-options

| 选项                                | 说明                                 |
| --------------------------------- | ---------------------------------- |
| `-result-dir` / `-r`              | 指定结果输出目录                           |
| `-search-dir`                     | 指定符号/二进制文件搜索目录                     |
| `-source-search-dir`              | 指定源码搜索目录                           |
| `-knob` / `-k`                    | 设置分析类型专属的高级参数                      |
| `-duration`                       | 限制采集时长                             |
| `-data-limit`                     | 限制采集数据大小（MB）                       |
| `-start-paused`                   | 以暂停状态启动采集，需手动用 `command` action 恢复 |
| `-resume-after`                   | 采集暂停指定秒数后自动恢复                      |
| `-target-pid` / `-target-process` | 按 PID 或进程名附加到已运行的进程                |
| `-analyze-system`                 | 系统级采集（分析所有进程），仅 HWE 分析类型可用         |
| `-follow-child`                   | 同时分析目标启动的子进程                       |
| `-trace-mpi`                      | 跟踪 MPI 代码，自动按 host 区分结果目录          |
| `-no-summary`                     | 采集完成后不在 stdout 输出 Summary 报告       |
| `-quiet` / `-q`                   | 抑制非必要的提示信息                         |
| `-verbose` / `-v`                 | 输出更详细的运行日志                         |

### `report` 常用参数

| report\_name | 说明                   |
| ------------ | -------------------- |
| `summary`    | 总体性能概要               |
| `hotspots`   | 按采集到的指标列出耗时最多的程序单元   |
| `callstacks` | 按调用栈/函数展示完整栈数据       |
| `top-down`   | 从 `main()` 开始的调用序列报告 |
| `timeline`   | 按时间区间展示指标数据          |
| `hw-events`  | 硬件事件总数               |
| `gprof-cc`   | 带调用关系的函数耗时报告（含子函数）   |

常用修饰参数：`-group-by`（分组维度，如
`module`/`function`）、`-filter`（按模块/线程等过滤）、`-format`（输出格式，如
csv）、`-report-output`（写入文件而非 stdout）、`-column`（指定展示列）、`-limit`（限制展示条目数）

> **注意**：`-report <report_name>` 每次只生成一种报告类型，不像 GUI 打开结果后可以在各个 View/Tab
> 间随意切换查看。`report` action 本身**没有提供一次性生成全部报告类型的选项**（没有类似 `all`
> 的参数）；需要多种报告时只能分别多次执行 `-report`，或自行写脚本遍历 report\_name
> 列表，例如：
> 
> ``` bash
> for r in summary hotspots callstacks top-down timeline; do
>   vtune -report $r -r r000hs -format csv -report-output ${r}.csv
> done
> ```
> 
> `-collect` 结束后默认额外输出的 Summary（可用 `-no-summary`
> 关闭）也只是附带的一种，并不等于所有报告都已生成。

## 典型工作流

1.  在目标机器/容器上用 `-collect` 采集数据，落盘到 `-result-dir` 指定目录
2.  用 `-report` 直接在命令行查看摘要，或把结果目录拷回本地，用 `vtune-gui` 打开做可视化分析
3.  需要对比优化前后效果时，把两次结果目录都传给 `-report`，一次性看差异
4.  在 GUI 的 Configure Analysis 窗口里，可以用底部的 **Command Line** 按钮直接生成对应的 CLI
    命令，便于迁移到脚本/CI 里复用

## Reference

  - [Intel® VTune™ Profiler Command Line
    Interface](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/command-line-interface.html)
  - [vtune
    Actions](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/actions.html)
  - [Option Descriptions and General
    Rules](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/option-descriptions-and-general-rules.html)
  - [Get Started with Intel® VTune™
    Profiler](https://www.intel.com/content/www/us/en/docs/vtune-profiler/get-started-guide/2026-0/overview.html)
