---
Title | Tools Vtune AT IO
-- | --
Created @ | `2026-10-14T07:18:27Z`
Updated @| `2026-10-14T07:18:27Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/67)

---
# Vtune Input and Output

- **I/O 分析**：定位存储/网络 I/O 密集型应用的瓶颈，同时覆盖硬件层（PCIe/DDIO/内存带宽）和软件层（DPDK/SPDK/Linux Kernel I/O）。属于 Input and Output Analysis Group，详见 [Analysis Type 索引](/0062_Tools_Vtune_AT)
- **仅支持 Linux/FreeBSD**；完整指标集仅在 Intel Xeon 处理器上可用

## 核心指标

### 平台层（硬件事件，应用无关）
- 外部 PCIe 设备/集成加速器的 I/O 带宽消耗，含 Intel® Data Direct I/O（Intel® DDIO）带宽与 Memory-Mapped I/O 流量
- DDIO 利用效率
- 内存带宽消耗
- Intel® Ultra Path Interconnect（Intel® UPI）带宽消耗

### 软件层（OS/API 相关）
- **DPDK**：针对轮询线程的 Rx/Event Dequeue 统计，包括 Rx Batch Statistics、Rx Spin Time、DPDK Event Dequeue Statistics、DPDK Event Dequeue Spin Time；其中 Event Dequeue Spin Time 反映 `rte_event_dequeue_burst()` 返回空结果的比例，可用于定位 event-driven pipeline 里的负载不均和瓶颈
- **SPDK**：估算 SPDK Effective Time 和 SPDK Latency，定位 SPDK 设备吞吐利用不足的情况，Summary 窗口按操作类型分组展示
- **Linux Kernel I/O**：直接分析 Linux 内核 I/O 栈

## 采集知识点（CLI knob）

- `dpdk`（true/false，默认 false）：采集 DPDK 指标，要求 DPDK 编译时启用 VTune Profiler 支持
- `spdk`（true/false，默认 false）：采集 SPDK 指标，要求 SPDK 编译时启用 VTune Profiler 支持
- `kernel-stack`（true/false，默认 false）：采集 Linux 内核 I/O 栈

## 典型工作流

1. 先跑一次不带额外 knob 的基础采集，看平台层指标（PCIe/DDIO/内存带宽/UPI）是否有明显异常
2. 若应用基于 DPDK，开启 `dpdk` knob，重点看 Spin Time 类指标判断轮询线程是否大量空转
3. 若应用基于 SPDK，开启 `spdk` knob，对比 SPDK Effective Time 和 Latency 判断设备吞吐是否打满
4. 怀疑是内核 I/O 路径问题时，开启 `kernel-stack` knob 做更细粒度分析

## CLI 示例

```bash
# attach 到运行中的进程，采集 SPDK 指标
vtune -collect io -knob iommu=true -knob mmio=false -knob spdk=true --target-process=<process_name>
```

## Reference
- [Input and Output Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-2/input-and-output-analysis.html)
- [Analyze DPDK Applications](https://www.intel.com/content/www/us/en/developer/articles/technical/dpdk-performance-analysis-with-intel-vtune-profiler.html)
- [Analyze SPDK Applications](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/analyze-spdk-applications.html)

