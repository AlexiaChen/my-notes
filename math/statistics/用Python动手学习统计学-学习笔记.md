
## 需要的python工具

- numpy
- pandas
- scipy

- matplotlib
- seaborn

## 统计学基础

### 统计学

- 统计学的目标 1. 描述现有数据（描述统计）  2. 估计未知数据（使用现有数据推断未知数据）（统计推断）
- 样本，现有数据
- 总体，包含现有数据也包含未知数据的全部数据集合

### 获取样本的过程

- 随机变量，根据随机法则变化的量叫做随机变量
- 样本值，由随机变量得到的具体数值叫样本值
- 抽样，从总体中获取样本叫做抽样
- 随机抽样/简单随机抽样。随机选择总体中各个元素的方法叫作随机抽样
- 样本容量，样本的大小或现有数据的个数叫作样本容量
- 普查，调查完整的总体叫做普查
- 抽样调查，只调查总体的一部分

### 抽样过程的抽样描述

- 概率，一般用P表示，获得某个数据的概率为P(某个数据)， 比如 P(1 < 体长 < 2) = xxx
- 概率分布，表示随机变量与概率之间的关系，有时直接称为分布。
- 服从某个概率分布，当某个数据和某种概率分布相符时，我们说xx服从xx概率分布
- 总体分布，总体服从的概率分布

> 通过简单随机抽样从总体中获取1个样本的过程，可以看作获取1个服从总体分布的随机变量的过程。

- 无限总体，掷骰子这样的实验的结果范围无限大，就是无限总体。

### 描述统计基础

- 定量变量/数值变量，数值之间的差距所代表的意义是等价的。比如，鱼的体长，条数等数据都属于定量变量
- 离散变量，属于定量变量的一种，只取整数的数据。比如鱼的条数
- 连续变量，属于定量变量的一种，有小数点的数据。比如鱼的体长。体长这个数值，是不可能用精确地整数描述的，总会有后面小数位数的差距，都是四舍五入，模糊计算得到的。
- 分类变量，不能定量的数据。类似数据的种类，比如，鱼的种类，有鲤鱼，罗非鱼等等。

- 组，在定量变量中，把数值分为几个范围，这些范围就叫做组。
- 组中值，组中最大值和最小值之间的中间数值，比如 \[1.5, 2.5\) 这个组中值就是2

- 频数，某个数据出现的次数
- 频数分布，每个组中数据的频数排列
- 频率，频数占总数的比例

- 累计频数，把组从小到大排列，将频数相加，得到的和就是累计频数。比如求组3的；累计频数就是把组1和组2，组3的频数相加所得
- 累计频率，同上，只是累加的是频率
- 直方图，表示频数的分布

- 统计量，用于统计数据的值叫作统计量。抽样的目的是推断总体，所以需要先分析样本的特征，分析样本的特征一个是图表，一个是计算数据的统计量。
- 均值，就常用的统计量。一般被用来代表已知数据（样本）的值，也叫代表值
- 期望值，跟均值等价，实质是一样的，也是统计量。只是统计学中，更喜欢用期望值替代均值。符号记作$\mu$, 你可以把期望值理解为能够用于未知数据的均值
- 方差，用来表示数据与均值(期望值)之间相差多少。当期望值和数据之间相差较大时，它们就不能很好地代表数据了。方差小，数据正好处在均值附近。方差大，数据远离均值。符号记作 $\sigma^2$

> 只通过均值和方差未必能够准确把握数据形状，这时候还是需要借助画图，直方图这样的手段

### 总体分布的推断

在总体很小的时候，直接数数。但是在总体很大的时候，就不行了。

- 做假设，先假设总体分布呈现某一种分布。比如正态分布。这个比较看经验，选择的分布要与实际的数据不太差太多。比如你从你的样本中的数据，也发现了它服从某种分布，你才可以假设总体服从某种分布。

### 概率质量函数与概率密度函数

- 概率质量函数，在将数据作为参数时，所得的函数值(返回值)是概率的函数。比较复杂，实践中很少使用
- 概率密度，因为离散变量可以直接计算概率，而连续变量不能直接计算概率，所以用概率密度代替概率，可以把它看作用于连续变量的类似于概率的概念。

> 取概率密度在4到5的区间上的积分，基本等同于求4到5的区间上无数个概率密度的总和。毕竟是用于连续变量的工具。

