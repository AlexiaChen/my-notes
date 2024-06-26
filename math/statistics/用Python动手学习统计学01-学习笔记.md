
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

可以看到，自由度（样本容量）越大，方差越接近1，t分布就越接近标准正态分布。样本容量越小，t分布就远离标准正态分布 ^32b2d6

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

```python
be_included_array = np.zeros(20000, dtype = "bool")
# 执行2万次求95%置信区间操作
# 如果置信区间包含总体均值(4), 就为True
np.random.seed(1)
norm_dist = stats.norm(loc = 4, scale = 0.8)
for i range(0, 20000):
	sample = norm_dist.rvs(size = 10)
	df = len(sample) - 1
	mu = sp.mean(sample)
	std = sp.std(sample, ddof = 1)
	se = std / sp.sqrt(len(sample))
	interval = stats.t.interval(0.95, df, mu, se)
	if(interval[0] <= 4 amd interval[1] >= 4):
		be_included_array[i] = True

# 0.948, 最终，包含了总体均值(4)的置信区间的比例约为0.95
sum(be_included_array) / len(be_included_array)
```

### 假设检验

#### 假设检验

通过样本对总体进行统计学上的判断叫做假设检验（有时就叫检验），其特征是用概率论的语言来描述判断。

- 有很多种，不同的方法对应不同的研究对象
- 统计推断的实践中经常用到假设检验
- 即可用用于我们自己的分析任务，也可以用于解读他人的检验结果

**通过样本来给总体下判断原本就很困难**

#### 单样本$t$检验

- 研究对象： 均值
- 研究目标： 均值是否与某个值存在差异

例子： 有一种薯片包装，号称标称50g。然而，事实是不可能每包都是恰好50g，可能在50g上下浮动。每包都会有偏差，但是我们希望薯片包装的均值是50g，正如厂家号称的那样。就可以用这个检验方法来检查。

#### $t$检验的直观解释

假设检验中经常使用显著性差异这个词，就是字面意思。你做检验，要关注是否是有显著差异。

接下来直观解释下$t$检验中的显著性差异。还是以薯片举例子。

- 进行了大量调查： 薯片包装个数的样本容量大
- 使用精密的仪器测量： 数据偏差(方差)小
- 均重距离50g很远:  均值的差异大

在$t$检验中，如果满足上述3个条件，那么可以认为存在显著性差异。

#### 均值差异大不代表存在显著性差异

根据之前的$t$检验的判断逻辑，均值差异大，并不代表存在显著差异。还可能是样本容量小，还有使用的仪器不精密，或者是坏了。

#### $t$ 值

将之前提到的3个条件放在一起，并且3个条件都满足的情况下，就可以用下列公式:

$$
t值 = \frac{样本均值 - 对比值}{标准差 ÷ \sqrt{样本容量}} = \frac{样本均值 - 对比值}{标准误差}
$$

如果套用在薯片的例子，就是对比值为薯片厂商号称的50g。

如果t值大，就可以认为均重与50g存在显著性差异。当样本均值远小于对比值时，t值也很小，这个时候，应认为t值的意义在于它的绝对值。

#### 假设检验的结构：零假设与备择假设

假设检验的过程是先提出一个假设，再基于数据客观判断是否拒绝它。

一开始提出来并用于拒绝的对象叫零假设。
和零假设对立的假设叫做备择假设。

例子：

- 零假设：薯片的平均重量是50g
- 备择假设： 薯片的平均重量不是50g

#### $p$值

样本与零假设之间的矛盾的指标就是$p$ 值。p值越小，零假设和样本之间越矛盾。

p值的表示形式是概率。p值和置信区间类似，在完全相同的条件下反复地抽样并计算t值后才能显示出它的含义。

#### 显著性水平

当p值小于显著性水平时，拒绝零假设。拒绝零假设的标准叫作显著性水平，显著性水平多使用5%这个数。显著性水平也叫做危险率。

