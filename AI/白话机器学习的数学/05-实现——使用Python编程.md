---
title: 第5章 实现——使用Python编程
book: 白话机器学习的数学
chapter: 5
tags: [机器学习, Python, NumPy, scikit-learn, 回归实现, 分类实现, 正则化]
---

# 第5章 实现——使用 Python 编程

> [!summary] 本章主线
> 手写模型不是为了在生产中重新发明 scikit-learn，而是为了把公式、数据形状、参数更新和评估结果连成一条可观察的程序链。理解这条链之后，成熟库才不再像黑盒。

## 5.1 使用 Python 实现

### 这一节在回答什么

如何把前面看到的模型变成可以运行、检查和复现的程序？

### 我的理解

一个最小机器学习程序通常有六层：

1. 读入并检查数据；
2. 划分训练、验证和测试；
3. 拟合预处理器；
4. 定义模型和损失；
5. 训练并记录参数或损失；
6. 用与训练隔离的数据和业务指标评估。

NumPy 适合帮助我们理解数组运算和梯度；scikit-learn 适合稳定地完成数据划分、Pipeline、交叉验证和成熟求解器。二者的关系像“透明的实验模型”和“可靠的工程工具”，不是非此即彼。

### 第一性原理洞见

**实现的本质不是把公式翻译成语法，而是把每一个假设变成一个可以被检查的对象。** 数据形状、标签编码、损失、学习率和随机种子都应能被观察。

### 工业联系

生产训练代码还要记录数据快照或时间范围、特征版本、依赖版本、随机种子、超参数、评估集构造方式和模型文件校验值。模型服务代码要复用同一套特征变换，避免训练—服务偏差。

### 最小工程约定

- 函数命名区分 fit、transform、predict 和 evaluate；
- 训练函数返回参数、损失历史和必要的诊断信息；
- 所有随机过程显式设置随机种子；
- 训练集之外的数据只调用 transform，不重新 fit 预处理器；
- 先写一个简单的基线，再增加复杂度。

## 5.2 回归

### 5.2.1 确认训练数据

#### 这一节在回答什么

在写回归代码前，如何确认数据确实是模型以为的样子？

#### 我的理解

先确认特征矩阵和目标向量的形状。若有 n 条样本、p 个特征，通常 $X$ 的形状是 $n\times p$，$y$ 的形状是 $n$。每一行应是一条样本，每一列应是一个特征，不能把行列含义弄反。

再检查缺失值、重复样本、异常值、目标范围、时间顺序和训练—测试分布。广告例子中还要确认点击量的时间窗口一致，不能把短窗口和长窗口直接混在一起。

~~~python
import numpy as np

def inspect_regression_data(X, y):
    X = np.asarray(X, dtype=float)
    y = np.asarray(y, dtype=float).reshape(-1)
    assert X.ndim == 2
    assert y.ndim == 1
    assert len(X) == len(y)
    print("X shape:", X.shape)
    print("y shape:", y.shape)
    print("missing in X:", np.isnan(X).sum())
    print("missing in y:", np.isnan(y).sum())
    print("target range:", float(y.min()), float(y.max()))
    return X, y
~~~

#### 第一性原理洞见

**数据检查是在确认“程序正在回答我以为的问题”。** 如果一列其实是曝光量、一列其实包含未来信息，模型可以正常运行，却在回答错误的问题。

#### 工业联系

生产数据检查应包括 schema、单位、缺失率、取值范围、时间新鲜度和分布漂移。Google 的生产机器学习指南把数据管道和训练—服务一致性放在模型性能之前，原因就是错误输入会让任何模型失效。

### 5.2.2 作为一次函数实现

#### 这一节在回答什么

如何用最少的 NumPy 代码实现一次函数和最小二乘拟合？

#### 我的理解

先把截距合并进设计矩阵，再使用稳定的最小二乘求解器。不要手写矩阵求逆；理解正规方程的目的即可，数值稳定性应交给线性代数库。

