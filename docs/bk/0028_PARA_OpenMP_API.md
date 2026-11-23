-----

| Title     | PARA OpenMP API                                   |
| --------- | ------------------------------------------------- |
| Created @ | `2021-08-07T03:56:52Z`                            |
| Updated @ | `2024-04-07T02:32:34Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/28) |

-----

# OpenMP API

  - API 分类
      - 环境设定获取
      - 锁定
      - 计时

## 环境变量获取

| 函数                     | 描述                                                   |
| ---------------------- | ---------------------------------------------------- |
| omp\_set\_num\_threads | 设置即将发生的并行区域中的线程数，除非由 num\_threads 子句重写。              |
| omp\_get\_num\_threads | 返回并行区域中的线程数。                                         |
| omp\_get\_max\_threads | 返回一个整数，该整数等于或大于当在代码中没有定义 num\_threads 的并行区域时可使用的线程数。 |
| omp\_get\_thread\_num  | 返回线程团队内执行的线程的线程号。 ID \[0, 1, 2, ...\]                |
| omp\_get\_num\_procs   | 返回调用函数时可用的处理器数。                                      |
| omp\_in\_parallel      | 如果从并行区域内调用，则返回非零值。                                   |
| omp\_set\_dynamic      | 指示可在即将推出的并行区域中使用的线程数进行调整。                            |
| omp\_get\_dynamic      | 返回一个值，该值指示是否可以在运行时调整即将存在的并行区域中的可用线程数。                |
| omp\_set\_nested       | 启用嵌套并行度。                                             |
| omp\_get\_nested       | 返回一个值，该值指示是否启用嵌套并行度。                                 |

## 锁定

| 函数                       | 描述                      |
| ------------------------ | ----------------------- |
| omp\_init\_lock          | 初始化简单锁。                 |
| omp\_init\_nest\_lock    | 初始化锁。                   |
| omp\_destroy\_lock       | 取消锁。                    |
| omp\_destroy\_nest\_lock | 取消 a.17 锁。              |
| omp\_set\_lock           | 阻止线程的执行，直到有可用锁为止。       |
| omp\_set\_nest\_lock     | 阻止线程的执行，直到有可用锁为止。       |
| omp\_unset\_lock         | 释放锁。                    |
| omp\_unset\_nest\_lock   | 释放一个 a.17 锁。            |
| omp\_test\_lock          | 尝试设置锁，但不阻止线程的执行。        |
| omp\_test\_nest\_lock    | 尝试设置 a.17 锁，但不会阻塞线程的执行。 |

## 计时

| 函数              | 描述                    |
| --------------- | --------------------- |
| omp\_get\_wtime | 返回一段时间内所用时间的值（以秒为单位）。 |
| omp\_get\_wtick | 返回处理器时钟计时周期之间的秒数。     |

## Reference

  - [OMP
    函数](https://docs.microsoft.com/zh-cn/cpp/parallel/openmp/reference/openmp-functions?view=msvc-160)
  - [OpenMP-API-Specification-5.0](https://www.openmp.org/wp-content/uploads/OpenMP-API-Specification-5.0.pdf)
