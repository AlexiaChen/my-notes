---
title: "第8章 神经网络与深度学习的应用：手写数字识别"
chapter: 8
tags: [MNIST, 计算机视觉, CNN, 卷积, 池化, Dropout, ReLU]
depends_on: [[07-神经网络与深度学习]]
next: [[09-无监督学习]]
---

# 第8章 神经网络与深度学习的应用（手写数字识别）

## 先纠正一个名称：MNIST

目录中出现的 “MINST” 是常见拼写错误，标准名称是 **MNIST**（Modified National Institute of Standards and Technology database）。它包含 28×28 灰度手写数字图像，是学习图像分类的经典基准，不等于真实工业数据的难度。

## 本章的第一性原理

同一份图像数据，可以被不同方式表示：

```text
28×28 像素 → 展平为 784 维向量 → 全连接网络
28×28 像素 → 保留局部邻域 → 卷积/池化网络
```

准确率的提升不只是“网络更大”，而是模型把图像的空间结构、局部性和一定程度的位置不敏感性写进了架构。

## 8.1 MNIST 数据集

```python
import numpy as np
import keras

(x_train, y_train), (x_test, y_test) = keras.datasets.mnist.load_data()
print(x_train.shape, y_train.shape)  # (60000,28,28), (60000,)
print(x_test.shape, y_test.shape)    # (10000,28,28), (10000,)
```

像素值通常是 0 到 255，先缩放到 0 到 1：

```python
x_train = x_train.astype("float32") / 255.0
x_test = x_test.astype("float32") / 255.0
```

还要检查类别分布、重复样本、训练/测试是否有泄漏，并随机显示样本：

```python
import matplotlib.pyplot as plt
plt.imshow(x_train[0], cmap="gray")
plt.title(f"label={y_train[0]}")
plt.axis("off")
plt.show()
```

数据预处理不是“为了让代码能跑”，而是改变优化问题的尺度。若把 0 到 255 直接输入网络，梯度和初始化的相对尺度会更难控制。

## 8.2 二层前馈神经网络模型

最简单的基线先把图像展平：

```python
from keras import layers

mlp = keras.Sequential([
    keras.Input(shape=(28, 28)),
    layers.Flatten(),
    layers.Dense(128, activation="sigmoid"),
    layers.Dense(10, activation="softmax"),
])
mlp.compile(
    optimizer="adam",
    loss="sparse_categorical_crossentropy",
    metrics=["accuracy"],
)
```

`Flatten` 把空间坐标变成一条向量，网络不再知道相邻像素是相邻的。它仍能识别 MNIST，说明全连接层有很强的拟合能力；但参数量大、对平移敏感，也不容易扩展到更大的图像。

基线的意义：先有一个简单、可复现、速度快的参照，再判断卷积结构到底带来了什么。

## 8.3 ReLU 激活函数

$$
\operatorname{ReLU}(x)=\max(0,x).
$$

它在正区间梯度为 1，通常比 sigmoid 更不容易在深层网络中出现梯度消失，也更容易计算。负输入的梯度为 0，可能造成“死亡 ReLU”，所以初始化、学习率和数据标准化仍然重要。

```python
layers.Dense(128, activation="relu")
```

选择激活函数的第一性问题不是“哪个最流行”，而是：梯度是否能有效传播？输出范围是否与任务匹配？数值是否稳定？

## 8.4 空间过滤器

图像中的边缘、笔画和角点是局部模式。一个小过滤器在图像上滑动，对每个局部窗口做加权求和，得到特征图：

$$
S(i,j)=\sum_{u,v}K(u,v)X(i+u,j+v).
$$

早期可手写一个边缘过滤器体验其效果：

```python
from scipy.signal import convolve2d

edge = np.array([[-1, -1, -1],
                 [ 0,  0,  0],
                 [ 1,  1,  1]], dtype=float)
feature = convolve2d(x_train[0], edge, mode="same")
plt.imshow(feature, cmap="gray")
plt.show()
```

深度学习中，过滤器的数值不必手工设计，而是通过损失函数学习。关键先验是局部连接和权重共享：同一个笔画模式可以出现在图像不同位置。

## 8.5 卷积神经网络

卷积层的输入通常是 `(height, width, channels)`。一个简单 CNN：

```python
cnn = keras.Sequential([
    keras.Input(shape=(28, 28, 1)),
    layers.Conv2D(32, kernel_size=3, activation="relu"),
    layers.MaxPooling2D(pool_size=2),
    layers.Conv2D(64, kernel_size=3, activation="relu"),
    layers.MaxPooling2D(pool_size=2),
    layers.Flatten(),
    layers.Dense(10, activation="softmax"),
])
```

输入要增加通道维：

```python
x_train_cnn = x_train[..., None]
x_test_cnn = x_test[..., None]
```