~~~python
def fit_linear_regression(X, y):
    X = np.asarray(X, dtype=float)
    y = np.asarray(y, dtype=float).reshape(-1)
    X_bias = np.c_[np.ones(len(X)), X]
    weights, residuals, rank, singular_values = np.linalg.lstsq(
        X_bias, y, rcond=None
    )
    return weights

def predict_linear_regression(X, weights):
    X = np.asarray(X, dtype=float)
    X_bias = np.c_[np.ones(len(X)), X]
    return X_bias @ weights
~~~

这里参数向量第一项是截距，其余项是特征权重。预测是矩阵乘法，正好对应前面模型的表达式：

$$
\hat{y}=w_0+\sum_{j=1}^{p}w_jx_j
$$

#### 第一性原理洞见

**代码最重要的透明性，是每个数组运算都能回到一个人话动作：加上基准、按权重计分、比较误差。**

#### 工业联系

生产中会优先使用库提供的 LinearRegression、Ridge 或专门求解器，因为它们处理数值稳定性、稀疏矩阵、多目标和边界情况更成熟。手写版本适合单元测试和学习，不应直接替代经过验证的实现。

### 5.2.3 验证

#### 这一节在回答什么

如何确认回归程序不是只在训练数据上看起来很好？

#### 我的理解

至少要分出训练集和测试集；若要调超参数，再增加验证集或使用交叉验证。指标要和业务场景相配，并与一个简单基线比较，例如始终预测训练目标均值。

~~~python
from sklearn.model_selection import train_test_split
from sklearn.metrics import mean_absolute_error, mean_squared_error, r2_score

X_train, X_test, y_train, y_test = train_test_split(
    X, y, test_size=0.2, random_state=42
)

w = fit_linear_regression(X_train, y_train)
y_pred = predict_linear_regression(X_test, w)

mae = mean_absolute_error(y_test, y_pred)
rmse = mean_squared_error(y_test, y_pred) ** 0.5
r2 = r2_score(y_test, y_pred)
print({"MAE": mae, "RMSE": rmse, "R2": r2})
~~~

如果数据有时间顺序，不要直接使用随机切分，应改为按时间切分；如果同一用户有多条记录，应考虑按用户分组。

#### 第一性原理洞见

**验证代码是在模拟未来，而不是给训练代码打分。** 测试集的价值来自它没有参与模型和方案选择。

#### 工业联系

离线验证最好保留一组真正的时间外或业务外测试数据；模型上线后还要用延迟标签回填真实误差。只保存一个训练分数，无法判断系统是否值得上线。

### 5.2.4 多项式回归的实现

#### 这一节在回答什么

怎样加入弯曲趋势，同时避免手工拼接特征时造成数据泄漏或顺序错误？

#### 我的理解

多项式特征扩展、标准化、模型拟合和评估应组成一个 Pipeline。Pipeline 的重要性不在于代码更短，而在于交叉验证的每一折都会只用该折训练部分拟合特征变换。

~~~python
from sklearn.pipeline import Pipeline
from sklearn.preprocessing import PolynomialFeatures, StandardScaler
from sklearn.linear_model import Ridge

poly_model = Pipeline([
    ("poly", PolynomialFeatures(degree=2, include_bias=False)),
    ("scale", StandardScaler()),
    ("regressor", Ridge(alpha=1.0))
])

poly_model.fit(X_train, y_train)
poly_pred = poly_model.predict(X_test)
print("poly MAE:", mean_absolute_error(y_test, poly_pred))
~~~

多项式扩展增加的是表示能力；Ridge 则限制因扩展而产生的过大权重。二者经常一起使用，因为高次特征很容易造成方差上升。

#### 第一性原理洞见

**工程上的模型不是一个孤立公式，而是一条从原始输入到预测的完整变换链。** 只保存最后一个回归器而忘记保存特征扩展，等于丢失了模型的一半。

#### 工业联系

