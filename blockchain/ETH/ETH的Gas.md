ETH的gas就是让这个世界计算机运行的燃料。但是Gas到底是什么呢？为什么这么重要呢？

注意 - 在2021年8月5日 ETH 执行了伦敦硬分叉，除其他事项外，还彻底改变了以太坊上gas的工作方式。这条线只提到伦敦硬分叉/EIP-1559后的世界。[EIP 1559 FAQ - HackMD (ethereum.org)](https://notes.ethereum.org/@vbuterin/eip-1559-faq)

ETH是世界计算机：这是一个全球共享的工具，存在于由1000多台计算机组成的网络中，每台计算机都运行着一个本地版本的以太坊虚拟机（EVM）。

任何人都可以在任何时间，以任何理由访问世界计算机。永远。

世界计算机诞生得到现在，运行很慢，虽然我们已经到了一个无限制扩展的时代。但ETH
 主网将永远保持缓慢。快速的系统奖励快速的参与者，而那些有更多机会获得资本和技术的人将永远是最快的。

ETH将永远保持缓慢的速度，让每个人都有公平的机会。也就是说，计算机速度慢的后果是计算资源的稀缺。gas是以太坊用来公平和有效地分配这些资源的机制。

一台（物理）计算机的工作原理是通过数以百万计的晶体管发送电力（有成本），最终汇总到人类的输出。虚拟计算机（如EVM）抽象化了所有这些，但从概念上讲，这个想法是成立的。EVM有计算成本。

触及世界计算机的每项活动都可以归结为EVM可读的机器代码。

该字节码是由每一个具有特定Gas成本的操作组成的。gas衡量执行特定操作所需的计算努力量。

为了实际运行一项交易，你必须向世界电脑提供足够的气体来执行它。

你的钱包 (@MetaMask,@Rabby_io等）可以检查你的pending交易，并给你一个（通常是很好的）估计，执行它需要多少gas。

这个气体被世界计算机所消耗，就像电力被（物理）计算机所消耗一样；它永远消失了。ETH
通过 "燃烧 "gas，或将其发送到一个永久无法恢复的地址来实现这一目的。

但世界计算机并不是靠利他主义运行的。它是一个由独立节点组成的网络，这些节点在本地运行EVM，通过一个被称为 "共识 "的过程保持其副本的同步。

node operator并不是为了感情而参与其中。我们是为了$ETH。[Haym 在 Twitter: "(1/23) Coordination in The World Computer: @ethereum Consensus The Merge is weeks away... are you caught up on Proof of Work (PoW), Proof of Stake (PoS) and the systems that form the backbone of Ethereum? Read up anon, time is running short. The world is about to change forever https://t.co/BRtRkwunjL" / Twitter](https://twitter.com/SalomonCrypto/status/1554500348231880704)

小费(tip)激励节点在区块中包含交易；如果没有小费，让区块空着只是为了收集区块奖励，在经济上是可行的。这也允许用户表达紧迫性：需要快速执行的交易可以支付更多的小费。

看一看一个 ETH交易。你可以看到到底发生了什么。

一个交易有一个gas成本，gas有一个价格，用户为gas出价（`maxFeePerGas`），并向节点提供小费（`maxPriorityFee`）。[Haym 在 Twitter: "(1/18) @ethereum Fundamentals: Transactions Sent $ETH? LP'ed into an AMM? Deployed a new contract? Everything you do on the World Computer leaves an on-chain record. Ever wonder what's inside your transactions? A field-by-field guide to the atomic unit of Ethereum computing https://t.co/hg4rK01naY" / Twitter](https://twitter.com/SalomonCrypto/status/1568092433803808770)


gas很复杂，在进行交易时，你需要遵循两个市场：\$ETH（以美元计价）和gas（以\$ETH/GWEI计价）。

但要这样想：你的电脑关心电力成本吗？为什么世界电脑会关心$ETH的成本？

Gas是一个抽象概念，允许我们对计算费用与$ETH的估值有一个明显的价值层。

世界计算机是一个全球共享的、可怕的资源。Gas是我们如何划分EVM的单位，然后我们让市场来分配它。

隐含的问题是：这种稀缺性从何而来？

从技术上讲，每个区块都有一个目标gas limit，它根据网络使用情况（最大2倍）向上或向下调整。甚至从技术上讲，这个gas limit是由网络中的节点的硬件要求来限制的。

ETH是建立在一个单一的核心原则上：去中心化。从去中心化中产生可信的中立性。从可信的中立性中产生了对世界的统治。去中心化来自于允许任何人运行一个节点。

最后，Gas费用保持 ETH的安全。通过要求每个txn的费用，垃圾邮件(spam)的攻击很快就变得不可行了。无限循环或其他计算的浪费很快就会被烧毁。

而更高的Gas费=更多的$ETH燃烧=更高的staking率=更多的经济安全。