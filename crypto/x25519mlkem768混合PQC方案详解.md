
## 引言：量子威胁与密码学演进

随着量子计算技术的快速发展，传统密码学面临着前所未有的挑战。Shor算法能够在量子计算机上高效解决大整数分解和离散对数问题，这直接威胁着我们目前广泛使用的RSA和椭圆曲线密码系统的安全性。

**关键时间点**：
- 1994年：Peter Shor提出量子算法
- 2016年：NIST启动后量子密码标准化项目

其结果，就是四个抗量子密码方案被标准化了。

其中两个数字签名方案：

- 模格(Modules-lattice-based)数字签名标准(ML-DSA), 源自CRYSTALS-Dilithium算法
- 无状态哈希(Stateless-Hash-based)数字签名标准(SLH-DSA), 源自SPHINCS+算法

两个是采用密钥封装机制(KEM)的加密密钥交换方案：

- 模格密钥封装机制(ML-KEM), 源自Kyber算法
- 汉明准循环码(HQC， Hamming Qusai-Cyclic)

> Hamming码是一类线性纠错码，由Richard Hamming在1950年发明, 可以检测和纠正单位错误,基于奇偶校验原理,具有最小汉明距离为3的特性。

> 准循环码是循环码的推广，生成矩阵的每一行都是前一行循环移位的结果，具有良好的代数结构和实现效率，适合硬件实现

> Hamming Quasi-Cyclic的特性,  结合了Hamming码和准循环码的优点, 不做详细解释


在这个背景下，ML-KEM（Module-Lattice-Based Key-Encapsulation Mechanism）作为抗量子密钥封装机制应运而生。然而，从传统密码系统向后量子系统的过渡需要谨慎的策略，混合密钥交换就是其中的重要解决方案。

## ML-KEM核心原理详解

### 什么是ML-KEM？

ML-KEM是基于**模格密码学**的密钥封装机制，它不依赖于传统的大整数分解或离散对数问题，而是基于**模学习带错误问题（Module-LWE）**的困难性。

### 数学基础

**1. 格(Lattic)**

一个格就是一个欧几里得向量空间的一个离散子群，符号记为 $\mathbb{R}^n$,  $\mathbb{R}$为实数集合，n代表维数。即表示该空间中的点所需的坐标数，换句话说，格是一组规则排列**在多维空间中的点**，遵循重复的模式。如果维度为 2 ，格就是一种包含点的2D网格，如图

![[Pasted image 20251113152555.png]]

格有一个原点，比如上图，可以看作是一种“网格中的起点”，允许格的生成。我们还区分了基(basis)，一组允许生成其他点的向量，可以将其视为网格中的“运动规则”。

在这些格中（尤其是在高维度中），存在非常困难的计算问题需要解决（即使使用量子计算机），例如**最短向量问题 （SVP），** 它包括找到最接近基的点。我们还可以引用**最近向量问题 （CVP），** 它旨在找到离给定点最近的点。解决这些问题的困难是由于高维空间中存在大量的可能性（非常大）。因此，SVP 和 CVP 的复杂性是指数级的。

> 欧几里得向量空间就是标准的n维实数空间，也就是我们所熟知的2D平面  3D空间，当然，维数可以更高。群的概念不用多说，无非就是那几个概念。熟悉密码学的都知道。

 2D格的例子
 
  具体例子：
  - 基向量：$\mathbf{b}_1 = (2, 0)$，$\mathbf{b}_2 = (1, 1)$
  - 格点：所有形如 $m(2,0) + n(1,1)$ 的点，其中 $m,n \in \mathbb{Z}$

> 显然格点就是基向量的线性组合。

3D格的例子

类似于晶体结构中的原子排列

它的基生成形式

给定线性无关的基向量 $\mathbf{b}_1, \mathbf{b}_2, \ldots, \mathbf{b}_n \in \mathbb{R}^n$，格定义为：

$$
\mathcal{L} = \sum_{i=1}^{n} c_i \mathbf{b}_i \mid c_i \in \mathbb{Z}
$$
矩阵生成形式

 如果 $\mathbf{B} = [\mathbf{b}_1, \mathbf{b}_2, \ldots, \mathbf{b}_n]$ 是基矩阵，那么：

  $$
  \mathcal{L} =  \mathbf{B} \cdot \mathbf{c} \mid \mathbf{c} \in \mathbb{Z}^n
  $$

