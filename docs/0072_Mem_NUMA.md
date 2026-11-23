---
Title | Mem NUMA
-- | --
Created @ | `2026-10-16T08:35:35Z`
Updated @| `2026-10-16T08:35:35Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/72)

---
# Mem NUMA

> 本文是 [内存优化](0037_Mem) 的子篇，讲多路服务器上的 NUMA（Non-Uniform Memory Access）内存放置问题——这是多 Socket 平台上最容易踩、也最容易修的一类性能问题。

## 一、NUMA 架构基础

- **UMA**：所有核心到同一个内存池的距离相等（FSB 时代）；
- **NUMA**：每个 Socket 自带内存控制器（IMC）和 DIMM 通道，构成一个 NUMA node；跨 Socket 访问要走 **UPI**（Ultra Path Interconnect，Skylake-SP 起取代 QPI）。

实测的本地 vs 远端代价（Intel MLC 测量，不同 SKU 有差异）：

| 指标 | 本地 | 远端 | 比值 |
|---|---|---|---|
| 空载延迟 | ~81 ns | ~129-131 ns | ≈ 1 : 1.6 |
| 带宽 | ~63.5 GB/s | ~30.5 GB/s | ≈ 2 : 1 |

**带宽的惩罚比延迟更重**——因为 UPI 的宽度远小于它前面的那些 DDR 通道。

### Sub-NUMA Clustering（SNC）

把单个 Socket 的 LLC 和内存通道按地址范围切成 2-3 个 cluster，每个 cluster 对外暴露成一个独立的 NUMA node（双路 + SNC2 = 4 个 node）。**Xeon 6（Granite Rapids）默认开启 SNC3**，关掉则变成 HEX 模式（每 Socket 一个域）。

- 应用本身已经做过 NUMA 优化 → 开 SNC（HPC/STREAM 类负载能再涨 2-4%）；
- 应用完全没有 NUMA 感知 → 建议 HEX，避免被切得更碎之后远端访问更多；
- 要求 DIMM 对称插满，可用 `lscpu` / `numactl -H` 确认实际拓扑。

> 另一个常见陷阱是 BIOS 里的 **Node Interleaving**——打开后它会把整个系统伪装成 UMA，拓扑信息对软件完全不可见，所有 NUMA 调优都失效。

## 二、first-touch 策略：最经典的 NUMA Bug

`malloc`/`mmap` 只保留虚拟地址空间，**物理页是在第一次写（write fault）时才真正分配的，落在执行这次缺页的那个线程所在的 node 上**——注意是"第一次写的线程"，不是"调用 malloc 的线程"。

```c
// 典型错误：串行初始化
double *a = malloc(N * sizeof(double));
for (i = 0; i < N; i++) a[i] = 0.0;    // 单线程跑完 → 整个数组都落在 node 0

#pragma omp parallel for
for (i = 0; i < N; i++) a[i] = ...;    // 所有非 node-0 的线程都在做远端访问
```

后果是双重的：非 node-0 的线程全部远端访问，而且所有流量都挤在 node 0 这一个 IMC 上。

```c
// 正确做法：用和后续计算完全相同的并行划分做初始化
#pragma omp parallel for schedule(static)
for (i = 0; i < N; i++) a[i] = 0.0;    // 每个线程 first-touch 自己要处理的那部分

#pragma omp parallel for schedule(static)
for (i = 0; i < N; i++) a[i] = ...;    // 现在全是本地访问
```

实测效果：STREAM Triad 从 21.80 GB/s 提升到 43.49 GB/s（32 线程）。

**注意事项**：
- **线程绑定是前提**——线程如果会被调度器迁移走，first-touch 的放置就白做了；
- `calloc`、串行 `memset`、以及 C++ 的 `std::vector`/容器构造函数都会**悄悄地预先触碰内存**，直接破坏 first-touch 策略。

## 三、实用工具

```bash
# 拓扑与放置
numactl --hardware                        # 拓扑、每 node 内存量、距离矩阵
numactl --cpunodebind=0 --membind=0 ./app # 严格绑定：node0 的 CPU + node0 的内存
numactl --interleave=all ./app            # 页在所有 node 间轮询分配
numactl --localalloc ./app                # 内核默认（first-touch）
numactl --preferred=1 ./app               # 软提示，不够时静默回退到其他 node
numactl --physcpubind=0-7 ./app           # 绑定到指定的逻辑 CPU

