### 图文解释

这就是ETH扩展达到100k txn每秒的指南，这真正让ETH变成了世界计算机。这是区块链扩容技术的未来。

![[Pasted image 20220917155515.png]]

从状态通道，plasma，optimistic rollup一路探索过来，它们的缺点都说腻了。

optimistic rollup是非常强大的，在提供中心化系统的性能的的同时，还能实现对ETH的结算。

然而，他们的乐观性质需要一个很大的权衡：在结算完成之前必须有一个巨大的挑战窗口(可能7天)。

ZK-Rollups解决了这个问题。

与optimistic rollup类似，首先用户将资产存入ETH主网上的智能合约。然后，ZK-rollup运营商(operator)在rollup链上铸造同等数量的资产，并将其交给存款人。

在ETH主网上，这些资产仍处于托管状态。

![[Pasted image 20220917155859.png]]

也和optimistic rollups一样，ZK-rollups的结算和资产权利的去中心化来自于 ETH。这使得rollup链可以在不牺牲信任度的情况下更加集中化。

从用户的角度来看，执行时间和gas成本比使用ETH主网要便宜得多。

![[Pasted image 20220917160044.png]]

在这里，optimistic和zk-rollup都将交易组捆绑在一起，并将压缩后的版本发布到ETH主网。

optimistic的rollups会天真地接受这些捆绑的交易，它会假设其中的txns是有效的，因为它乐观，是由作恶就会罚款这个经济模型来支撑的乐观心态。


在rollup链本身，你可能永远不会注意到挑战期......但当在链上和链下跨时，它就成为一个主要问题。

现代optimistic rollups需要一个7天的挑战期。即使在Trad-Fi中，7天结算也会被认为是不可行的。

ZK-rollups试图解决最终问题：我们如何能在链上接受批次的那一刻就最终确定它？

答案是。零知识证明（Zero-Knowledge Proofs）。

 ZKP是一类数学证明，它允许一方（证明者）向另一方（验证者）证明一个语句是真实的，同时也确保证明者不给验证者任何验证者没有的信息。

ZKP是密码学研究的最前沿，细节很多。

你需要知道的是：ZKP证明的生成是漫长而艰难的，但证明的验证是快速而简单的。

![[Pasted image 20220917160711.png]]
![[Pasted image 20220917160721.png]]

每隔一段时间，rollup operator将捆绑以前的交易并建立一个ZK证明（链外）。然后，它将向链上的验证合约提交该批次和证明。如果合约接受了证明，那么这批交易就会立即完成。相当于plasma和optimistic rollup都是靠经济模型，靠人性来预防欺诈。而zkRollup是依靠数学证明。

![[Pasted image 20220917160826.png]]

ZK-rollups具有optimistic rollups的所有数据压缩收益，而且更多。

ZK-proof的引入在数学上带来了批次有效的确定性；确认链的加密完整性所需的许多区块数据都可以不使用。

发布一批需要ZKP的验证，并提供即时结算。因此，rollup可以即时处理提款（不需要挑战窗口）。

![[Pasted image 20220917161126.png]]

如果你退一步看一下 ETH扩容的演变。你会看到一个主题：卸载执行，同时将结算锚定在ETH主网上。

从状态通道（单一用途，目的）到ZK-rollups（持久的状态，通用，即时结算）。都是要把交易的执行放到L2网络，不在ETH主网。

### 文字解释

[Zero-Knowledge rollups | ethereum.org](https://ethereum.org/en/developers/docs/scaling/zk-rollups)

