---
Title | OPT PARA SYCL ESIMD
-- | --
Created @ | `2024-01-15T14:47:08Z`
Updated @| `2024-04-07T02:23:41Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/11)

---
# ESIMD
- ESIMD  - `Explicit SIMD` SYCL extension
- 用于 Intel GPUs 架构的高效底层编程扩展，用于编写显示向量化代码
- APIs - `loads/stores/gathres/scatters/ explicit cache hints/ gpu inline assembly`
- `相当于 SYCL subgroup size=1` - 每个subgroup 映射到一个 `硬件 thread`
- 只能用于 **Intel GPUs & Host Device**
- **Intel GPU ISA** - `Intel GPU Instruction Set Architecture`

## API
- SIMD Vector
- simd_view
- Reduction func
- Memory Access
  - SLM 
  - Atomics
- Math 
- Dot Product Accumulate Systolic - DPAS 

---

- 哪些可以直接 `Run on Host Device` ?


- 常用函数？
  - replicate_xxxx<>()
  - bit_cast_view<>()
  - merge() 


## Reference
- [Explicit SIMD Programming Extension for DPC++](https://github.com/intel/llvm/blob/sycl/sycl/doc/extensions/supported/sycl_ext_intel_esimd/sycl_ext_intel_esimd.md)
- [ESIMD API](https://intel.github.io/llvm-docs/doxygen/group__sycl__esimd.html)
- [examples](https://github.com/intel/llvm/blob/sycl/sycl/doc/extensions/supported/sycl_ext_intel_esimd/examples/README.md)
- 
