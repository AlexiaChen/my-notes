![[Pasted image 20221007163716.png]]

欢迎回到Stake N' Eggs节目。上周我们深入研究了处于Optimistic Rollups技术前沿的项目，如Optimism、Arbitrum、Boba Network和Metis，但今天我们探讨的是Rollups技术的另一面，即zk-Rollups。虽然以太坊第一层，也就是所谓的共识层，明确利用链上计算来处理交易和验证区块，但Zk-rollup解决方案在第二层，也就是第二层引入了链外功能。  
  
我们将深入研究的第一个zk-rollup是zkSync [zkSync — Rely on math, not validators | zkSync — Rely on math, not validators](https://zksync.io/) 。 zkSync由总部位于德国的Matter Labs于2019年12月创建，并于2020年6月首次发布。这个第一次迭代能够扩展到每秒300笔交易。在zkSync 2.0的开发过程中，他们发布了第一个版本的zkEVM，实现了以太坊虚拟机（EVM）在Rollups环境中的兼容性。零知识技术与EVM的兼容性可以说是零知识采用的最大障碍，但最近zkSync、Polygon和Scroll宣布了解决方案，据称可以解决这个问题。  

![[Pasted image 20221007164039.png]]
  
在撰写本报告时，根据正上方的图片，zkSync在生态系统中的总价值锁定（TVL）方面排名第六，并且是目前仅次于Metis和Loopring的第三便宜的Rollups解决方案，见下图。

![[Pasted image 20221007164051.png]]

zk-rollups设法比其同行表现得更好的原因之一是，它们在以太坊主链之外处理许多交易，并创建一个SNARK（Succinct Non-Interactive Argument of Knowledge），这是一个加密证明，使用户能够证明它拥有特定的数据而不展开其细节。它提供的有效性证明，被发送到以太坊主网，同时通过使用Merkle Trees保持数据的链外存储。  
  
Merkle Trees只由与智能合约相关的最重要的数据组成，与第1层解决方案相比，访问和要求输出信息的频率要低得多。这为区块链节省了大量的处理能力和时间。因此，由于较少的区块链容量被用于交易验证，gas费用减少，使得第2层解决方案，特别是zk-rollups，成为小型交易的首选解决方案。  
  
使得zk-rollups能够比第1层区块链更快地成功验证交易的技术是Merkle Trees的实现。Merkle Trees是一种重要的数学结构，它允许区块链确保没有人可以在zk-rollup的链上记录中伪造数据。通常，一个Zk-rollup由两个Merkle Trees组成，它们都存储在智能合约中，或者换句话说，在链上。一个树专门用于存储账户，而另一个树则存储所有的余额。zk-rollup产生和使用的任何其他类型的数据都存储在链外。  

##### Optimistic和zkRollups的主要区别

与Optimistic Rollups相比，zkRollups的关键优势在于链上处理的速度。由于zkRollups没有等待时间，在此期间交易的真实性可能会受到质疑，因此它们被发布到底层区块链账本(L1)上的速度比Optimistic Rollups的交易快。

然而，zkRollups所使用的加密有效性证明需要依靠大量的哈希功率来计算。因此，那些链上活动有限的程序可能会发现使用Optimistic Rollups解决方案更有利。

现在我们对底层技术有了基本的了解，也知道了zkRollups和Optimistic Rollups之间的主要区别，让我们来看看本文旨在揭示的真正问题......

##### 为什么是zkSync？

随着其宣布zkEVM，一个可以处理任何以太坊智能合约的Rollups，允许开发人员能够在zkSync上构建，就像他们在以太坊或任何其他EVM兼容协议上一样，Matter Labs坚持认为，它将是第一个将完全EVM兼容的zkRollups推向市场的团队。

在过去两周内，zkSync团队宣布了他们有史以来的第一个路线图，zkSync 2.0（包括他们的zkEVM解决方案的升级）的目标是在10月上线，距离上述宣布还有100天。

观察到的最重要的收获是以下几点。

- 如前所述，最重要的是，zkSync与EVM兼容，网络支持Solidity和Vyper，不需要重新进行安全审核。

- 移植到zkSync应该是毫不费力的，该公司声称99%的工具将立即开箱工作。

- 动态费用 - zkSync在他们的路线图中表示，他们相信开发者只需为他们使用的东西付费。这就是为什么他们包括所谓的 "费用建模系统的重大升级"，以确保以最准确的方式收取费用。


  ##### 发布zkSync 2.0的升级

在EVM兼容性升级后，路线图指出，他们将有一个压力测试期，然后是一个公平启动的入职过程，以允许开发人员访问该协议。

> "目标将是首先推出主网......在没有外部项目的情况下，我们将系统通过一系列真金白银的压力测试，这将有助于我们验证生产系统是否正常工作并达到预期效果。[100 Days to Mainnet. We are proud to announce zkSync 2.0… | by Matter Labs | Matter Labs (matter-labs.io)](https://blog.matter-labs.io/100-days-to-mainnet-6f230893bd73)
> 
> 在压力测试期结束后，路线图进一步解释说，ZkSync "将仔细执行公平启动入职流程。公平启动意味着我们欢迎所有的生态系统项目，我们不会参与挑选优胜者或偏爱者。这里的想法是，我们要确保我们能够明智地处理新的生态系统合作伙伴的入职问题。我们希望我们的每一个生态系统合作伙伴都有一个高质量的体验，并且随着每次入职，我们希望改善我们的系统、流程和支持。在这个阶段，用户访问将保持有限。

路线图的目标是在2022年底前实现完整的2.0主网，并在EOY前进一步宣布zkSync 3.0。

当zkSync与Polygon和Scroll竞争，以获得第一个全功能主网zkEVM协议的能力时，所有这些解决方案都有一件事是Vitalik Buterin的支持，他几周前在巴黎的ETH CC上说，"zk-rollups降低了安全风险，即使rollup的容量非常大"，"zkEVMs将使很多事情在L1和L2更加惊人"。

##### zkSync原生代币？

在写这篇文章的时候，zkSync还没有一个原生代币。然而，该公司在其tokenomics页面 [Tokenomics | zkSync Documentation](https://docs.zksync.io/userdocs/tokenomics/) 上明确表示，将有一个原生的zkSync代币，它将被用于抵押，以及成为zkSync网络中的validator。在主网启动和压力测试期结束后，可能会有一个空投。


文档资料 [zksync/docs at master · matter-labs/zksync (github.com)](https://github.com/matter-labs/zksync/tree/master/docs)