- 概率密度函数。将数据作为参数时，所得函数值(返回值)是概率密度的函数。概率密度函数不要随意选择，不然无法符合总体分布。

- 正态分布。定义域是整个实数轴上的实数。均值附近的概率密度较大。数据距离均值越远，概率密度越小。概率密度的大小相对于均值左右对称。
- 参数（概率分布的参数），用于定义概率分布的值，概率分布的形状会随着参数的变化而变化。若将随机变量x，则正态分布的概率密度函数记为$N(x)$ 或 $N(x | \mu, \sigma^2)$，因为正态分布有两个参数，就是均值和方差

> 根据中心极限定理证明，很多样本呈现正态分布，正态分布确实方便计算，但是也不能死板套用，有时候也会使用二项分布或者泊松分布。二项分布适用于硬币正反面这种结果只有两种形式的数据。泊松分布多用于个数，次数等只取大于0的整数的数据。
> 
> 很多统计学的书籍会花费大量时间来讲解概率分布的种类及性质，因为概率分布的种类可以直接代表总体分布的类型。在统计学分析中，人们会将常用的分布用于总体分布，但是如果不在一定程度上假设概率分布的种类，分析就很难继续进行下去，用错了分布，数据分析就会偏离实际。

如何推断总体分布呢？两步，第一步确定分布的种类，第二步是估计分布的参数。估计分布参数可以用python来计算。

如何估计分布的参数呢？ 估计参数最直接的思路就是把样本的统计量看作是总体分布的参数。但是这个估计出来的可能也会有误差，所以误差也要考虑进行。

### 统计量的计算

- 均值，一般是算术平均值

$$
\mu = \frac{1}{N} 
\sum_{i=1}^{N}x_i
$$

- 期望值，本质上就是平均。$x_i$ 表示样本中的具体数值， $p_i$是取值的概率。

$$
\mu = \sum_{i=1}^{N} P(x_i) \cdot x_i
$$

> 总体均值和样本均值不一样，总体均值是未知的，样本均值是已知的，但是总体均值与样本均值会有差距，但是不会偏离。也就是说，两者之间出现正或负的微小差距的情况基本各占一半，当扩大样本容量，样本均值会逐渐接近总体均值。

- 样本方差

$$
\sigma^{2} = \frac{1}{N} 
\sum_{i=1}^{N} (x_i - \mu)^{2}
$$

$$
\sigma^{2} = \sum_{i=1}^{N} P(x_i) \cdot (x_i - \mu)^{2}
$$

其中$x_i - \mu$叫偏差，偏差的平方$(x_i - \mu)^{2}$ 一般用来表示数据与期望值（均值）之间的距离。所以方差公式结合前面的均值/期望值的公式模板套用，本质上方差就是数据与期望值之间的距离的均值。

- 无偏方差，因为样本方差和总体方差之间有偏离，无偏方差就是为了修正这个偏离而出现的。无偏方差比样本方差更大

$$
\sigma^{2} = \frac{1}{N - 1} 
\sum_{i=1}^{N} (x_i - \mu)^{2}
$$

- 标准差，一般就是对方差取平方根得出，这里的方差是无偏方差。

$$
\sigma = \sqrt{\sigma^{2}} = \sqrt{\frac{1}{N - 1} 
\sum_{i=1}^{N} (x_i - \mu)^{2}}
$$

### 概率论基础

- 集合，这个从小学到大。不赘述。
- 元素，集合里面的元素。

- 样本点，可能发生的实验结果。符号记作 $\omega$ 例子：掷色子的样本点为1，2，3，4，5，6
- 样本空间，样本点的总体集合。符号记作 $\Omega$， 例子，掷色子的样本空间是{1,2,3,4,5,6}
- 事件，样本空间的子集，事件有交事件也有并事件。
- 基本事件，只有一个样本点组成且不可再分解的事件。例子，掷色子的基本事件，{1},{2},(3),{4},{5},{6}
- 复合事件，含有多个样本点且可以再分解为多个基本事件的事件。例子，掷色子得偶数{2,4,6}为一个复合事件，同理，奇数也一样
- 空事件，不含任何样本点的事件

- 概率的公理化定义,只要满足以下三个条件，就可以称为概率
    - 对于所有事件A， $0 <= P(A) < 1$
    - $P(\Omega) = 1$
    - 对于互斥事件 $A_1, A_2,..., P(A_1 \cup A_2 \cup ...) = P(A_1) + P(A_2) + ...$