# 监控
numastat                                  # numa_hit/numa_miss/numa_foreign
numastat -p <pid> -c                      # 单进程在各 node 上的内存分布
lscpu | grep -i numa                      # 快速看 node 数量与 CPU 归属
lstopo --of png > topo.png                # hwloc，可视化完整拓扑
cat /proc/<pid>/numa_maps                 # 每个 VMA 的策略与实际页分布
```

**关键计数器含义**：
- `numa_hit`：成功分配在期望的 node 上；
- `numa_miss`：本想分配在别的 node，结果落在了这个 node；
- `numa_foreign`：本想分配在这个 node，结果跑到别处去了（和某个 miss 互为镜像）。

`numa_miss` 持续上升说明内存压力大或放置策略有问题。

**libnuma API**（链接 `-lnuma`，头文件 `<numa.h>`）：

```c
numa_available();                  // 先检查系统是否支持
numa_alloc_onnode(size, node);     // 在指定 node 上分配
numa_alloc_local();                // 在当前 node 上分配
numa_run_on_node(n);               // 把当前线程限制到 node n 的 CPU 上
numa_set_preferred(n);             // 设置首选 node
numa_node_of_cpu(cpu);             // 查某个 CPU 属于哪个 node
numa_free(ptr, size);
```

底层对应的系统调用是 `mbind(2)`（按地址范围）、`set_mempolicy(2)`（按线程）、`get_mempolicy(2)`。

## 四、常见优化模式

```bash
# OpenMP 标准写法
export OMP_PLACES=cores OMP_PROC_BIND=spread   # 每核一个线程，跨 node 铺开
# Intel 运行时等价写法
export KMP_AFFINITY=granularity=fine,scatter,verbose
export KMP_AFFINITY="explicit,proclist=[0,1,4,5],verbose"
```

`scatter` ≈ `spread`，`compact` ≈ `close`；加 `verbose` 可以打印实际绑定结果来核对。

**怎么选策略**：

| 负载特征 | 推荐做法 | 原因 |
|---|---|---|
| 延迟敏感、数据可按线程私有划分 | 绑核 + localalloc + 并行 first-touch | 让每个线程只访问自己 node 的内存 |
| 带宽受限、数据共享且局部性差（DB buffer pool、分析类、大 GEMM） | `--interleave=all` | 用延迟换总带宽，避免单个 IMC 成为热点 |

Interleave 的本质是"让所有访问都变得同样平庸"——双 node 时会有 50% 变成远端访问。由于本地:远端带宽约为 5:3，简单的 50/50 并不是最优；Linux 6.9+ 支持**加权交织**（weighted interleave），按约 62% 本地的比例分配更贴合硬件实际比值。

## 五、用 VTune 诊断 NUMA 问题

跑 **Memory Access 分析**（开启内存对象分析，Object Size Threshold 设为 1），重点看：

- `Memory Bound → DRAM Bound → Local DRAM` / **Remote DRAM** / **Remote Cache**（各自的停顿周期占比）；
- `LLC Miss Count → Local/Remote DRAM Access Count`、`Remote Cache Access Count`，以及 **NUMA: % of Remote Accesses** 指标；
- Summary → **Bandwidth Utilization Histogram**，把 Bandwidth Domain 切到 **UPI**，判断互联是否已经饱和；
- `DRAM Bound` 还会进一步细分成 Bandwidth-Bound / UPI-Utilization-Bound / Latency 三类成因。

详见 [Vtune Memory Access](0059_Tools_Vtune_AT_MA)，整体排查路径见 [Vtune Pipeline](0071_Tools_Vtune_Pipeline)。

## 六、常见坑

### AutoNUMA Balancing

内核的自动 NUMA 平衡（`kernel.numa_balancing`）会把页临时标记为 `PROT_NONE`、通过采样 hinting fault 来判断访问来源，同一 node 连续访问 2 次后就迁移该页。**采样 fault 很便宜（约 10 μs），但页迁移本身不便宜**。

建议关闭的场景：已经用 numactl/cpuset 做了显式绑定、应用本身是 NUMA-aware、以及实时/延迟敏感型负载。

```bash
sysctl -w kernel.numa_balancing=0
# 或内核命令行 numa_balancing=disable
```

（新内核上值 `2` 表示内存分层模式；也可以只调 `numa_balancing_scan_*` 系列参数而不是直接关掉。）

### 容器与虚拟机

- Kubernetes 下要同时满足三个条件才能真正拿到 NUMA 亲和性：`cpuManagerPolicy: static`（且 Pod 是 Guaranteed QoS、CPU 必须是整数——小数会静默失去绑定）、`topologyManagerPolicy: single-numa-node`、`topologyManagerScope: pod`。没有配 Topology Manager 时，kubelet 会静默地跨 node 装箱；配了之后由于调度器本身不感知拓扑，又可能出现 `TopologyAffinityError` 准入失败；
- 虚拟机里 guest 的 vNUMA 拓扑必须和 host 实际拓扑对应，否则在 guest 里做的所有 NUMA 调优都是自欺欺人。

### 大页与 NUMA 的相互影响

全局的 `nr_hugepages` 会在各 node 间均分，并且**会静默跳过那些没有足够连续内存的 node**。要按 node 精确预留应该写：

```bash
echo 512 > /sys/devices/system/node/node0/hugepages/hugepages-2048kB/nr_hugepages
```

1GB 大页必须在启动时预留。另外要注意：**一个大页整体属于某一个 node**——它能解决 TLB miss，但一个远端的 2MB 大页依然是远端的，页粒度变粗反而让 NUMA 放置更难精细调整，所以大页有可能**加剧** NUMA 不均衡。详见 [Mem TLB](0073_Mem_TLB)。

## Reference
- [numactl(8)](https://linux.die.net/man/8/numactl)
- [numa(7)](https://man7.org/linux/man-pages/man7/numa.7.html)
- [numastat - Kernel Docs](https://docs.kernel.org/admin-guide/numastat.html)
- [NUMA API for Linux（Andi Kleen）](https://halobates.de/numaapi3.pdf)
- [Intel Memory Latency Checker (MLC)](https://www.intel.com/content/www/us/en/developer/articles/tool/intelr-memory-latency-checker.html)
- [VTune Cookbook: NUMA Impact in Multiprocessor Systems](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2024-0/numa-impact-in-multiprocessor-systems.html)
- [How To Befriend NUMA（OpenMP）](https://www.openmp.org/wp-content/uploads/SC18-BoothTalks-vanderPas.pdf)
- [Kubernetes Topology Manager](https://www.kubernetes.io/docs/tasks/administer-cluster/topology-manager/)