格$\mathcal{L}$是离散的，同一个格也可以有不同的基：

  - 好基：基向量"短"且"近似正交"
  - 坏基：基向量"长"且"接近平行"


而ML-KEM中的格是模格: 

$$
\mathcal{L} =  \mathbf{A} \cdot \mathbf{s} + \mathbf{e} \mid \mathbf{s}, \mathbf{e} \in \mathcal{R}^k
$$

  其中：
  - $\mathbf{A}$：公开矩阵
  - $\mathbf{s}$：私钥向量
  - $\mathbf{e}$：小误差向量
  
其安全性原理:

 - 公钥：看起来像随机向量（隐藏了格结构）
  - 私钥：知道格的"好基"，容易解决困难问题
  - 攻击者：只知道"坏基"，难以解决格问题

**2 LWE和MLWE**

##### 什么是LWE（Learning With Errors）？

**LWE（Learning With Errors）**，即"带错误学习"，是一种基于格的困难问题，由Oded Regev在2005年提出。这个问题的核心在于解决带有噪声的线性方程组。

###### 数学定义

给定：
- 素数 $q$
- 随机矩阵 $\mathbf{A} \in \mathbb{Z}_q^{m \times n}$（$m \times n$ 维矩阵，元素是模 $q$ 的整数）
- 秘密向量 $\mathbf{s} \in \mathbb{Z}_q^n$（需要寻找的目标）
- 误差向量 $\mathbf{e} \in \mathbb{Z}_q^m$（小的随机误差）

LWE问题描述：**已知** $\mathbf{A}$ 和 $\mathbf{b} = \mathbf{A} \cdot \mathbf{s} + \mathbf{e}$，**求**秘密向量 $\mathbf{s}$。

###### 关键特点

1. **误差的添加**：
   $$
   \mathbf{b} = \underbrace{\mathbf{A} \cdot \mathbf{s}}_{\text{纯线性部分}} + \underbrace{\mathbf{e}}_{\text{噪声干扰}}
   $$

2. **困难性来源**：
   - 没有误差时：$\mathbf{b} = \mathbf{A} \cdot \mathbf{s}$ 是普通的线性方程组，容易求解
   - 有误差时：$\mathbf{b} = \mathbf{A} \cdot \mathbf{s} + \mathbf{e}$ 变得极其困难

3. **与格问题的等价性**：
   - LWE等价于在特定格上求解最近向量问题（CVP）
   - 误差 $\mathbf{e}$ 使得 $\mathbf{b}$ 不在格上，但很接近某个格点

###### 直观理解

想象一个2D例子：
```
格点 ●     ●     ●     ●
      ╲   ╱ ╲   ╱ ╲   ╱
    b ●─●─●─●─●─●─●  (带有噪声的点)
      ╱   ╲ ╱   ╲ ╱   ╲
     ●     ●     ●     ●
```

- 格点：$\mathbf{A} \cdot \mathbf{s}$ 的所有可能值
- $\mathbf{b}$：加了噪声 $\mathbf{e}$ 后的点，稍微偏离了真正的格点
- **任务**：找到离 $\mathbf{b}$ 最近的格点，从而确定 $\mathbf{s}$

##### 什么是MLWE（Module LWE）？

**MLWE（Module Learning With Errors）**是LWE的变种，它在环上的模（module）中操作，而不是直接在整数向量中操作。

#### 环（Ring）的概念

**环**是一种代数结构，包含一个集合和两种二元运算：加法和乘法。

**重要性质**：
- 加法和乘法满足结合律
- 加法满足交换律，但乘法**不一定**满足交换律
- 存在加法单位元（0）和乘法单位元（1）

**常见例子**：
1. **整数环 $\mathbb{Z}$**：我们熟悉的整数，加法和乘法
2. **多项式环 $\mathbb{R}[x]$**：实系数多项式
3. **多项式商环 $\mathbb{Z}_q[x]/(x^n + 1)$**：ML-KEM中使用的环