- 概率是什么？可以简单认为就是频率的极限，因为只有无穷无尽地实验下去，才可以知道精确地概率。

> 比如掷色子，1/6的概率表示，扔6次，大概就会有一次出现你期望的结果。但是也有可能扔6次，看不到你期望的结果，只有不停地扔下去，你的实验结果才可以逼近1/6。

> 注意，贝叶斯学派对于概率的解释不是这样的，他们任务概率是主观，个人主观决定的，所以有一种统计学叫贝叶斯统计学，依赖贝叶斯定理。

- 条件概率，以事件B为前提条件，事件A发生的概率叫做条件概率，记作:

$$
P(A | B) = \frac{P(A \cap B)}{P(B)}
$$

- 独立事件，当$P(A \cap B) = P(A) \cdot P(B)$成立时

### 随机变量与概率分布

- 随机变量与样本值。 $P(X = x)$， 其中X就是随机变量，x是随机变量的具体值，也就是样本值。
- 离散型概率分布与概率质量函数
$$
P(X = x_i) = f(x_i), i = 1,2,3,...
$$

此时，称X服从离散型概率分布，称函数$f(x_i)$为概率质量函数。

$$
\sum_{i = 1}^{\infty} f(x_i) = 1, and f(x_1) >= 0
$$

- 概率密度

$$
P(x <= X <= x + \Delta x)
$$

其中，当$\quad \Delta x \to 0$, 可由$P(x) \cdot \Delta x$计算出概率，P(x)就是x的概率密度

- 连续型概率分布与概率密度函数

$$
P(a <= X <= b) = \int_{a}^{b} f(x) dx
$$

此时，称X服从连续型概率分布，函数$f(x)$为概率密度函数。 连续型随机变量X，取得特定值的概率永远为0，所以都是用积分求落在某范围内的概率。

$$
\int_{-\infty}^{+\infty} f(x)dx = 1 \quad and \quad f(x) >= 0
$$

> 离散型和连续型其实无非就是加法和积分的区别，二者其实都是在做同一件事情

- 正态分布的概率密度函数

$$
N(x | \mu, \sigma^2) = \frac{1}{\sqrt{2 \pi \sigma^2}}e^{-\frac{(x - \mu)^2}{2 \sigma^2}}
$$

可以用随机变量X，表明随机变量X服从XX分布，比如 $X \sim N(x| \mu, \sigma^2)$

- 使用概率密度计算期望值的方法。

> 前面的期望值都是针对离散的数据进行计算的，所以现在学习连续的数据，怎样算期望/均值

$$
\mu = \int_{-\infty}^{+\infty} f(x) \cdot xdx = \int_{-\infty}^{+\infty} N(x| \mu,\sigma^2) \cdot xdx
$$

## 用Python进行数据分析

### 使用Python进行描述统计 - 单变量

```python
import numpy as np
import scipy as sp
# 精度设置
%precision 3

fish_data = np.array([2,3,1,6,...])

# 计算数据总和
sp.sum(fish_data)

# 计算样本容量
len(fish_data)

# 计算期望/均值
N = len(fish_data)
sum_value = sp.sum(fish_data)
mu = sum_value / N
# or
sp.mean(fish_data)

# 计算样本方差
sigma_2 = sp.sum((fish_data - mu) ** 2) / N
# or
sigma_2 = sp.var(fish_data, ddof = 0)

# 计算样本的无偏方差
sigma_2 = sp.sum((fish_data - mu) ** 2) / (N - 1)
# or
sigma_2 = sp.var(fish_data, ddof = 1)

# 计算标准差
sigma = sp.sqrt(sigma_2)
# or
sigma = sp.std(fish_data, ddof = 1)

# 标准化，把数据集合的均值转化为0，把数据集合的标准差(方差)转换为1
standard = (fish_data - mu) / sigma

# 最大值和最小值
sp.amax(fish_data)
sp.amin(fish_data)
# 中位数, 中位数对于极端值带来的影响更具有抵抗性
sp.median(fish_data)

# 四分位数，把数据按照升序排列，处在25%和75%位置的数
from scipy import stats
stats.scoreatpercentile(fish_data, 25)
stats.scoreatpercentile(fish_data, 75)
```

### 使用python进行描述统计 - 多变量

在进行多变量分析之前，需要了解整洁数据。

