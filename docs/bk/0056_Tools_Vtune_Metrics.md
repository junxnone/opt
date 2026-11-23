-----

| Title     | Tools Vtune Metrics                               |
| --------- | ------------------------------------------------- |
| Created @ | `2024-04-20T19:01:02Z`                            |
| Updated @ | `2026-10-14T09:21:43Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/56) |

-----

# Vtune Metrics

本篇是 VTune 常用 metric 的速查入口，只给每个指标一句话的直观解释；深入的方法论和逐项拆解见 [0047
TMA](0047_OPT_TMA)、[0058 Microarchitecture
Exploration](0058_Tools_Vtune_AT_Microarchitecture)、[0059 Memory
Access](0059_Tools_Vtune_AT_MA) 三篇，这里不重复展开。

## CPU Metrics

### 顶层时间类

| Metric          | 含义                                      |
| --------------- | --------------------------------------- |
| Elapsed Time    | 从采集开始到结束的墙钟时间                           |
| CPU Time        | CPU 实际执行应用代码的时间                         |
| Effective Time  | CPU Time 中真正用于用户代码的部分（排除 Spin/Overhead） |
| CPU Utilization | 应用使用的逻辑核占系统全部逻辑核的百分比，100% 表示全程占满所有逻辑核   |

### 指令效率类（判断"算得快不快"）

| Metric                                   | 含义                                                                                   |
| ---------------------------------------- | ------------------------------------------------------------------------------------ |
| IPC（Instructions Per Cycle）              | 每周期平均退休指令数，现代超标量 CPU 理论上限约 4；HPC 场景 IPC≈1 通常可接受，具体阈值因应用而异                            |
| CPI（Clockticks per Instructions Retired） | IPC 的倒数视角，CPI\<1 多为指令密集型，CPI\>1 多提示存在停顿/访存瓶颈；应和基线对比而非看绝对值，向量化可能合理地推高 CPI，需结合退休指令数一起看 |

### TMA 四大类

| Metric          | 一句话含义                                                                              |
| --------------- | ---------------------------------------------------------------------------------- |
| Retiring        | Pipeline Slots 中真正产出有效 uOp 退休的占比，越高越好，但 Microcode Assists 这类"有害但计入 Retiring"的例外要注意 |
| Bad Speculation | 因分支预测错误/机器清空而浪费的 Pipeline Slots 占比                                                 |
| Front-End Bound | 前端取指/译码跟不上，供给不了足够 uOp 给后端                                                          |
| Back-End Bound  | 后端执行资源（计算单元或访存）跟不上，是 Core Bound 和 Memory Bound 的父类                                 |
| Core Bound      | Back-End Bound 中非访存类瓶颈，如执行端口过载、乱序资源不足、长延迟运算链依赖                                     |
| Memory Bound    | Back-End Bound 中因访存（L1/L2/L3/DRAM 命中或带宽）导致的停顿，细分指标见 [0059](0059_Tools_Vtune_AT_MA) |

### 其他常见指标

| Metric                | 含义                                                   |
| --------------------- | ---------------------------------------------------- |
| CPU Frequency         | 用 APERF/MPERF 寄存器在采样间隔内估算出的平均逻辑核频率                   |
| False Sharing         | 多线程访问同一 cache line 的不同数据引发的无效 cache 跳动               |
| Spin Time / Wait Time | 线程空转等待（自旋锁、同步原语）/ 阻塞等待（如 I/O、锁）所占时间，多线程场景判断同步开销的关键指标 |

### 访存层级细分

| Metric                                     | 含义                                                         |
| ------------------------------------------ | ---------------------------------------------------------- |
| L1/L2/L3 Bound                             | 分别代表因未命中对应层级 cache 而导致的停顿占比，层级越往后延迟越高                      |
| DRAM Bound                                 | 停顿在主存（DRAM）访问上的占比，常伴随 Memory Bandwidth/Memory Latency 两个子类 |
| L1/L2/L3 Hit Rate、LLC Miss Count           | 各级 cache 的命中率与 LLC 未命中次数，用于判断数据局部性好坏                       |
| Contested Accesses / Data Sharing          | 多核对同一 cache line 的竞争性访问 / 数据共享导致的一致性开销，常见于多线程共享数据结构        |
| 4K Aliasing                                | load 被此前 store 以 4K 地址偏移"伪命中"而重发，通常可忽略，除非向上传导到 L1 Bound    |
| DTLB/UTLB Overhead                         | 一级/统一 TLB 未命中导致的地址翻译开销，大页（Huge Pages）通常能缓解                 |
| Split Loads / Split Stores                 | 跨 cache line 边界的读/写，建议按 64 字节对齐数据                          |
| Store Bound / Store Latency / Lock Latency | 存储操作、长延迟存储未命中、锁相关存储分别导致的停顿                                 |
| Loads Blocked by Store Forwarding          | load 因等待同地址的前序 store 转发数据而被阻塞                              |
| Local/Remote DRAM、Remote Cache（NUMA）       | 本地内存访问 vs 跨 Socket 的远程内存/缓存访问，远程访问延迟显著更高，是多路服务器的常见瓶颈       |
| Memory Bandwidth / Memory Latency          | 分别代表受限于 DRAM 带宽上限 / 受限于 DRAM 访问延迟两种不同的访存瓶颈成因               |

### 前端取指细分

| Metric                            | 含义                                                                                                                     |
| --------------------------------- | ---------------------------------------------------------------------------------------------------------------------- |
| ICache Misses / ITLB Overhead     | 指令缓存未命中 / 指令 TLB 未命中，常见于代码体积过大（热点函数内联过度）的场景                                                                            |
| BACLEARS                          | 分支目标缓冲区（BTB）预测被后级分支预测器纠正所浪费的周期                                                                                         |
| DSB/MITE/LSD 相关指标                 | 分别衡量 uOp 由"解码后 µop 缓存（DSB）"/"传统译码路径（MITE）"/"循环流检测器（LSD，又称 Loop Cache）"供给的比例，DSB 命中率低、频繁在 DSB↔MITE 间切换通常提示热点代码超出 DSB 容量 |
| Length Changing Prefixes (LCP)    | 变长指令前缀导致译码器重新计算指令长度的停顿                                                                                                 |
| MS Switches / Microcode Sequencer | 切入微码时序器（走慢速译码路径）导致的开销，常由复杂/罕见指令触发                                                                                      |

### 乱序执行与端口

| Metric                               | 含义                                                                       |
| ------------------------------------ | ------------------------------------------------------------------------ |
| Divider                              | 除法/开方等长延迟运算单元（DIV unit）占用周期的比例                                           |
| Port Utilization / Port 0-7          | 各执行端口（ALU、Load、Store-address、Store-data、分支等）被 uOp 占用的周期比例，用于定位具体哪类运算资源过载 |
| Cycles of 0/1/2 Ports Utilized       | 每周期实际派发到执行端口的 uOp 数分布，0 端口占用比例高常提示长延迟指令（如除法）阻塞流水线                        |
| Slow LEA Stalls / Flags Merge Stalls | 三操作数 LEA 指令 / 标志位合并（如 shift+cl）带来的额外延迟                                   |

### 向量化 / FPU

| Metric                           | 含义                                             |
| -------------------------------- | ---------------------------------------------- |
| FPU Utilization                  | FPU 被占用并以满向量宽度退休浮点指令的周期占比，100% 表示每周期都在跑满宽度向量运算 |
| FP Arithmetic（Scalar/Vector/x87） | 标量 / 向量 / x87 浮点运算指令分别占总退休浮点 uOp 的比例           |
| Vector Capacity Usage            | 浮点运算中实际使用的向量宽度相对硬件最大向量宽度的比例，低说明代码未充分向量化或向量宽度不足 |
| Vector Instruction Set           | 实际使用的向量指令集（SSE/AVX2/AVX-512 等）                 |
| SP/DP GFLOPS、SP FLOPs per Cycle  | 每秒/每周期完成的单精度/双精度浮点运算数，衡量向量代码生成与执行效率的直接指标       |

### 多线程调度与同步

| Metric                                                            | 含义                                                                      |
| ----------------------------------------------------------------- | ----------------------------------------------------------------------- |
| Wait Time / Wait Count / Wait Rate                                | 线程因阻塞类 API（锁、I/O、barrier 等）而等待的总时间/次数，以及平均每次等待耗时；Wait Rate 偏低常提示线程间竞争激烈 |
| Spin Time                                                         | 线程忙等（自旋锁、busy-wait barrier）消耗的 CPU 时间，通常是无效工作                           |
| Overhead Time（含 Creation/Scheduling/Reduction/Atomics/Tasking 子项） | 线程库自身管理并行任务（创建线程、调度、归约、原子操作等）产生的开销，而非用户代码本身                             |
| Lock Contention                                                   | 多线程争抢同一把锁导致的等待，是 Spin/Wait Time 里最常见的成因之一                               |
| Imbalance or Serial Spinning                                      | 因负载不均导致部分线程提前完成、在 barrier 处空转等待其他线程                                     |
| Thread Oversubscription                                           | 同时工作的线程数超过可用逻辑核数，会引入额外的抢占/上下文切换开销                                       |
| Serial Time（outside parallel regions）                             | 应用在并行区域之外（如 OpenMP master 线程单独执行）花费的时间，直接限制并行加速比（对应 Amdahl 定律里的串行部分）    |
| OpenMP Region Time / Potential Gain                               | 各 OpenMP 并行区域的耗时，及消除负载不均后理论上可节省的时间                                      |
| MPI Busy Wait Time / Imbalance                                    | MPI rank 间等待通信、负载不均导致的忙等时间，常用于定位 HPC 多进程扩展性瓶颈                           |

### NUMA / 多路系统

| Metric                          | 含义                                                         |
| ------------------------------- | ---------------------------------------------------------- |
| Local DRAM/Cache Access（Count）  | 本地内存/缓存访问次数或占比                                             |
| Remote DRAM/Cache Access（Count） | 跨 Socket 远程内存/缓存访问，往往是多路服务器上性能回退的主因，建议配合 NUMA-aware 内存分配优化 |
| Remote / Local DRAM Ratio       | 远程与本地内存访问的比例，越高说明 NUMA 亲和性越差                               |
| UPI Utilization Bound           | 因 Socket 间互联（UPI）带宽接近上限而产生的停顿                              |

### HPC 网络 Fabric

| Metric                              | 含义                                               |
| ----------------------------------- | ------------------------------------------------ |
| Incoming/Outgoing Bandwidth Bound   | 节点间网络收/发带宽接近理论上限导致的停顿                            |
| Incoming/Outgoing Packet Rate Bound | 节点间网络收/发包速率过高导致的停顿，细分可用 Packet Rate Histogram 查看 |

## GPU Metrics

GPU Compute/Media Hotspots 分析里，所有 metric 分组都会包含下面这组追踪 EU 活跃度的核心指标（新架构里
EU 改称 XVE，含义不变）：

| Metric                     | 含义                                                                                           |
| -------------------------- | -------------------------------------------------------------------------------------------- |
| EU Array Active            | EU 中至少一个线程在执行指令的周期占比，越高越好                                                                    |
| EU Array Stalled           | EU 有线程占用但被阻塞（多为等待显存子系统数据）的周期占比，优化目标通常是降低它、提高 Active；但带宽瓶颈型 kernel 应以逼近峰值带宽为目标，而不是一味追求 Active |
| EU Array Idle              | EU 完全没有线程可调度的周期占比，为 0 说明 GPU 负载已经比较饱和                                                        |
| EU Threads Occupancy       | 所有 core、所有线程槽位里"有线程被调度"周期数的归一化总和，偏低说明工作负载在线程间分布不均                                            |
| Occupancy / Peak Occupancy | 实际并发线程数相对硬件理论峰值的比例；受限常见原因是每个 workgroup 申请的 SLM（共享本地内存）过大，或 Global Size 太小                    |
| Computing Threads Started  | 本次采集期间启动的 GPU 线程（work-item）总数，用于判断 kernel 是否被拆分成了过多的短小任务                                     |

Compute Basic/Extended 分组额外关注 EU 内部流水线与本地内存：

| Metric                                                  | 含义                                                                         |
| ------------------------------------------------------- | -------------------------------------------------------------------------- |
| EU IPC Rate                                             | GPU EU 每周期退休指令数，CPU IPC 概念在 GPU 侧的对应指标                                     |
| EU 2 FPU Pipelines Active / EU Send Pipeline Active     | EU 内双 FPU 流水线 / 负责显存访问与跨 EU 通信的 Send 流水线的活跃周期占比，用于判断 kernel 是计算密集还是访存/通信密集 |
| Shared Local Memory (SLM) Read/Write Bandwidth          | kernel 对共享本地内存（类似 CPU 侧的 scratchpad）的读写带宽                                  |
| Untyped/Typed Memory Read/Write Bandwidth（Transactions） | 分别对应 Buffer（untyped）与 Image（typed）两类显存访问的带宽/事务数                            |
| Untyped/Typed Reads/Writes Coalescence                  | 访存请求被合并（coalesce）执行的比例，合并度低意味着访存模式不连续、带宽利用率差                               |

Overview 分组则关注显存带宽与采样器：

| Metric                                                        | 含义                                    |
| ------------------------------------------------------------- | ------------------------------------- |
| Memory Read/Write Bandwidth、GPU Memory Texture Read Bandwidth | GPU 对显存的读/写带宽，以及纹理（Texture）读带宽        |
| L3 Miss Rate/Ratio、LLC Miss Rate/Ratio due GPU Lookups        | GPU 侧 L3/LLC 未命中率，衡量 GPU 对共享末级缓存的命中情况 |
| Sampler Busy / Sampler Is Bottleneck                          | 纹理采样单元的繁忙度，以及是否已成为瓶颈（多见于图形/媒体负载）      |

Render Basic（图形管线）分组补充像素/顶点着色器维度：

| Metric                                                   | 含义                                          |
| -------------------------------------------------------- | ------------------------------------------- |
| PS/VS EU Active / EU Stall                               | 像素着色器（PS）/顶点着色器（VS）对应 EU 的活跃/停顿周期占比         |
| Samples Killed in PS / Samples Written / Samples Blended | 像素着色阶段被丢弃/实际写出/参与混合（blend）的采样点数量，用于图形管线效率分析 |

## Reference

  - [CPU Metrics Reference
    (2024-1)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2024-1/cpu-metrics-reference.html)
  - [GPU Metrics Reference
    (2023-2)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-2/gpu-metrics-reference.html)
  - [GPU EU Array
    Usage](https://software.intel.com/content/www/us/en/develop/documentation/vtune-help/top/reference/gpu-metrics-reference/gpu-eu-array-usage.html)
  - [EU Threads
    Occupancy](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/eu-threads-occupancy.html)
  - [Software Optimization for Intel
    GPUs](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2025-4/software-optimization-for-intel-gpus.html)
