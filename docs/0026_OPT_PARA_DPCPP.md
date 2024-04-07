---
Title | OPT PARA DPCPP
-- | --
Created @ | `2021-10-28T08:00:47Z`
Updated @| `2024-04-07T02:24:47Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/26)

---


# DPC++

- **DPC++** - `Data Parallel C++` - 是 Intel 的 SYCL 实现
- 用于数据并行编程 & 异构编程
- C++ 标准 > `C++17`
-  = `ISO C++` + `SYCL standard` + `extensions`
- 实现了 [SYCL](/0031_OPT_PARA_SYCL) 接口
  - 使用 [OpenCL](/0034_OPT_PARA_OpenCL) 后端加速 Intel CPU/iGPU/FPGAs
  - 使用 `CUDA+PTX` 后端加速 Nvidia GPUs




## DPCPP VS OPENCL/CUDA/HIP

name | Description
-- | --
OPENCL/CUDA/HIP | - Low-level<br>- Separate source for device code<br>- Device code only
`DPC++(SYCL)`/OpenMP | - Higher-level<br>- Single source<br>- Host  and Device code


## Reference
- [Intel® oneAPI DPC++/C++ Compiler Developer Guide and Reference](https://www.intel.com/content/www/us/en/develop/documentation/oneapi-dpcpp-cpp-compiler-dev-guide-and-reference/top/optimization-and-programming/intel-oneapi-level-zero.html)
- [oneAPI DPC++ Compiler documentation](https://intel.github.io/llvm-docs/GetStartedGuide.html)
- [Intel Data Parallel C++ (and SYCL 2020) Tutorial](https://github.com/jeffhammond/dpcpp-tutorial)
- [SYCL 1.2.1 spec](https://www.khronos.org/registry/SYCL/specs/sycl-1.2.1.pdf)
- [DPCPP API](https://intel.github.io/llvm-docs/doxygen/index.html) 
- [Intel llvm Implementaion](https://github.com/intel/llvm/tree/sycl/)
- [Level Zero](https://dgpu-docs.intel.com/technologies/level-zero.html)
- [DPCPP Reference](https://oneapi-src.github.io/DPCPP_Reference/index.html)
- [training - intel](https://techdecoded.intel.io/quickhits/overview-of-oneapi-dpc-programming)