#### 模（Module）的概念

**模**是向量空间（线性空间）的推广，但空间中的标量来自环(Ring)而不是域(Field)。

**直观理解**：
- **向量空间**：可以加向量，可以用**数/标量**乘向量
- **模**：可以加"向量"，可以用**环元素**乘"向量"

#### ML-KEM中的多项式商环详解

ML-KEM使用的多项式商环是 $R_q = \mathbb{Z}_q[x]/(x^n + 1)$，这是理解MLWE的关键。

**多项式商环的元素**：**等价类**，每个等价类包含所有在模 $(x^n + 1)$ 意义下相等的多项式。

##### 环元素的具体实例

**以 $q = 3329$, $n = 4$ 为例**：

原始多项式：$f(x) = 2x^5 + 7x^3 + 4x + 1$

**在环 $R_q$ 中的等价多项式**：

1. **模 $x^4 + 1$ 约简**：
   - $x^4 \equiv -1$ （模 $x^4 + 1$）
   - $x^5 = x \cdot x^4 \equiv x \cdot (-1) = -x$

2. **约简过程**：
   $$
   \begin{align}
   f(x) &= 2x^5 + 7x^3 + 4x + 1 \\
   &\equiv 2(-x) + 7x^3 + 4x + 1 \pmod{x^4 + 1} \\
   &= -2x + 7x^3 + 4x + 1 \\
   &= 7x^3 + 2x + 1 \pmod{x^4 + 1}
   \end{align}
   $$

3. **系数模 $q$ 约简**：
   - 所有系数都小于 $q = 3329$，所以无需进一步约简

**最终结果**：在环 $R_q$ 中，元素 $[2x^5 + 7x^3 + 4x + 1] = [7x^3 + 2x + 1]$

##### 环元素的标准表示

**标准形式**：次数 < $n$ 的多项式
$$
[a_0 + a_1x + a_2x^2 + \cdots + a_{n-1}x^{n-1}]
$$

**ML-KEM中的典型元素**：

元素1：$[5 + 13x - 7x^2 + 1023x^3 + \cdots + 27x^{255}]$

元素2：$[100 - 25x + 3000x^2 + 7x^3 + \cdots + 15x^{255}]$

##### 环运算的具体例子

**1. 加法运算**：

设：
- $A = [3 + 2x + x^2]$
- $B = [1 + 4x + 2x^2]$

$A + B = [(3+1) + (2+4)x + (1+2)x^2] = [4 + 6x + 3x^2]$

**2. 乘法运算**（需要模 $x^n + 1$ 约简）：

设：
- $A = [1 + x]$
- $B = [1 + x + x^2]$
- 模：$x^3 + 1$ ($n=3$)

计算过程：
$$
\begin{align}
A \times B &= (1 + x)(1 + x + x^2) \\
&= 1 + 2x + 2x^2 + x^3 \\
\text{约简： } x^3 &\equiv -1 \pmod{x^3 + 1} \\
&= 1 + 2x + 2x^2 - 1 \\
&= 2x + 2x^2
\end{align}
$$

最终：$A \times B = [2x + 2x^2]$

##### ML-KEM中的实际应用

**典型的环元素结构**：

私钥向量 $\mathbf{s} = [s_0, s_1, \ldots, s_{k-1}]$

其中每个 $s_i = [a_{i,0} + a_{i,1}x + \cdots + a_{i,255}x^{255}]$

系数 $a_{i,j} \in \{-1, 0, 1\}$ （小系数多项式）

**误差多项式的例子**：

误差向量 $\mathbf{e} = [e_0, e_1, \ldots, e_{k-1}]$

其中：
- $e_0 = [2 - x + 3x^2 - x^3 + \cdots + x^{255}]$
- $e_1 = [-1 + 2x - x^2 + 0x^3 + \cdots - 2x^{255}]$

##### 为什么选择 $x^n + 1$？

1. **良好的代数性质**：
   - $x^n + 1$ 是不可约多项式（对适当的 $n$）
   - 保证环具有理想的数学性质

2. **高效的运算**：
   - 乘法约简：$x^n \equiv -1$，非常简单
   - 可以使用快速傅里叶变换（FFT）加速

