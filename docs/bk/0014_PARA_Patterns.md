-----

| Title     | PARA Patterns                                     |
| --------- | ------------------------------------------------- |
| Created @ | `2023-03-10T08:30:25Z`                            |
| Updated @ | `2024-04-07T02:25:58Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/14) |

-----

# Parallel Patterns

## Map

  - 简单的并行 Kernel
  - 每个输入通过某个函数映射到独立的输出

![image](media/5b9a64f4ced3ba51a4f60623d02a068f0059dd51.png)

## Stencil

  - 结构化的数据
  - 输入和相邻的输入应用 `stencil` 后产生一个输出

![image](media/0d4b90ba0088790d88a56efb989471b2870e165a.png)

## Reduction

  - 合并数据
  - sum/min/max

![image](media/2041c2e78ffe58c98b55f6ff95494984e79b6279.png)

## Scan

![image](media/245799936c644a36f3b0e159e9a1ad9d65df9195.png)

## Pack

  - 将连续输入元素通过 `Boolean` 筛选后输出到连续位置

![image](media/1657807a64e26efef29b09e55c93d34ed0c69489.png)

## Unpack

  - 和 `Pack` 相反
  - 产生不连续的输出

![image](media/a3619655b34dd6950364350b16d4e061b6872256.png)