#### $t$ 检验与 $t$ 分布的关系

前面我们已经知道t值的公式了 [[#$t$ 值]] ， 以薯片举例子，假设总体服从均值为50的正态分布，则样本的t值服从t分布。[[#^32b2d6]]  

使用t分布的累积分布函数可以得到当总体均值为50时，t值小于 | t | 的概率，这个概率用$\alpha$ 表示。$1 - \alpha$ 就是当总体均值为50时t值大于 | t | 的概率。 $1 - \alpha$  越小，t值大于 | t |的概率就越小，更容易出现显著性差异。

#### 单侧检验与双侧检验

- 单侧检验，比如检验薯片均重是否小于50g叫做单侧，此时不考虑它是否大于50g。与之相对的另一侧大于50g也是单侧检验
- 双侧检验，检验薯片均重是否和50g存在差异，叫做双侧检验

单侧检验和双侧检验的p值不同，一般认为只进行单侧检验不太合理，所以一般使用双侧检验。

#### p值的计算

计算双侧检验的p值

设样本对应的t值为$t_\mu$。使用t分布的累积分布函数可以得到当总体均值为50时，t值小于$t_\mu$的概率，这个概率用$\alpha$ 表示。

$$
p = (1 - \alpha) * 2
$$

为什么要乘以2呢？是因为双侧经验，要计算薯片均重不是50g的概率，就要考虑大于和小于50g的两种情况。如果是单侧检验，那么就不用乘以2。p值就是 $1 - \alpha$ 。

#### t检验的实现-环境准备

```python
import pandas as pd
import numpy as np
import scipy as sp
from scipy import stats
from matplotlib import pyplot as plt
import seaborn as sns
sns.set()

%precision 3

# 在Jupyter notebook里面显示图形
%matplotlib inline
```

```python
# 假设有薯片样本文件
junk_food = pd.read_csv("file_path")["weight"]
junk_food.head()
```

进行单样本t检验

- 零假设: 薯片的均重是50g
- 备择假设：薯片的均重不是50g

设定显著性水平为5%，如果p值小于0.05，就拒绝零假设，并认为薯片均重50g存在显著差异

#### t检验的实现-计算t值

回顾t值计算公式， [[#$t$ 值]]

```python
# 计算样本均值
mu = sp.mean(junk_food)
# 自由度，一般是样本容量减一, 目前这个df没有用，但是计算p值会用到
df = len(junk_food) - 1
# 标准误差
sigma = sp.std(junk_food, ddof = 1)
se = sigma / sp.sqrt(len(junk_food))
# t值
t_value = (mu - 50) / se
```

#### t检验的实现-计算p值

假设总体分布服从正态分布，那么t值就服从t分布，所以可用使用t分布的累积分布函数。回顾下p值的计算方法 [[#p值的计算]]

```python
alpha = stats.t.cdf(t_value, df = df)
# 如果p值小于0.05.那么认为零假设存在显著差异
p = (1 - alpha) * 2
```

或者更方便地可以计算p值

```python
# 输出的statistic为t值，pvalue为p值
stats.ttest_lsamp(junk_food, 50)
```

#### 通过模拟实验计算p值

可以通过一些随机数生成样本，让样本服从正态分布。

```python
size = len(junk_food)
sigma = sp.std(junk_food, ddof = 1)
t_value_array = np.zeros(50000)
np.random.seed(1)
norm_dist = stats.norm(loc = 50, scale = sigma)
for i range(0, 50000):
	sample = norm_dist.rvs(size = size)
	sample_mean = sp.mean(sample)
	sample_std = sp.std(sample, ddof = 1)
	sample_se = sample_std / sp.sqrt(size)
	t_value_array[i] = (sample_mean - 50) / sample_se

# 这5W个t值中大于t_mu 的t值所占的比例，这个比例乘以2就是p值
(sum(t_value_array > t_value) / 50000) * 2
```

### 均值差的检验

#### 双样本t检验

之前研究的是单样本t检验，研究对象是单变量的，比如1种薯片的均重。

现在介绍如何判断2种变量的均值是否有差异。

比如，在研究吃药前和吃药后体温是否有变化（吃药前的体温均值和吃药后的体温均值，配对样本t检验），或者大鱼钩和小鱼钩所钓的鱼的体长是否有差异（双样本t检验），就会用到均值差的检验。

#### 配对样本t检验

配对样本t检验用于研究两个不同条件下对同一对象进行测量所得到的值的区别，比如之前提到的吃药前后的体温

要服药前后的体温举例子，如果前后的体温差的均值不为0，则认为服药前后体温有变化。

在配对样本t检验中，要先求出数据的差，再检验这些差值的均值是否为0.也就是说，将问题转化为了单样本t检验。

#### 环境准备

参考 [[#t检验的实现-环境准备]]

```python
paired_test_data = pd = read_csv("file_path")
```

- 零假设：服药前后体温不变
- 备择假设：服药前后体温变化

显著水平为5%，如果p值小于0.05，就拒绝零假设，并认为服药前后体温存在显著性差异

#### 实现配对样本t检验

主要思路是，拿到服药前后的体温向量，然后作差。

```python
before = paired_test_data.query('medicine == "before"')["body_temperature"]
after = paired_test_data.query('medicine == "after"')["body_temperature"]
# 转化为向量数组
before = np.array(before)
after = np.array(after)
# 计算差值, 也就是diff向量
diff = after = before
# 单样本t检验观察体温变化的均值与不变化（也就是0）是否存在差异
# 输出的statistic为t值，pvalue为p值
stats.ttest_lsamp(diff, 0)
# or
# 输出的statistic为t值，pvalue为p值, 同上
stats.ttest_rel(after, before)
```

#### 独立样本t检验

独立样本t检验的关注重点是两组数据均值的差，而配对样本t检验则是先求数据的差值，再进行单样本t检验。这就是二者的不同。

设x和y是要检验的两个变量，比如它们分别为大钩子小钩子调到的鱼的体长。

那么独立样本t检验的公式：

$$
t = \frac{\mu_x - \mu_y}{\sqrt{{\sigma_x}^2 / m + {\sigma_y}^2 / n}}
$$

其中， $\mu_x$ 为x的样本均值，$\mu_y$为y的样本均值。m为x的样本容量，n为y的样本容量。${\sigma_x}^2$ 为x的无偏方差，${\sigma_y}^2$ 为y的无偏方差。

双变量的t分布的自由度会变得复杂，此时使用Welch的近似方法来计算p值，这种方法叫作Welch检验

#### 实现独立样本t检验

下面就对之前的服药前后的体温数据进行独立样本t检验，严格来说，独立样本t检验不适合这样的数据，但是为了方便，所以

```python
# 均值
mean_before = sp.mean(before)
mean_after = sp.mean(after)
# 无偏方差
sigma_before = sp.var(before, ddof = 1)
sigma_after = sp.var(after, ddof = 1)
# 样本容量
m = len(before)
n = len(after)
# t值
t_value = (mean_after - mean_before) / sp.sqrt(sigma_before / m + sigma_after / n)
```

或者

```python
# 输出的statistic为t值，pvalue为p值
stats.ttest_ind(after, before, equal_var = False)
```

这个p值与配对样本t检验的p值不同，所以即使对于同一组数据进行相同目的的t检验，不同的检验方法得到的p值也可能不同。

#### 独立样本t检验-同方差

有些传统教材指出，要先检查数据的同方差性，再根据情况进行相应的t检验。

不过，忽略两者是否同方差，直接以方差不同为前提进行检验也无大问题。我们可以直接对任意双变量数据使用Welch检验。

上面的函数`stats.ttest_ind` 的参数`equal_var` 为`False` ，这就表示进行方差不同的t检验，即Welch检验。

#### p值操纵

单是均值差的检验就存在多种方法，目的相同，手段不同。计算出来的p值也可能不同。所以除了这两种方法，还有曼-惠特尼的U检验等方法可以用于均值差的检验。单是，并不是说各种检验方法都可以很容易得到显著性差异。

因为不同检验算法的p值可能不同，有些人会不停地调整检测算法和更换样本来检验，以期望达到心中的目的。相当于掩耳盗铃。

不应该只关注p值而试出来的主观希望的数值。任意改变p值的行为就叫做p值操纵。无需伪造或者篡改数据，只要反复更换检验方法，就可以轻易的修改p值。

目前阶段，发现p值操纵的行为是很困难的，也没有防止方法，甚至有人提出应当禁用一切假设检验。

我们分析数据的目的不是得到心中想要的结果，而是实事求是。

### 列联表检验（卡方检验）

有时也叫做 $\chi ^{2}$  检验。

#### 使用卡方检验的好处

比如，你经营了一家网站，要判断购买按钮和客服按钮的颜色对点击率的影响。得到的数据比如是，蓝色按钮点击人数是20，红色按钮点击人数是10。显然，蓝色按钮人数更多，那么你可能觉得蓝色更合适。

但是，这个数据存在严重的缺点，它并没有记录未点击按钮的数据。

包含未点击按钮的人数的数据，这种数据表就是列联表，也叫交叉列联表。

如果包含了未点击按钮的数据，你就可以分别计算出蓝色按钮的点击率和红色按钮的点击率，有可能这两种按钮的点击率是相同的。如果样本容量大一些，那么你就可以通过点击率做出判断，这时就可以说XX色按钮更受欢迎了，完美。

如果其中的XX色按钮的样本容量太少了，就又会导致错误的结论判断，在研究这类问题的时候，列联表的作用很大。

#### 例题

你可以把红色按钮和蓝色按钮的点击次数和未点击次数做一个Excel表格，并写上一些合计的数值。如果不出现极端情况，那么红色按钮和蓝色按钮的点击率是不一样的，但是要判断二者是否有显著差异，还是需要通过假设检验来判断。

#### 计算期望频数

我们的目的是证明不同颜色的按钮的吸引力不同，换句话说，就是不同按钮的吸引力有显著性差异。反向思考一下，如果按钮的颜色对吸引力完全没有影响，我们会得到什么样的数据。这样的数据叫做期望频数。

首先，我们先把例题中的数据，做成一个期望频数的表，也就是故意把红色按钮和蓝色按钮的点击和未点击的比例调成一致，比如1:9。然后列一份表格。展示未点击和点击的人数。但是其红蓝色按钮的未点击和点击人数的比例一致。

这个表就是期望频数表。

最后，考察期望频数和观测频数之间的区别。如果差别大，就可以认为按钮的颜色影响按钮的吸引力。

注意，**卡方检验要求所有的期望频数不能小于5。 所以表格中的红蓝色按钮的无论是点击还是未点击的人数，都不能少于5。**

#### 计算观察频数和期望频数的差

$$
\chi ^{2} = \sum_{i = 1}^2\sum_{j = 1}^2\frac{(O_{ij} - E_{ij})^2}{E_{ij}}
$$

其中$O_{ij}$是第i行第j列的观测频数，$E_{ij}$是第i行第j列的期望频数。计算的结果叫$\chi^2$ 统计量。

这个公式用人话说就是，观测频数表格与期望频数表格，对应格子的差值的平方除以期望频数表格对应格子而得到的值的累加。这个累加就是统计量。

#### 计算p值

```python
import pandas as pd
import numpy as np
import scipy as sp
from scipy import stats
from matplotlib import pyplot as plt
import seaborn as sns
sns.set()

%precision 3

# 在Jupyter notebook里面显示图形
%matplotlib inline
# 之前计算X^2统计量的值
x_2 = 6.667
# 使用自由度为1的X^2分布的累积分布函数计算p值
1 - sp.stats.chi2.cdf(x = x_2, df = 1)
```

如果得到的结果小于0.05， 那么就认为按钮的颜色显著地影响了按钮的吸引力。换句话说就是，红色按钮和蓝色按钮的吸引力有显著差异。

#### 实现卡方检验

```python
click_data = pd.read_csv("file_path")
print(click_data)
# 输出如下
#       color  click  freq
#   0    blue  click   20
#   1    blue    not   230
#   2     red   click   10
#   3     red    not     40

# 将数据转换为列联表
cross = pd.pivot_table(
	data = click_data,
	values = "freq",
	aggfunc = "sum",
	index = "color",
	colums = "click"
)
print(cross)

# 计算卡方检验（禁用修正）. 输出的结果依次是X^2统计量，p值，自由度和期望频数
sp.stats.chi2_contingency(cross, correction = False)
```

### 检验结果的解读

假设检验很方便，但是容易被滥用，所以有必要学习解读判断的方法。

#### p值小于0.05时的表述方式

当p值小于0.05时，数据存在显著差异，比如在检验薯片均重是否是50g时，表述是：薯片的均重与50g存在显著性差异。

#### p值大于0.05时的表述方式

当p值大于0.05，不能拒绝零假设，此时的表述比较特殊。，比如在检验薯片的均重是否是50g时，表述是：薯片的均重和50g没有显著性差异。但是你不能说是相同。

#### 关于假设检验的常见误区

下列观点是错误的

- p值越小，差异越大。
- p值大于0.05，所以没有差异
- “1 - p值”是备择假设正确的概率

下面会一一反驳

#### p值小不代表差异大

因为影响t值的因素有很多，数据越集中，标准差越小，t值就越大。样本容量越大，t值就越大。t值肯定影响p值，所以只看一眼p值就下结论，是不稳妥的。

#### p值大于0.05不代表没有差异

通过p值可以确定零假设错误的概率，所以有一个显著性水平值。但是无法确认零假设正确的概率。所以你不能说，大于0.05，就代表没有差异。

#### 第一类错误和第二类错误

零假设正确却拒绝了零假设的行为叫第一类错误。反过来，零假设错误却接受了零假设的行为叫做第二类错误。

在假设检验中，出现第一类错误的概率就是p值，p值越小，就说明出现第一类错误的概率就越小，所以我们可以说，当p小于显著性水平值的时候，我们可以给出拒绝零假设的判断。

#### 假设检验的非对称性

第一类错误是可控的，第二类错误的概率是不可控的，这叫做假设检验的非对称性。

假设家宴无法确定第二类错误的概率。

所以假设检验只能确定第一类错误。

#### 在检验之前确定显著性水平

之前的案例中，一般我们都设定显著性水平为5%。这个在检验开始之前，就应该确定这个数值。虽然5%和1%常常被作为显著性水平的数值，但是至于为何选取它们，并没有特别的理由。

一定要注意，不同领域所用的显著性水平的数值不同，要结合不同的领域的规定来检验。比如，生物学研究领域，也是用5%作为显著性水平。但是其他领域，不保证。

#### 统计模型的选择

学习相关的知识，可以缓解一些由检验的非对称性导致的问题。这个就比较高级了。后面会讲。

#### 假设检验有什么用

之前说了，统计模型的选择可能可以缓解假设检验带来的非对称性问题，假设检验相当于比较初级的方法，所以有可能未来会被淘汰。这里只是有可能被淘汰。

但是，目前，脱离假设检验，数据分析会变得很困难而低效。

#### 假设是否正确

在t检验中，我们假设了总体服从正态分布。如果总体不服从正态分布，那么计算出来的p值就是错误的。这个大前提是一切的保证。

所以，我们无时无刻都要考虑如何才能减少样本和假设之间的偏离。

在采用传统的均值差检验等方法在解决上述偏离问题会显得比较无力。这样，我们就需要学习统计模型，它可以结合实际灵活地进行数据分析，可以说它是统计学的新标准。