线上服务必须保存完整 Pipeline 或等价的版本化变换。特征扩展还会造成维度爆炸，应监控内存、延迟和新特征的稳定性。

### 5.2.5 随机梯度下降法的实现

#### 这一节在回答什么

如何把一次梯度更新写成可以观察的训练循环？

#### 我的理解

下面是一个教学版的线性回归 SGD。它使用小批量和均方误差梯度；真实工程中还要加入标准化、学习率调度、早停、检查点和验证监控。

~~~python
def fit_sgd_regression(X, y, epochs=100, learning_rate=0.01,
                       batch_size=32, seed=42):
    X = np.asarray(X, dtype=float)
    y = np.asarray(y, dtype=float).reshape(-1)
    X_bias = np.c_[np.ones(len(X)), X]
    rng = np.random.default_rng(seed)
    weights = np.zeros(X_bias.shape[1])
    history = []

    for _ in range(epochs):
        order = rng.permutation(len(X_bias))
        for start in range(0, len(order), batch_size):
            batch = order[start:start + batch_size]
            xb = X_bias[batch]
            yb = y[batch]
            error = xb @ weights - yb
            gradient = 2.0 * xb.T @ error / len(batch)
            weights -= learning_rate * gradient

        full_error = X_bias @ weights - y
        history.append(float(np.mean(full_error ** 2)))

    return weights, history
~~~

每一轮的损失历史应与验证损失一起看。若训练损失震荡，先检查学习率和特征尺度；若训练持续下降、验证先降后升，考虑过拟合和早停。

更新的数学形式是：

$$
w\leftarrow w-\eta\nabla J(w)
$$

#### 第一性原理洞见

**训练循环是一个反馈系统：预测产生误差，误差产生方向，方向改变参数，参数再影响下一次预测。** 记录反馈过程，才能知道系统是在学习、震荡还是记忆噪声。

#### 工业联系

大规模稀疏数据适合分批读入、增量训练或使用成熟的 SGD 实现。模型更新还需要可回滚、可重放和可比较，不能让一个随机训练循环直接覆盖线上模型。

## 5.3 分类——感知机

### 5.3.1 确认训练数据

#### 这一节在回答什么

感知机的输入和标签如何准备？

#### 我的理解

感知机常用 -1/+1 标签，因为更新条件和方向表达更自然。若原始标签是 0/1，应在训练函数中明确转换，并在输出阶段再转换回业务类别。

~~~python
def prepare_perceptron_data(X, y):
    X = np.asarray(X, dtype=float)
    y = np.asarray(y).reshape(-1)
    y_pm = np.where(y == 0, -1, 1)
    X_bias = np.c_[np.ones(len(X)), X]
    return X_bias, y_pm
~~~

要特别检查类别是否真的只有两类、是否有空类别、是否存在重复样本和标签冲突。图像大小这类教学数据还要画散点图，看看线性边界是否有机会存在。

#### 第一性原理洞见

**分类数据确认的重点不是“能不能转成数字”，而是“这些数字是否保留了可分的证据”。**

### 5.3.2 感知机的实现

#### 这一节在回答什么

如何把“分错才更新”写成循环？

#### 我的理解

使用 -1/+1 标签时，如果 y 与当前总分的乘积不大于 0，就更新权重。记录每轮错误数，可以观察数据是否趋于可分。

~~~python
def fit_perceptron(X, y, epochs=20, learning_rate=1.0):
    X_bias, y_pm = prepare_perceptron_data(X, y)
    weights = np.zeros(X_bias.shape[1])
    errors_history = []

    for _ in range(epochs):
        errors = 0
        for xi, yi in zip(X_bias, y_pm):
            if yi * (xi @ weights) <= 0:
                weights += learning_rate * yi * xi
                errors += 1
        errors_history.append(errors)

    return weights, errors_history

