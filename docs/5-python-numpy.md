# 第5章：科学计算 — NumPy 与向量化思维

## :thinking: AI 时代，你真正需要掌握什么？

问 ChatGPT「如何用 NumPy 做矩阵乘法」，它会立刻给你正确答案。问它「如何对一组图像做归一化」，它也能写出漂亮的代码。

所以，NumPy 的 API 你不需要死记硬背。

但是，当 AI 生成了一段科学计算代码，你能判断它是否**正确**、是否**高效**吗？

- 这段代码用了 Python 循环还是向量化操作？性能差 100 倍你能发现吗？
- Broadcasting 的结果是你想要的 shape 吗？
- 这里用 C order 还是 Fortran order 更快？
- `np.dot` 在这里是矩阵乘法还是点积？

**这才是你需要内化的东西**：向量化思维、广播规则、内存模型。这些是 AI 写不出来的判断力。

---

## :star: 为什么需要 NumPy？

### Python list 的局限

C++ 程序员都懂数组：连续内存、缓存友好、SIMD 加速。Python 的 `list` 不是这样的：

```python
# Python list 存的是"对象引用"，不是数据本身
nums = [1.0, 2.0, 3.0, 4.0]
# 每个元素是一个 Python float 对象，在堆上随机分布
# 遍历需要：解引用 → 检查类型 → 取值 → 下一步
```

```cpp
// C++ vector 存的是连续的数据
std::vector<double> nums = {1.0, 2.0, 3.0, 4.0};
// 内存：[1.0][2.0][3.0][4.0] — 连续，SIMD 友好
```

这个差异导致什么结果？

```python
import time

# 对 100 万个数求平方
n = 1_000_000
data = list(range(n))

# 方式 1：Python list comprehension
start = time.perf_counter()
result = [x * x for x in data]
t1 = time.perf_counter() - start

# 方式 2：NumPy
import numpy as np
arr = np.arange(n)

start = time.perf_counter()
result = arr * arr
t2 = time.perf_counter() - start

print(f"Python list: {t1*1000:.1f} ms")
print(f"NumPy:       {t2*1000:.1f} ms")
print(f"加速比:       {t1/t2:.0f}x")
# 典型输出：
# Python list: 85.3 ms
# NumPy:       1.2 ms
# 加速比:       71x
```

:bulb: 为什么 NumPy 这么快？三个原因：
1. **连续内存** — ndarray 就是 C 数组，缓存命中率高
2. **SIMD 指令** — 底层调用 AVX/SSE，一次处理 4-8 个 double
3. **跳过解释器** — 运算在 C 层完成，Python 只负责调度

### NumPy vs C++ 数组的关系

你可以把 `np.ndarray` 理解为 C++ 的 `std::vector<T>` 的 Python 包装，加上大量线性代数操作。

```cpp
// C++ 视角
double* data = new double[n];  // 原始数组
// NumPy ndarray ≈ 这个指针 + shape + stride + dtype 的元数据
```

---

## :star: NumPy 基础

### ndarray — 核心数据结构

```python
import numpy as np

# 从 list 创建
a = np.array([1, 2, 3, 4, 5])
print(a.dtype)   # int64
print(a.shape)   # (5,)
print(a.ndim)    # 1

# 二维数组（矩阵）
m = np.array([[1, 2, 3],
              [4, 5, 6]])
print(m.shape)   # (2, 3) — 2行3列
print(m.ndim)    # 2
print(m.size)    # 6 — 元素总数
```

### dtype — 类型是你的责任

C++ 中类型是编译期确定的。NumPy 中你需要主动管理：

```python
# 默认推断
a = np.array([1, 2, 3])          # int64
b = np.array([1.0, 2.0, 3.0])    # float64
c = np.array([1, 2, 3], dtype=np.float32)  # 显式指定

# 常用 dtype
np.float32   # 单精度浮点，等同 C++ float
np.float64   # 双精度浮点，等同 C++ double（默认）
np.int32     # 32位整数
np.int64     # 64位整数
np.complex128 # 复数

# 类型转换
arr = np.array([1, 2, 3], dtype=np.int32)
arr_f = arr.astype(np.float64)

# ⚠️ 陷阱：整数溢出不报错
small = np.array([127], dtype=np.int8)
print(small + 1)   # [128] → 溢出为 -128！
```