- 整洁数据，类似于数据库的二维表格，每个变量构成一列，每个单元格代表一个数值，每项观察构成一行，每种类型的观察分析构成一整张表格
- 杂乱数据，无法分析，整洁数据以外的都是杂乱数据，把杂乱数据变为整洁数据所花费的时间和精力超出很多人的想象

> 给别人分析之前，最好给整洁数据

```python
import pandas as pd
import scipy as sp

%precision 3

# pandas的数据帧就是一张二维表格，只是这张二维表格不一定是整洁数据，
multi_fish_data = pd.read_csv("path.csv")
print(multi_fish_data)
# 可以像类似数据库的groupby操作，假设表格中有一列叫种类 species
group = multi_fish_data.groupby("species")
print(group.mean())
print(group.std(ddof = 1))
# 得出最重要的几个统计量
group.describe()
```

- 协方差，研究两个连续变量之间的关系时，使用的统计量。
    - 协方差大于0，两个变量是正相关关系
    - 协方差小于0，两个变量是负相关关系
    - 协方差等于0，两个变量不相关

$$
Cov(x,y) = \frac{1}{N}\sum_{i = 1}^{N}(x_i - \mu_x)(y_i - \mu_y)
$$

$$
Cov(x,y) = \frac{1}{N - 1}\sum_{i = 1}^{N}(x_i - \mu_x)(y_i - \mu_y)
$$

其中$\mu_x, \mu_y$是变量x，y的均值，N是样本容量。

- 协方差矩阵，把多个变量的方差和协方差放在一起形成的矩阵

$$
CovMatrix(x,y) = \\
\begin{bmatrix} 
\sigma_{x}^2 & Cov(x,y)  \\
Cov(x,y) & \sigma_{y}^2 \\
\end{bmatrix} \\
$$

下面用python进行协方差计算

```python
import pandas as pd
import scipy as sp

%precision 3

cov_data = pd.read_csv("path.csv")
print(cov_data)
# 读取列x和y的值
x = cov_data["x"]
y = cov_data["y"]
# 样本容量
N = len(cov_data)
# 均值
mu_x = sp.mean(x)
mu_y = sp.mean(y)
# 协方差
cov = sum((x - mu_x) * (y - mu_y)) / N
# or
cov = sum((x - mu_x) * (y - mu_y)) / (N - 1)

# 协方差矩阵
sp.cov(x,y, ddof = 0)
# ot
sp.cov(x,y,ddof  = 1)
```

- 皮尔逊积矩相关系数(皮尔逊相关系数)， 对于两个变量x，y的相关系数，可以简单看成是协方差的标准化

$$
\rho_{xy} = \frac{Cov(x,y)}{\sqrt{\sigma_x^2 \sigma_y^2}} = \\
\frac{Cov(x,y)}{\sigma_x \sigma_y}
$$

> 皮尔逊相关系数，其实就是把协方差Normalize到最大值为1，最小值为-1之间，不然协方差会难以使用，皮尔逊相关系数是对协方差加以修正。其相关系数的绝对值越大，越说明相关性越高。越接近0，相关性越低

- 相关矩阵，把多个变量的相关系数放在一起得到的矩阵

$$
Cov(x, y) = \\
\begin{bmatrix} 
1 & \rho_{xy}  \\
\rho_{xy} & 1 \\
\end{bmatrix} \\
$$

下面用python来计算

```python
import pandas as pd
import scipy as sp

%precision 3

cov_data = pd.read_csv("path.csv")
print(cov_data)
# 读取列x和y的值
x = cov_data["x"]
y = cov_data["y"]
# 样本容量
N = len(cov_data)
# 均值
mu_x = sp.mean(x)
mu_y = sp.mean(y)
# x,y的方差
sigma_2_x = sp.var(x, ddof = 1)
sigma_2_y = sp.var(y, ddof = 1)
# 计算相关系数
cov = sum((x - mu_x) * (y - mu_y)) / (N - 1)
rho = cov / sp.sqrt(sigma_2_x * sigma_2_y)
# 计算相关矩阵
sp.corrcoef(x,y)
```

### 基于matplotlib和seaborn的数据可视化

只观察数据的统计量难以把握数据特征，因此，数据可视化是数据分析中必要的任务。

