# 椭圆曲线的Koblitz形式

之前我写过一篇关于在有限域GP(p)椭圆曲线的文章， p是素数，所以GF(p)也叫素域。还有一种在GF(2^n)这样的有限域下的曲线。如果大家知道有限域的定义就明白了，这种形式的有限域显然适用更广，是更generalize的表示。因为有限域的定义是GF(p^k)，这里的p就是素数，素数的k次方。所以GF(2^n)显然是有限域，2是素数，这种特有的有限域又叫二元域(binary field)。

这种形式有什么优点呢？  下面有个Quora上的密码学博士[这样回答](https://www.quora.com/What-are-the-advantages-of-elliptic-curve-cryptography-using-binary-fields-over-prime-fields-and-vice-versa):

```txt
Security:
There are no known attacks that greatly weaken either class of curves. However, there is some concern about binary curves because of the recent improvements in attacking discrete logarithms over small-characteristic fields. These attacks don't yet matter for elliptic curves unless you have a pairing. So don't use pairings on small-characteristic fields. But it might be worth going with a prime-field curve for long-term security.

Speed:
Binary curves are smaller and faster in hardware than prime-field ones. This is because they have shorter formulas, and because binary operations have no carries, and because binary squaring is very cheap (it's linear). A recent binary curve implementation paper can be found at https://eprint.iacr.org/2013/131.pdf

.

Prime-field curves are usually faster on general-purpose CPUs, because those CPUs usually have a giant integer multiplier circuit, and they don't usually have a giant binary multiplier circuit. However, as the above paper shows, Intel's PCLMULQDQ gives access to a giant binary multiplier (especially on Haswell) and can move things in favor of binary curves.

The fastest prime-field curves these days are Edwards curves. See, for example, http://www.iacr.org/archive/asiacrypt2008/53500329/53500329.pdf. The downside of Edwards curves is that they're slightly special (they have cofactors divisible by 4). For example, the NIST curves cannot be put in Edwards form.

Either sort of curve can have its speed improved by using fancy shapes (Edwards, Lambda, etc) and, for certain curves, endomorphisms. See, for example, https://eprint.iacr.org/2011/608.pdf

.

Edit: David Wong suggested that I mention Montgomery curves. These are a very simple, fast curve shape that's mainly suitable for Elliptic Curve Diffie Hellman. They're equivalent (isomorphic and isogenous) to Edwards curves, so you can use the Montgomery form of a given curve for ECDH and the Edwards form for other protocols that don't work well in Montgomery form. Compared to Edwards form, Montgomery form is about as fast and much simpler for ECDH, slower for keygen, and both slower and more complicated for other situations. The most popular Montgomery curve is Curve25519

.

Another edit at the same time: the cofactor of 4 can be removed by various means to create a prime-order group, so this is only a small downside. But there still isn't a profitable way to write the NIST curves in Edwards form.

A third edit: The fastest keygen algorithm I know is the "signed all bits set combs" algorithm. It is described in my paper at Cryptology ePrint Archive: Report 2012/309

.

Intellectual property:
Elliptic curves are still somewhat IP-encumbered. For example, the point compression patent is probably expiring this July. It is my impression that binary elliptic curves are more encumbered than prime-field ones. This is because Certicom put a lot of work into them in the '90s, and patented everything. (They patented prime-field stuff too, but more of that was known already.) I could be mistaken about this, though, as I haven't researched it recently.
```

啊，无非就是在硬件CPU上实现更方便，也更快，因为公式“更小”,所以你搜一些论文可以看到在FPGA上实现Koblitz曲线的快速幂算法之类的。当然素域上的椭圆曲线在通用CPU上也可以更快，当然了，目前最快的就是爱德华曲线了，它与蒙特哥马利曲线同构，蒙特哥马利曲线适合做ECDH，爱德华曲线可以做其他协议，两者的优点直接结合了，当然最流行的蒙特哥马利曲线就是Curve25519了，所以才有了x25519和Ed25519。

所以，话说回来，言而总之，这样的形式也叫Koblitz曲线。

Koblitz曲线的代数方程表示为以下,这是定义在GF(2^n)这样的有限域下:

```txt
y^2 + x*y = x^3 + a*x^2 + b, (b≠0)
```

定义下素域下的Koblitz曲线是:

```txt
y^2 = x^3 + a*x + b
```

像secp256k1就是Koblitz曲线，其中的k就是koblitz的意思。

同样，曲线上这些点也构成一个交换群，无穷远点为O。既然是交换群，我们首先定义点的逆元：

设点P的坐标为(x, y), 那么-P的坐标就是(x, -(x+y))。  你可以代入具体数值看看，-P的点也在曲线上。

这个-P的坐标是有推导证明的，在此就不证明了。

既然是交换群，自然也满足点的加法的封闭性, 设两点P(x1,y1), Q(x2,y2),加法结果是点R(x3,y3), 那么加法公式就是:

```txt
ß = (y1-y2)/(x1-x2)
x3 = ß^2 + ß – x1 – x2 – a
y3 = ß*(x1 – x3) – x3 – y1
```


![[Pasted image 20221005204332.png]]


有了加法，自然就能定义2倍标量乘法，2*P = P + P,  自然可以代入P+Q的加法公式，点2P(x2,y2)的公式是:

```txt
ß = (3*(x1)^2 + 2*a*x1 – y1)/(2*y1 + x1)
x2 = ß^2 + ß – 2*x1 – a
y2 = ß*(x1 – x2) – x2 – y1
```

![[Pasted image 20221005204358.png]]

