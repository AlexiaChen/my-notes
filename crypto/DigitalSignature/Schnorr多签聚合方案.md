## 前言

之前写过[Schnorr单个签名的原理和算法](https://github.com/AlexiaChen/AlexiaChen.github.io/issues/123) ，现在写个多签方案，是由于毕竟BTC已经这样做了，为了加快验证签名的速度还有减小签名和公钥的占用Tx的空间。所以还是要讲解下已经落地的应用。当然，现在越来越多的链已经用BLS签名了，这个更好，等放到以后讲解吧。

BTC之前采用的ECDSA的多签方案，这种多签方案不能一次验证签名，需要对应的公钥一次一次的校验，这样签名越多，速度越慢，而且之前也提到过了，ECDSA的门限化比较黑科技，有技巧性，就是麻烦。Schnorr签名就不一样了，对于多签，相当方便做聚合，是由于其签名公式是线性的，这带来了优点。

## 正文

假设x, x1, x2,... x_n为私钥，其对应的公钥为X, X1,X2, ... , X_n （X_i = x_i*G,   G是生成元）
m是被签名消息
H是密码学Hash函数，做随机预言机来生成challenge的。

首先回顾下Schnorr单签名算法:

- Prover(签名者)生成签名(R, s) = (r\*G, r + H(X || R || m)\*x) ，其中r是determine nonce随机数，Prover生成的
- Verifier(验证者)通过等式校验签名  s*G = R + H(X || R || m)*X

好的接下来就是，传统的Schnorr多签方案了:

- 设X为所有参与方公钥X_i相加，因为公钥一般是曲线上点群中的点
- 每个参与多签的签名者随机选择一个自己的随机数r_i， 然后计算R_i = r*G, 把R_i共享给别的参与者
- 设R是所有参与方的R_i的相加，因为R_i也是曲线上点群上的点
- 然后每个参与者计算自己的s_i = r_i + H(X, R, m)*x_i
- 最终聚合的签名就是(R, s), s就是所有参与者s_i相加的结果
- 最后，通过聚合的一个签名和一把聚合公钥，一次校验等式 s*G = R + H(X || R || m)*X,   其中X就是所有公钥的聚合

看上面貌似对Schnorr签名的线性特点有点懵，下面来推导一下:

签名聚合：

(R_1, s_1) = (r_1\*G, H(X || R || m)\*x_1)
(R_2, s_2) = (r_2\*G, H(X || R || m)\*x_2)
...
(R, s) = (r\*G, H(X || R || m)\*x)

验证签名聚合：

s_1\*G = R_1 + H(X || R || m)\*X_1
s_2\*G = R_2 + H(X || R || m)\*X_2
...
s_n\*G = R_n + H(X || R || m)\*X_n

把上面的等式全部叠加起来，因为是标量乘法，所以:

=> (s_1 + s_2 + ... + s_n)\*G = (r_1 + r2 + ... + r_n)\*G + H(X || R || m)\*(X_1 + X_2 + ... + X_n)
=> s\*G = R + H(X || R || m)\*X

这样由于线性关系，就可以叠加起来了，并且校验方无法知道签名是不是聚合签名，无法追踪，因为签名是一个签名了，公钥也是一把公钥，在椭圆曲线上与单个签名一样。

以上看起来太完美了，但是这个方案是不安全的，所以BTC对其做了改进，先来说说为什么不安全吧。

为什么不安全呢？因为正是因为多签参与方的公钥聚合的线性特点导致的。

考虑以下场景。Alice和Bob想一起产生一个2-2的多重签名。Alice有一对公私钥对（x_a，X_a），Bob也有（x_b，X_b）。然而，没有什么能阻止Bob声称他的公钥是X_b' = X_b -  X_a。如果他这么做了，其他人会认为X_a + X_b'是Alice和Bob需要合作才能签约的聚合公钥。不幸的是，这个总和等于X_b，因为 X_b' + X_a = X_b, Bob可以自己就签成这个多签。这称为盗贼密钥攻击，避免这种攻击的一种方法是要求Alice和Bob首先证明他们实际拥有与其所声明的公钥对应的私钥。

这种攻击的详细论文，在这里，叫[On the Security of Two-Round Multi-Signatures](https://eprint.iacr.org/2018/417.pdf)。

基于以上论文，BTC才对Bellare-Neven多签方案作出了改进，并且也发了一篇论文[Simple Schnorr Multi-Signatures with Applications to Bitcoin](https://eprint.iacr.org/2018/068.pdf)。

算法如下:

- 首先，拿到所有多签参与方的公钥，并计算公钥的Hash，L = H(X_1 || X_2 || X_3, ..., || X_n)
- 然后再计算 X = H(L || X_1)\*X_1 + H(L || X_2)\*X_2 + ... +  H(L || X_n)\*X_n
- 每个参与者生成自己的R_i = r_i\*G 并共享出去让其他参与者知道
- 然后就可以计算R = R_1 + R_2 + ... + R_n
- 每个参与者可以计算自己的 s_i = r_i + H(X || R || m)\*H(L, X_i)\*x_i
- 然后最终的签名可以聚合为 (R, s), 其中s是所有s_i相加
- 签证签名，验证等式s\*G = R + H(X || R || m)\*X是否成立

注意，以上的X就不是简单的公钥相加，而是每把公钥有自己对应的Hash绑定，相当于一种承诺(commitment)，这种就防止不可篡改了，不可能发生密钥盗窃攻击。

## 参考

- BlockStream. [Key Aggregation for Schnorr Signatures](https://blockstream.com/2018/01/23/en-musig-key-aggregation-schnorr-signatures/)