```python
import pandas as pd
import numpy as np

%precision 3

from matplotlib import pyplot as plt

x = np.array([0,1,2,3,4,5,6,7,8,9])
y = np.array([2,3,4,3,5,4,6,7,4,8])

# pyplot的折线图
plt.plot(x, y, color = 'black')
plt.title("lineplot")
plt.xlabel("x")
plt.ylabel("y")
# 保存图为文件
plt.savefig("path")

import seaborn as sns
sns.set()

# 用seaborn给折线图添加背景，让折线图更清晰
plt.plot(x, y, color = 'black')
plt.title("lineplot seaborn")
plt.xlabel("x")
plt.ylabel("y")

# 直接用seaborn画直方图
fish_data = np.array([2,3,3,4,4,4,4,5,5,6])
# 横轴分成5组，纵轴是横轴数据的频数，禁用kde核密度估计
sns.distplot(fish_data, bins = 5, color = 'black', kde = False)

# 核密度估计是为了解决直方图的一个缺点诞生的，这个缺点就是
# 直方图的形状会随着组的大小变化而剧烈变动，有时候完全无法体现数据特征
# 这时候就是一条平滑的曲线与直方图配合，纵轴也变了，直方图的面积相当于概率，总和是1
sns.distplot(fish_data,  color = 'black')

# csv中有两列，一列是鱼的种类，一列是鱼的体长，是整洁数据
fish_multi = pd.read_csv("path.csv")
print(fish_multi)
# 各类鱼的各种统计量
fish_multi.groupby("species").describe()
# 把各类鱼的体长数据分别存到新的变量中
length_a = fish_multi.query('species = "A"')["length"]
length_b = fish_multi.query('species = "B"')["length"]
# 分别画出每个体长变量的直方图，那么鱼A和鱼B的体长数据的直方图就在一起了
sns.distplot(length_a, bins = 5, color = 'black', kde = False)
sns.distplot(length_b, bins = 5, color = 'gray', kde = False)

# 直方图主要用于单变量的，多变量的图有其他更高效的图
# 如果读取的是整洁数据，列名应该与随机变量的名称一致

# 箱形图，箱子中心线段表示中位数，上下底边表示上四分位数和下四分位数，箱子
# 上下边缘表示数据范围
sns.boxplot(x = "species", y = "length", data = fish_multi, color = 'gray')

# 小提琴图，与箱型图类似，用核密度估计的结果替换了箱子
# 图中的平滑曲线就是核密度估计的结果，可以简单理解为小提琴图用横向放置的直方图代替了箱型图
# 中的箱子
sns.violinplot(x = "species", y = "length", data = fish_multi, color = 'gray')

# 条形图，各条高度表示均值，条中的黑线叫做误差线，代表均值的置信区间
sns.barplot(x = "species", y = "length", data = fish_multi, color = 'gray')

# 散点图，用来表示两种定量变量组合起来的数据
cov_data = pd.read_csv("path.csv")
# 有两列数据，分别为x和y
print(cov_data)
# 这里的散点图会顺便把皮尔逊相关系数给计算出来
sns.jointplot(x = "x", y = "y", data = cov_data, color = 'black')

# 散点图矩阵，表示两个以上的变量的数据可视化
# 使用seaborn内置的鸢尾花数据
iris = sns.load_dataset("iris")
iris.head(n = 3)
# 为了观察数据特征，求每种鸢尾花的各数据均值
iris.groupby("species").mean()
# 画散点图矩阵,不同种类用不同颜色，矩阵的主对角线是直方图
sns.pairplot(iris, hue="species", palette='gray')
```

### 用python模拟抽样

```python
import pandas as pd
import numpy as np
import scipy as sp
from scipy import stats
from matplotlib import pyplot as plt
import seaborn as sns
sns.set()

%precision 3
```

- 抽样过程，样本就是随机变量，它的取值会随机变化
- 简单的模拟抽样与大量数据的模拟抽样

```python
# 湖中鱼的体长数据，是全量的，假设湖中就5条鱼
fish_5 = np.array([2,3,4,5,6])
# 随机抽1条鱼
np.random.choice(fish_5, size = 1, replace = False)
# 随机抽3条鱼
np.random.choice(fish_5, size = 3, replace = False)

big_data = pd.read_csv("path.csv")["length"]
np.random.choice(big_data, size = 10, replace = False)
```

- 放回抽样与不放回抽样。把抽出的样本再放回总体再重新抽样叫放回抽样，反之就是不放回，上面的代码`replace = False`就是不放回
- 计算一组数据的概率密度，并且画出概率密度曲线

