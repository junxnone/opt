---
Title | Tools Vtune DebugAPI
-- | --
Created @ | `2023-07-20T06:27:25Z`
Updated @| `2026-10-14T08:34:57Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/10)

---
# Vtune ITTJIT APIs
- ITT API - `Instrumentation and Tracing Technology API` - 控制收集跟踪数据的范围，用户在代码中手动标记 task/frame 等逻辑单元，让 VTune 在时间线上按这些标记展示性能数据
- JIT Profiling API - 让 JIT/托管运行时在代码生成时把"这段地址对应哪个方法/源文件"报告给 VTune，从而分析 JIT 生成代码（而非静态编译的二进制）的性能

> **注意**：自 VTune 2025.4 起，ITT API 与 JIT Profiling API 已转为开源组件，不再作为 VTune Profiler 自身维护的闭源部分，源码、issue、贡献都在 [intel/ittapi](https://github.com/intel/ittapi) 仓库进行。两套 API 的使用方式不受此变化影响，VTune 仍然可以正常消费它们产生的数据。

## ITT API

- 适用场景：你的代码是静态编译的，但想在 VTune 时间线上标出自定义的逻辑范围（例如一次渲染里的"烟雾绘制" vs "阴影计算"），而不是只看到函数级别的采样结果。
- 核心概念：
  - **Domain（域）**：`__itt_domain_create()` 创建，大多数 ITT API 调用都需要指定一个 domain，用来隔离不同模块/库各自的标记，避免互相干扰。
  - **String Handle（字符串句柄）**：`__itt_string_handle_create()` 创建，用于标识 task 名称；相比直接传字符串，句柄方式开销更低、效率更高。
  - **Task（任务）**：`__itt_task_begin()` / `__itt_task_end()` 成对调用，标记一段逻辑工作的开始和结束，是最常用的标记粒度。
  - 此外还有 Frame API（标记一帧渲染）、Thread Naming API（给线程命名，方便在时间线上识别）等配套 API。
- 最小示例：
  ```c
  #include <ittnotify.h>

  __itt_domain* domain = __itt_domain_create("Example.Domain.Global");
  __itt_string_handle* handle_work = __itt_string_handle_create("work");

  __itt_task_begin(domain, __itt_null, __itt_null, handle_work);
  // ... 需要被标记/分析的代码 ...
  __itt_task_end(domain);
  ```
- 实际在 PyTorch 里的用法（隐式/显式调用、`emit_itt()` 作用域）参见 [0064 Debug pytorch with vtune](0064_Tools_Vtune_pytorch.md)。

### 常用方法

除了 Domain/String Handle/Task 这三个最基础的函数，ITT API 还按场景提供了一批配套函数：

| 分类 | 常用函数 | 作用 |
| --- | --- | --- |
| Domain API | `__itt_domain_create(name)` | 创建/获取一个域，用于隔离不同模块的标记；域的 `flags` 置 0 可整体禁用该域的采集开销 |
| String Handle API | `__itt_string_handle_create(name)` | 创建字符串句柄，相同 name 多次调用返回同一句柄，比直接传字符串更省开销 |
| Task API | `__itt_task_begin(domain, taskid, parentid, name)` / `__itt_task_end(domain)` | 标记一段逻辑任务的开始/结束，需要在分析设置里勾选 **Analyze user tasks** 才能在结果里看到 |
| Frame API | `__itt_frame_begin_v3(domain, id)` / `__itt_frame_end_v3(domain, id)` | 标记"帧"（一段不重叠的时间区间，如一次渲染/一次事务），在 Timeline 里单独成一条 track，常用于游戏渲染、图形应用、逐事务性能分析 |
| Thread Naming API | `__itt_thread_set_name(name)` / `__itt_thread_ignore()` | 给当前线程起一个有意义的名字；或把某个服务线程整体从分析结果里排除 |
| Event API | `__itt_event_create(name, namelen)` → `__itt_event_start(event)` / `__itt_event_end(event)` | 标记一次性事件或一段持续事件，在 Timeline 上显示为刻度/带时长的横线 |
| Counter API | `__itt_counter_create(name, domain)` | 定义一个用户自定义计数器（如 SoC 某子模块的自定义硬件计数），随采集过程写入数值 |
| Collection Control API | `__itt_pause()` / `__itt_resume()` / `__itt_detach()` | 在代码里临时暂停/恢复数据采集，或彻底摘除采集器；暂停期间 Domain/String Handle/Thread Naming 仍然生效 |
| User-Defined Synchronization API | `__itt_sync_create(addr, objtype, objname, attribute)` 等 | 让 VTune 识别自定义的同步原语（非标准 Windows/POSIX 同步对象），配合 Threading 分析类型统计等待/竞争 |

## JIT Profiling API

- 适用场景：代码是在运行时动态生成/编译的（JIT），VTune 事先不知道这些地址对应什么源码，需要由生成代码的一方主动上报映射关系。典型场景：
  - JavaScript 引擎的 trace-based JIT
  - OpenCL™ 应用的运行时内核编译
  - Java*/.NET* 等托管执行环境
  - 自定义的 ISV JIT 引擎
- 工作原理：VTune 安装包里包含一个静态库/桩实现（stub），JIT 引擎在生成代码时通过这个静态部分把"方法 ID、方法名、源文件、加载地址、代码大小"等信息写入 trace 文件；采集结束后 VTune 用这份 trace 文件把采样结果映射回方法/源码行。若未安装 VTune，桩实现什么都不做，不影响程序正常运行。
- 需要设置的环境变量（把桩实现替换成真正的采集器）：
  - Windows：`INTEL_JIT_PROFILER32` / `INTEL_JIT_PROFILER64` 指向 `<install-dir>\bin32(64)\runtime\ittnotify_collector.dll`
  - Linux：指向 `<install-dir>/lib32(64)/runtime/libittnotify_collector.so`
- 最小示例（方法级上报）：
  ```c
  #include <jitprofiling.h>

  if (iJIT_IsProfilingActive() != iJIT_SAMPLING_ON) {
    return;
  }

  iJIT_Method_Load jmethod = {0};
  jmethod.method_id = iJIT_GetNewMethodID();
  jmethod.method_name = "method_name";
  jmethod.class_file_name = "class_name";
  jmethod.source_file_name = "source_file_name";
  jmethod.method_load_address = code_addr;
  jmethod.method_size = code_size;

  iJIT_NotifyEvent(iJVM_EVENT_TYPE_METHOD_LOAD_FINISHED, (void*)&jmethod);
  ```
- 除了最基本的方法级上报，API 还支持：
  - **Split Functions（分裂函数）**：同一 method_id 多次上报不同地址区间，用于资源受限环境下的重新 JIT（re-JIT）场景。
  - **Inline Functions（内联函数）**：用 `iJIT_Method_Inline_Load` 把内联方法与其 parent_method_id 关联，还原多层内联调用关系。
- 采集方式：JIT Profiling 需要搭配 Launch Application 目标 + 基于事件的采样（event-based sampling）使用。

### 常用方法

| 函数/结构体 | 作用 |
| --- | --- |
| `iJIT_IsProfilingActive()` | 查询当前是否正在被 VTune 采集（`iJIT_SAMPLING_ON` 才需要继续上报，避免生产环境空跑开销） |
| `iJIT_GetNewMethodID()` | 申请一个全局唯一的 `method_id`，后续所有事件都用它关联同一个方法 |
| `iJIT_NotifyEvent(event_type, data)` | 上报事件的统一入口，`event_type` 决定 `data` 要传哪种结构体 |
| `iJVM_EVENT_TYPE_METHOD_LOAD_FINISHED` + `iJIT_Method_Load` | 上报一个方法：`method_id`/`method_name`/`class_file_name`/`source_file_name`/`method_load_address`/`method_size` |
| `iJIT_Method_Load_V2` | `iJIT_Method_Load` 的扩展版，额外支持关联模块名（module name），用于区分不同 JIT 引擎生成的同名函数 |
| `iJVM_EVENT_TYPE_METHOD_INLINE_LOAD_FINISHED` + `iJIT_Method_Inline_Load` | 上报一个内联方法，需指定 `parent_method_id` 还原多级内联层级 |
| `iJVM_EVENT_TYPE_METHOD_UNLOAD_START` | 通知某个 `method_id` 已被卸载/失效（典型于 re-JIT 覆盖旧代码区间时） |
| `iJVM_EVENT_TYPE_SHUTDOWN` | 通知 JIT 引擎即将退出，VTune 据此结束对该引擎的 trace 记录 |

## 典型工作流

1. 静态编译代码 → 用 ITT API 在关键路径插入 `domain`/`task begin-end` 标记；JIT/托管运行时代码 → 在代码生成器里插入 JIT Profiling API 调用上报方法信息。
2. 正常编译、链接 ITT/JIT 的桩库（`libittnotify`），确保 VTune 环境变量（JIT 场景）能找到真正的采集器。
3. 用 `vtune -collect hotspots`（或其他基于采样的分析类型）正常采集，VTune 会自动识别并消费这些标记/trace 数据。
4. 在 GUI 的 Timeline 里查看自定义 task/frame，或在 Bottom-up/Caller-Callee 里看到以方法名呈现的 JIT 代码热点，而不是裸地址。

## Reference
- [API Support - Vtune Profiler User Guide (2026-1)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/api-support.html)
- [ITT API / JIT Profiling API Source - intel/ittapi](https://github.com/intel/ittapi)
- [Profiling PyTorch workloads with the ITT API (PyTorch Tutorials)](https://docs.pytorch.org/tutorials/recipes/profile_with_itt.html)

