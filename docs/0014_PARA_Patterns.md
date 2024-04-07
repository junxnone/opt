---
Title | PARA Patterns
-- | --
Created @ | `2023-03-10T08:30:25Z`
Updated @| `2024-04-07T02:25:58Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/14)

---
# Parallel Patterns

## Map
- 简单的并行 Kernel
- 每个输入通过某个函数映射到独立的输出

![image](https://user-images.githubusercontent.com/2216970/224264522-94968d83-271b-4457-a523-71bce5c545b6.png)


## Stencil
- 结构化的数据
- 输入和相邻的输入应用 `stencil` 后产生一个输出

![image](https://user-images.githubusercontent.com/2216970/224264774-62296496-86de-4b3f-a5e7-7afa6911b29c.png)

## Reduction
- 合并数据
- sum/min/max

![image](https://user-images.githubusercontent.com/2216970/224265274-f923d868-be4e-4a1b-ad94-b392699f186e.png)

## Scan


![image](https://user-images.githubusercontent.com/2216970/224265998-3ab39320-8aca-452a-b89e-38ececee3059.png)

## Pack
- 将连续输入元素通过 `Boolean` 筛选后输出到连续位置


![image](https://user-images.githubusercontent.com/2216970/224270190-8a8062c2-15d0-4e69-99f4-66ec9d67beaf.png)

## Unpack
- 和 `Pack` 相反
- 产生不连续的输出

![image](https://user-images.githubusercontent.com/2216970/224270218-131838ee-672f-4e08-8bfe-852486009562.png)