```python
x = [2,3,1,...]
plt.plot(x,
		stats.norm.pdf(x = x, loc = sp.mean(x), scale = sp.std(x, ddof = 1),
		color = 'black'))
```

- 抽样过程的抽象描述

如果总体分布为正态分布，那么我们可以认为我们的抽样也服从正态分布，可以用函数，生成一组，服从正态分布的随机数

```python
# 生成样本容量为10的，均值为4，标准差为0.8的，服从正态分布的一组随机数
sample_norm = stats.norm.rvs(loc = 4, scale = 0.8, size = 10)
```

- 直方图的无线细分，那么所表现的图形与正态分布的曲线基本等价。当总体容量远大于样本容量，无须进行校正。所以推荐用`norm.rvs`来模拟抽样
- 假设总体服从正态分布是否恰当

> 严格地说，总体分布不是正态分布，但是，实践上多会假设总体分布是正态分布。通常我们难以对总体进行普查，所以一般都会先绘制样本的直方图，再判断总体是否与所假设的分布差别过大

### 样本统计量的性质

借助程序在相同的条件下进行无数次抽样。

- 样本分布是样本所服从的概率分布

```python
import pandas as pd
import numpy as np
import scipy as sp
from scipy import stats
from matplotlib import pyplot as plt
import seaborn as sns
sns.set()

%precision 3

# 实例化一个正态分布
population = stats.norm(loc = 4, scale = 0.8)
# 做10000次抽样试验, 每次样本容量是10，计算每个样本均值
sample_mean_array = np.zeros(10000)
np.random.seed(1)
for i in range(0,10000):
	sample = population.rvs(size = 10)
	sample_mean_array[i] = sp.mean(sample)

# 样本均值的均值, 其值与总体均值相近
sp.mean(sample_mean_array)
# 样本均值的直方图
sns.distplot(sample_mean_array, color = 'black')
```

- 样本容量越大，样本均值越接近总体均值
- 样本容量越大，样本均值越集中在总体均值附近

```python
# size: 每次抽样的样本容量
# n_trail: 抽样的次数
# return: n次抽样的样本均值数组
def calc_sample_mean(size, n_trail):
	sample_mean_array = np.zeros(n_trail)
	np.random.seed(1)
	for i in range(0,n_trail):
		sample = population.rvs(size = size)
		sample_mean_array[i] = sp.mean(sample)
	return sample_mean_array
```

 - 样本容量越大，样本均值的标准差就越小，标准差越小就能得到更集中更可信的样本均值。
 - 标准误差，样本均值的标准差的理论值, 可以看到，样本容量越大，标注误差就越小。标准误差可以用来估计样本均值与总体均值的差异，标准误差越小，说明样本均值越接近总体均值。除以样本容量的根，是因为这样可以消除样本容量的影响，使得不同大小的样本可以进行比较。如果直接使用标准差，那么样本容量越大，样本均值越稳定，但是这并不反应样本均值与总体均值的差异。

$$
标准误差(Standard Error) = \\
\frac{\sigma}{\sqrt{N}}
$$

> 样本均值的标准差必然小于总体的标准差

- 样本方差的均值一般来说是偏离总体方差的，需要采用无偏方差修正，无偏方差的均值可以看作是总体方差
- 样本容量越大，其无偏方差就越接近总体方差
- 无偏性，估计量的期望值相当于真正的参数的特性
- 一致性，样本容量越大，估计量越接近真正的参数的特性
- 样本均值和样本的无偏方差具有适合作为参数估计量的性质，样本均值和无偏方差都具有无偏性和一致性。
- 大数定律，样本容量越大，样本均值越接近总体均值
- 中心极限定理，对于任意总体分布，样本容量越大，随机变量的和的分布越接近正态分布

```python
# 进行50000次抽样，每次样本容量为10000的，抛硬币实验
n_size = 10000
n_trail = 50000

coin = np.array([0,1])
count_coin = np.zeros(n_trail)
np.random.seed(1)
for i range(0, n_trail):
	count_coin[i] = sp.sum(
		np.random.choice(coin, size = n_size, replace = True)
	)

# 绘制直方图，该直方图接近正态分布
sns.distplot(count_coin, color = 'black')
```

因为样本均值，需要先求出样本数值的和，再除以样本容量。所以你可以认为样本均值的分布接近正态分布。应当明确的就是，这里的服从正态分布的只是样本值的和。比如，抛硬币，明显总体分布是一个二项分布，那么样本容量无穷大的样本依旧是二项分布。

