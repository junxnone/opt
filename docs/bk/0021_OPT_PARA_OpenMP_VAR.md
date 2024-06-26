-----

| Title     | OPT PARA OpenMP VAR                               |
| --------- | ------------------------------------------------- |
| Created @ | `2021-11-05T09:34:58Z`                            |
| Updated @ | `2024-04-07T02:24:28Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/21) |

-----

# 变量

## Reference

  - [OpenMP Internal Control
    Variables](https://www.openmp.org/spec-html/5.0/openmpse13.html)

## 对变量的声明

| Keyword      | Description                           |
| ------------ | ------------------------------------- |
| private      | 其列出来的变量对于线程私有                         |
| firstprivate | `private` + 对于线程局部存储的变量，其初值是进入并行区之前的值 |
| lastprivate  | `private` + 并行区里的值在最后会赋值给并行区前面的变量     |
| default      | 自定义一个并行区的默认的变量的作用范围                   |
| shared       | 其列出来的变量对于所有线程共享(**默认值**)              |
| reduction    | 对于各个线程私有的变量，在并行区结束时通过某种运算归一           |

## 环境变量 ICV(Internal Control Variables)

| 环境变量              | Description                                                                    |
| ----------------- | ------------------------------------------------------------------------------ |
| OMP\_SCHEDULE     | 指定调度方式<br>static/dynamic/guided/auto                                           |
| OMP\_NUM\_THREADS | 指定线程数 \[1,N\]                                                                  |
| OMP\_DYNAMIC      | 动态设定 `dyn-var` ICV \[TRUE/FALSE\]                                              |
| OMP\_PROC\_BIND   | thread affinity policy<br>`bind-var` ICV \[TRUE/FALSE, master, close, spread\] |
| OMP\_NESTED       | 是否嵌套                                                                           |