def predict_perceptron(X, weights):
    X = np.asarray(X, dtype=float)
    X_bias = np.c_[np.ones(len(X)), X]
    scores = X_bias @ weights
    return np.where(scores >= 0, 1, 0)
~~~

更新公式为：

$$
w\leftarrow w+\eta yx
$$

它只在错误样本上执行。若数据线性不可分，错误数可能不会归零，停止条件就不能只依赖“没有错误”，还要设置最大轮数或验证表现。

#### 第一性原理洞见

**手写感知机最值得观察的不是最终权重，而是每个错误样本怎样改变边界。** 一次更新的意义可以被画出来、打印出来和反驳出来。

### 5.3.3 验证

#### 这一节在回答什么

怎样验证感知机得到的边界不是训练数据的偶然结果？

#### 我的理解

在验证集上计算准确率只是起点，还要看混淆矩阵、正类召回率和阈值附近的样本。感知机输出的是符号类别，没有天然概率，因此如果业务需要排序或风险分层，感知机可能不是合适的最终模型。

~~~python
from sklearn.metrics import accuracy_score, confusion_matrix

perceptron_weights, error_history = fit_perceptron(X_train, y_train)
pred = predict_perceptron(X_test, perceptron_weights)
print("accuracy:", accuracy_score(y_test, pred))
print("confusion matrix:")
print(confusion_matrix(y_test, pred))
~~~

#### 第一性原理洞见

**验证感知机是在验证一条边界能否稳定工作，而不是验证训练循环有没有执行。**

## 5.4 分类——逻辑回归

### 5.4.1 确认训练数据

#### 这一节在回答什么

逻辑回归要求的数据与感知机有什么不同？

#### 我的理解

逻辑回归通常使用 0/1 标签，输出正类概率。特征仍应完成数值化、缺失处理和尺度检查；类别不平衡、标签延迟和同一用户重复记录需要在数据切分时处理。

如果特征有极端值，Sigmoid 的总分可能迅速进入饱和区，梯度变得很小。标准化、稳健缩放和异常值检查能改善优化，但不能替代业务上的异常分析。

### 5.4.2 逻辑回归的实现

#### 这一节在回答什么

如何用 NumPy 从零实现一个带 L2 正则化的逻辑回归？

#### 我的理解

先实现数值稳定的 Sigmoid，再用预测概率减标签形成梯度。下面的正则化只施加在非截距权重上。

~~~python
def stable_sigmoid(z):
    z = np.asarray(z, dtype=float)
    out = np.empty_like(z)
    positive = z >= 0
    out[positive] = 1.0 / (1.0 + np.exp(-z[positive]))
    exp_z = np.exp(z[~positive])
    out[~positive] = exp_z / (1.0 + exp_z)
    return out

def fit_logistic_regression(X, y, epochs=1000, learning_rate=0.05,
                            l2_strength=0.0):
    X = np.asarray(X, dtype=float)
    y = np.asarray(y, dtype=float).reshape(-1)
    X_bias = np.c_[np.ones(len(X)), X]
    weights = np.zeros(X_bias.shape[1])
    history = []

    for _ in range(epochs):
        probability = stable_sigmoid(X_bias @ weights)
        error = probability - y
        gradient = X_bias.T @ error / len(X_bias)
        gradient[1:] += l2_strength * weights[1:]
        weights -= learning_rate * gradient

        eps = 1e-12
        clipped = np.clip(probability, eps, 1.0 - eps)
        loss = -np.mean(
            y * np.log(clipped) + (1.0 - y) * np.log(1.0 - clipped)
        )
        loss += 0.5 * l2_strength * np.sum(weights[1:] ** 2)
        history.append(float(loss))

    return weights, history

def predict_logistic_probability(X, weights):
    X = np.asarray(X, dtype=float)
    X_bias = np.c_[np.ones(len(X)), X]
    return stable_sigmoid(X_bias @ weights)

def predict_logistic_label(X, weights, threshold=0.5):
    probability = predict_logistic_probability(X, weights)
    return (probability >= threshold).astype(int)