### 创建数组的常用方式

```python
# 等差数列（类比 C++ for 循环填充）
np.arange(10)           # [0, 1, 2, ..., 9]
np.arange(0, 1, 0.1)    # [0.0, 0.1, ..., 0.9]
np.linspace(0, 1, 11)   # [0.0, 0.1, ..., 1.0]，包含终点，共11个

# 特殊矩阵
np.zeros((3, 4))         # 3×4 全零矩阵
np.ones((2, 3))          # 2×3 全一矩阵
np.eye(4)                # 4×4 单位矩阵
np.full((3, 3), 7.0)     # 3×3 全为 7.0

# 随机数
np.random.rand(3, 4)     # [0, 1) 均匀分布
np.random.randn(100)     # 标准正态分布
np.random.randint(0, 10, size=(3, 3))  # 整数随机

# 从已有 shape 创建
a = np.array([[1, 2], [3, 4]])
np.zeros_like(a)         # 与 a 同 shape 的零数组
np.ones_like(a)
```

---

## :star: 向量化思维 — 本章核心

这是 C++ 程序员最需要转变的思维方式。

### 循环思维 vs 向量化思维

**问题**：对数组每个元素应用 `f(x) = 2x² + 3x - 1`

```cpp
// C++ 思维：一个个处理
std::vector<double> y(n);
for (int i = 0; i < n; i++) {
    y[i] = 2 * x[i]*x[i] + 3 * x[i] - 1;
}
```

```python
# ❌ 直接翻译 C++ —— 可运行，但慢
x = np.linspace(0, 10, 1_000_000)
y = np.zeros_like(x)
for i in range(len(x)):
    y[i] = 2 * x[i]**2 + 3 * x[i] - 1

# ✅ 向量化 —— 把操作作用于整个数组
y = 2 * x**2 + 3 * x - 1  # 就这一行！
```

向量化版本不只是更短，它快了 **50-100 倍**。

### 为什么向量化快？

```
Python 循环的每次迭代：
  Python 解释器 dispatch
  → 检查 x[i] 的类型
  → 调用 Python float 的 __mul__
  → 创建临时 Python 对象
  → 垃圾回收
  → 重复 100 万次

NumPy 向量化：
  Python 调度一次
  → C 层循环 + SIMD（AVX2 可以一次处理 4 个 double）
  → 结果直接写入预分配内存
```

### 向量化的本质：消灭 Python 层循环

```python
# 场景：找出数组中所有大于均值的元素的平方和

data = np.random.randn(1_000_000)

# ❌ Python 循环
result = 0
mean = sum(data) / len(data)
for x in data:
    if x > mean:
        result += x * x

# ✅ 向量化（三行搞定，快 100x）
mask = data > data.mean()    # 布尔数组
result = (data[mask] ** 2).sum()
```

### 通用函数 ufunc

NumPy 的数学函数都是向量化的（称为 ufunc）：

```python
x = np.linspace(0, 2 * np.pi, 1000)

# ✅ 向量化数学函数
y = np.sin(x)          # 不是 math.sin！
z = np.exp(-x) * np.cos(2 * x)
w = np.sqrt(x**2 + 1)

# ❌ 陷阱：用了 math 模块
import math
# math.sin(x)  # 这会报错，math 函数不接受数组
# 应该用 np.sin(x)
```

---

## :star: 索引和切片

### 基础索引