3. **密码学安全性**：
   - $x^n + 1$ 定义的环具有足够复杂的结构
   - 支持构造困难的格问题

#### MLWE的定义

现在理解了多项式商环，MLWE问题可以更清楚地表述：

给定：
- 环 $R_q = \mathbb{Z}_q[x]/(x^n + 1)$
- 随机矩阵 $\mathbf{A} \in R_q^{k \times k}$（元素是多项式的矩阵）
- 秘密向量 $\mathbf{s} \in R_q^k$（多项式向量，每个元素是小系数多项式）
- 误差向量 $\mathbf{e} \in R_q^k$（小系数多项式向量）

MLWE问题：**已知** $\mathbf{A}$ 和 $\mathbf{t} = \mathbf{A} \cdot \mathbf{s} + \mathbf{e}$，**求**秘密向量 $\mathbf{s}$。

#### MLWE vs LWE的优势

| 特性   | LWE      | MLWE     |
| ---- | -------- | -------- |
| 操作对象 | 整数       | 多项式（环元素） |
| 密钥大小 | $O(n^2)$ | $O(n)$   |
| 计算效率 | 较慢       | 更快       |
| 安全性  | 经典       | 同等安全，更高效 |

#### 为什么MLWE更高效？

1. **结构利用**：多项式的代数结构使得可以用快速算法（如FFT）
2. **紧凑性**：一个多项式可以代表多个整数的"压缩"信息
3. **并行性**：多项式运算天然适合并行计算

### 在ML-KEM中的应用

ML-KEM的核心就是基于MLWE问题：

```
密钥生成：
    A (公开矩阵，元素为环中多项式) + s (私钥向量，小系数多项式) + e (误差多项式) → t (公钥向量)

封装：
    A + t → (共享密钥, 密文)

解封装：
    s + 密文 → 共享密钥
```

**安全性保证**：
- 知道私钥 $\mathbf{s}$：容易解封装（因为有格的"好基"）
- 不知道私钥 $\mathbf{s}$：难于从公钥恢复私钥（MLWE困难问题）

### ML-KEM三大算法

#### 1. 密钥生成（KeyGen）  

```python
def ML_KEM_KeyGen():
    # 步骤1：生成随机多项式矩阵 A ($k \times k$维，k为参数)
    A = generate_random_polynomial_matrix(k)

    # 步骤2：生成私钥向量 s (k个小系数多项式)
    s = generate_small_polynomial_vector(k)

    # 步骤3：生成误差向量 e (k个小系数多项式)
    e = generate_small_polynomial_vector(k)

    # 步骤4：计算公钥部分 t = A·s + e mod q
    t = matrix_vector_multiply(A, s, q)
    t = add_vectors(t, e, q)

    # 私钥 = (A, s)，公钥 = (A, t)
    private_key = (A, s)
    public_key = (A, t)

    return private_key, public_key
```

**详细解释**：
- **随机矩阵$\mathbf{A}$**：作为公开参数，所有人都知道
- **私钥$\mathbf{s}$**：小的多项式向量，系数通常在$\{-1, 0, 1\}$范围内
- **误差$\mathbf{e}$**：小的随机误差，用于增加安全性
- **公钥$\mathbf{t}$**：看起来像随机向量，但隐藏了$\mathbf{s}$的信息

#### 2. 封装（Encaps）

```python
def ML_KEM_Encaps(public_key):
    A, t = public_key

    # 步骤1：生成随机消息 m (32字节数据)
    m = generate_random_message()

    # 步骤2：将m编码为多项式
    m_poly = encode_to_polynomial(m)

    # 步骤3：生成随机向量 r, e1, e2
    r = generate_small_polynomial_vector(k)
    e1 = generate_small_polynomial_vector(k)
    e2 = generate_small_polynomial()

    # 步骤4：计算密文组件
    u = transpose_matrix_vector_multiply(A, r, q)  # u = A^T·r
    u = add_vectors(u, e1, q)                      # u = A^T·r + e1

    v = dot_product(t, r, q)                      # v = t^T·r
    v = add_polynomials(v, e2, q)                 # v = t^T·r + e2
    v = add_polynomials(v, m_poly, q)             # v = t^T·r + e2 + m

    # 步骤5：生成共享密钥
    shared_secret = hash_function(m)

    # 步骤6：密文 = (u, v)
    ciphertext = (u, v)

    return shared_secret, ciphertext
```

