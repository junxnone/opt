---
Title | PARA OpenMP UseCase
-- | --
Created @ | `2021-11-05T09:31:24Z`
Updated @| `2024-04-07T02:31:02Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/22)

---

# OpenMP UseCase
- 头文件  `include <omp.h>`
- 格式 `#pragma omp <编译关键字> [ 子句[ [,] 子句]…… ]`


## 设定线程数量
- 设置环境变量 `OMP_NUM_THREADS=N ./your/program`
- 编译指令中设定 `#pragma omp parallel num_threads(N)

## parallel
- 每个 thread 都执行 并行区域 code

## for
- 分发 loop iteration 到所有 thread

```
#pragma omp parallel
#pragma omp for
```
```
#pragma omp parallel for
```

## sections

```
#pragma omp parallel sections
```
- single
- task

## Reference

