-----

| Title     | OPT Loop                                          |
| --------- | ------------------------------------------------- |
| Created @ | `2021-12-14T07:42:33Z`                            |
| Updated @ | `2024-04-07T03:17:16Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/49) |

-----

# Loop

  - 编译器可以自动优化一部分循环
  - Loop Pipelining
  - Loop Unrolling - 循环展开

## Loop Pipelining

| No Loop![image](media/dfecad3465f2365e02375c942c3c7f6a3a77a447.png) | ![image](media/50e605b1c8725f275b4501ce54313eb111877393.png) |
| ------------------------------------------------------------------- | ------------------------------------------------------------ |

## Loop Unrolling

  - 代码复制，用以减少循环分支指令执行的次数
  - 更多的指令级并行
  - 更多寄存器重用，提高访存效率
  - 展开方式
      - 编译器自动循环展开(LLVM - `-funroll-loops`)
      - pragma 定义展开(`#pragma clang loop unroll(enable)`)
      - 手动展开
  - 需要考虑数据依赖

<!-- end list -->

    accum = 0;
    #pragma unroll N
    for (size_t i=0; i<4; i++) {
      accum += data[i];
    }
    sum_out = accum;

| unroll/nounroll | Execution                                                    |
| --------------- | ------------------------------------------------------------ |
| nounroll        | ![image](media/d33c5e28b2faf365849f00bc3ef338c611e73257.png) |
| Unroll 2        | ![image](media/1130b47a4c3fa7beb320ce1c8e08e99076a76586.png) |
| Fully Unroll    | ![image](media/6798e907b5b5d17b173235ee435c409cc1074387.png) |

| Basic Loop                                                   | Unrolled Loop                                                | Pipelined Loop                                               |
| ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| ![image](media/7693eb40490469501a9324ca6585c01d2f94ede6.png) | ![image](media/3e3748cecb650978d5f0525adf40d9049c27b74b.png) | ![image](media/a243279ed12372f5cf9183ed43b9e9535c69e628.png) |

## Reference

  - [HLS Loop
    Optimizations](https://learning.intel.com/developer/learn/course/external/view/elearning/242/hls-loop-optimizations-part-3-of-7)
  - [循环优化 -
    先进编译实验室](https://space.bilibili.com/1540261574/channel/collectiondetail?sid=693322)
