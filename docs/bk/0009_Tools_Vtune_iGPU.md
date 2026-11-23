-----

| Title     | Tools Vtune iGPU                                 |
| --------- | ------------------------------------------------ |
| Created @ | `2023-05-25T13:19:54Z`                           |
| Updated @ | `2024-04-07T02:18:54Z`                           |
| Labels    | \`\`                                             |
| Edit @    | [here](https://github.com/junxnone/opt/issues/9) |

-----

# Intel Vtune iGPU Analysis

## Setup

  - Intel Metric Discovery API Library
      - Windows 集成在 Drivers 里面
      - [Linux Install](https://github.com/intel/metrics-discovery)
  - Enable Permissions
  - Enable GPU utilization events (i915 ftrace events)

## Reference

  - [Set Up System for GPU
    Analysis](https://www.intel.com/content/www/us/en/docs/vtune-profiler/user-guide/2023-0/set-up-system-for-gpu-analysis.html)
  - [oneAPI GPU Optimization
    Guide](https://www.intel.com/content/www/us/en/docs/oneapi/optimization-guide-gpu/2023-0/overview.html)
  - [Using the Command-Line Interface to Analyze the Performance of a
    SYCL\* Application running on a
    GPU](https://www.intel.com/content/www/us/en/docs/vtune-profiler/cookbook/2023-1/profiling-gpu-from-cli.html)
  - [Optimize Applications for Intel® GPUs with Intel® VTune™
    Profiler](https://www.intel.com/content/www/us/en/developer/articles/technical/optimize-applications-for-intel-gpus-with-intel-vtune-profiler.html)
  - [Get Ultimate GPU Application Performance in 7
    Steps](https://www.intel.com/content/www/us/en/developer/articles/technical/get-ultimate-gpu-performance-in-7-steps-with-vtune.html)
  - [GPU Architecture Terminology for Intel® Xe
    Graphics](https://www.intel.com/content/www/us/en/developer/articles/technical/gpu-terminology-for-intel-xe.html)