~~~

逻辑回归的二元对数损失可以写成：

$$
L=-\left[y\log(p)+(1-y)\log(1-p)\right]
$$

#### 第一性原理洞见

**从零实现的价值，是让“概率、损失、梯度和正则化”在同一个函数里相遇。** 任何一个词如果没有对应代码，就还没有真正落地。

#### 工业联系

成熟库会处理更多求解器、停止条件、多类别、类别权重和数值边界。手写版本应通过小数据和数值梯度检查后，再用于学习或单元测试。

### 5.4.3 验证

#### 这一节在回答什么

如何同时评价逻辑回归的概率、标签和阈值？

#### 我的理解

不要只调用一次 predict 得到 0/1 结果。先保存概率，再在验证集上选择阈值，并分别计算 ROC-AUC、精确率、召回率、F1、对数损失和校准表现。

~~~python
from sklearn.metrics import (
    roc_auc_score, precision_score, recall_score,
    f1_score, log_loss, confusion_matrix
)

logistic_weights, loss_history = fit_logistic_regression(
    X_train, y_train, l2_strength=0.01
)
probability = predict_logistic_probability(X_test, logistic_weights)
pred = (probability >= 0.5).astype(int)

print("AUC:", roc_auc_score(y_test, probability))
print("log loss:", log_loss(y_test, probability))
print("precision:", precision_score(y_test, pred, zero_division=0))
print("recall:", recall_score(y_test, pred, zero_division=0))
print("F1:", f1_score(y_test, pred, zero_division=0))
print(confusion_matrix(y_test, pred))
~~~

阈值选择不能使用最终测试集反复试验。应在训练交叉验证或独立验证集上根据业务成本选择，再在测试集上做一次报告。

#### 第一性原理洞见

**概率模型的验证要把“相信程度”和“行动结果”分开。** 同一组概率可以配合不同阈值服务不同业务场景。

### 5.4.4 线性不可分分类的实现

#### 这一节在回答什么

如何让线性分类器处理需要弯曲边界的简单问题？

#### 我的理解

可以先把输入扩展成多项式特征，再训练逻辑回归。下面示例用 Pipeline 把特征扩展、标准化和带 L2 的分类器绑在一起。

~~~python
from sklearn.linear_model import LogisticRegression

nonlinear_logistic = Pipeline([
    ("poly", PolynomialFeatures(degree=2, include_bias=False)),
    ("scale", StandardScaler()),
    ("classifier", LogisticRegression(C=1.0, max_iter=2000))
])

nonlinear_logistic.fit(X_train, y_train)
nonlinear_pred = nonlinear_logistic.predict(X_test)
print("nonlinear accuracy:", accuracy_score(y_test, nonlinear_pred))
~~~

这并没有让逻辑回归本身变成任意非线性模型，而是改变了输入坐标。若特征扩展后仍不够，才考虑树、核方法或神经网络等更强的表示和边界。

#### 第一性原理洞见

**所谓“线性不可分的解决方案”，常常是在寻找一个更适合描述现实的坐标系。** 复杂模型的第一步也可以理解为自动学习坐标系。

#### 工业联系

多项式扩展适合小规模、低维、可解释的实验；高维生产特征容易维度爆炸，通常更适合专门的交互设计、树模型或学习表示。

### 5.4.5 随机梯度下降法的实现

#### 这一节在回答什么

如何把逻辑回归改成小批量随机更新？

#### 我的理解

逻辑回归的批量梯度可以换成每个小批量的梯度。要注意每轮打乱样本、保持学习率适当、记录验证对数损失，并在正则化和梯度平均的系数上与目标函数保持一致。