**详细解释**：
- **消息$m$**：最终将变成共享密钥的随机数据
- **随机向量$\mathbf{r}$**：用于"混淆"操作，使密文看起来随机
- **误差向量$\mathbf{e_1}, \mathbf{e_2}$**：提供额外的安全性
- **$\mathbf{u}$和$\mathbf{v}$**：构成密文的两个部分
#### 3. 解封装（Decaps）

```python
def ML_KEM_Decaps(private_key, ciphertext):
    A, s = private_key
    u, v = ciphertext

    # 步骤1：计算消息的噪声版本
    m_noisy = dot_product(transpose_vector(s), u, q)  # m_noisy = s^T·u
    m_noisy = subtract_polynomials(v, m_noisy, q)     # m_noisy = v - s^T·u

    # 步骤2：解码得到原始消息
    m = decode_from_polynomial(m_noisy)

    # 步骤3：生成共享密钥
    shared_secret = hash_function(m)

    return shared_secret
```

**数学原理**：

因为：$\mathbf{t} = \mathbf{A} \cdot \mathbf{s} + \mathbf{e}$

所以：$\mathbf{t}^T \cdot \mathbf{r} = (\mathbf{A} \cdot \mathbf{s} + \mathbf{e})^T \cdot \mathbf{r} = \mathbf{s}^T \cdot \mathbf{A}^T \cdot \mathbf{r} + \mathbf{e}^T \cdot \mathbf{r}$

在解封装时：

$\mathbf{s}^T \cdot \mathbf{u} = \mathbf{s}^T \cdot (\mathbf{A}^T \cdot \mathbf{r} + \mathbf{e_1}) = \mathbf{s}^T \cdot \mathbf{A}^T \cdot \mathbf{r} + \mathbf{s}^T \cdot \mathbf{e_1}$

因此：$\mathbf{v} - \mathbf{s}^T \cdot \mathbf{u} = (\mathbf{t}^T \cdot \mathbf{r} + \mathbf{e_2} + m) - \mathbf{s}^T \cdot \mathbf{u}$

$= (\mathbf{s}^T \cdot \mathbf{A}^T \cdot \mathbf{r} + \mathbf{e}^T \cdot \mathbf{r} + \mathbf{e_2} + m) - (\mathbf{s}^T \cdot \mathbf{A}^T \cdot \mathbf{r} + \mathbf{s}^T \cdot \mathbf{e_1})$

$= \mathbf{e}^T \cdot \mathbf{r} + \mathbf{e_2} - \mathbf{s}^T \cdot \mathbf{e_1} + m$

由于所有误差项都很小，可以通过解码恢复出$m$

## 混合密钥交换的设计思想 

### 为什么需要混合？

**1. 渐进式迁移**
- 不能一夜之间替换所有加密系统
- 需要向后兼容性
- 减少部署风险

**2. 安全性保证**

安全条件：
- IF 传统算法安全 AND 抗量子算法安全 THEN 系统安全
- IF 传统算法不安全 AND 抗量子算法安全 THEN 系统安全
- IF 传统算法安全 AND 抗量子算法不安全 THEN 系统安全

**3. 性能平衡**
- 传统算法：快速但未来不安全
- 抗量子算法：安全但当前较慢
- 混合方案：平衡安全性和性能

### 混合策略

**1. 串联策略（Series Combination）**
```
Final_Secret = KDF(Traditional_Secret || PQ_Secret)
```

**2. 并行策略（Parallel Combination）**
同时执行传统和抗量子算法，然后组合结果

**3. 嵌套策略（Nested Combination）**
用传统密钥加密抗量子通信的元数据

TLS 1.3采用串联策略，因为它最简单且安全性证明相对直接。

## X25519MLKEM768详细技术步骤

### 完整的TLS 1.3握手流程

#### 阶段1：客户端准备工作

