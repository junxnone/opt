---
Title | LIB OneAPI oneDPL
-- | --
Created @ | `2021-12-27T09:29:14Z`
Updated @| `2024-04-07T03:09:36Z`
Labels | ``
Edit @| [here](https://github.com/junxnone/opt/issues/43)

---
# Intel OneAPI oneDPL

- DPL - `DPC++ Library`
- 包含内容
  - [Standard C++ APIs](https://www.intel.com/content/www/us/en/docs/onedpl/developer-guide/2022-0/tested-standard-c-apis.html)
  - [Parallel STL Algorithms](https://www.intel.com/content/www/us/en/developer/articles/guide/get-started-with-parallel-stl.html)
  - [Extensions APIs](https://www.intel.com/content/www/us/en/docs/onedpl/developer-guide/2022-0/parallel-api.html)

## 使用 Buffer
- 当调用多次 dpl APIs 时 如果使用 Host 变量，则存在每次调用 API, Data 都会经历一次 `Host --> Device --> Host`
- 使用 Buffer 可以避免重复 copy

```
    buffer buf(v);
    auto buf_begin = oneapi::dpl::begin(buf);
    auto buf_end   = oneapi::dpl::end(buf);

    oneapi::dpl::for_each(make_device_policy(q), buf_begin, buf_end, [](int &a){ a *= 3; });
    oneapi::dpl::sort(make_device_policy(q), buf_begin, buf_end);
```

## 使用 USM
- 使用 USM 指针
- 使用 USM Allocators

## Reference
- [oneAPI DPC++ Library (oneDPL) Guide](https://oneapi-src.github.io/oneDPL/index.html)
- [code - Github](https://github.com/oneapi-src/oneDPL)
- [C++ Standard Execution Policies](https://en.cppreference.com/w/cpp/algorithm/execution_policy_tag_t)

## Brief
- DPCPP High-level API
- Components
  - Parallel API
  - API for DPC++ Kernels
  - Macros




