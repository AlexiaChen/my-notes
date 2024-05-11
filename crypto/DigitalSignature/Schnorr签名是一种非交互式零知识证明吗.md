
是的，实际上，Schnorr的签名方案最初被描述为一种非交互式协议。我认为关于交互性的混淆源于同一篇论文首次描述了一个交互式身份验证方案，可以看作是针对空消息的签名方案的特殊化。

在这两种方案中，挑战(challenges)可以通过交互方式或使用Hash函数生成。非交互式变体可以被视为Fiat-Shamir变换的应用，尽管Schnorr并没有将其描述为这样。

请注意，虽然更一般形式的Fiat-Shamir变换涉及随机预言机(Random Oracle)假设，但特别是Schnorr签名方案具有较弱要求



> 注意，但其他一些数字签名，比如RSA就不依赖零知识证明。

在《Real World Cryptography》一书中，写到:

> The best way to understand how signatures work in cryptography is to understand where they come from. For this reason, let’s take a moment to briefly introduce ZKPs and then I’ll get back to signatures.

大意就是说，理解签名的工作原理就是要理解签名是如何来的，所以作者打算简要介绍一下零知识证明(ZKP)。但是作者并没有从技术上说明数字签名方案就是来源于ZKP，只是说为了了解数字签名方案的来源，要提一下ZKP，并不是说数字签名依赖/基于ZKP。


虽然数字签名的最初的proposals并不基于ZKP。

数字签名的概念首次在《  [New Directions in Cryptography](https://ee.stanford.edu/%7Ehellman/publications/24.pdf)》中提出，该书提出使用trapdooer permutation生成签名（然而，他们没有给出这样一个permutation的示例）。

第一个实际提出的签名算法是RSA；事实上它是基于一种trapdoor permutation（而不是ZKP）。

但是，现在，许多签名算法确实基于非交互式零知识证明(NIZK)，但这并不是数字签名的起源的方式（即使跳过RSA作为例子，在所有的数字签名算法中也不普遍适用，也就是说，还有很多数字签名算法并不依赖或来自ZKP）。


#### 一个想用零知识证明的现实例子

> 假设Alice写了一本书，并且在这本书中公开了一个公钥，并且后来想要证明她是这本书的作者。她可以用自己的私钥签署任意的对手方的挑战性消息，而其他人可以验证那些已签署的消息。Alice正在证明她知道一个私钥，同时不泄露任何相关信息。

这样算零知识证明吗？其实前面已经说过，不是所有的数字签名都是基于ZKP，但是在这里例子中，即使你用其他的基于ZKP的数字签名，在这个例子里也不是零知识的。

这不是零知识证明。特别地，你以挑战的签名形式泄露了信息。这里的信息就是**Alice拥有的私钥**这个信息，这是Verifier所没有的东西，因此它是一种“学习”，也就是**Verifier学到了Alice拥有私钥这个知识**。

这可能有两个意义。假设我想向你证明我写了这本书，但又不希望你能够说服其他人与那位写书的人互动过（如果我学到了你拥有私钥这个只是，我就可以说服别人了）。通过这个协议，可以证明你与作者互动过，但在真正的零知识证明中无法实现。另一个问题是挑战可能被恶意生成，以提供有意义的签名。

> Your digital signature method is not zero knowledge because Alice just revealed that she knows the private key. Even if she didn't reveal what the private key is.
> 
> A common explanation of zero knowledge is the story of the Ali Baba cave. The paper goes in depth, starting in the "Jealous Reporter" section, to highlight that not only is the secret hidden, but also the knowledge that someone could have the secret is also hidden.
> 
> this has a practical implication. If you know a valuable secret like the key to unlock 100 BTC, you don't want anyone to know you have that key. If people find out, then a malicious actor may put you in a hostage / ransom situation to force you to reveal the key.
> 
> [http://pages.cs.wisc.edu/~mkowalcz/628.pdf](http://pages.cs.wisc.edu/~mkowalcz/628.pdf)

对于这个Alice写书证明自己这个例子，你可能需要看一下Fiat和Shamir的论文《How to prove yourself》。零知识证明实际上比这里所描述的更多，因为Verifier在协议执行过程中**完全不会学到任何东西**。这里的公私钥数字签名的解决方案，足够用于计算安全的身份证明，但并非是零知识的。

零知识证明基本上允许Alice以超出合理怀疑的方式说服某人，她知道某个特定信息（即，对某个问题的答案），而不透露具体是什么信息。

