---
title: "第2章 Python基础知识"
chapter: 2
tags: [Python, NumPy, ndarray, 数据表示]
depends_on: [[01-学习前的准备]]
next: [[03-数据可视化]]
---

# 第2章 Python基础知识

## 本章真正要学的不是语法

本章的核心是：**把现实中的样本可靠地表示成程序能计算的数组**。四则运算、变量、`if`、`for`、函数只是表面；真正会在机器学习中反复出现的是类型、形状、索引、广播、拷贝与数据保存。

## 2.1 四则运算

### 2.1.1 四则运算的用法

```python
a, b = 7, 2
print(a + b, a - b, a * b, a / b)
print(a // b, a % b)  # 整除与余数
```

除法 `/` 通常得到浮点数，`//` 是向下取整。数据预处理里，整数除法可能无意中丢掉小数；先确认 `dtype`，再选择运算。

### 2.1.2 幂运算

```python
print(2 ** 3)
print(2 ** 0.5)
```

幂运算会出现在平方误差、范数、标准差、指数函数和多项式基函数中。`x ** 2` 是逐元素平方；对数组不要把它和矩阵平方混为一谈。

## 2.2 变量

### 2.2.1 利用变量进行计算

变量名把一个值绑定到可读的概念上：

```python
temperature = 23.5
humidity = 0.62
comfort = temperature - 4 * humidity
```

机器学习里，`X`、`y`、`w`、`b` 通常分别表示输入、目标、权重、偏置；命名虽短，但必须在笔记中说明含义、单位和形状。

### 2.2.2 变量的命名

变量名应表达语义，使用小写加下划线；常量可用大写。避免用 `l`、`O`、`I` 这类容易混淆的名字，也不要覆盖 `sum`、`list`、`type` 等内置名称。

```python
n_samples = 100
n_features = 3
learning_rate = 0.01
```

### 2.2.3 一行定义多个变量（扩展）

```python
n, d = 120, 4
width, height = 28, 28
```

同时赋值本质是解包。它适合表达成组的形状参数，但如果一行太长或每个变量语义不同，应拆开写。

## 2.3 类型

### 2.3.1 类型的种类

常见类型有 `int`、`float`、`bool`、`str`、`list`、`tuple` 和 NumPy 的 `ndarray`。类型影响能做的运算、内存大小和模型输入是否有效。

```python
label = 1                 # int
probability = 0.85        # float
is_valid = True           # bool
name = "mnist"           # str
features = [1.0, 2.0]     # list
```

### 2.3.2 检查类型

```python
print(type(probability))
import numpy as np
x = np.array([1, 2, 3], dtype=np.float32)
print(type(x), x.dtype, x.shape)
```

`type()` 告诉你对象类别，`dtype` 告诉你数组元素类型，`shape` 告诉你结构。三者都要检查。

### 2.3.3 字符串

字符串常用于列名、类别名、文件路径和日志，不应直接拿来做数值计算。类别字符串进入模型前要编码，并固定编码字典，避免训练和线上类别编号错位。

```python
class_names = ["cat", "dog"]
print(f"class={class_names[0]}")
```

## 2.4 `print` 语句

### 2.4.1 `print` 语句的用法

```python
print("shape:", x.shape)
print("first three:", x[:3])
```

打印是最小的可观测性工具。训练循环中至少打印步数、损失、学习率和验证指标。

### 2.4.2 同时显示数值和字符串的方法1

```python
epoch = 3
loss = 0.127
print("epoch", epoch, "loss", loss)
```

这种写法简单，适合快速检查。

### 2.4.3 同时显示数值和字符串的方法2

```python
print(f"epoch={epoch:02d}, loss={loss:.4f}")
```

格式化字符串适合日志和结果对比。固定小数位可以避免只因显示精度不同而误判模型变化。

## 2.5 `list`（数组变量）

### 2.5.1 `list` 的用法

```python
items = [10, 20, 30]
items.append(40)
print(items[0], items[-1], len(items))
```

列表可以修改，适合收集对象或保存不同长度的记录；数值批量运算应尽早转换为 `ndarray`。

### 2.5.2 二维数组

```python
table = [[1, 2, 3], [4, 5, 6]]
X = np.array(table, dtype=float)
print(X[1][2])  # Python 嵌套索引
print(X[1, 2])  # NumPy 多轴索引
```

二维数组可以看成表格，但不要忘记语义：行通常是样本，列通常是特征。

### 2.5.3 创建连续的整数数组

```python
np.arange(5)          # [0 1 2 3 4]
np.arange(2, 10, 2)   # [2 4 6 8]
np.linspace(0, 1, 5)  # 等间距浮点数
```

`arange` 按步长生成，`linspace` 按元素个数生成。绘图网格常用 `linspace`，索引和批次常用 `arange`。

## 2.6 `tuple`（数组）

### 2.6.1 `tuple` 的用法

```python
shape = (28, 28)
```

元组不可变，适合表达固定结构，例如数组的 `shape`、坐标或函数返回的一组值。

### 2.6.2 读取元素

