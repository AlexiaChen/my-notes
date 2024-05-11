
[BrainSTARK, Part 0: Introduction | BrainSTARK (aszepieniec.github.io)](https://aszepieniec.github.io/stark-brainfuck/)
## Part-0  介绍

本教程教读者如何设计一个图灵完备的zk-STARK引擎，包括虚拟机、证明者和验证者。由于Brainfuck具有众所周知且简单的指令集，因此选择它作为目标语言，但是本教程介绍的设计模式适用于任意指令集架构。读者应该能够轻松重新编译这些工具，并生成他们选择的语言的STARK引擎。


### STARKs是什么

STARK是近年来密码学领域最令人兴奋的进展之一。简而言之，STARK引擎将计算分为两个部分：Prover和Verifier。Prover运行原始计算，并额外输出一个称为Proof的加密数据字符串。Verfier读取此Proof，并通过验证它可以以密码学确定性判断计算声明(computational claim)是否正确。这个惊人的卖点在于：与直接重新又运行一遍计算相比，Verfifier可以用更少的资源检查此证据。

- 如果原始计算需要过多的时间或内存，则验证者仍然非常快速。
- 如果原始计算涉及秘密输入，则可以使Proof成为零知识的，也就是零知识证明，即不泄露任何关于它们的信息，并且Verifier仍然能够进行验证。

STARK和STARK引擎之间的区别在于后者配备了虚拟机，并应能够对任意计算进行证明和验证。

![[Pasted image 20240428144834.png]]

> 	所以你可以看到，以太坊的zkSNARK来做L2 还是zkSTARK 做的L2 都是对交易的验证计算用零知识证明进行了压缩，使之在L1上不需要重新执行交易进行验证。只用验证零知识证明就可以了。


> 另外zkSNARK中的S表示Succinct这个单词表示的简洁，其实就是类似相比重新执行一遍计算而言，代价小很多，是简洁的。当然，可以换一种学术点的表达方式，就是Proof的长度和要证明的statement和witness的长度是polylog(n)关系的，这里的n是statement和witness的长度，polylog(n)就是Proof的长度，是以n的对数为变量的多项式关系。[algorithm - What is the meaning of O( polylog(n) )? In particular, how is polylog(n) defined? - Stack Overflow](https://stackoverflow.com/questions/1801135/what-is-the-meaning-of-o-polylogn-in-particular-how-is-polylogn-defined)


### 背景

本教程假设读者对SNARKs有一定的了解，特别是STARKs，并且熟悉与后者相关的数学知识。如果需要更基础的介绍，请参阅[Anatomy of a STARK, Part 0: Introduction | Anatomy of a STARK (aszepieniec.github.io)](https://aszepieniec.github.io/stark-anatomy/) 教程或其中引用的资料。本教程在术语、源代码和功能方面基于[Anatomy of a STARK, Part 0: Introduction | Anatomy of a STARK (aszepieniec.github.io)](https://aszepieniec.github.io/stark-anatomy/) 进行拓展。

### 路线图

第一部分：  [[01-STARK引擎]] 审查了STARK中涉及的基本原则和概念，介绍了整体设计和工作流程，并介绍了将要使用的具体工具。
第二部分： [[02-Brainfuck语言]] 描述了我们正在为之构建STARK引擎的语言。
第三部分： [[03-Brainfuck虚拟机算术化]] 描述了针对Brainfuck的算术化过程，并编译先前开发的STARK引擎工具以构建适用于Brainfuck的STARK引擎。
第四部分：  [[04-下一步]] 总结本教程并提供一些发人深省的观点、想法和下一步建议。
第五部分：  [[05-攻击]] 后来添加。它涵盖了一个攻击和修复方法。

### 支持Python代码

教程文本包含了一些Python代码片段，以说明关键原则。要了解所有内容如何整体运作，请查看[AlexiaChen/stark-brainfuck: Tutorial for designing and impementing a STARK-compatible VM, along with a fully functional Brainfark instruction set architecture, virtual machine, prover, and verifier. (github.com)](https://github.com/AlexiaChen/stark-brainfuck)

这个教程和支持的代码的目的是教育性质的，因此对于性能方面的问题基本上不予考虑。确实，对于一个高效的STARK引擎来说，您可能不希望使用Python，更别提支持Brainfuck了。
### 伴侣

这个教程就是以上代码库的指南



