---
Title | Tools Vtune AT XPU
-- | --
Created @ | `2026-10-14T07:18:38Z`
Updated @| `2026-10-14T07:18:38Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/68)

---
# Vtune XPU Offload Analysis

- **预览特性（preview）**：目前仍是预览阶段，采集结果的数据格式不保证向后兼容
- **统一的 Offload 分析入口**：在一次分析里同时评估应用在 CPU/GPU/NPU 上的 offload 效果，判断瓶颈是在 Host 端还是 Device 端。自 2026.1 起取代了此前分离的 **GPU Offload Analysis** 和 **NPU Exploration Analysis**（两者已废弃并入本分析类型）。属于 Accelerators Analysis Group，详见 [Analysis Type 索引](/0062_Tools_Vtune_AT)
- 适合已用 SYCL/OpenMP/OpenCL/Level Zero 做异构计算，想确认 offload 是否划算、瓶颈在 Host 还是 Device 的场景；若此前环境里只有旧版 GPU Offload/NPU Exploration，建议直接升级到本分析类型

## 配置要点

1. 在 Accelerators 分组下选择 **XPU Offload (preview)**
2. WHAT 面板：指定应用路径和参数
3. HOW 面板：选择要分析的 Target Devices（CPU/GPU/NPU 可多选）
4. 采集选项：
   - **Trace computing programming APIs**：跟踪 SYCL、Level Zero、OpenCL、Intel VPL 等编程 API，用于分析运行在 GPU/NPU 上的程序；开启后会对 CPU 性能有一定影响
   - **Collect host stacks**：采集 CPU 侧调用栈，识别关键路径；可用来检查 GPU/NPU 计算任务对应的 CPU 端调用栈，评估 offload 的效率

## 采集模式

- **Time-based 模式**：系统级采集指标（类似 CPU uncore metrics），开销较低，适合大规模负载；该模式需要安装 Level Zero backend 及 NPU 驱动，但应用本身不需要使用 Level Zero（计算任务除外）

## 近期版本更新（了解即可，不同版本能力有差异）

- 2026.1：加速了结果 finalize 的速度
- 2026.2：NPU profiling 开启时会采集 NPU 功耗 trace 数据
- 2026.4：新增对 Xe3P 架构 GPU（Nova Lake-P）的支持，含新的 stall sampling 数据格式；改进了 Level Zero 的 trace 能力（支持 command list 复用、`zeCommandListImmediateAppendCommandListsExp`、SYCL graphs）；新增 Kernel GRF size 上报，便于判断 GRF 模式是否限制了 occupancy

## 典型工作流

1. 先看整体 Host/Device 时间占比，判断瓶颈更偏向 CPU 端还是 XPU 端
2. 若怀疑是 offload 调用本身开销大，开启 Trace computing programming APIs，细看 API 调用耗时
3. 若需要定位发起 offload 调用的具体代码位置，开启 Collect host stacks
4. 确认是 XPU-bound 后，进一步用 GPU Hotspots 分析类型定位 Device 端具体热点

## CLI 示例

```bash
vtune -collect xpu-offload -- ./your_app
```

## Reference
- [XPU Offload Analysis (Preview)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2025-4/xpu-offload-analysis.html)
- [Intel® VTune™ Profiler Release Notes and New Features](https://www.intel.com/content/www/us/en/developer/articles/release-notes/vtune-profiler/current.html)

