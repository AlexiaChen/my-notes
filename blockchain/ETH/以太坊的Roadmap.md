在约 24 小时内，以太坊将永远改变。但合并并不是结束，它只是标志着一个新的篇章；一个有更多改进。以太坊正在成为世界计算机！ 将以太坊从 12 提升到 100,000 txns/秒的计划指南。

![[Pasted image 20220915083418.png]]

 2008年，现代金融失败了。当世界在他周围燃烧的时候，中本聪发布了他的白皮书和 
BTC诞生了。在2015年，VitalikButerin实现了中本聪的愿景：去中心化、无信任的计算---ETH
，世界计算机。

与世界计算机的互动是缓慢和昂贵的......目前。这些仍然是早期的，从我们今天的位置到一个明确的路径是 ETH 达到其全部潜力。但在我们向前看之前，我们必须了解我们所处的位置。

![[Pasted image 20220915083741.png]]

要了解 ETH作为世界计算机，你必须把以太坊虚拟机（EVM）理解为一个状态机。状态机是一个监测系统状态（state）的系统；状态被交易（对系统状态的原子性改变）所改变。

![[区块链状态系统]]


在大量的元数据下，一个ETH的区块只是一捆EVM交易。每次创建一个新的区块，EVM都会取一个区块，并对区块中的每个交易应用状态转换函数(STF)，以生成其新的状态。

![[ETH的PoS区块]]

为了进行计算（执行交易），世界计算机需要gas，就像普通计算机消耗电力一样。gas的价格是$ETH/GWEI，是一种会计工具，用来表示世界计算机上计算资源的成本。

![[ETH的Gas]]

一般来说，世界计算机有两种类型的计算资源。

- 执行 - 执行交易和改变EVM状态的成本
- 存储--将数据写进 ETH的存储（区块链）的成本

将这两者结合起来，就得到了gas成本。

每个区块都有一个gas limit，代表每个区块间隔期间的可用计算空间的数量。

这个数字很低，使得成本很高，执行很慢......相当于每个区块消耗的gas上限太低，成本很大。

...但这是有目的的! 每个人都应该能够运行一个节点，即使使用廉价的硬件。

ETH是建立在一个单一的核心原则上：去中心化。从去中心化中产生可信的中立性。从可信的中立性带来全球主导地位。去中心化来自于允许任何人运行一个节点。

像整个技术史上的许多次想法一样，答案是专业化。ETH主网将始终保持缓慢和昂贵，优先考虑去中心化和安全。二级网络（L2）扩展解决方案将专注于性能。

扩展解决方案的演变应该被理解为在将结算固定在主网的同时卸载执行(offload execution)的迭代努力。从状态通道（单一用途/目的）开始，到ZK-rollups（持久状态，通用，即时结算）结束。

![[ETH的扩容技术]]

为了这个主题，让我们定义一个在主网上"结算"的txn是什么含义

- 所有的转移(transfer转账)都已经完成（反映在状态中），并且不能被撤销
- 在链上存在一个不可改变的、可恢复的txn的记录
- 在主网上解决了（最终）所有权问题

Rollups（optimistic和ZK）是最有希望的范式，给予ETH扩展性提供支撑。

Rollups是独立的、高性能的（集中式）区块链，通过发布其state root和txn的（压缩）副本，在主网上结算所有txn。

 state root是一个（一个Merkle Root）单行的，用来表示巨大的数据量。rollup operator可以将其区块链的整个state（GBs大小）表示为一个单一的字符串（字节），并定期向主网发布一份副本。

![[Merkle Tree和其证明]]

每次state root被发布到主网时，rollup都在说。"这是这个区块生成时的rollup state(rollup你可以理解成某种state的压缩以及证明)"。

这就像一个永久的保存点。任何人都可以指着这个区块说："在这一刻，我在这个rollup上拥有X"。

这样我们看到rollups的第一个巨大收益的地方：他们可以改变EVM的状态（通过发布他们的state root）而不消耗 ETH (L1) 主网上的计算资源。基于执行的gas成本下降到（接近）零。相当于把大量交易的状态转移函数的执行放到了L2的rollup上。

不幸的是，发布state root是不够的。state root只能让你验证一个txn是否发生，它并不包含实际的txn（相当于state root只是一个证明）。因此，rollups也必须发布rollup的txn数据--尽管是以高度压缩的形式。

![[区块链扩容之Optimistic Rollups]]

然后，这些压缩的txns被捆绑（rolled up）并输入到一个 "input "字段中。这个字段是在一个ETH的txn交易中。input字段用于传递信息给智能合约（然后智能合约使用它来改变EVM的状态）。input字段它不会直接改变EVM的状态。

![[ETH的交易]]

因此，在ETH mainnet上存在一个（压缩的，但可恢复的）发生在rollup上的每个txn的记录。

有了rollup state root和世界计算机上的txn副本（在input字段上的），我们可以实现我们的第三个也是最后一个需求：（最终）所有权在mainnet上得到解决。

Optimistic和ZK-Rollups的运作方式不同，但想法是相似的：无论operator发生什么，你总是可以在主网上使用将永久可用的数据提取(withdraw)你的资产。就这样，在ETH的结算已经完成。

使用这种rollup的范式。可以在今天ETH的基础上进行数量级的扩展。rollups可以随心所欲地中心化，甚至只是一台超级计算机，然而只要他们沉淀(扎根)到世界计算机，就能获得所有的好处。

ETH的最终愿景就是: 一个多链的未来，但这些链将被拴在世界电脑上。

这个未来是建立在以太坊的路线图中的。首先，EIP-4844将彻底改革gas市场；rollups将为他们发布到主网的数据获得一个专门的channel（和gas市场）。

Rollups将不再与arb bots和NFT mints竞争。[Haym 在 Twitter: "(1/25) @ethereum Scalability: The Roadmap to 100k Transactions per Second Over the next 3-5 years, Ethereum will evolve from a primitive blockchain into the backbone of the internet. Your guide to: - The Merge - EIP-4844 (proto-danksharding) - Enshrined PBS - Danksharding" / Twitter](https://twitter.com/SalomonCrypto/status/1559402384526258176)


在EIP-4844之后，世界计算机将收到一个名为Danksharding的升级。这项技术将把区块链分成多个分片，提供更多的区块容量。[Haym 在 Twitter: "(19/25) As the name suggests, proto-danksharding is a step on the road to Danksharding which will unlock huge amounts of @ethereum's scalability. EIP-4844 creates a channel for data, danksharding uses data sampling tech to reduce the resources system uses to validate it." / Twitter](https://twitter.com/SalomonCrypto/status/1559402438372630528)


世界计算机正在朝着以rollup为中心的未来发展。ETH主网将保持对其使命的专注和专业：去中心化，公平访问。随着时间的推移，L1上的txns将发展为几乎完全是rollups和核心的DeFi协议。

 最终，一些协议会意识到，rollup实际上只是一个智能合约，允许你卸载执行(把交易的执行放到链下)，同时仍然保持链上结算/价值。在这一点上，他们推出只是一个时间问题。

![[Pasted image 20220915092025.png]]