```python
a = np.array([10, 20, 30, 40, 50])

# 与 Python list 类似
print(a[0])     # 10
print(a[-1])    # 50
print(a[1:4])   # [20, 30, 40]
print(a[::2])   # [10, 30, 50]

# 二维数组
m = np.arange(12).reshape(3, 4)
# [[ 0  1  2  3]
#  [ 4  5  6  7]
#  [ 8  9 10 11]]

print(m[1, 2])     # 6 — 第1行第2列（0-indexed）
print(m[1])        # [4, 5, 6, 7] — 整行
print(m[:, 2])     # [2, 6, 10] — 整列
print(m[0:2, 1:3]) # 子矩阵
```

### :exclamation: 切片是视图，不是拷贝！

这是 NumPy 与 Python list 的关键区别：

```python
a = np.array([1, 2, 3, 4, 5])

# Python list：切片是拷贝
lst = [1, 2, 3, 4, 5]
sliced = lst[1:4]
sliced[0] = 99
print(lst)     # [1, 2, 3, 4, 5] — 原 list 不变

# NumPy：切片是视图（view）
arr = np.array([1, 2, 3, 4, 5])
view = arr[1:4]
view[0] = 99
print(arr)     # [ 1 99  3  4  5] — ⚠️ 原数组被修改了！

# 需要拷贝时，用 .copy()
copy = arr[1:4].copy()
copy[0] = 0
print(arr)     # 不变
```

:bulb: 这是 NumPy 的设计哲学：避免不必要的内存拷贝，让你显式控制。C++ 程序员应该很熟悉这个理念——类似引用 vs 值语义。

### 花式索引（Fancy Indexing）

```python
a = np.array([10, 20, 30, 40, 50])

# 用整数数组索引
idx = [0, 2, 4]
print(a[idx])   # [10, 30, 50]

# 二维索引
m = np.arange(12).reshape(3, 4)
rows = [0, 2]
cols = [1, 3]
print(m[rows, cols])   # [m[0,1], m[2,3]] = [1, 11]

# ⚠️ 花式索引返回拷贝，不是视图
fancy = a[[0, 2, 4]]
fancy[0] = 99
print(a)  # 不变！
```

### 布尔索引 — 最强大的索引方式

```python
data = np.array([3, -1, 4, -1, 5, -9, 2, 6])

# 生成布尔掩码
mask = data > 0
print(mask)   # [ True False  True False  True False  True  True]

# 用掩码过滤
print(data[mask])   # [3, 4, 5, 2, 6]

# 简写（常用）
print(data[data > 0])        # 正数
print(data[data % 2 == 0])   # 偶数

# 组合条件（用 & | ~，不能用 and or not！）
print(data[(data > 0) & (data < 5)])   # 0 到 5 之间

# 配合赋值：替换所有负数为 0
data[data < 0] = 0
print(data)   # [3, 0, 4, 0, 5, 0, 2, 6]
```

---

## :star: 广播机制（Broadcasting）

Broadcasting 是 NumPy 最强大也最容易出错的特性。

### 直觉理解

```python
a = np.array([1, 2, 3])
b = 10

# 标量与数组：b 被"广播"到与 a 相同 shape
print(a + b)   # [11, 12, 13]
# 等价于 a + np.array([10, 10, 10])
```

```python
# 二维 + 一维
m = np.array([[1, 2, 3],
              [4, 5, 6]])   # shape (2, 3)
v = np.array([10, 20, 30])  # shape (3,)

print(m + v)
# [[11, 22, 33],
#  [14, 25, 36]]
# v 被广播到每一行
```

### Broadcasting 规则（重要！）

两个数组做运算时，从**右边**对齐维度，逐维比较：
- 相同 → OK
- 其中一个为 1 → 那个维度被广播（拉伸）
- 其他 → 报错

```
shape (2, 3) + shape (3,)
       ↓ 右对齐
(2, 3) + (1, 3)  ← (3,) 被当作 (1, 3)
  ↓ 第0维：2 vs 1，1被广播为2
(2, 3) + (2, 3)  ← OK!
```

