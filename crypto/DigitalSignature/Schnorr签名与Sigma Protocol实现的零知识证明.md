之前在https://github.com/AlexiaChen/AlexiaChen.github.io/issues/120 这里写过Chaum-Pedersen 零知识证明方案，所以提到过Sigma Protocol 中的Equality of discrete logs 关系证明。我们来看看Schnorr签名与它有什么关系，其实Schnorr签名可以看作是proof of knowledge of secret key corresponding to a public key ( x in g^x)，而被签名的消息m增加作为hash函数的输入，以此来生成challenge。而且它们的原型就是交互式的，用hash函数生成了challenge而已，所以Schnorr签名我感觉也是一种零知识证明。

先来看Schnorr签名，再看Sigma Protocol中的proof of knowledge of discrete log吧。

## Schnorr Signature

设椭圆曲线G点为G， pk = sk.G或者也可以表述成pk = G^sk。在非对称加密中，签名无非的意义就是一段描述与该私钥对应的公钥的一段证明嘛，所以才能是私钥签名，公钥验签。也就是满足pk = G^sk的约束，但是不用暴露sk，因为sk是witness。是Prover向Verifier要证明的东西。

Schnorr签名流程：

设Prover的公钥为P，私钥为k， P = G^k 待签名的消息为m

- Prover生成一个determine nonce的随机数r
- Prover创建一个关于随机数r的公钥R， R = G^r
- Prover用Hash函数生成一个随机challenge c = Hash(R || P || m)
- Prover生成签名数据(R,s), 其中s = r +  k*c， 并把签名数据发送给Verifier。签名数据是一个二元组。
- Verifier因为已知G，P， R，s， c，可以通过等式 G^s = R + P^c。  Verifier是知道c的，因为可以它知道R P m，可以通过Hash(R || P || m)算出来。


注: **Hash函数的选取要与公私钥在同一个范围，比如公私钥是一段256 bit的无符号大整数，那么Hash函数也要选输出为256 bit的函数，如SHA256。  而且Hash函数有要求，一定要选择密码学Hash。密码学Hash在一定程度下才能充当随机预言机(Random Oracle), 这样才可以在随机预言机模型下讨论协议的安全性。**  但是在去年有篇论文[《Does Fiat-Shamir Require a Cryptographic Hash Function?》](https://eprint.iacr.org/2020/915.pdf) 指出，Fiat-Shamir变换可以不用密码学Hash，这就比较牛逼了，小白的我还是保持观望状态。

整个过程没有暴露私钥k，还有随机数r，并且整个过程是非交互式的，感谢Fiat-Shamir变换。 若不引入随机数r，那么就会有问题，相当于泄露私钥k了。所以每次签名的时候，随机数r都需要重新生成，随之的R也不一样。所以，对于相同消息m，每次Schnorr签名后的的数据肯定不一样。 随机数r很重要，切记。如果不明白，网络上有详细推导。

## proof of knowledge of discrete log(Schnorr identification protocol)

这个零知识证明的关系约束是 y = g^x。  其中g是生成元，x是witness不能暴露，y对应到Schnorr里面就是公钥。

- Prover随机生成一个随机数r，并创建一个承诺(commitment), t = g^r
- Prover用Hash函数生成一个随机challenge c = Hash(g || y || t)
- Prover生成一个回复 s = r + x * c, 并把二元组(t, s)发送给Verifier
- Verifier根据收到的t，s。还有一开始最知道的y和g来通过Hash函数计算c，最后验证等式 g^s = y^c * t

## 总结

你会发现，Schnorr签名其实就相当于proof of knowledge of discrete log的简单修改，只有最后一步校验的时候校验等式不一样，等式不一样的原因其实就是Schnorr签名的Hash函数的输入不一样，Schnorr没有把G点算进去，而是把消息m作为输入。而
proof of knowledge of discrete log则是把g算进去了，而没有签名的消息m。所以Schnorr签名是做了修改的proof of knowledge of discrete log的零知识证明。

换句大白话来说吧， proof of knowledge of discrete log中在不泄露x的情况下，需要零知识地证明y就是x通过g^x得到的这样的关系上的证明。而Schnorr签名则是要零知识地证明签名消息m确实是经过私钥x签出来的，所以Schnorr在关系证明y = g^x的约束上引入了外部的一个待签名的数据m的约束,，关系约束性更强了。 