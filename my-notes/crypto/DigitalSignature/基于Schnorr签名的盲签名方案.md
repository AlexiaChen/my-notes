# Schnorr盲签名

## 前言

之前有写过Schnorr签名其实就是一种零知识证明协议，都会一个challenge生成 https://github.com/AlexiaChen/AlexiaChen.github.io/issues/123    也写过盲签名的大概原理，就是让签名者不知道所
签名的明文消息m，也不知道签名后的签名数据是什么。https://github.com/AlexiaChen/AlexiaChen.github.io/issues/132

下面我们就讲解一种基于Schnorr的盲化签名方案。盲化签名本质上就是要对盲化函数和去盲函数进行选择。

## 实现

因为盲签名一般是两方协议，有客户和签名者的概念，我们这里把客户变成User（Client），签名者变成Server，来讨论。

前提条件:

 假设Server有一个私钥$x$，公钥 $X = x * G$. 当然，G就是椭圆曲线上的生成元(基点)。

1. Server生成随机数nonce

Server生成一个随机的秘密值$k$和公有nonce $R = k*G$ 。 R当然就是一个曲线上的随机点了，然后把着两个值保存下来。然后把nonce $R$和公钥$X$发送给User(Client)。

2. Client生成盲化因子

User选择一段消息$m$，这个消息也就是待签名的数据明文。然后生成三个随机标量值$\alpha, \beta, t$, 这些标量将用来盲化(伪装)我们要求Server端签署的消息$m$。  

> 当然这些标量，包括nonce的bit长度都是一样的，最好是uint256表示。R虽然是一个随机点，但是也可以表示成标量。

3. Client盲化这些因子

- 盲化nonce R:  $R' = R + \alpha * G + \beta * X$
- 盲化公钥X: $X' = X + t * G$
- 创建一个challenge c: $c = H(X' || R' || m)$, $H$是一种密码学Hash函数，比如SHA256。
- 然后盲化challenge c: $c' = c + \beta$

4. Server签名

Server通过自己的私钥$x$和随机秘密值$k$生成签名$s = k + c' * x$。然后把签名$s$发给Client。

5. Client去盲签名s

Client利用之前的盲化因子和challenge c去盲s： $s' = s + \alpha + c*t$。  去盲后的签名实际上就是$(R', s')$

6. 验证签名

这里的验证签名，是除了client和server，其他任何人都可以验证，毕竟盲化的公钥$X'$是公开的.验证签名其实就是验证以下公式是否相等:

$$
VerifySignature(R',s', c, X') := s' * G = R' + c * X'
$$

注意，这里的c是直接用别人共享过来值c，不是通过Hash计算出来的，因为验签人也不知道m是什么。

## 各类数值的十六进制的具体代入例子

- nonce R `83814f977c81e0d3bce35a0789c90e804fd3e1cb0e6ad71e797ef2adf9a98567`
- $\alpha$ `4d6df5468daccafe8a8ca4a7ead25907f47ed76f63e098808141ceea97cb4139`
- $\beta$ `086a1d986288a10efee980f1c54393a49eb7282749f6faf8d41b1a7ec5388f47`
- $t$ `ff63ac5c4f8a070eaa7e5ec364579ce81d57c296a94210c9b29baac4d6b55122`
- $R'$ `02cf069dead6d52f261b0548768201b615c770f55b14c29df425d05d06854b4058`
- $X'$ `0263a4be2a134b5bd0599f1d9a8fe2270f74677ca5f657302f2e275fc20b481327`
- $c'$ `1b572e3d1568a340986db8bdf388a97f7c38fb1dce3b7fab9d94f5f01be0c788`
- $s'$ `db6730485a402e1385a22f49b3c0984a132da02f1ce8d8cda829604002e04b33`

## 公式推导

![d316700c2c00f5d676e96c4fef37048](https://user-images.githubusercontent.com/8574915/187062996-86961ff6-4c53-4843-927c-a72e8f9b9c93.png)


## 参考

- https://suredbits.com/schnorr-applications-blind-signatures/