```python
# 更复杂的例子
a = np.ones((3, 1, 4))   # shape (3, 1, 4)
b = np.ones((5, 4))      # shape (5, 4)
# 右对齐：(3, 1, 4) + (1, 5, 4)
# 广播后：(3, 5, 4)
print((a + b).shape)   # (3, 5, 4)
```

### 实用场景：列归一化

```python
# 每列减去列均值（常见的数据预处理操作）
data = np.array([[1.0, 2.0, 3.0],
                 [4.0, 5.0, 6.0],
                 [7.0, 8.0, 9.0]])   # shape (3, 3)

col_mean = data.mean(axis=0)   # shape (3,) — 每列的均值
normalized = data - col_mean   # shape (3, 3) - shape (3,) → Broadcasting!

print(col_mean)      # [4. 5. 6.]
print(normalized)
# [[-3. -3. -3.]
#  [ 0.  0.  0.]
#  [ 3.  3.  3.]]
```

### :exclamation: 常见陷阱

```python
a = np.array([1, 2, 3])    # shape (3,)
b = np.array([[1], [2]])   # shape (2, 1)

# 这会成功，但结果是 (2, 3)，你想要的吗？
print((a + b).shape)   # (2, 3)
# [[2, 3, 4],
#  [3, 4, 5]]

# 如果你想要逐元素相加，shape 必须完全匹配或是 (1,) 或标量
```

---

## :star: 常用变形操作

### reshape — 改变形状不改变数据

```python
a = np.arange(12)   # [0, 1, 2, ..., 11]

# reshape 返回视图（尽可能）
m = a.reshape(3, 4)   # 3×4 矩阵
print(m)
# [[ 0  1  2  3]
#  [ 4  5  6  7]
#  [ 8  9 10 11]]

# -1 让 NumPy 自动计算
a.reshape(2, -1)    # (2, 6)，-1 自动推断为 6
a.reshape(-1)       # 展平为一维，等同 a.ravel()

# 增加维度（用于 Broadcasting）
v = np.array([1, 2, 3])        # shape (3,)
col = v.reshape(-1, 1)         # shape (3, 1) — 列向量
row = v.reshape(1, -1)         # shape (1, 3) — 行向量
# 等价简写：
col = v[:, np.newaxis]
row = v[np.newaxis, :]
```

### transpose — 转置

```python
m = np.arange(6).reshape(2, 3)
print(m.shape)      # (2, 3)
print(m.T.shape)    # (3, 2) — 转置

# 多维数组
t = np.ones((2, 3, 4))
print(t.transpose(2, 0, 1).shape)   # (4, 2, 3)
```

### concatenate / stack — 拼接

```python
a = np.array([[1, 2], [3, 4]])
b = np.array([[5, 6], [7, 8]])

# 按行拼（竖向）
print(np.concatenate([a, b], axis=0))
# [[1, 2], [3, 4], [5, 6], [7, 8]]

# 按列拼（横向）
print(np.concatenate([a, b], axis=1))
# [[1, 2, 5, 6], [3, 4, 7, 8]]

# vstack / hstack（常用简写）
np.vstack([a, b])   # 等同 axis=0
np.hstack([a, b])   # 等同 axis=1

# stack：新增一个维度再拼
np.stack([a, b], axis=0).shape    # (2, 2, 2)
np.stack([a, b], axis=2).shape    # (2, 2, 2)
```

### split — 分割

```python
a = np.arange(12).reshape(3, 4)

# 按列分割成 2 份
left, right = np.hsplit(a, 2)   # 每份 (3, 2)

# 按行分割
top, bottom = np.vsplit(a, [1]) # 分割点在第1行
# top: shape (1, 4), bottom: shape (2, 4)
```

---

## :star: 线性代数

### 矩阵乘法

```cpp
// C++ with Eigen
MatrixXd C = A * B;   // 矩阵乘法
double d = v.dot(w);  // 向量点积
```

