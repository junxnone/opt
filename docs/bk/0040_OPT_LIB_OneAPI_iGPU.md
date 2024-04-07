-----

| Title     | OPT LIB OneAPI iGPU                               |
| --------- | ------------------------------------------------- |
| Created @ | `2021-07-26T07:30:03Z`                            |
| Updated @ | `2024-04-07T03:08:44Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/40) |

-----

# OneAPI iGPU

## Reference

  - [适合GPU运算的类型](https://blog.csdn.net/fortuna_i/article/details/81183971)

## Brief

  - 足够多的task，使计算资源占满
  - 最小化host(CPU) 和 device(GPU) 同步
  - 最小化数据传输
  - 保持数据存在更快的memory中

-----

  - 尽量保持数据在加速器内存中
  - 访问持续性的内存块
  - 结构化数据块

-----

  - Slice
      - 8x SubSlices(Gen11) / 6x SubSlices(Gen12)
  - SubSlice
      - 8x EU(Gen11)/ 16x EU(Gen12)
  - EU - `execution units`
      - 7x threads
          - 128 GRF - `General purpose registers`
      - 2x SIMD ALU - `Single Instruction Multiple Data Arithmetic Logic
        Units`
          - 4x 32 FP/INT 计算 或 8x 16 FP 计算

## 适合 GPU 的运算

  - 大量的轻量级运算
  - 耦合度低的并行运算
  - 计算密集型
  - 浮点型运算

## UseCase