```python
shape = (60000, 28, 28)
samples, height, width = shape
print(samples, height, width)
```

解包时变量数量必须匹配。这个约束反而能及早暴露数据格式变化。

### 2.6.3 长度为1的 `tuple`

```python
one = (28,)  # 逗号不可省略
not_tuple = (28)
print(type(one), type(not_tuple))
```

在 NumPy 中，`(n,)` 表示一维向量形状；`(n, 1)` 表示二维列向量，二者在广播和矩阵乘法中不同。

## 2.7 `if` 语句

### 2.7.1 `if` 语句的用法

```python
score = 0.73
if score >= 0.8:
    decision = "accept"
else:
    decision = "review"
```

分类模型输出概率后，业务规则通常就在这里发生。模型概率不是最终决策；阈值和人工复核策略应单独记录。

### 2.7.2 比较运算符

`<`、`<=`、`==`、`!=`、`>=`、`>` 会产生布尔值。浮点数不要直接用 `==` 判断计算结果相等，使用容差：

```python
np.isclose(0.1 + 0.2, 0.3)
```

## 2.8 `for` 语句

### 2.8.1 `for` 语句的用法

```python
losses = [0.9, 0.6, 0.4]
for loss in losses:
    print(loss)
```

循环适合表达训练轮次、文件处理和控制流。对大规模数组，优先考虑 NumPy 向量化，以减少 Python 层循环开销。

### 2.8.2 `enumerate` 的用法

```python
for epoch, loss in enumerate(losses, start=1):
    print(epoch, loss)
```

`enumerate` 同时给出索引和元素，比手写计数器更不容易错。

## 2.9 向量

### 2.9.1 NumPy 的用法

```python
import numpy as np
x = np.array([1.0, 2.0, 3.0])
print(x + 1, x * 2, np.exp(x))
```

NumPy 的 `ndarray` 是同质的多维数组；很多数学函数以逐元素方式作用于它。它是后续把公式翻译为代码的主要载体。

### 2.9.2 定义向量

```python
x = np.array([1, 2, 3], dtype=float)
```

一维数组的 `shape` 是 `(3,)`，不是 `(1, 3)`。形状差异会影响转置和矩阵乘法。

### 2.9.3 读取元素

```python
print(x[0])
print(x[-1])
```

Python 从 0 开始索引；负索引从末尾读取。索引错误经常导致特征错位，尤其在手写特征工程时。

### 2.9.4 替换元素

```python
x[1] = 99
```

数组的元素可以原地替换，但要注意是否有其他变量共享这块内存。

### 2.9.5 创建连续整数的向量

```python
indices = np.arange(10)
```

它常用于批次索引、绘图横轴和构造实验数据。

### 2.9.6 `ndarray` 的注意事项

最重要的三件事：

1. `*` 是逐元素乘法，`@` 才是矩阵乘法；
2. 切片可能返回 view，必要时使用 `.copy()`；
3. 广播会自动扩展形状，结果正确与否取决于轴的语义。

```python
X = np.arange(6).reshape(2, 3)
w = np.array([1., 10., 100.])
print(X * w)  # 每一列分别缩放
```

## 2.10 矩阵

### 2.10.1 定义矩阵

```python
A = np.array([[1., 2.], [3., 4.]])
```

二维 `ndarray` 是矩阵的程序表示，但后续更高维数据通常称为张量。

### 2.10.2 矩阵的大小

```python
print(A.shape)  # (2, 2)
print(A.size)   # 4 个元素
```

`shape` 是每个轴的长度，`size` 是元素总数。模型出错时先看 shape，不要只看 `size`。

### 2.10.3 读取元素

```python
print(A[0, 1])
print(A[:, 0])
```

逗号分隔的索引对应不同轴；`:` 表示该轴全部取出。

### 2.10.4 替换元素

```python
A[0, 1] = 20
A[:, 0] = 0
```

切片赋值可以一次修改一行或一列，但需要确认左侧形状和右侧值能正确广播。

### 2.10.5 生成元素为0和1的 `ndarray`

```python
zeros = np.zeros((2, 3))
ones = np.ones((2, 3))
identity = np.eye(3)
```

这些数组是初始化参数、构造单位矩阵和设计矩阵的基础。

### 2.10.6 生成元素随机的矩阵

```python
rng = np.random.default_rng(0)
W = rng.normal(0, 0.1, size=(3, 2))
```

固定随机种子有利于调试，但不要把一次随机结果当成模型稳定性的证明；应在多个种子上重复实验。

### 2.10.7 改变矩阵的大小

```python
v = np.arange(6)
M = v.reshape(2, 3)
flat = M.reshape(-1)
```

`reshape` 只能改变视图解释，元素数量必须相同。图像从 `(28, 28)` 展平为 `(784,)` 时，必须确认像素顺序与后续模型一致。

## 2.11 矩阵的四则运算

### 2.11.1 矩阵的四则运算

```python
A = np.array([[1., 2.], [3., 4.]])
B = np.array([[10., 20.], [30., 40.]])
print(A + B)
print(A - B)
print(A * B)  # 逐元素
print(A / B)
```

