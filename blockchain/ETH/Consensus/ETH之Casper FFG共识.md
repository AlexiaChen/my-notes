以太坊共识: Casper FFG

世界计算机通过权益证明（Proof of Stake）进行协调；验证者将ETH代币置于权益中，以参与该系统。但实际上这个系统是什么，它是如何实现共识的？那么我们请进一步来看。

以太坊最终性(确认性)的指南。

![[Pasted image 20221004204632.png]]

以太坊存在于由1000多台电脑组成的网络之间，每台电脑都运行着一个本地版本的以太坊虚拟机（EVM）。

EVM的所有副本都保持完全同步。任何单独的EVM都是世界计算机共享状态的一个窗口。

EVM通过一个称为共识的过程保持同步。共识机制是区块链用来保持其所有节点同步的系统。

直到最近，以太坊一直使用工作证明（PoW）；大约两周前，我们经历了合并，转而使用权益证明（PoS）。

PoW和PoS都是指定一个 "leader "节点（Validator）的系统，并允许他们向前推进（区块高度/编号增长）他们的EVM副本（使用来自像你和我这样的用户提交的交易）。

一旦他们完成了尽可能多的交易，他们就会创建一个区块。

一个区块是在每个节点成为leader期间在EVM内发生的所有交易的完整记录，也就是由leader来出块。

任何其他节点都可以读取该区块，处理交易，并使其EVM的副本与leader的相匹配。

在正常运行期间，每隔~12秒就会选择一个leader，然后他建立并提出一个区块。

然而，网络可能会失去同步性，而且不止一个节点可能会广播一个有效产生的区块，这样就会导致冲突，如果这样，那么该选择哪一个区块呢？

![[Pasted image 20221004210149.png]]

这些选择之一被称为分叉，而分叉可以增长。这样就涉及到分叉选择，你总要选择你自己的一条路。

即使是小的分叉也是对分布式系统的生存威胁。如果没有一个真正的区块链，每个节点上的独立EVM必须决定执行哪些txns。

 这给网络的节点和网络的用户都带来了一个问题。

节点不能确定哪条是通过区块树（分叉了）的真正路径。

用户不能确定他们的txns是否在 "真正的 "路径上；如果他们不是，就不会被计算在内。

友好的最终性Gadget（FFG）Casper是由V神和Virgil Griffith在2019年推出的，以提供区块的确定性。

另外，你们所有人都应该知道Virgil Griffith是谁；他正在为他的ETH的信仰付出代价。[Casper the Friendly Finality Gadget](https://arxiv.org/pdf/1710.09437.pdf) ^9ab86f

 Casper FFG是一个存在于区块提议机制之上的进程；它负责最终确定这些区块，将真正的以太坊
 区块链经典化(cononizing)，其实就是相当于选择一个某个条特定的区块路径，作为以太坊的大家承认的链。

Casper由两个主要部分组成：最终确定算法和惩罚方案。

最终确定算法借鉴了拜占庭将军问题所衍生的研究领域。

该领域寻找 "当没有人能够验证其他成员的身份时，网络成员如何能够就特定的现实达成一致？"的答案。这里请参考 [[拜占庭容错和其实战性PBFT解决方案]]

Casper来自于拜占庭将军问题的一个特殊解决方案，称为实用拜占庭容错（PBFT）。

PBFT是一个两次投票的过程。在两次投票之后，总是可以达成共识（只要大于$2/3$的Validators是诚实的）。
.
在其核心中，PBFT使用投票方案来提供数学上的确定性，即一个分布式网络已经做出了一个集体的、最终的决定。

这就是Casper应用的原则：为以太坊网络提供最终决定的确定性。


.Casper不是最终确定区块，而是最终确定 "检查点"，每N个区块一次（对PoS来说N=32个区块）。

Casper等价于PBFT的投票被称为 "supermajority link"，意味着大于$2/3$的Validators确认了检查点的有效性。

![[Pasted image 20221004210930.png]]

pBFT中的两轮投票被称为prepare和commit；在提交结束时，行动是最终的。

在Casper中，这些想法有不同的名称：如果一个检查点被确认过一次，它就是 "合理的(justified)"。如果它已经被确认了两次，那就是 "最终确定(finalized)"。

每次到达检查点时，每个Validator节点负责评估该区块并创建一个投票。投票只是简单标记源检查点、目标块、有效性证明(proof of balidity)和validator的签名。

![[Pasted image 20221004211304.png]]


.这些投票既是一种特权，也是作为validator的责任。

成功的投票可以获得ETH代币的奖励，但错过(missing)的投票将招致ETH代币的惩罚。

恶意投票将招致更大的惩罚，并将validator从网络中删除。

![[Pasted image 20221004211425.png]]

![[Pasted image 20221004211434.png]]


在PBFT和围绕拜占庭将军问题的其他研究的基础上，我们可以从数学上证明，这些属性完善了一个强大的BFT系统。

这个证明是在Casper的原始论文中[[#^9ab86f]]

 BFT系统有两个特性：safety（一个节点运行的任何txn都会被所有节点运行）和liveness，也就是活性（共识不能停滞）。 ^eab037

Casper FFG不是一个完整的BFT算法（因此称为 "gadget"）。它将保证safety，但liveness取决于提议机制(proposal mechanism)。

然而，Casper确实提供了新的属性。

- 责任性(accountability)，可以检测到违反Casper的行为，并且可以确定违反者的身份
- 防御性(Defenses)，针对远距离修改攻击(long range revision attacks)和大于$1/3$的节点离线的情况
- 动态(Dynamic)，节点可以被添加或删除

Casper FFG是一个从几十年的研究中得出的PoS系统--然而它并不完全完整。

其一，没有区块提议方法。其二，Casper并不指导Validator在需要做出选择时选择哪个检查点。

幸运的是，研究并没有在2019年停止；事实上，它仍然持续到今天，建立在PBFT和Casper的工作之上。

而在不久之后，Casper将变得足够成熟，可以占据中心位置，与LMD-GHOST合并，并向以太坊提供共识  
.