~~~python
def fit_logistic_sgd(X, y, epochs=20, learning_rate=0.05,
                     batch_size=32, l2_strength=0.01, seed=42):
    X = np.asarray(X, dtype=float)
    y = np.asarray(y, dtype=float).reshape(-1)
    X_bias = np.c_[np.ones(len(X)), X]
    rng = np.random.default_rng(seed)
    weights = np.zeros(X_bias.shape[1])
    history = []

    for _ in range(epochs):
        order = rng.permutation(len(X_bias))
        for start in range(0, len(order), batch_size):
            batch = order[start:start + batch_size]
            xb = X_bias[batch]
            yb = y[batch]
            probability = stable_sigmoid(xb @ weights)
            gradient = xb.T @ (probability - yb) / len(batch)
            gradient[1:] += l2_strength * weights[1:]
            weights -= learning_rate * gradient

        full_probability = stable_sigmoid(X_bias @ weights)
        clipped = np.clip(full_probability, 1e-12, 1.0 - 1e-12)
        loss = -np.mean(
            y * np.log(clipped) + (1.0 - y) * np.log(1.0 - clipped)
        )
        history.append(float(loss))

    return weights, history
~~~

#### 第一性原理洞见

**小批量训练把“全体数据的结论”换成“当前一小群样本提供的证据”，所以它必须靠重复、平均和验证来抵消偶然性。**

#### 工业联系

scikit-learn 的 SGDClassifier 使用 log_loss 时可以训练逻辑分类器，并支持稀疏输入与 partial_fit。对于在线学习，还要决定旧数据如何保留、标签何时成熟和模型怎样回滚。

## 5.5 正则化

### 5.5.1 确认训练数据

#### 这一节在回答什么

为什么正则化实验前更需要检查特征尺度和数据划分？

#### 我的理解

正则化直接惩罚权重大小，所以如果一列用元、一列用千次曝光，惩罚就不公平。应先在训练部分拟合 StandardScaler，再把同一变换应用到验证和测试。

还要确认训练集和验证集不包含重复样本、不共享未来信息，且类别比例与业务情境一致。否则正则化强度选得再好，也只是在错误实验上优化。

### 5.5.2 不应用正则化的实现

#### 这一节在回答什么

如何建立一个没有正则化的对照组？

#### 我的理解

对照组很重要，因为没有它就无法知道正则化到底改变了什么。用手写逻辑回归时把 l2_strength 设为 0；用 scikit-learn 时，应明确选择没有惩罚的配置，并确认当前版本和求解器支持该配置。

~~~python
unregularized_weights, unregularized_history = fit_logistic_regression(
    X_train, y_train, l2_strength=0.0
)
unregularized_probability = predict_logistic_probability(
    X_test, unregularized_weights
)
~~~

比较时不要只看训练损失。要同时看验证对数损失、权重大小、概率是否极端、阈值后的混淆矩阵和不同随机切分的波动。

### 5.5.3 应用了正则化的实现

#### 这一节在回答什么

如何把正则化加入模型，并通过验证选择强度？

#### 我的理解

手写实现中，正则化梯度加入非截距权重；成熟库中通常通过参数控制。下面示例使用 Pipeline 和交叉验证搜索 Ridge 的 alpha。分类时若使用 LogisticRegression，参数 C 的方向与 alpha 相反，需要看清文档。

~~~python
from sklearn.model_selection import GridSearchCV

regularized_model = Pipeline([
    ("scale", StandardScaler()),
    ("classifier", LogisticRegression(max_iter=2000))
])

search = GridSearchCV(
    regularized_model,
    param_grid={"classifier__C": [0.01, 0.1, 1.0, 10.0]},
    scoring="f1",
    cv=5
)
search.fit(X_train, y_train)
print("best params:", search.best_params_)
print("test score:", search.score(X_test, y_test))
~~~

在回归中，Ridge 的目标可写成：

$$
J(w)=\sum_{i=1}^{n}(y_i-\hat{y}_i)^2+\alpha\sum_{j=1}^{p}w_j^2
$$

在分类中，正则化应与 log loss、类别权重和阈值选择共同验证。最优超参数不是一个永恒常数，它依赖数据规模、特征工程、标签噪声和业务成本。

