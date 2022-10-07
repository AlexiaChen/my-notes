![[Pasted image 20221007175138.png]]

Starkware是一家位于以色列的公司，为以太坊生态系统建立第二层扩展解决方案，使用zk-STARKs来实现计算安全，该技术是由联合创始人Eli Ben-Sasson和Michael Riabzev发明的。此外，另一位联合创始人Alessandro Chiesa被认为是第一个在2012年提出zk-SNARKs理论的人。

zk-SNARK是Zero-Knowledge Succinct Non-Interactive Argument of Knowledge的缩写，指的是一种证明架构，在这种架构中，人们可以证明对特定知识（如密匙）的所有权，而无需披露这些知识，也无需与证明者或验证者互动。

zk-STARKs（零知识可扩展透明知识论证）[STARK - Starkware](https://starkware.co/stark/) 是可扩展和透明的加密证明，简单地说，它是一个声称快速和高度可扩展的证明系统，同时依赖于不太繁琐但更安全的加密假设。

![[Pasted image 20221007175238.png]]

上面的图表由101Blockchains.com创建，突出了两个零知识学派之间的一些关键区别。

Starkware正在建立两个主要产品。StarkNet和StarkEx。

##### 什么是StarkNet

StarkNet是一个无权限和去中心化的ZK-Rollup第二层解决方案。像其他的ZK和Optimisitic Rollups一样，StarkNet执行交易并将交易数据分批转发回以太坊主网，由STARK证明来保障。因此，StarkNet受益于以太坊的高安全性、去中心化和可组合性，但由于其与EVM兼容，可以在以太坊上扩展交易。StarkNet Alpha于2021年11月29日在以太坊主网上线 [StarkNet Alpha, Now on Mainnet!. TL;DR | by StarkWare | StarkWare | Medium](https://medium.com/starkware/starknet-alpha-now-on-mainnet-4cf35efd1669) ，尽管它目前只适用于白名单合约。

##### 什么是StarkEx

StarkEx [StarkEx - Starkware](https://starkware.co/starkex/) 是一个第二层的扩展解决方案，是为特定的DApps特别是DeFi交易应用而定制的，如dYdX（最大的永续DEX）、rhino.fi（原DeversiFi）从以太坊上的多个L2的一个钱包中获取DeFi机会）、Sorare（一个全球幻想足球游戏，你可以用官方授权的数字卡玩，每周赚取奖品）和Immutable X（NFT的第二层，承诺游戏、应用和市场的零气体费用、即时交易和扩展性）。  
  
广义上讲，StarkEx是StarkNet的许可版本，意味着它是为协议的使用而定制的，而StarkNet是一个去中心化的zk-rollup，将允许任何人在没有许可的情况下在上面部署他们的智能合约。  
  
StarkEx目前可以在三种数据可用性模式下使用。  
  
DApps可以选择通过使用其标准的zkRollup在链上rollup他们的数据 ，通过使用称为Validium的解决方案在链外处理数据，或者使用这两者的混合体，称为Volition。例如，dYdX依靠zkRollup模式向其交易者提供快速和廉价的交易，而rhino.fi、Immutable X和Sorare使用Validium。  
  
这种可选性是由Starkware产品所使用的编程语言Cairo实现的 [Cairo for Developers - Starkware](https://starkware.co/cairo/) 。此外，共享Prover（SHARP）技术确保一批交易之间的gas成本可以共享，从而使Starkware的gas费用非常低。  
  
根据Starkware网站在撰写本文时的数据，StarkEx在其应用程序中的总价值锁定（TVL）为6.48亿美元，交易结算价值为6,570亿美元，促进了约2.05亿笔交易和6170万个NFT的铸造。  
  
Starkware的网站提供了一个强有力的销售宣传。任何用例，任何规模。  [Use Cases - Starkware](https://starkware.co/use-cases/) 
  
无论你有什么想法，实现它的编程工具已经存在。Cairo，图灵完备的编程语言，可以为任何使用案例快速部署。它有助于提供与L1完全互操作的L2解决方案。

无论项目的规模如何，STARK技术都能为其提供动力。今天的STARK解决方案给你提供了比以前更强的扩展能力；第一代STARK解决方案从根本上扩展了区块链，并通过从一个应用程序中批处理数以千计的交易，并通过一个证明来处理它们，从而降低了gas成本。现在，STARK解决方案也将来自几个不同应用的批次打包到一个证明中。这相当于区块链与他人共享一辆出租车，以提高效率。

#### 文档资料

[STARK - Starkware](https://starkware.co/stark/)  主要是关于密码学，ZKP的技术。

