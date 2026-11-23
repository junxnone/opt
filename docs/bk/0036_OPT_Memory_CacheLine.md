-----

| Title     | OPT Memory CacheLine                              |
| --------- | ------------------------------------------------- |
| Created @ | `2022-04-27T02:59:04Z`                            |
| Updated @ | `2024-04-07T02:42:58Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/36) |

-----

## Reference

  - [CPU Cache
    Optimization](https://zzqcn.github.io/perf/cpu_cache.html)
    \[[code](https://github.com/zzqcn/storage/tree/main/code/c/cache_opt)\]
  - [伪共享（False Sharing）](https://zhuanlan.zhihu.com/p/55917869)
  - [False Sharing - Intel
    Vtune](https://www.intel.com/content/www/us/en/develop/documentation/vtune-cookbook/top/tuning-recipes/false-sharing.html)

## Brief

  - \[\[Cache\]\]
  - 结构体定义导致的每次 Cache 读取效率低
      - 结构体中包含无用变量
      - 结构体中的数据类型混排对齐
      - 数据行列访问
  - **False Sharing**: 多个线程访问同一 `cache line` 的不同数据

## Optimization

### 无用变量

  - 结构体中包含无用变量，导致读取到的有用数据减少，需要更多的读取时间

<!-- end list -->

    struct DATA1
    {
        int a;
        int b;
        int c;
        int d;
    };
    
    struct DATA2
    {
        int a;
        int b;
    };

| Define | 排列                                                           |
| ------ | ------------------------------------------------------------ |
| Data1  | ![image](media/f2c1b01134bb3ffc73c945e61ba4f6b2dc59d047.png) |
| Data2  | ![image](media/18f79b0cc6688d13fc7e993c86cf90fe71b6390b.png) |

### 数据类型混排对齐

  - 数据类型对齐导致的空间浪费，导致需要读取更多次数

<!-- end list -->

    struct DATA3
    {
        char a;
        int b;
        char c;
    };
    struct DATA4
    {
        int b;
        char a;
        char c;
    };

| Define | 排列                                                           |
| ------ | ------------------------------------------------------------ |
| Data3  | ![image](media/5614dc9eb6c306b317eee729824b132420d26e53.png) |
| Data4  | ![image](media/36824f269f79f67b9274d1f96a1880c1d1c611f9.png) |

### 数据行列访问

  - 内存以行存储数据, 按行访问数据，每次 Cache 读取能够取到更多有效数据

<!-- end list -->

    // 按列访问
    char * p;
    p = new char[SIZE];
    for (long x=0; x<sRowSize; x++) {
        for (long y=0; y<nbRows; y++) {
            p[x+y*sRowSize]++;
        }
    }

    // 按行访问
    char * p;
    p = new char[SIZE];
    for (long y=0; y<nbRows; y++) {
        for (long x=0; x<sRowSize; x++) {
            p[x+y*sRowSize]++;
        }
    }

| Mode            | 按列访问                                                                                                                       | 按行访问                                                                                                                       |
| --------------- | -------------------------------------------------------------------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------- |
| Cache Line 读取区域 | <img width=200 src="https://user-images.githubusercontent.com/2216970/165432374-f045fb72-6d56-4b93-be94-b4f8493d0ca7.png"> | <img width=200 src="https://user-images.githubusercontent.com/2216970/165432478-0e33ef14-8511-4c15-a2cb-7f1600210b34.png"> |

### False Sharing

![image](media/c487f2dfe70b54969c3e0aba0271112c962027b0.png)
