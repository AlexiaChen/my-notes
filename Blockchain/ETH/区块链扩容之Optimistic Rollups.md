Optimistic Rollup是当今区块链最重要的扩容技术之一。

![[Pasted image 20220917152113.png]]


再次之前的技术是Plasma。Plasma是一个独立的区块链，有自己的虚拟机和状态。

每隔一段时间一次，plasma链将建立状态Merkle树并将Merkle root发布到 ETH mainnet。因此，整个plasma链的（压缩）记录存在于ETH mainnet上。

为了提取资产，一个用户提交该资产的Merkle证明。

虽然plasma提供了对状态通道的改进，但它们有一个共同的（巨大的）弱点：都需要参与所有权。如果所有者不关心一项资产，那么涉及该资产的结果可能是 "无效的"。

但还有一个更大的问题：数据可用性。

plasma链只将状态Merkle root发布到主网；它没有发布生成欺诈证明所需的任何交易或块数据。

Rollups的答案是：把所有的东西都发布到ETH主网（这里的主网代表的是你要扩容的主网）。

用户通过将资产存入在ETH主网上的智能合约，与Optimistic Rollup进行互动。. 然后，Rollup operator(运营商)在rollup链上铸造同等数量的资产，并将其交给存款人。（思想类似plasma）

在ETH主网上，这些资产仍处于托管状态(暂时不可取出)。

![[Pasted image 20220917152755.png]]


因为rollup依赖于 ETH来实现去中心化的产权和结算，所以rollup operator可以更加中心化。

从用户的角度来看，交易的执行时间和gas成本比使用ETH主网要便宜得多。

![[Pasted image 20220917152909.png]]

到此你就知道了，rollup和plasma基本上是一回事。两者都是高性能、中心化的区块链，通过托管智能合约锚定在 ETH上。

Plasma每隔一段时间向ETH主网发布一次Merkle root，向ETH主网写入一个无可指责的记录。

plasma链到此为止；唯一发布到ETH主网的是Merkle root。

Merkle root是一个可以用来证明交易的单个Hash。你只能证明一个txn在Merkle tree中，你不能仅仅从Merkle tree的root部搜索txn。

在正常运行期间，这不是一个问题。每次plasma运营商(operator)发布一个新的Merkle root，他们也会向每个资产所有者发送必要的Merkle分支。

如果用户需要创建欺诈证明，他们必须依靠运营商提供区块数据

但是，如果有一天plasma运营商不这么干了呢？

一个恶意的运营商可以很容易地进行无效的交易，并隐藏必要的数据来创建防欺诈。

如果运营商被要求在主网上提供txn数据，这就不可能了。

rollups是解决plasma的数据可用性问题的方法

将所有数据放在链上，任何人都可以根据自己的意愿在本地处理rollup中的所有业务，允许他们检测欺诈、启动提款或亲自开始生产TXN批次。

rollup operator向合约发布一批交易（包括以前的和新的state root）。

合约检查所提交的上一个state root是否与它的当前state root的副本相匹配，如果是，则将合同的state root切换到所提供的state root。

![[Pasted image 20220917153641.png]]


rollups扩容ETH主要是在两个方面: 

- 首先，通过将计算转移到高性能链上，与执行有关的gas成本大幅减少
- 第二，他们能够以高度压缩的形式发布txns，大大降低了向ETH主网发布数据的gas成本。

![[Pasted image 20220917153906.png]]

但有一个突出的问题：智能合约如何知道新的state root是正确的？

一个Optimistic(乐观的) rollup假设所有的批次都是有效的...... 但它留下了一个挑战窗口。

任何人在链上发现欺诈行为的人都可以发布欺诈证明，证明该批次是无效的，应该被撤销。

因此，rollup假设了良好的行为，但依靠不受信任的行为者的经济激励来维持其完整性。(作恶就被罚款)

要撤销(withdraw)，用户向合约发送消息并启动撤销的txn。这改变了rollup的状态；一个新的批次/state root被发布在链上。

在挑战窗口过后，txn被最终确定，用户可以提取他们的资产。

![[Pasted image 20220917154359.png]]

plasma和rollups都建立在相同的原则上：卸载执行，同时将结算固定(确认)在 ETH主网
. 但是，rollups通过解决数据可用性实现了巨大的飞跃。

最重要的结果是：资产不再需要所有者。

Rollups可以运行一个EVM。

一个与EVM兼容/等价的rollup是理想的：所有ETH的属性只是更便宜和更快。

...但我们可以做得更好，不是吗？如果我们不是 "Optimistic"，而是想立即解决事情呢？

基于零知识的验证是可能的吗？下面就会提到一个终极目标 zk-Rollups。