CNN 的三个核心归纳偏置：

1. **局部性**：局部邻域更可能形成有意义的模式；
2. **权重共享**：相同过滤器可在不同位置检测相同模式；
3. **层级组合**：边缘组合成笔画，笔画组合成数字形状。

因此 CNN 往往用更少参数获得更适合图像的表示。卷积在严格数学上常与互相关的实现约定混用；对学习结果来说，关键是过滤器和数据共同训练。

## 8.6 池化

最大池化在窗口中取最大值：

```python
layers.MaxPooling2D(pool_size=(2, 2))
```

它减少空间尺寸和计算量，并让模型对小幅位置变化不那么敏感。代价是丢失细节；对需要像素级定位的任务，不能盲目池化。

平均池化保留窗口平均趋势，最大池化更像“这个局部是否出现了强响应”。选择哪种池化取决于任务和网络设计，不是固定口诀。

## 8.7 Dropout

训练时以一定概率随机屏蔽单元：

```python
layers.Dropout(0.5)
```

第一性原理是限制模型过度依赖某一组特征，近似训练许多共享参数的子网络；推理时通常关闭 Dropout，并使用相应的缩放约定。它是正则化，不是增加信息，也不能修复标签错误和数据泄漏。

```python
cnn = keras.Sequential([
    keras.Input(shape=(28, 28, 1)),
    layers.Conv2D(32, 3, activation="relu"),
    layers.MaxPooling2D(2),
    layers.Conv2D(64, 3, activation="relu"),
    layers.MaxPooling2D(2),
    layers.Flatten(),
    layers.Dropout(0.5),
    layers.Dense(10, activation="softmax"),
])
```

## 8.8 融合了各种特性的 MNIST 识别网络模型

一个完整实验可这样写：

```python
model = keras.Sequential([
    keras.Input(shape=(28, 28, 1)),
    layers.Conv2D(32, 3, padding="same", activation="relu"),
    layers.MaxPooling2D(2),
    layers.Conv2D(64, 3, padding="same", activation="relu"),
    layers.MaxPooling2D(2),
    layers.Flatten(),
    layers.Dropout(0.5),
    layers.Dense(10, activation="softmax"),
])
model.compile(
    optimizer="adam",
    loss="sparse_categorical_crossentropy",
    metrics=["accuracy"],
)
history = model.fit(
    x_train_cnn, y_train,
    validation_split=0.1,
    epochs=10,
    batch_size=128,
)
test_loss, test_acc = model.evaluate(x_test_cnn, y_test, verbose=0)
```

评估不仅看 `test_acc`，还应看混淆矩阵、各数字类别的召回率、错误样本图和置信度。MNIST 上的高分不代表模型懂得数字概念，更不代表它能直接用于真实手写、扫描件或工业字符；真实部署需要域内测试、数据增强和持续监控。

## 工业应用：从数字识别到视觉质检

这章的思想可以迁移到 OCR、票据识别、零件缺陷检测和医学图像分类：

- 保留空间结构，优先考虑卷积或其他视觉架构；
- 训练/验证/测试按设备、批次或时间隔离，避免同一物体的近重复图像泄漏；
- 记录输入尺寸、通道顺序、归一化方式和标签规则；
- 线上关注误报/漏报成本、延迟、置信度分布和新工况，而不只看离线准确率；
- 新类别或新相机出现时，重新评估分布偏移。

## 本章练习

1. 比较 MLP 与 CNN 的参数量和测试误差。
2. 可视化第一层卷积核的输出，观察它们是否响应边缘/笔画。
3. 去掉池化或 Dropout，记录训练与验证曲线的变化。
4. 画出最高置信度但预测错误的 MNIST 图片，思考模型为何会自信地错。
5. 把 MNIST 的训练/测试切分改为按数字书写者或模拟批次切分，讨论泛化差异。

## 关键连接

- [[07-神经网络与深度学习]]：CNN 仍然是前馈网络，只是加入了图像结构先验。
- [[03-数据可视化]]：特征图、训练曲线和错误样本是模型调试的窗口。
- [[06-有监督学习-分类]]：十分类的 softmax 与交叉熵直接复用。
- [[10-本书小结]]：输入表示、模型假设和评估协议共同决定结果。

## 英文补充

- [Keras Simple MNIST convnet](https://keras.io/examples/vision/mnist_convnet/)
- [TensorFlow beginner MNIST quickstart](https://www.tensorflow.org/tutorials/quickstart/beginner)
- [CS231n：Convolutional Networks](https://cs231n.github.io/convolutional-networks/)
- [D2L：Convolutional Neural Networks](https://classic.d2l.ai/chapter_convolutional-neural-networks/index.html)
- [Google：Activation functions](https://developers.google.com/machine-learning/crash-course/neural-networks/activation-functions)

