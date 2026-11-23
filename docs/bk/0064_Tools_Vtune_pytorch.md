-----

| Title     | Tools Vtune pytorch                               |
| --------- | ------------------------------------------------- |
| Created @ | `2026-02-27T03:26:23Z`                            |
| Updated @ | `2026-10-14T08:34:59Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/64) |

-----

# Debug pytorch with vtune

  - ITT API 本身的通用概念（Domain/String Handle/Task API 等）参见 [0010 Vtune
    ITTJIT APIs](0010_Tools_Vtune_DebugAPI.md)，本篇只讲 PyTorch 对它的集成方式。
  - 适用版本：PyTorch \>= 1.13（ITT API 自 1.13 起集成进 PyTorch）。
  - 隐式调用：默认情况下，当 ITT 功能启用时，所有通过 PyTorch 算子注册机制注册的算子都会被 ITT
    功能自动标记（时间线上表现为以 `aten::` 开头的标签）。
  - 显式调用：如果需要自定义标记，用户可以明确使用 [PyTorch
    文档](https://pytorch.org/docs/stable/profiler.html#intel-instrumentation-and-tracing-technology-apis)
    中提到的 Python API 来标记所需范围，**不需要**直接调用底层的 ITT C/C++ API。

> The ITT API had been integrated into PyTorch since 1.13.

  - 要对 PyTorch 脚本进行性能分析，建议将所有手动步骤（包括激活 Python 环境和设置所需的环境变量）都封装到一个 bash
    脚本中，然后对这个 bash 脚本进行性能分析。

## 显式调用

  - 为了启用显式调用，预期会被标记的代码应在 `torch.autograd.profiler.emit_itt()` 作用域下调用。例如：

<!-- end list -->

``` python
with torch.autograd.profiler.emit_itt():
  <code-to-be-profiled...>
```

  - 在 `emit_itt()` 作用域内，还可以用更细粒度的 API 给某个区域打自定义标签（二选一）：
      - 成对调用 `torch.profiler.itt.range_push(name)` /
        `torch.profiler.itt.range_pop()`
      - 或者用作用域写法 `with torch.profiler.itt.range(name):`

## 启动 VTune 分析

  - GUI 方式：打开 VTune，点击 **Configure
    Analysis…**，配置面板分三部分：**WHERE**（在哪台机器上跑）、**WHAT**（要分析的程序路径——推荐直接指向前面封装好的
    bash 脚本）、**HOW**（选择分析类型）。
  - CPU 场景：HOW 选择 `Hotspots`（或其他合适的 CPU 分析类型）。
  - XPU/GPU 场景：HOW 选择 `GPU Offload` 分析类型，其余步骤与 CPU 场景一致。
  - 远程/Web UI 方式：也可以不开 GUI 客户端，用 VTune Profiler Server 以网页形式查看/操作：
    ``` bash
    vtune-backend --web-port=8080 --allow-remote-access --enable-server-profiling
    ```

## 查看结果

  - 采集成功后，打开结果的 **Platform** 标签页查看时间线：
      - 主线程显示为一行 Python 线程，下方是各个 OpenMP 线程。
      - 以 `aten::` 开头的标签是 PyTorch 隐式标记的算子；`iteration_N` 这类标签是用
        `range_push/range_pop` 或 `range()` 显式标记的区域。
      - 标为 `convolution`/`oneDNN reorder` 的红色方框，是 Intel® oneAPI Deep
        Neural Network Library（oneDNN）自己打的标记，不是 PyTorch 算子标签。
      - 每个线程行里的棕色条表示该时刻的 CPU 占用率（条的高度占比 ≈ CPU 利用率），可以据此判断：各线程 CPU
        利用率是否充分、多线程之间是否均衡、OpenMP 线程启动/结束时是否存在抖动（同步不佳）。
  - XPU 场景下，时间线顶部会额外显示 GPU Computing Queue，可以看到各个 XPU Kernel 被派发进 GPU
    队列的情况。

## 示例代码

`sample.py`：两个算子（`Conv2d` + `Linear`）构成的小模型，推理 3 次，每次迭代用 `range()` 标记为
`iteration_N`（CPU/XPU 切换只需解开注释里的 `.to("xpu")`）：

``` python
# sample.py
import torch
import torch.nn as nn

class ITTSample(nn.Module):
    def __init__(self):
        super(ITTSample, self).__init__()
        self.conv = nn.Conv2d(3, 5, 3)
        self.linear = nn.Linear(292820, 1000)

    def forward(self, x):
        x = self.conv(x)
        x = x.view(x.shape[0], -1)
        x = self.linear(x)
        return x

def main():
    m = ITTSample()
    # 如需分析 XPU，取消下面两行注释
    # m = m.to("xpu")
    x = torch.rand(10, 3, 244, 244)
    # x = x.to("xpu")
    with torch.autograd.profiler.emit_itt():
        for i in range(3):
            with torch.profiler.itt.range(f'iteration_{i}'):
                m(x)

if __name__ == '__main__':
    main()
```

`launch.sh`：VTune 的 WHAT 配置里直接指向这个脚本，脚本内部负责激活环境变量再调用 `sample.py`：

``` bash
#!/bin/bash
BASEFOLDER=$(cd -- "$(dirname -- "${BASH_SOURCE[0]}")" &> /dev/null && pwd)
<Activate a Python environment>
cd ${BASEFOLDER}
python sample.py
```

## Reference

  - [Profiling PyTorch workloads with The Instrumentation and Tracing
    Technology (ITT)
    API](https://docs.pytorch.org/tutorials/recipes/profile_with_itt.html)
  - [PyTorch ITT API
    (torch.profiler.itt)](https://pytorch.org/docs/stable/profiler.html#intel-instrumentation-and-tracing-technology-apis)
  - [Run Intel® VTune™ Profiler Server
    (2026-1)](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2026-1/run-vtune-profiler-server.html)