```python
def client_prepare():
    # 1. 生成ML-KEM-768密钥对
    mlkem_priv, mlkem_pub = ML_KEM_KeyGen_768()

    # 2. 生成X25519密钥对
    x25519_priv = generate_random_32_bytes()
    x25519_pub = X25519_scalar_mult(x25519_priv, BASE_POINT)

    # 3. 生成备用X25519密钥对（纯传统方案）
    fallback_x25519_priv = generate_random_32_bytes()
    fallback_x25519_pub = X25519_scalar_mult(fallback_x25519_priv, BASE_POINT)

    return {
        'mlkem_private': mlkem_priv,
        'mlkem_public': mlkem_pub,
        'x25519_private': x25519_priv,
        'x25519_public': x25519_pub,
        'fallback_private': fallback_x25519_priv,
        'fallback_public': fallback_x25519_pub
    }
```

**数据大小分析**：
- ML-KEM-768公钥：1184字节
- X25519公钥：32字节
- 混合公钥：1216字节
- 纯X25519公钥：32字节

#### 阶段2：ClientHello消息构造

```python
def construct_client_hello(client_keys):
    client_hello = {
        'version': 'TLS 1.3',
        'supported_groups': [
            'X25519MLKEM768',  # 优先选择混合方案
            'X25519'           # 备用传统方案
        ],

        'key_shares': {
            # 混合密钥共享：ML-KEM公钥 || X25519公钥
            'X25519MLKEM768': concatenate(
                client_keys['mlkem_public'],    # 1184字节
                client_keys['x25519_public']    # 32字节
            ),

            # 纯X25519备用
            'X25519': client_keys['fallback_public']  # 32字节
        },

        'random': generate_random_32_bytes(),
        'session_id': generate_random_bytes(),
        'cipher_suites': ['TLS_AES_256_GCM_SHA384'],
        'extensions': {...}
    }
    return client_hello
```


#### 阶段3：服务器处理

```python
def server_process_client_hello(client_hello):
    # 1. 检查支持的密钥交换组
    if 'X25519MLKEM768' in client_hello['supported_groups']:
        selected_group = 'X25519MLKEM768'
        use_hybrid = True
    elif 'X25519' in client_hello['supported_groups']:
        selected_group = 'X25519'
        use_hybrid = False
    else:
        raise Exception("No supported key exchange group")

    # 2. 生成服务器端密钥
    server_x25519_priv = generate_random_32_bytes()
    server_x25519_pub = X25519_scalar_mult(server_x25519_priv, BASE_POINT)

    # 3. 提取客户端公钥
    if use_hybrid:
        client_hybrid_share = client_hello['key_shares']['X25519MLKEM768']
        client_mlkem_pub = client_hybrid_share[:1184]    # 前1184字节
        client_x25519_pub = client_hybrid_share[1184:]   # 后32字节
    else:
        client_x25519_pub = client_hello['key_shares']['X25519']

    return {
        'selected_group': selected_group,
        'use_hybrid': use_hybrid,
        'server_x25519_private': server_x25519_priv,
        'server_x25519_public': server_x25519_pub,
        'client_mlkem_public': client_mlkem_pub if use_hybrid else None,
        'client_x25519_public': client_x25519_pub
    }
```

#### 阶段4：服务器计算共享秘密

```python
def server_compute_shared_secrets(server_data):
    if server_data['use_hybrid']:
        # 1. 计算X25519共享秘密
        x25519_shared = X25519_scalar_mult(
            server_data['server_x25519_private'],
            server_data['client_x25519_public']
        )

        # 2. ML-KEM封装
        mlkem_shared, mlkem_ciphertext = ML_KEM_Encaps_768(
            server_data['client_mlkem_public']
        )

        # 3. 组合共享秘密
        combined_secret = concatenate(x25519_shared, mlkem_shared)

        # 4. 生成最终共享密钥
        final_secret = HKDF_Extract(None, combined_secret)

        # 5. 准备服务器Hello的key_share
        server_key_share = concatenate(
            server_data['server_x25519_public'],  # 32字节
            mlkem_ciphertext                      # 1088字节
        )  # 总计1120字节

        return {
            'final_secret': final_secret,
            'server_key_share': server_key_share,
            'x25519_shared': x25519_shared,
            'mlkem_shared': mlkem_shared,
            'mlkem_ciphertext': mlkem_ciphertext
        }
    else:
        # 纯X25519方案
        x25519_shared = X25519_scalar_mult(
            server_data['server_x25519_private'],
            server_data['client_x25519_public']
        )

        final_secret = HKDF_Extract(None, x25519_shared)

        return {
            'final_secret': final_secret,
            'server_key_share': server_data['server_x25519_public']
        }
```