#### 第一性原理洞见

**正则化实验的重点不是找到神奇的系数，而是观察模型从“追随样本”转向“保持稳定”的过程。** 把这条过程画出来，比只记录最优参数更能迁移到新问题。

#### 工业联系

模型注册系统应保存选择出的超参数、交叉验证方案、评估指标、特征版本和阈值。上线后若分布改变，不能直接沿用旧参数而不重做验证。

## 5.6 后话

### 这一节在回答什么

学完这些线性模型之后，下一步应该怎样继续？

### 我的理解

本书的前五章把机器学习最小骨架讲清楚了：

$$
\text{数据}
\rightarrow
\text{模型}
\rightarrow
\text{损失}
\rightarrow
\text{优化}
\rightarrow
\text{评估}
\rightarrow
\text{部署与监控}
$$

后续学习可以沿三条路延伸：

1. **更好的表示**：特征工程、词向量、图像编码器和神经网络；
2. **更强的边界**：树模型、支持向量机、集成方法；
3. **更严谨的系统**：校准、因果推断、时间外验证、漂移监控和公平性评估。

但无论模型多复杂，都仍然要回答同一组问题：它看到了什么、把什么算作错误、如何更新、如何证明没有只记住训练数据，以及上线后怎样知道自己变了。

### 第一性原理洞见

**复杂模型不是对基础模型的否定，而是把“表示、边界和优化”做得更强；评估和问题定义的责任不会消失。**

### 工业联系

真正的机器学习工程是闭环系统，而不是一次训练命令。Google 的 Rules of ML 强调先建设稳定的端到端管道、明确目标和监控训练—服务偏差；这也是从本书玩具例子走向生产系统时最重要的升级。

## 本章完整练习路线

1. 用 NumPy 手写线性回归，画出预测线和损失历史。
2. 用相同数据对比正规方程和 SGD，观察数据规模、学习率和标准化的影响。
3. 手写感知机，记录每一轮错误样本和权重变化。
4. 手写逻辑回归，加入数值稳定 Sigmoid 和梯度检查。
5. 对不可分数据加入二次特征，比较有无正则化的边界。
6. 用 Pipeline、交叉验证和时间外测试重做一次实验。
7. 把离线指标转换为每天误报、漏报、人工审核量或金额损失，写一份上线前评估报告。

## 实现检查表

- [ ] 训练、验证、测试的角色清楚；
- [ ] 预处理只在训练折上 fit；
- [ ] 训练和服务使用同一套特征变换；
- [ ] 标签编码与损失、预测函数一致；
- [ ] 学习率、随机种子、轮数和停止条件可复现；
- [ ] 记录训练与验证损失，而不是只记录最终分数；
- [ ] 分类报告包含混淆矩阵、精确率、召回率和阈值；
- [ ] 正则化强度与阈值通过验证数据选择；
- [ ] 测试集没有被反复试用；
- [ ] 上线后有数据分布、模型年龄、预测分布和真实效果监控。

## 英文补充资料

- [scikit-learn Linear Models](https://scikit-learn.org/stable/modules/linear_model.html)
- [scikit-learn Pipeline](https://scikit-learn.org/stable/modules/generated/sklearn.pipeline.Pipeline.html)
- [scikit-learn StandardScaler](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.StandardScaler.html)
- [scikit-learn Ridge](https://scikit-learn.org/stable/modules/generated/sklearn.linear_model.Ridge.html)
- [scikit-learn PolynomialFeatures](https://scikit-learn.org/stable/modules/generated/sklearn.preprocessing.PolynomialFeatures.html)
- [scikit-learn Common pitfalls](https://scikit-learn.org/stable/common_pitfalls.html)
- [Google Production ML Systems: Monitoring](https://developers.google.com/machine-learning/crash-course/production-ml-systems/monitoring)
- [Google Rules of Machine Learning](https://developers.google.com/machine-learning/guides/rules-of-ml)
