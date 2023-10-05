## 前言

密码学很多问题都可以归结到计算困难问题，这些问题也是密码学安全的根基。这些根基就是论文里面安全证明的一些前提假设(assumption)。比如，RSA依赖大质因数分解，椭圆曲线生成公私钥对，通过公钥难以计算私钥这个问题一般是依赖离散对数难题( Discrete Logarithm)。

## 正文

以下三个难题都是定义在群上的(半群也可以，但是群上会更有趣)，其中g是生成元:

- 离散对数难题(DL): 对于等式g^x = y，  给定y，难以找到对应的x。但是给定x，求解y很容易。
- Computational Diffie-Hellman problem(CDH): 对于两个等式y1 = g^x1 , y2 = g^x2， 知道y1和y2, 不知道x1和x2，找到y = g^(x1*x2)很困难，即求解y很困难
- Decisional Diffie-Hellman problem(DDH): 对于三个等式y1 = g^a , y2 = g^b, y3 = g^c。 知道y1,y2,y3，不知道a,b,c 判断g^c = g^(a*b)等式是否相等。

感觉上面三个问题，是一个比一个难，因为从等式上看，都隐含了你首先要解决离散对数难题(DL),后面的就迎刃而解了，那就感觉DL问题是最难的，解决最难的，其他就轻松了。感觉困难程度是 DL > CDH和DDH。

那么到底哪个难题更难呢？

好了，我们用Generic Group Model(GGM)下来入手这个问题，给定一个预言机(Oracle)可以高效地解决DL问题，那么你就可以基于这个构造一个Solver来解决CDH问题： y = g^(x1*x2) = y1^x2， 那么就可以通过y2求解出x2，那么经过一次正向运算，就可以轻松得到y，CDH问题解决！如果你轻松解决了CDH，那么你看到了，判断DDH的等式g^c = g^(a*b)是否相等也很容易，因为你可以解决g^(a*b)的问题了。这三个问题层层嵌套递进，所以定义了一个链条。也就是:

- 如果DDH困难，那么CDH一定也困难
- 如果CDH困难，那么DL一定困难

所以DDH假设相比CDH假设是一个更健壮的假设，而CDH假设又比DL更健壮。但是从困难程度来说，就是相反的，DL > CDH > DDH。为什么这顺序是相反的？按理来说越困难越“健壮”啊，因为这里的健壮的定义是这样的，如果给定一个Oracle能解决DDH问题，那么它不一定能解决CDH，能解决CDH问题，那么也不一定能解决DL问题。这里的健壮是，DDH是第一道安全防线一样的存在，攻破了第一道防线(DDH)，还有第二道防线(CDH)，还有第三道防线(DL)。

## FAQ

以上的几个问题真的困难吗？我不知道，首先要定义什么是困难。这里的困难说的也只是临时性的，现阶段的，谁知道以后会怎样，因为现在我们都没有证明P = NP或者P != NP。也难保这困难也是大家理解的真困难。所以由于我们人类的认知，技术有限，所以都是在给定已知的条件（限制和理想条件）下进行分析和证明，比如在Generic Group Model(GGM)下分析，因为很多密码学方案的假设都是在GGM下证明安全的。我们人类的认知有限，所以也只能靠预设的模型框架来理解问题和世界，物理学也是一样的。

GGM到底是什么？

GGM全名是Generic Group Model，我们都知道DL,CDH,DDH都是定义在群上的，这个群大家都了解了，有椭圆曲线上的交换群，还有其他各种特定的群结构，多种多样。GGM中的Generic就是把群的具体结构剥离，类似群的泛型。在GGM中证明安全，那么算法方案可以适配到各种群里面，这是一个高层次的抽象模型。但是要注意，在GGM中证明的结论，不一定适用于具体的群。例如，ECDSA的安全性就是在GGM下证明的，但是没有在椭圆曲线群下证明安全过，不过从实践上来看，目前是安全的，没有被人攻破过。

## 参考

- [What is the relation between Discrete Log, Computational Diffie-Hellman and Decisional Diffie-Hellman?](https://crypto.stackexchange.com/questions/1493/what-is-the-relation-between-discrete-log-computational-diffie-hellman-and-deci)
- [Explanation of the Decision Diffie Hellman problem.](https://crypto.stackexchange.com/questions/5254/explanation-of-the-decision-diffie-hellman-ddh-problem)