中心极限定理一般只用于寻找均值等特定的值，当总体部分不服从正态分布时，应该使用广义线性模型。

### 正态分布及其应用

```python
import pandas as pd
import numpy as np
import scipy as sp
from scipy import stats
from matplotlib import pyplot as plt
import seaborn as sns
sns.set()

%precision 3
```

正态分布的概率密度可以回顾一下 

$$
N(x | \mu, \sigma^2) = \frac{1}{\sqrt{2 \pi \sigma^2}}e^{-\frac{(x - \mu)^2}{2 \sigma^2}}
$$

```python
N = 1 / (sp.sqrt(2 * sp.pi * sigma ** 2)) * sp.exp(- (x - mu) ** 2 / 2 * sigma ** 2)

# or
# 均值为4，标准差为0.8， x为3的位置时的正态分布的概率密度的值
stats.norm.pdf(lock\ = 4, scale = 0.8, x = 3)

# or
norm_dist = stats.norm.pdf(lock\ = 4, scale = 0.8)
norm_dist.pdf(x = 3)
```

- 样本小于等于某值的比例，就是求小于等于这个值的数据个数和样本容量的比值

```python
# 从N(x | 4, 0.8^2)的正态分布中抽样，样本容量为10万
np.random.seed(1)
sample_norm = stats.norm.rvs(loc = 4, scale = 0.8, size = 100000)
# 求小于等于3的比例
sp.sum(sample_norm <= 3) / len(sample_norm)
```

- 累积分布函数，也叫分布函数。对于随机变量X，当x为实数时，F(X)叫做累积分布函数。

$$
F(X) = P(X \leq x)
$$

简单来说，累积分布函数可以计算随机变量小于等于某个值的概率，用这个公式，就不需要像刚才pyhon中计算数据的个数了，比如

$$
P(x \leq 3) = \int_{- \infty}^3 \frac{1}{\sqrt{2 \pi \sigma^2}}e^{-\frac{(x - \mu)^2}{2 \sigma^2}}dx
$$

```python
# 计算小于等于3的累积分布
stats.norm.cdf(loc = 4, scale = 0.8, x = 3)
```

这样就可以无须计算数据的个数，只计算积分就可以得到概率，这也是假设总体服从正态分布的优点。

- 左侧概率与百分位数

数据小于等于某个值的概率叫做左侧概率。借助累积分布函数可以直接得到左侧概率。与之相反，可以得到某个概率的某个值，这个值就叫百分位数，也叫做左侧百分位数。

```python
# 计算左侧概率为2.5%的百分位数
stats.norm.ppf(loc = 4, scale = 0.8, q = 0.025)

# 比如，计算之前P( x <= 3)的百分位数,正反方向都做
left = stats.norm.cdf(loc = 4, scale = 0.8, x = 3)
r = stats.norm.ppf(loc = 4, scale = 0.8, q = left)
assert(r == left)
```

- 标准正态分布，均值为0，方差/标准差 为1的正态分布叫作标准正态分布，即 $N(x | 0, 1)$
- $t$值 ，是一个统计量。本质上就是样本均值减去总体均值，再除以标准误差

$$
t = \frac{\widehat{\mu} - \mu}{\widehat{\sigma} / \sqrt{N}}
$$
其中 $\widehat{\sigma}$ 为实际样本的无偏标准差

t值的公式与标准化公式类似，标准化就是把均值转化为0，把方差转化为1

- $t$值的样本分布，主要用于样本容量较小的地方。

```python
# 做1万次抽样，把一万次的样本算出来的t值放到一个t值数组里面，求t值的样本分布

np.random.seed(1)
t_value_array = np.zeros(10000)
norm_dist = stats.norm(loc = 4, scale = 0.8)
for i range(0, 10000):
	sample = norm_dist.rvs(size = 10)
	sample_mean = sp.mean(sample)
	sample_std = sp.std(sample, ddof = 1)
	sample_se = sample_std / sp.sqrt(len(sample))
	t_value_array[i] = (sample_mean - 4) / sample_se

# 画出t值的分布,一般来说，t值的边缘比较接近正态分布曲线
sns.distplot(t_value_array, color = 'black')
x = np.arrange(start = -8, stop = 8.1, step = 0.1)
plt.plot(x, stats.norm.pdf(x = x), color = 'black', linestyle = 'dotted')
```