```python
A = np.random.randn(3, 4)
B = np.random.randn(4, 5)
v = np.array([1.0, 2.0, 3.0])
w = np.array([4.0, 5.0, 6.0])

# Python 3.5+ 有 @ 运算符
C = A @ B           # 矩阵乘法，shape (3, 5)
d = v @ w           # 向量点积 = 32.0

# 等价写法
C = np.dot(A, B)
d = np.dot(v, w)

# ⚠️ 重要区分：
# np.dot(A, B) — 矩阵乘法（当 A, B 是二维时）
# A * B        — 逐元素乘法（element-wise）！
print((A * np.ones_like(A)).shape)   # (3, 4)，不是矩阵乘法
```

### np.linalg — 线性代数工具箱

```python
from numpy import linalg

A = np.array([[2.0, 1.0],
              [1.0, 3.0]])

# 行列式
print(linalg.det(A))        # 5.0

# 逆矩阵
A_inv = linalg.inv(A)
print(A @ A_inv)            # 单位矩阵（近似）

# 特征值和特征向量
eigenvalues, eigenvectors = linalg.eig(A)
print(eigenvalues)          # [1.38, 3.61]（近似）

# 解线性方程组 Ax = b
b = np.array([3.0, 5.0])
x = linalg.solve(A, b)      # 比 inv(A) @ b 更数值稳定
print(A @ x)                # 验证：应等于 b

# SVD 分解
U, s, Vh = linalg.svd(A)    # A = U @ diag(s) @ Vh

# 矩阵范数
print(linalg.norm(A))           # Frobenius 范数
print(linalg.norm(A, ord=2))    # 谱范数
```

### 与 C++ Eigen/BLAS 的对比

| 操作 | C++ Eigen | NumPy |
|------|-----------|-------|
| 矩阵乘法 | `A * B` | `A @ B` |
| 逐元素乘 | `A.cwiseProduct(B)` | `A * B` |
| 转置 | `A.transpose()` | `A.T` |
| 求逆 | `A.inverse()` | `linalg.inv(A)` |
| 特征值 | `EigenSolver` | `linalg.eig(A)` |

:bulb: NumPy 底层调用 BLAS/LAPACK，大矩阵运算性能接近 C++ Eigen。

---

## :star: SciPy 简介

SciPy 是在 NumPy 基础上构建的科学计算库，相当于 C++ 里 Boost 之于标准库的关系。

```
NumPy  → 数组基础设施（就像 C++ 的 STL 容器）
SciPy  → 算法和工具（就像 Boost/GSL）
```

### 常用子模块

```python
# 优化（scipy.optimize）
from scipy import optimize

def f(x):
    return (x - 2)**2 + 1

result = optimize.minimize_scalar(f)
print(result.x)   # ≈ 2.0（最优解）

# 非线性方程求根
def equation(x):
    return x**3 - x - 2

root = optimize.brentq(equation, 1, 2)
print(root)   # ≈ 1.5214

# 信号处理（scipy.signal）
from scipy import signal

# 设计低通滤波器（类比 C++ 里用 DSP 库）
b, a = signal.butter(4, 0.1, btype='low')
filtered = signal.filtfilt(b, a, noisy_signal)

# 统计（scipy.stats）
from scipy import stats

data = np.random.randn(1000)
# t 检验
t_stat, p_value = stats.ttest_1samp(data, popmean=0)
print(f"p-value: {p_value:.4f}")

# 正态分布 PDF
x = np.linspace(-4, 4, 100)
pdf = stats.norm.pdf(x, loc=0, scale=1)
```

### 什么时候用 SciPy？

- **数值优化**：梯度下降、最小化、方程求根
- **信号处理**：滤波器设计、FFT、卷积
- **统计检验**：t 检验、卡方检验、相关性分析
- **插值**：`scipy.interpolate`
- **积分**：`scipy.integrate`（数值积分、ODE 求解器）
- **稀疏矩阵**：`scipy.sparse`

---

## :star: 性能提示

### NumPy 快的时候

