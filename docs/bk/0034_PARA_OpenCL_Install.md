-----

| Title     | PARA OpenCL Install                               |
| --------- | ------------------------------------------------- |
| Created @ | `2020-03-18T08:05:25Z`                            |
| Updated @ | `2024-04-07T02:34:47Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/34) |

-----

# OpenCL Install

## Install on Ubuntu

    wget https://github.com/intel/compute-runtime/releases/download/20.10.16087/intel-gmmlib_19.4.1_amd64.deb
    wget https://github.com/intel/compute-runtime/releases/download/20.10.16087/intel-igc-core_1.0.3471_amd64.deb
    wget https://github.com/intel/compute-runtime/releases/download/20.10.16087/intel-igc-opencl_1.0.3471_amd64.deb
    wget https://github.com/intel/compute-runtime/releases/download/20.10.16087/intel-opencl_20.10.16087_amd64.deb
    wget https://github.com/intel/compute-runtime/releases/download/20.10.16087/intel-ocloc_20.10.16087_amd64.deb

    sudo dpkg -i *.deb
    sudo usermod -a -G video $(whoami)

## Tips

  - 可以使用 `clinfo` 查看 CL device

<!-- end list -->

    sudo apt install clinfo

> 有可能需要 `sudo`

    $ clinfo
    Number of platforms                               1
      Platform Name                                   Intel(R) OpenCL HD Graphics
      Platform Vendor                                 Intel(R) Corporation
      Platform Version                                OpenCL 2.1
      Platform Profile                                FULL_PROFILE
      Platform Extensions                             cl_khr_byte_addressable_store cl_khr_fp16 cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_icd cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atom
    ics cl_intel_subgroups cl_intel_required_subgroup_size cl_intel_subgroups_short cl_khr_spir cl_intel_accelerator cl_intel_driver_diagnostics cl_khr_priority_hints cl_khr_throttle_hints cl_khr_create_command_queue cl_intel_subgroups_char
    cl_intel_subgroups_long cl_khr_fp64 cl_khr_subgroups cl_khr_il_program cl_intel_spirv_device_side_avc_motion_estimation cl_intel_spirv_media_block_io cl_intel_spirv_subgroups cl_khr_spirv_no_integer_wrap_decoration cl_intel_unified_share
    d_memory_preview cl_khr_mipmap_image cl_khr_mipmap_image_writes cl_intel_planar_yuv cl_intel_packed_yuv cl_intel_motion_estimation cl_intel_device_side_avc_motion_estimation cl_intel_advanced_motion_estimation cl_khr_int64_base_atomics c
    l_khr_int64_extended_atomics cl_khr_image2d_from_buffer cl_khr_depth_images cl_intel_media_block_io cl_khr_3d_image_writes cl_intel_va_api_media_sharing
      Platform Host timer resolution                  1ns
      Platform Extensions function suffix             INTEL
    
      Platform Name                                   Intel(R) OpenCL HD Graphics
    Number of devices                                 1
      Device Name                                     Intel(R) Gen9 HD Graphics NEO
      Device Vendor                                   Intel(R) Corporation
      Device Vendor ID                                0x8086
      Device Version                                  OpenCL 2.1 NEO
      Driver Version                                  20.10.16087
      Device OpenCL C Version                         OpenCL C 2.0
      Device Type                                     GPU
      Device Profile                                  FULL_PROFILE
      Device Available                                Yes
      Compiler Available                              Yes
      Linker Available                                Yes
      Max compute units                               24
      Max clock frequency                             1050MHz
      Device Partition                                (core)
        Max number of sub-devices                     0
        Supported partition types                     None, None
      Max work item dimensions                        3
      Max work item sizes                             256x256x256
      Max work group size                             256
      Preferred work group size multiple              32
      Max sub-groups per work group                   32
      Sub-group sizes (Intel)                         8, 16, 32
      Preferred / native vector sizes
        char                                                16 / 16
        short                                                8 / 8
        int                                                  4 / 4
        long                                                 1 / 1
        half                                                 8 / 8        (cl_khr_fp16)
        float                                                1 / 1
        double                                               1 / 1        (cl_khr_fp64)
      Half-precision Floating-point support           (cl_khr_fp16)
        Denormals                                     Yes
        Infinity and NANs                             Yes
        Round to nearest                              Yes
        Round to zero                                 Yes
        Round to infinity                             Yes
        IEEE754-2008 fused multiply-add               Yes
        Support is emulated in software               No
      Single-precision Floating-point support         (core)
        Denormals                                     Yes
        Infinity and NANs                             Yes
        Round to nearest                              Yes
        Round to zero                                 Yes
        Round to infinity                             Yes
        IEEE754-2008 fused multiply-add               Yes
        Support is emulated in software               No
        Correctly-rounded divide and sqrt operations  Yes
      Double-precision Floating-point support         (cl_khr_fp64)
        Denormals                                     Yes
        Infinity and NANs                             Yes
        Round to nearest                              Yes
        Round to zero                                 Yes
        Round to infinity                             Yes
        IEEE754-2008 fused multiply-add               Yes
        Support is emulated in software               No
      Address bits                                    64, Little-Endian
      Global memory size                              6619983872 (6.165GiB)
      Error Correction support                        No
      Max memory allocation                           3309991936 (3.083GiB)
      Unified memory for Host and Device              Yes
      Shared Virtual Memory (SVM) capabilities        (core)
        Coarse-grained buffer sharing                 Yes
        Fine-grained buffer sharing                   No
        Fine-grained system sharing                   No
        Atomics                                       No
      Minimum alignment for any data type             128 bytes
      Alignment of base address                       1024 bits (128 bytes)
      Preferred alignment for atomics
        SVM                                           64 bytes
        Global                                        64 bytes
        Local                                         64 bytes
      Max size for global variable                    65536 (64KiB)
      Preferred total size of global vars             3309991936 (3.083GiB)
      Global Memory cache type                        Read/Write
      Global Memory cache size                        524288 (512KiB)
      Global Memory cache line size                   64 bytes
      Image support                                   Yes
        Max number of samplers per kernel             16
        Max size for 1D images from buffer            206874496 pixels
        Max 1D or 2D image array size                 2048 images
        Base address alignment for 2D image buffers   4 bytes
        Pitch alignment for 2D image buffers          4 pixels
        Max 2D image size                             16384x16384 pixels
        Max planar YUV image size                     16384x16352 pixels
        Max 3D image size                             16384x16384x2048 pixels
        Max number of read image args                 128
        Max number of write image args                128
        Max number of read/write image args           128
      Max number of pipe args                         16
      Max active pipe reservations                    1
      Max pipe packet size                            1024
      Local memory type                               Local
      Local memory size                               65536 (64KiB)
      Max number of constant args                     8
      Max constant buffer size                        3309991936 (3.083GiB)
      Max size of kernel argument                     1024
      Queue properties (on host)
        Out-of-order execution                        Yes
        Profiling                                     Yes
      Queue properties (on device)
        Out-of-order execution                        Yes
        Profiling                                     Yes
        Preferred size                                131072 (128KiB)
        Max size                                      67108864 (64MiB)
      Max queues on device                            1
      Max events on device                            1024
      Prefer user sync for interop                    Yes
      Profiling timer resolution                      83ns
      Execution capabilities
        Run OpenCL kernels                            Yes
        Run native kernels                            No
        Sub-group independent forward progress        Yes
        IL version                                    SPIR-V_1.2
        SPIR versions                                 1.2
      printf() buffer size                            4194304 (4MiB)
      Built-in kernels                                block_motion_estimate_intel;block_advanced_motion_estimate_check_intel;block_advanced_motion_estimate_bidirectional_check_intel;
      Motion Estimation accelerator version (Intel)   2
        Device-side AVC Motion Estimation version     1
          Supports texture sampler use                Yes
          Supports preemption                         No
      Device Extensions                               cl_khr_byte_addressable_store cl_khr_fp16 cl_khr_global_int32_base_atomics cl_khr_global_int32_extended_atomics cl_khr_icd cl_khr_local_int32_base_atomics cl_khr_local_int32_extended_atom
    ics cl_intel_subgroups cl_intel_required_subgroup_size cl_intel_subgroups_short cl_khr_spir cl_intel_accelerator cl_intel_driver_diagnostics cl_khr_priority_hints cl_khr_throttle_hints cl_khr_create_command_queue cl_intel_subgroups_char
    cl_intel_subgroups_long cl_khr_fp64 cl_khr_subgroups cl_khr_il_program cl_intel_spirv_device_side_avc_motion_estimation cl_intel_spirv_media_block_io cl_intel_spirv_subgroups cl_khr_spirv_no_integer_wrap_decoration cl_intel_unified_share
    d_memory_preview cl_khr_mipmap_image cl_khr_mipmap_image_writes cl_intel_planar_yuv cl_intel_packed_yuv cl_intel_motion_estimation cl_intel_device_side_avc_motion_estimation cl_intel_advanced_motion_estimation cl_khr_int64_base_atomics c
    l_khr_int64_extended_atomics cl_khr_image2d_from_buffer cl_khr_depth_images cl_intel_media_block_io cl_khr_3d_image_writes cl_intel_va_api_media_sharing
    NULL platform behavior
      clGetPlatformInfo(NULL, CL_PLATFORM_NAME, ...)  Intel(R) OpenCL HD Graphics
      clGetDeviceIDs(NULL, CL_DEVICE_TYPE_ALL, ...)   Success [INTEL]
      clCreateContext(NULL, ...) [default]            Success [INTEL]
      clCreateContextFromType(NULL, CL_DEVICE_TYPE_DEFAULT)  Success (1)
        Platform Name                                 Intel(R) OpenCL HD Graphics
        Device Name                                   Intel(R) Gen9 HD Graphics NEO
      clCreateContextFromType(NULL, CL_DEVICE_TYPE_CPU)  No devices found in platform
      clCreateContextFromType(NULL, CL_DEVICE_TYPE_GPU)  Success (1)
        Platform Name                                 Intel(R) OpenCL HD Graphics
        Device Name                                   Intel(R) Gen9 HD Graphics NEO
      clCreateContextFromType(NULL, CL_DEVICE_TYPE_ACCELERATOR)  No devices found in platform
      clCreateContextFromType(NULL, CL_DEVICE_TYPE_CUSTOM)  No devices found in platform
      clCreateContextFromType(NULL, CL_DEVICE_TYPE_ALL)  Success (1)
        Platform Name                                 Intel(R) OpenCL HD Graphics
        Device Name                                   Intel(R) Gen9 HD Graphics NEO
    
    ICD loader properties
      ICD loader Name                                 OpenCL ICD Loader
      ICD loader Vendor                               OCL Icd free software
      ICD loader Version                              2.2.11
      ICD loader Profile                              OpenCL 2.1

## Reference

  - [intel compute-runtime
    release](https://github.com/intel/compute-runtime/releases)