#### 阶段5：ServerHello消息构造

```python
def construct_server_hello(server_result, server_data):
    server_hello = {
        'version': 'TLS 1.3',
        'selected_group': server_data['selected_group'],
        'key_share': server_result['server_key_share'],
        'random': generate_random_32_bytes(),
        'cipher_suite': 'TLS_AES_256_GCM_SHA384',
        'extensions': {
            'supported_versions': ['TLSv1.3'],
            'key_share': server_data['selected_group']
        }
    }
    return server_hello
```

#### 阶段6：客户端完成密钥交换

```python
def client_complete_key_exchange(client_keys, server_hello):
    # 1. 解析服务器key_share
    server_key_share = server_hello['key_share']

    if len(server_key_share) == 1120:  # 混合方案
        # 提取服务器X25519公钥
        server_x25519_pub = server_key_share[:32]

        # 提取ML-KEM密文
        mlkem_ciphertext = server_key_share[32:]

        # 计算X25519共享秘密
        client_x25519_shared = X25519_scalar_mult(
            client_keys['x25519_private'],
            server_x25519_pub
        )

        # ML-KEM解封装
        client_mlkem_shared = ML_KEM_Decaps_768(
            client_keys['mlkem_private'],
            mlkem_ciphertext
        )

        # 组合共享秘密
        combined_secret = concatenate(client_x25519_shared, client_mlkem_shared)

        # 生成最终共享密钥
        final_secret = HKDF_Extract(None, combined_secret)

        return {
            'final_secret': final_secret,
            'client_x25519_shared': client_x25519_shared,
            'client_mlkem_shared': client_mlkem_shared,
            'use_hybrid': True
        }

    else:  # 纯X25519方案
        client_x25519_shared = X25519_scalar_mult(
            client_keys['fallback_private'],
            server_key_share
        )

        final_secret = HKDF_Extract(None, client_x25519_shared)

        return {
            'final_secret': final_secret,
            'client_x25519_shared': client_x25519_shared,
            'use_hybrid': False
        }
```

#### 阶段7：TLS 1.3密钥调度

```python
def tls13_key_schedule(final_secret, transcript_hash):
    # 1. 早期密钥（0-RTT）
    early_secret = HKDF_Extract(None, final_secret)

    # 2. 握手密钥
    handshake_secret = HKDF_Extract(
        Derive_Secret(early_secret, "derived", Hash("")),
        Hash("")
    )

    # 3. 客户端握手流量密钥
    client_handshake_secret = Derive_Secret(
        handshake_secret,
        "c hs traffic",
        transcript_hash
    )

    # 4. 服务器握手流量密钥
    server_handshake_secret = Derive_Secret(
        handshake_secret,
        "s hs traffic",
        transcript_hash
    )

    # 5. 应用密钥
    application_secret = Derive_Secret(
        handshake_secret,
        "c ap traffic",
        transcript_hash
    )

    return {
        'client_handshake_key': HKDF_Expand(client_handshake_secret, "key", 32),
        'server_handshake_key': HKDF_Expand(server_handshake_secret, "key", 32),
        'client_application_key': HKDF_Expand(application_secret, "key", 32),
        'server_application_key': HKDF_Expand(application_secret, "key", 32)
    }
```

### 数据流图

```
客户端                            服务器
 |                                 |
 |  ClientHello (混合公钥1216字节)  |
 |------------------------------>  |
 |                                 |
 |           ServerHello           |
 |          (混合响应1120字节)      |
 | <------------------------------ |
 |                                 |
 | 双方计算相同的final_secret       |
 |                                 |
 | 加密应用数据传输                 |
 |==============================> |
```


