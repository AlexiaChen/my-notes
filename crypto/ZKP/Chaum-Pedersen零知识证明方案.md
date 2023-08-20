这就是著名的Chaum-Pedersen零知识证明方案，在我的mpvss-rs里面用到了: 
https://github.com/AlexiaChen/mpvss-rs/blob/9861e7dc71296bffcf07c6001ff2315b96689221/src/dleq.rs#L66

里面设这个证明协议叫DLEQ。

下面来讲解下这个方案是怎么回事，首先这个知识证明把其看做是一个接受一个四元组的函数协议，该参数满足如下约束:

```
DLEQ(g, g^a, g^b, g^ab) => DLEQ(g1,h1, g2,h2) 

h1 = g1^a
h2 = g2^a = (g^b)^a = g^(a*b)

只要以上关系约束能满足，Prover就可以向Verifier证明它知道a。不满足，则验证Proof就是错的。
这个a就是所谓的witness
```

g 一般来说是q素阶群上的生成元。然后Prover， Verifier这两个角色都知道以上这个参数约束可以证明一些东西。

好了，如果满足以上约束，那么就可以通过这个协议来做零知识的证明了，这个协议干了些什么呢？就是Prover想向Verifier证明我知道一个a, 并且a满足以上参数关系的约束，但是不需要把这个a公布出来，Verifier就能在DLEQ上验证Prover的秘密是否属实。协议的流程如下:

- Prover生成一个determine nonce，也就是随机数r ,然后计算承诺a1 = g1^r , a2 = g2^r
- Verifier生成一个随机数挑战c发送给Prover，其中 c ∈ Zq
- Prover拿到c后，计算z = r + a*c mod q， 然后把z发送给verifier
- Verifier拿到z后，检查两个验证公式是否两边相等： g1^z = (h1^c) * a1 和 g2^z = (h2^c) * a2

verifier最后一旦等式验证通过，就说明Prover知道的a确实满足DLEQ的参数约束是真命题。并且Prover并没有向Verifier泄露a。

当然，在 MPVSS中，对其做这个协议做了一个简单的修改，本质都是一样的。这里的a就是mpvss中的alpha，并且challenge c这个c的挑战也是随机数，只是在MPVSS被修改成了用hash函数来得到这个随机数，这样就可以消除第二步Verifier的工作，也就是Prover在第二步中用hash函数来得到这个随机数c，这样前三步的工作都是Prover做了，Prover最后就可以提供一个Proof，然后Verifier在最后一步校验。为什么这样呢？因为这个原生的零知识证明方案是交互式的，第二步Verifier要在线，构造起来不实用，所以可以通过[Fiat-Shamir变换(Fiat-Shamir heuristic)](https://blog.csdn.net/qq_26360165/article/details/92844431) 把交互式变为非交互式的零知识证明，MPVSS就用到了这点。

以下是Chaum-Pedersen Scheme的python实现:

```python
import random
import sys

q=10009

c=random.randint(1,1000)
r=random.randint(1,1000)

if (len(sys.argv)>1):
        r=int(sys.argv[1])

g=3
a=10
b=13
A=pow(g,a,q)
B=pow(g,b, q)
C=pow(g,(a*b),q)

a1=pow(g,r,q)
a2=pow(B,r,q)

z=(r+a*c) % q


print("Prover and Verifier agree of (g,g^a, g^b and g^ab) =(",g,A,B,C,")")
print("\nProver generates secret random number (r)",r)
print("Prover sends y1 (g^r, B^r)=(",a1,a2,")")

print()


print("Verifier sends a challenge (c)=",c)

print("Prover computes z=r+ac (mod q)=",z)

print("\nVerifier now checks these are the same")
print("Verifier checks g^z=",pow(g,z,q))

print("Verifier checks A^c * a1=",(A**c * a1) % q)

print("\nVerifier now checks these are the same")
print("Verifier checks B^z=", pow(B,z,q))
print("Verifier checks C^c * a2=",(C**c * a1) % q)
```

看完上面的过程，如果了解的朋友就知道，其实上面这协议，本质就是Sigma Protocol，这种协议用于证明某个值(witness)具有某种关系(约束)，但是又不泄露这个值(witness)。其核心思想就是commitment，challenge，prove这三个步骤， 该协议可以实现各种各样的关系证明。

所以我也猜想在MPVSS中的DLEQ也就是sigma protocol中的Equality of discrete logs的缩写，当时看到也非常费解这么一个缩写。

**注意**: 以上协议流程里面随机数r，每次要不一样，就是每次生成Proof的时候r要不一样，不然相当于就泄露witness了，然后也即意味着Prover可预测challenge c了，因为c是基于hash构造的，它的不可预测性是基于随机数r的。详细推导可见参考资料。

## References:

- [什么是Fait-Shamir变换](https://www.cnblogs.com/zhuowangy2k/p/12246575.html)
- [Fiat-Shamir transform和零知识证明](https://blog.csdn.net/qq_38200023/article/details/79871285)
- [基于Sigma protocol实现的零知识证明protocol集锦](https://blog.csdn.net/mutourend/article/details/106391126)