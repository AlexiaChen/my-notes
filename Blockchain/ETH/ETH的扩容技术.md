状态通道 → Plasma → Optimistic Rollups → ZK-Rollups

这个路线将把以太坊从12 txns/秒扩展到100,000 txns/秒......而且成本比您现在支付的还要低。

TPS 从12 到 100000 TPS

![[Pasted image 20220915104244.png]]

BTC是建议无信任计算是可能的。ETH即世界计算机，是其交付。

世界计算机是缓慢的，故意的。这种慢表现在两个方面：执行不力和高gas价。

这给我们带来了一个框架，定义了 ETH的扩容路线：在最终结算到以太坊主网的同时，尽可能保持链外执行交易(链外执行状态转移STF)。

如果交易在以太坊上结算，那么它就获得了以太坊的所有属性。[Haym 在 Twitter: "(1/16) Global Finance Fundamentals: Settlement and the DTCC Who actually owns a stock/bond? What happens when you buy/sell a security? How do we handle the complexity of billions of trades every single day? Looking to understand world trade? This thread is for you!" / Twitter](https://twitter.com/SalomonCrypto/status/1560052032509071360)

状态通道是将执行转移到链外的第一次尝试。channel是两方或多方之间的一次性关系。双方在链上锁定资本，允许他们免费交换IOUs。

![[区块链扩容之状态通道]]

来自 ETH的角度来看，一个状态通道是两个txns（每个参与者）：打开和关闭。这些txns代表了更多发生在链外的计算，但最终会结算到主网。

状态通道提供了扩展性，但在应用上是有限的。

Plasma（链）的开发是为了解决这些问题（的一部分）。

Plasma是独立的区块链，其性能远高于（也更中心化）ETH。. 然而，它们通过将数据发布回主网而锚定在世界计算机上。也就是并没有降低一些成本。

![[区块链扩容技术之Plasma]]

Plasma比状态通道有巨大的改进。

- 可以向还没有选择加入的用户发送资产
- 支持持久状态（即使用户退出系统也会存在）。
- 数据定期发布在链上

但是，Plasma只是解决方案的一半。

完整的解决方案：rollups!

在plasma只发布state root（用于验证是否发生了txn的这样的单一事情）的地方，rollups发布了你完全重建链所需要的一切。

想象一下，整个区块链被挤压(rollup)到ETH的主链上 。

第一类rollup是Optimistic rollup。

Optimistic rollup假设所有发布到主网的txns都是有效的，所以它在链上记录它们。但是，为了以防万一，他们也留下了一个挑战窗口。

![[区块链扩容之Optimistic Rollups]]

该Optimistic Rollup创建了自己的区块链，任何人都可以观察它是否有欺诈行为。一旦发现，他们可以发布欺诈证明(fraund proof)，证明该批次（the batch，这个的batch是把一堆交易批量提交的意思）是无效的，应该被撤销。

其结果是：在挑战期（最长7天）过去之前，没有任何txn被最终确定（就是在ETH主链上的确认）
。本质就是为了提升交易性能和成本，这个拖得有点长了，让用户心焦。迟则生变。哈哈。

这给我们带来了区块链扩展的真正解决方案和未来的 ETH: ZK-Rollups。

像他们的Optimistic rollup这位兄弟一样，ZK-rollups将所有(压缩的)数据发布到主网，但他们也提供了一个零知识证明。

![[区块链扩容之zkRollups]]

ZKP代表了数学上的确定性，即在链上发布的任何信息都是有效的，并且在rollup中实际发生。如果证明成立，那么该交易在rollup上和 ETH上都被确认(final)


ZK-Rollups仍然是区块链技术的边缘；（我相信）今天还没有一个通用/EVM兼容的ZK-Rollup准备用于生产...。

但我们并不遥远，如果你仔细观察，你会发现有一两个测试网。你也会发现今年以来  有三家厂商都发布了zkEVM，支撑了zkRollup的框架。