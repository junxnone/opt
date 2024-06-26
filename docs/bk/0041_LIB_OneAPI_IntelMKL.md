-----

| Title     | LIB OneAPI IntelMKL                               |
| --------- | ------------------------------------------------- |
| Created @ | `2021-08-03T02:32:35Z`                            |
| Updated @ | `2024-04-07T03:10:26Z`                            |
| Labels    | \`\`                                              |
| Edit @    | [here](https://github.com/junxnone/opt/issues/41) |

-----

# IntelMKL

  - MKL - `Math Kernel Library`
  - Intel-Optimized Math Library for Numerical Computing
  - Functions
      - Linear algebra - 线性代数
          - \[\[BLAS\]\] - 基本线性代数
          - \[\[LAPACK\]\] - 高级线性代数
          - Sparse solvers
      - FFT - `fast Fourier transforms` - `1D/2D/3D`
      - RNG - `random number generator`
      - Summary Statics
      - data fitting
      - Vector Math
  - **Language Support:**
      - DPCPP - `DPCPP API`
      - CPP/C - `C API`
      - Fortran - `Fortran API`

## API

### BLAS

  - Level 1 - `Vector - Vector 操作`
  - Level 2 - `Matrix - Vector 操作`
  - Level 3 - `Matrix - Matrix 操作`

#### Levle 1

| fun    | Formula                                                                                                                     | Description               |
| ------ | --------------------------------------------------------------------------------------------------------------------------- | ------------------------- |
| asum   | $result = \\sum\_{i=1}^{n}(|Re(x\_i)| + |Im(x\_i)|)$                                                                        | 计算实数元素之和或者复数实部和虚部之和       |
| axpy   | $y \\leftarrow alpha \* x + y$                                                                                              | scalar-vector 相乘 加 vector |
| copy   | $y \\leftarrow x$                                                                                                           |                           |
| dot    | $result = \\sum\_{i=1}^{n}X\_iY\_i$                                                                                         | 点乘                        |
| sdsdot | $result = sb + \\sum\_{i=1}^{n}X\_iY\_i$                                                                                    | 点乘加                       |
| dotc   | $result = \\sum\_{i=1}^{n}\\overline{X\_i}Y\_i$                                                                             | 复数点乘(第一个参数共轭)             |
| dotu   | $result = \\sum\_{i=1}^{n}X\_iY\_i$                                                                                         | 复数点乘                      |
| nrm2   | $result = | x|$                                                                                                             | 欧式范数                      |
| rot    | $\\begin{bmatrix} x \\ y \\end{bmatrix}\\leftarrow \\begin{bmatrix} c*x + s*y\\ -s*x + c*y \\end{bmatrix}$                  |                           |
| rotg   | $\\begin{bmatrix}c & s \\ -s & c\\end{bmatrix}. \\begin{bmatrix}a \\ b\\end{bmatrix} =\\begin{bmatrix}r \\ 0\\end{bmatrix}$ |                           |
| rotm   | $\\begin{bmatrix}x\_i \\ y\_i\\end{bmatrix}= H \\begin{bmatrix}x\_i \\ y\_i\\end{bmatrix}$                                  |                           |
| rotmg  | $\\begin{bmatrix}x1 \\ 0\\end{bmatrix}= H \\begin{bmatrix}x1\\sqrt{d1} \\ y1\\sqrt{d2}\\end{bmatrix}$                       |                           |
| scal   | $x \\leftarrow alpha\*x$                                                                                                    | 标量向量乘                     |
| swap   | $\\begin{bmatrix} y\\x \\end{bmatrix} \\leftarrow \\begin{bmatrix} x\\y \\end{bmatrix}$                                     | 交换向量                      |
| iamax  | $Max(x\_i)$                                                                                                                 | Vector x 中绝对值最大的值         |
| iamin  | $Min(x\_i)$                                                                                                                 | Vector x 中绝对值最小的值         |

## UseCase

    icc  app.obj -L${MKLROOT}/lib/intel64 -lmkl_intel_lp64-lmkl_intel_thread -lmkl_core

## Reference

  - [Spec](https://spec.oneapi.io/versions/latest/index.html)
  - [oneMKL Interfaces](https://oneapi-src.github.io/oneMKL/index.html)
  - [Intel® oneAPI Math Kernel Library Link Line
    Advisor](https://www.intel.com/content/www/us/en/developer/tools/oneapi/onemkl-link-line-advisor.html)
  - [Get Started
    Guide](https://www.intel.com/content/www/us/en/develop/documentation/get-started-with-mkl-for-dpcpp/top.html)
  - [Developer Reference C
    API](https://www.intel.com/content/www/us/en/develop/documentation/onemkl-developer-reference-c/top.html)
