-----

| Title     | OPT PARA OpenMP Offload                           |
| --------- | ------------------------------------------------- |
| Created @ | `2021-11-03T05:56:24Z`                            |
| Updated @ | `2024-04-07T02:24:39Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/24) |

-----

## Reference

  - [Get Started with OpenMP\* Offload to GPU for the Intel® oneAPI
    DPC++/C++ Compiler and Intel® Fortran
    Compiler](https://www.intel.com/content/www/us/en/develop/documentation/get-started-with-cpp-fortran-compiler-openmp/top.html)
  - [OpenMP offload
    compilers](https://github.com/ye-luo/openmp-target/wiki/OpenMP-offload-compilers)
  - [Introduction to oneAPI and OpenMP\* Offload with
    C/C++](https://github.com/oneapi-src/oneAPI-samples/blob/master/DirectProgramming/C%2B%2B/Jupyter/OpenMP-offload-training/intro/intro.ipynb)
  - [Managing Device Data
    (C/C++)](https://github.com/oneapi-src/oneAPI-samples/blob/master/DirectProgramming/C%2B%2B/Jupyter/OpenMP-offload-training/datatransfer/datatransfer.ipynb)
  - [OpenMP\* Device Parallelism
    (C/C++)](https://github.com/oneapi-src/oneAPI-samples/blob/master/DirectProgramming/C%2B%2B/Jupyter/OpenMP-offload-training/parallelism/parallelism.ipynb)
  - [Programming Your GPU with
    OpenMP](https://github.com/UoB-HPC/openmp-tutorial/blob/master/omp_GPGPU_prog_SC21.pdf)

## Brief

  - OpenMP 支持 Device Code 在 Intel/AMD/NVIDIA GPU 上运行
  - OpenMP 4.0+ 开始支持

<!-- end list -->

    ...// Sequential Host Code
    
    #pragma omp target     //Target Region Executed on the Device
    {
        for (...) {
            ...;
        }
    }
    ...// More Sequential Host Code

    export OMP_TARGET_OFFLOAD={"MANDATORY" | "DISABLED" | "DEFAULT" }
    - MANDATORY:  run on GPU
    - DISABLED: run on CPU
    - DEFAULT GPU --> Fall back to CPU

### API

| API                        | Description                        |
| -------------------------- | ---------------------------------- |
| `omp_get_default_device()` | 返回默认的 `Target Device`              |
| `omp_set_default_device()` | 设置默认的 `Target Device`              |
| `omp_get_num_devices()`    | 返回 `Target Device` 的数量             |
| `omp_get_device_num()`     | 返回当前 `thread` 执行的 `Device Number`  |
| `omp_is_initial_device()`  | 返回当前 `thread` 是否执行在 `Host Device`  |
| `omp_get_initial_device()` | 返回 `Host Device` 的 `Device Number` |

### Compiler Option

| Compiler Option                      | Description                                                       |
| ------------------------------------ | ----------------------------------------------------------------- |
| `-fopenmp-targets=spir64`            | [Intel Compiler](/Intel_Compiler) 选项使编译器产生 `x86 + SPIR64` 用于 iGPU |
| `-fopenmp-targets=amdgcn-amd-amdhsa` | AMD AOMP                                                          |
| `-mp=gpu -gpu=cc70`                  | NVIDIA HPC                                                        |

## GPU & OpenMP

### iGPU Arch

  - 每个 GPU 包含多个 `Slice`
  - 每个 `Slice` 包含多个 `SubSlice`
  - 每个 `SubSlice` 包含多个 `EU`
  - 每个 `EU` 包含多个 `SIMD ALUs`

![image](media/cae65d89bb037b4ccd8aa8ff549eb0af3ffc65bf.png)

### OpenMP Map to GPU

  - `  #pragma omp teams num_teams(8) thread_limit(16) `
      - `num_teams`: 限制 team 个数
      - `thread_limit`: 限制 team 中的 threads 数量

| OpenMP | GPU Hardware                          |
| ------ | ------------------------------------- |
| SIMD   | SIMD Lane (Channel)                   |
| Thread | SIMD Thread mapped to an EU           |
| Team   | Group of threads mapped to a Subslice |
| League | Multiple Teams mapped to a GPU        |

### Example

    #pragma omp target teams distribute parallel for simd

  - **target**: 在 `target device` 上执行
  - **teams**: 创建多个 `teams(master threads)` (此处为 4 个)
  - **distrubute**: 分发 `loop iterations` 到 `teams(master thread)`
    (此处为每个team 32 iterations)
  - **parallel**: 为每个 `team(master thread)` 创建 threads (此处每个 team 4 个
    threads)
  - **for**: 分发 iterations 到 `team threads` (32 iterations --\> 4 team
    threads)
  - **simd**: 指定 执行 SIMD Instructions

![image](media/0b20f70029c8b88883275c88512d0d141bfc893e.png)

## Asynchronous Offloading

  - `#pragma omp target nowait`
  - `#pragma omp taskwait`

### Example

  - `#pragma omp target map(to:b,c,d) map(from:a) nowait`: Asyn Run on
    Target Device (iGPU)
  - `#pragma omp task`: Run other task on CPU
  - `#pragma omp taskwait`: Synchronization

<!-- end list -->

    #pragma omp target map(to:b,c,d) map(from:a) nowait
    {
        #pragma omp teams distribute parallel for simd
        for (i=0; i<500; i++) {
            a[i] = b[i] * c + d;
        }
    }
    
    #pragma omp task
        other_work();
    
    #pragma omp taskwait
        a0 = a[0];

## `OpenMP Offload` VS DPCPP(SYCL)

  - OpenMP 适合迁移 `C/Fortran` 应用
  - DPCPP(SYCL) 适合迁移 CUDA/OpenCL 应用