## ML-KEM参数集深度解析

### 三个安全级别对比

| 参数 | ML-KEM-512 | ML-KEM-768 | ML-KEM-1024 |
|------|------------|------------|-------------|
| 安全强度 | ~128位 | ~192位 | ~256位 |
| 等效AES | AES-128 | AES-192 | AES-256 |
| 矩阵维度k | $2\times2$ | $3\times3$ | $4\times4$ |
| 公钥大小 | 800字节 | 1184字节 | 1568字节 |
| 私钥大小 | 1632字节 | 2400字节 | 3168字节 |
| 密文大小 | 768字节 | 1088字节 | 1568字节 |

### 为什么叫"768"？

**误解澄清**：
- ❌ 768不是素数域的大小
- ❌ 768不是密钥的字节数
- ❌ 768不是多项式的度数

**正确含义**：
- ✅ 768代表**安全强度级别标识**
- ✅ 对应约**192位量子安全**
- ✅ $768 = 192 \times 4$（4是安全因子）

### NIST安全等级对应

| NIST安全级别 | 对称密钥强度 | 抗量子强度 | ML-KEM参数 |
|-------------|-------------|-----------|-----------|
| Level 1     | 128位       | ~128位    | ML-KEM-512 |
| Level 2     | 192位       | ~160位    | (无对应)   |
| Level 3     | 192位       | ~192位    | ML-KEM-768 |
| Level 4     | 256位       | ~224位    | (无对应)   |
| Level 5     | 256位       | ~256位    | ML-KEM-1024 |

### 参数选择指南

**ML-KEM-512适用场景**：
- 中等安全需求
- 带宽受限环境
- 性能敏感应用

**ML-KEM-768适用场景**：
- 高安全需求（推荐）
- 一般商业应用
- 平衡性能与安全

**ML-KEM-1024适用场景**：
- 最高安全需求
- 政府军事应用
- 长期数据保护

### 结语

ML-KEM与X25519的混合密钥交换代表了密码学从经典向后量子时代过渡的重要里程碑。这种方案在保持当前系统兼容性的同时，为未来量子威胁提供了有效的防护。

随着量子计算技术的发展，这种混合方案将在未来几年内成为安全通信的标准配置。理解并掌握这些技术，对于每一个从事网络安全、系统架构和软件开发的专业人士来说，都是必不可少的。


## 参考资源


- [NIST FIPS 203 - ML-KEM标准](https://csrc.nist.gov/pubs/fips/203/final)
- [IETF TLS混合密钥交换草案](https://datatracker.ietf.org/doc/draft-ietf-tls-ecdhe-mlkem/)
- [CRYSTALS-Kyber原始论文](https://eprint.iacr.org/2017/634)
- [draft-ietf-tls-ecdhe-mlkem-01 - Post-quantum hybrid ECDHE-MLKEM Key Agreement for TLSv1.3](https://datatracker.ietf.org/doc/draft-ietf-tls-ecdhe-mlkem/)
- [draft-ietf-tls-hybrid-design-16 - Hybrid key exchange in TLS 1.3](https://datatracker.ietf.org/doc/draft-ietf-tls-hybrid-design/16/)
- [后量子密码学教程](https://pqcrypto.org/)
- [NIST后量子项目](https://csrc.nist.gov/Projects/Post-Quantum-Cryptography)
- [量子计算威胁分析](https://www.nsa.gov/Research/Quantum-Computing/)
- [TLS 1.3 Hybrid Key Exchange using X25519Kyber768 / ML-KEM](https://www.netmeister.org/blog/tls-hybrid-kex.html)
- [Getting started with post-quantum cryptography: the ML-KEM key exchange | Thibaut Probst](https://thibautprobst.fr/en/posts/ml-kem/)
- [kyber-specification-round3-20210131.pdf](https://pq-crystals.org/kyber/data/kyber-specification-round3-20210131.pdf)

- [OpenSSL 3.x](https://www.openssl.org/)
- [liboqs](https://github.com/open-quantum-safe/liboqs)
- [PQCrypto](https://github.com/microsoft/PQCrypto)