```python
# ✅ 大数组的向量化运算
result = np.exp(large_array) * np.sin(another_array)

# ✅ 矩阵运算（调用 BLAS）
C = A @ B  # 对大矩阵，接近 Eigen 的速度

# ✅ 聚合操作
total = array.sum()
max_val = array.max(axis=0)
```

### NumPy 慢的时候

```python
# ❌ 小数组（Python 函数调用开销 > 计算本身）
small = np.array([1.0, 2.0, 3.0])
# 对 3 个元素用 NumPy 不比 Python 快

# ❌ 需要条件逻辑的复杂操作
for i in range(n):
    if condition[i]:
        result[i] = f(arr[i])
    else:
        result[i] = g(arr[i])
# 改写为：
result = np.where(condition, f(arr), g(arr))  # ✅

# ❌ 递推/依赖上一步的结果
# 如 Fibonacci、状态机 → 无法向量化，考虑 Numba/Cython
```

### 内存布局：C order vs Fortran order

```python
# C order（行优先，默认）：同行元素在内存中连续
# Fortran order（列优先）：同列元素在内存中连续

c_arr = np.array([[1, 2, 3], [4, 5, 6]], order='C')
f_arr = np.array([[1, 2, 3], [4, 5, 6]], order='F')

# 检查内存布局
print(c_arr.flags['C_CONTIGUOUS'])   # True
print(f_arr.flags['F_CONTIGUOUS'])   # True

# 按行遍历 C order 快（连续内存访问）
# 按列遍历 Fortran order 快

# 实践：如果你大量做 row-wise 操作，用 C order（默认即可）
# 如果大量做 column-wise 操作，考虑 Fortran order 或先 transpose
```

```cpp
// 类比 C++ 二维数组：
// row-major（C++默认）: arr[row][col] → arr[row * cols + col]
// column-major（Fortran）: arr[row][col] → arr[col * rows + row]
```

### 避免中间数组

```python
# ❌ 创建多个临时数组
a = np.ones(1_000_000)
b = np.ones(1_000_000)
c = np.ones(1_000_000)
result = a * b + c    # 先创建 a*b 的临时数组，再加 c

# ✅ 用 out 参数（高级用法）
tmp = np.empty_like(a)
np.multiply(a, b, out=tmp)
np.add(tmp, c, out=tmp)

# ✅ 对于复杂表达式，考虑 numexpr
# import numexpr as ne
# result = ne.evaluate('a * b + c')  # 单次遍历，无临时数组
```

### 快速诊断工具

```python
# 检查是否连续内存（影响性能）
arr = np.ones((100, 100))
print(arr.flags)

# 查看内存占用
print(arr.nbytes)   # 字节数

# 用 %timeit（Jupyter）或 timeit 模块
import timeit
t = timeit.timeit(lambda: arr.sum(axis=0), number=1000)
print(f"{t/1000*1000:.3f} ms per call")
```

---

## :link: 小结：C++ 程序员的 NumPy 备忘录

| C++ 概念 | NumPy 对应 |
|---------|-----------|
| `double arr[100]` | `np.zeros(100)` |
| `std::vector<double>` | `np.array([...])` |
| 指针运算/引用 | 切片（视图） |
| `memcpy` | `.copy()` |
| SIMD 循环 | 向量化运算 |
| Eigen/BLAS | `np.linalg` / `@` |
| 行优先内存 | C order（默认） |
| 列优先内存 | Fortran order |

**核心心法**：

1. :bulb: **消灭 Python 循环** — 凡是能用向量化替代的循环，都替代
2. :bulb: **切片是视图** — 需要独立副本时显式 `.copy()`
3. :bulb: **Broadcasting 从右对齐** — 脑子里想好 shape 再写代码
4. :bulb: **dtype 要匹配** — float32 vs float64 会影响精度和速度
5. :bulb: **AI 会写代码，但你要能判断它是否正确高效**

---

!!! tip "下一步"
    第6章将介绍 **Pandas** — 当你的数据不只是数字矩阵，而是带标签的表格时，NumPy 的下一层抽象。