逐元素运算要求形状相同或可广播，不等同于线性代数中的矩阵运算。

### 2.11.2 标量 × 矩阵

```python
print(0.1 * A)
```

标量乘矩阵会缩放每个元素。学习率、正则化系数和单位换算经常体现为这种缩放。

### 2.11.3 算术函数

```python
print(np.sqrt(A))
print(np.log(A))
print(np.sum(A), np.mean(A), np.max(A))
```

NumPy 的通用函数通常逐元素运行；聚合函数要明确 `axis`。例如 `X.mean(axis=0)` 是按样本聚合每个特征。

### 2.11.4 计算矩阵乘积

```python
X = np.array([[1., 2., 3.], [4., 5., 6.]])
w = np.array([0.1, 0.2, 0.3])
score = X @ w
print(score.shape)  # (2,)
```

这就是批量线性模型的核心：`(n_samples, n_features) @ (n_features,) -> (n_samples,)`。

## 2.12 切片

```python
X = np.arange(20).reshape(4, 5)
print(X[:2, :])     # 前两行
print(X[:, 1:4])    # 第2到第4列
print(X[::2, ::-1]) # 每隔一行，列反向
```

切片是取训练/测试子集、选择特征和图像裁剪的基础。`X[:n]` 通常是 view，若要隔离内存使用 `X[:n].copy()`。

## 2.13 替换满足条件的数据

```python
temperature = np.array([-2., 4., 40., np.nan])
temperature[temperature < 0] = 0
temperature = np.nan_to_num(temperature, nan=20.)
```

布尔索引把“规则”变成数组操作。工业中不要只写替换代码，还要记录：为什么这样处理、阈值来自哪里、是否会改变标签分布。

## 2.14 `help`

```python
help(np.reshape)
print(np.reshape.__doc__[:200])
```

查官方文档时重点看参数顺序、返回形状、是否原地修改、版本变更和异常条件。面对旧书代码，先查当前 API，而不是盲目改到“能运行”。

## 2.15 函数

### 2.15.1 函数的用法

```python
def square(x):
    return x * x

print(square(np.array([1, 2, 3])))
```

函数是可复用、可测试的假设。把数据清洗、模型前向计算和指标计算拆成函数，才能定位是数据、模型还是评估出了问题。

### 2.15.2 参数与返回值

```python
def standardize(X, mean=None, std=None):
    """返回标准化数据以及使用的统计量。"""
    if mean is None:
        mean = X.mean(axis=0)
    if std is None:
        std = X.std(axis=0) + 1e-8
    return (X - mean) / std, mean, std
```

预处理函数要区分训练阶段和推理阶段：均值/标准差只能从训练集估计，然后原样复用；不能用测试集重新计算，否则会把未来信息泄漏进模型。

## 2.16 保存文件

### 2.16.1 保存一个 `ndarray` 类型变量

```python
np.save("X.npy", X)
X_loaded = np.load("X.npy")
```

`.npy` 能保留数组形状和数据类型，适合单个数组。保存时最好在旁边写一份数据说明。

### 2.16.2 保存多个 `ndarray` 类型变量

```python
np.savez("dataset.npz", X=X, y=np.array([0, 1, 1]))
data = np.load("dataset.npz")
X2, y2 = data["X"], data["y"]
```

真实项目还要记录数据版本、单位、列顺序、缺失值规则和生成时间。一个没有元数据的二进制文件，未来很难被安全复用。

## 工业视角：形状是数据契约

很多“模型问题”其实是表示问题：时间戳时区不一致、缺失值被当成 0、类别编码顺序改变、图像通道顺序错了、训练和线上使用了不同标准化参数。模型只会忠实地放大这些错误。

```python
def inspect(name, a):
    a = np.asarray(a)
    print(name, "shape=", a.shape, "dtype=", a.dtype,
          "min=", np.nanmin(a), "max=", np.nanmax(a))

assert X.ndim == 2
assert len(X) == len(y)
assert np.isfinite(X).all()
```

## 本章练习

1. 生成形状为 `(100, 3)` 的随机数据，分别按列和按行标准化，比较结果。
2. 用一个小例子演示 `view` 与 `copy` 的差异。
3. 手写 `X @ w + b`，并打印每一步的形状。
4. 把一个 `list[list]` 数据保存成 `.npz`，重新加载后验证 `dtype`、`shape` 和数值一致。

## 关键连接

- [[03-数据可视化]]：数组不只是被计算，也要被画出来检查。
- [[04-机器学习中的数学]]：`@`、转置和 `shape` 会直接对应向量/矩阵公式。
- [[05-有监督学习-回归]]：`X @ w + b` 就是线性回归的前向计算。

## 英文补充

- [Python Tutorial](https://docs.python.org/3/tutorial/)
- [NumPy Quickstart](https://numpy.org/doc/stable/user/quickstart.html)
- [NumPy absolute basics](https://numpy.org/doc/stable/user/absolute_beginners.html)
- [NumPy ndarray reference](https://numpy.org/doc/stable/reference/arrays.ndarray.html)

