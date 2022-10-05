之前其实写了一篇[《深入了解Ed25519》](https://github.com/AlexiaChen/AlexiaChen.github.io/issues/103), 主要介绍了背景及来龙去脉，最后给了用Python实现的算法，没有列出签名公式。公式会更清晰些，这里补一下。

Ed25519的签名公式与Schnorr签名很像，可以理解为在扭曲爱德华曲线上用了Schnorr机制的签名，是Schnorr的变种，也继承了Schnorr的优点，就是相比ECDSA非常方便做门限化多签。

Ed25519就是EdDSA标准的具体实现了，有性能高，更安全等这些优点。经过 NIST 批准的曲线有多条，例如 secp256r1，secp256k1 等。但现有的 ECC 算法中的曲线被指存在后门。但是Ed25519采用的扭曲爱德华曲线参数完全公开，并说明了参数选取的意义，保证曲线中并未内置后门。同时 ED25519 采用 Schnorr 机制作为签名的构建方式，使得签名与验证的性能得到了巨大的提升。

## 具体算法

### 公私钥对的生成

首先公私秘钥长度为b，为256-bit， 生成元为G，Hash函数为SHA512, M是被签名的消息， l是一个大素数， 满足l.G = O , 就是对G点的标量乘法得到的是无穷远点，也就是I是曲线上子群的阶。设私钥为sk， 公钥为pk。

- 随机生成一个长度为256-bit的二进制数据作为私钥sk
- 对sk进行Hash，也就是 Hash(sk)， 产生了一个长度为2b的512-bit的输出数据h。其中低256-bit的数据x用于产生公钥(h_0到h_(b - 1))， 另外的高256-bit的数据为随机数k(h_b 到h_(2b-1))
- 将x_0 x_1 x_2的位设置为0，x_(b-1)的位设置为0， x_(b - 2)的位设置为1
- 最后通过标量乘法计算公钥 pk = x.G

通过上面的步骤看到了吗？与平时接触到的椭圆曲线不一样，平时的椭圆曲线的公私钥就是pk = sk.G。而这里的公私钥生成却对标量做了处理，不是直接使用私钥做为G点标量乘法的标量。

### 签名流程

进行签名时，需要用到上一步生成的私钥sk，公钥pk的步骤中得到的随机数k。

- 计算随机数r = Hash(k || M)      （Schnorr签名中的r是随机生成的，这里的r用了Hash充当随机预言机来生成,也就是通过随机生成的秘钥sk来Hash后的随机数再Hash来生成的）
- 计算随机点 R = G^r
- 计算challenge c, c = Hash(R || pk || M)
- 计算s， s = ( r + c*x ) mod l。
- 生成签名数据(R, s)

### 验签流程

Verifier只需要知道公钥pk， 还有(R, s), 消息M本身就可以验证签名是否正确，公式如下:

```
c =  H(R || pk || M)
G^s = R + c*pk
```

## 总结 

其实你可以发现，Ed25519的签名和验证流程几乎就是Schnorr签名的变种，但是随机数r没有像Schnorr那样采用本地的伪随机数生成器(PRG)生成，而是经过了sk私钥两次Hash的结果生成的，即随机数r已经完全脱离对随机数发生器(PRG)的依赖，避免了因为随机化问题而导致密钥的泄露与安全性问题。Hash函数的输出的随机肯定是与PRG不一样的，这个密码学Hash函数在这里充当了随机预言机，在统计上是随机的。

## References

- https://github.com/jedisct1/libsodium/issues/170
- https://soatok.blog/2022/05/19/guidance-for-choosing-an-elliptic-curve-signature-algorithm-in-2022/
- https://cendyne.dev/posts/2022-03-06-ed25519-signatures.html
- https://rebase.network/posts/700