- $t$分布，当总体服从正态分布时，$t$值的样本分布就是$t$分布。

假设样本容量为N， 那么自由度就是N - 1.  $t$分布的图形与自由度相关，如果自由度为n - 1，则t分布表示为$t(n - 1)$

$$
t(n - 1)的方差 = \frac{n - 1}{n - 3} 且  n > 3
$$

可以看到，自由度（样本容量）越大，方差越接近1，t分布就越接近标准正态分布。样本容量越小，t分布就远离标准正态分布

```python
# 标准正态分布
plt.plot(x, stats.norm.pdf(x = x), color = 'black', linestyle = 'dotted')
# t分布，df是自由度，之前的每次样本的容量为10，10 - 1 = 9
plt.plot(x, stats.t.pdf(x = x, df = 9), color = 'black')

sns.distplot(t_value_array, color = 'black'， norm_hist = True)
plt.plot(x, stats.t.pdf(x = x, df = 9), color = 'black', linestyle = 'dotted')
```

t分布的意义就是在总体方差未知的时候，也可以研究样本均值的分布。样本均值的分布就是之前python的10000次抽样的实验，每次抽样，算样本容量为10的均值的均值分布。

### 参数估计

> 所谓的参数就是指总体分布的参数，只要知道的参数，就能确定总体分布

```python
import pandas as pd
import numpy as np
import scipy as sp
from scipy import stats
from matplotlib import pyplot as plt
import seaborn as sns
sns.set()

%precision 3
```

- 点估计，直接指定总体分布的参数为某一值的估计方法。比如，用样本均值作为总体均值的估计量，因为样本均值具有无偏性和一致性，它才可以作为总体均值的估计量。

> 直接计算样本的均值作为总体均值的估量，直接计算样本的无偏方差作为总体方差的估计量

- 区间估计，估计值具有一定范围的估计方法。使用概率的方法计算这个范围。

因为估计值是一个范围，所以会引入估计误差。估计误差越小，区间估计的范围越小。样本容量越大，区间的范围越小。

- 置信水平与置信区间

置信水平是表示区间估计的区间可信度的概率，95% 99%都可以作为置信水平，满足某个置信水平的区间就是置信区间。对于同一组数据，置信水平越大，置信区间就越大。直观来说，要提高可信度，必然要扩大取值范围以保证安全。

- 置信界限，置信区间的上下界的值，上置信区间置，下置信区间值
- 置信区间的计算

$$
CI = \mu \pm z \cdot \frac{\sigma}{\sqrt{n}} = \mu \pm z \cdot Standard Error
$$

$$
Standard Error = \frac{\sigma}{\sqrt{n}}
$$

当然，可以用t分布来计算

```python
mu = sp.mean(fish)
df = len(fish) - 1
sigma = sp.std(fish, ddof = 1)
se = sigma / sp.sqrt(len(fish))
# 0.95是95%的置信水平，df是自由度，loc是样本均值， scale是标准误差
interval = stats.t.interval(alpha = 0.95, df = df, loc = mu, scale = se)
# 得出来的结果就是95%水平的置信区间
```

公式里面的z值都可以查表了，根据你选择的置信水平查就可以了，叫z-value table。就不需要计算繁杂的分位数这些量了。以为置信水平一般都会选择95%，不会乱选。

- 决定置信区间大小的因素

样本的方差越大，就表明数据更偏离均值，相应地均值就更不可信，从而导致置信区间更大，所以我们要尽可能让置信区间更小。置信区间越大，真正的总体均值的位置就更难确定，而样本容量越大，样本均值就越可信，进而置信区间就越小。

对于相同数据，置信水平越大，那么确保安全，置信区间也会越大

- 区间估计的解读

之前我们一直把置信水平95%，99%就叫置信水平。你可以理解为可信度，最终的置信区间的范围的含义其实要看样本均值的含义，看样本均值$\mu$本身是一个概率，还是某种身高体长的数值。比如，我有一个样本值为身高的样本数据，均值为5，那么假设它的95%水平的置信区间为 4.5 到 5.5,那么我们可以用人话来讲就是，这个置信区间的意思是，我们有95%概率的把握认为，这个身高范围会在4.5到5.5之间。 

### 假设检验



### 均值差的检验

### 列联表的检验

### 检验结果的解读

## 统计模型基础

## 正态线性模型

## 广义线性模型

## 统计学与机器学习
