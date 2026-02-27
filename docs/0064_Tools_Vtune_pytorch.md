---
Title | Tools Vtune pytorch
-- | --
Created @ | `2026-02-27T03:26:23Z`
Updated @| `2026-02-27T03:26:23Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/64)

---
# Debug pytorch with vtune

- 隐式调用：默认情况下，当 ITT 功能启用时，所有通过 PyTorch 算子注册机制注册的算子都会被 ITT 功能自动标记。
- 显式调用：如果需要自定义标记，用户可以明确使用 [PyTorch 文档](https://pytorch.org/docs/stable/profiler.html#intel-instrumentation-and-tracing-technology-apis) 中提到的 API 来标记所需范围。

> The ITT API had been integrated into PyTorch since 1.13.

- 要对 PyTorch 脚本进行性能分析，建议将所有手动步骤（包括激活 Python 环境和设置所需的环境变量）都封装到一个 bash 脚本中，然后对这个 bash 脚本进行性能分析。


## 显示调用

- 为了启用显式调用，预期会被标记的代码应在 `torch.autograd.profiler.emit_itt()` 作用域下调用。例如：

```
with torch.autograd.profiler.emit_itt():
  <code-to-be-profiled...>
```

## Reference
- [Profiling PyTorch workloads with The Instrumentation and Tracing Technology (ITT) API](https://docs.pytorch.org/tutorials/recipes/profile_with_itt.html)

