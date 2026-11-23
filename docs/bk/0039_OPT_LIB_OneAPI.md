-----

| Title     | OPT LIB OneAPI                                    |
| --------- | ------------------------------------------------- |
| Created @ | `2021-06-23T06:24:28Z`                            |
| Updated @ | `2024-04-07T03:08:41Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/39) |

-----

## OneAPI

### Compilers & Language

  - **C++ compiler Classic**
  - **DPC++/C++ Compiler** [DPCPP](/0036_OPT_PARA_DPCPP)
  - **Fortran Compiler**

### Libraries

  - **IPP**: `Integrated Performance Primitives` : 视觉/信号/安全/存储 优化库
  - **oneDPL**: 基于 `C++ STL` `Parallel STL` `Boost.Compute` `SYCL*` 的
    API
  - **oneMKL**: `Math Kernel Library` 高性能数学库实现
  - **oneTBB**: `Threading Building Blocks` 基于线程的并行编程库
  - **oneVPL**: `Video Processing Library` 视频处理加速库
  - **oneDAL**: `Data Analytics Library` 数据科学加速算法
  - **oneDNN**: `Deep Neural Network` 高性能实现的深度学习框架
  - **oneCCL**: `Collective Communications Library` 服务于深度学习框架的跨硬件通信库
  - **Intel MPI**: 通信相关

### Analyzers and Debuggers

  - **Advisor**: 设计和分析向量化并行化等高性能应用
  - **VTune Profiler**: 查找应用性能瓶颈的工具
  - **Inspector**: 定位和调试 线程/内存
  - **Trace Analyzer and Collector** MPI 调试

### CUDA 迁移工具

  - **DPCT**: `DPC++ Compatibility Tool` 迁移 `CUDA* code` 到 `SCYL Code`
  - \--\>[SYCLomatic](https://oneapi-src.github.io/SYCLomatic/index.html)
    \[[Code](https://github.com/oneapi-src/SYCLomatic)\]

## Reference

  - [OneAPI](https://www.intel.com/content/www/us/en/developer/tools/oneapi/overview.html)
  - [OneAPI Specification](https://spec.oneapi.io/versions/1.2-rev-1/)
    \[[repo](https://github.com/oneapi-src/oneAPI-spec)\]
  - [oneAPI-samples](https://github.com/oneapi-src/oneAPI-samples)
  - [oneAPI
    Components](https://www.intel.com/content/www/us/en/developer/tools/oneapi/components.html)

## Install with Linux

    wget https://apt.repos.intel.com/intel-gpg-keys/GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
    sudo apt-key add GPG-PUB-KEY-INTEL-SW-PRODUCTS.PUB
    echo "deb https://apt.repos.intel.com/oneapi all main" | sudo tee /etc/apt/sources.list.d/oneAPI.list
    sudo add-apt-repository "deb https://apt.repos.intel.com/oneapi all main"
    sudo -E apt install intel-basekit
