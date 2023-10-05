在不到1周的时间里，以太坊区块链将与信标链合并，世界计算机将从PoW过渡到PoS。区块链将永远不一样了。

![[Pasted image 20220915095031.png]]

ETH是世界计算机：一个全球共享的工具，存在于由1000多台计算机组成的网络之间，每台计算机都运行着一个本地版本的以太坊虚拟机（EVM）。

从2015年创世至今，以太坊一直与一个名为工作证明(PoW)的系统协调。

在几天后， ETH区块链将与信标链合并，创建新的canonical PoS $ETH。

正如 "合并(The Merge) "这个名字所暗示的，以太坊区块将是旧区块（执行层）和新的信标区块（共识层）的组合。

![[Pasted image 20220915095440.png]]
先决条件 -Merkle Tree

默克尔树：用于组织和加密巨大数据集的数据结构。默克尔证明可用于有效地验证数据集中是否存在的数据（确认一块数据的存在，而无需转移整个数据集）。[Haym 在 Twitter: "(1/13) Computer Science 201: Merkle Trees and Merkle Proofs If you want to understand @Bitcoin, @ethereum and blockchain technology, you need to learn: - How a Merkle trees expresses a large dataset - How a Merkle proof works - Why a Markle tree is so efficient https://t.co/RaXbVaMzKr" / Twitter](https://twitter.com/SalomonCrypto/status/1567638108937801728)


在PoS下，一个区块由三部分组成。

- 管理，区块的元数据
- 共识，协调信标链的层，提供PoS的密码学安全性
- 执行，区块的数据，（几乎）与PoW区块完全一致。

![[Pasted image 20220915095711.png]]


管理部分 - 有关区块的信息

下面的图片显示了所有的管理字段。我们将在下面讨论那些不明显的字段。

`state_root` - 默克尔树的root hash，用于存储信标链的状态（BeaconState）。

![[Pasted image 20220915095910.png]]

![[Pasted image 20220915095918.png]]

`randao_reveal` - 协议验证的随机性，在一个epoch内所有区块提议者(block proposer)之间产生。

随机性对信标链至关重要；安全性取决于能否以不可预测的方式统一选择区块提议者和委员会成员。(我之前做的FnFn区块链就是安全主链上有一个随机信标，就是这个了，是用Publicly Verifiable Secret Sharing产生的)。

`graffiti` - 一个（可选）32字节的字段，区块提议者可以在其中输入任何他们想要的东西。经常被矿池用来记录他们的区块。

`signature` - 区块提议者创建的签名，以承担责任（添加到区块链上，如果好的话，可以获得奖励，如果不好的话，会被砍掉）。通过结合BeaconState、BeaconBlock和提议者的私钥创建。

共识部分--协调和验证区块链共识和实施PoS的必要信息

所附图片显示了所有的共识字段。我们将在下面讨论非显而易见的字段。

![[Pasted image 20220915100518.png]]
![[Pasted image 20220915100528.png]]


`deposit_root` - Merkle树的root hash，该树存储了$ETH的存款到staking 合约中（成为验证者需要）。

`attestations` - 对该区块进行认证的所有验证者(validators)的列表

ETH PoS选出一个提案人，负责建立（或选择）一个区块并将其提交给网络。证实者审查该区块，如果它是有效的，就用他们的密钥签署它。

`attestations` - 这些签名的列表，用这个数据结构表示:

![[Pasted image 20220915100821.png]]

`deposits` - @beaconcha_in将这个字段定义为 "区块提议者在这个区块中包含的validator存款的数量"。有趣的是，我唯一能找到的非0值是在创世块中。

`voluntary_exits` - 从staking合约中提取的金额

`proposer_slashings`和`attester_slashings` -validator对网络采取了敌对行动（例如，提议或证明一个无效的区块）。网络会没收他们押注的$ETH的一部分，并将他们从validators的集合中弹出。

![[Pasted image 20220915101201.png]]

一个同步委员会是一个由512个validator组成的小组，每256个epochs（大约27小时）随机分配一次。这个委员会创建了一个高效的轻客户端所需的签名。

`sync_committee_bits` - 有效表示委员会的参与情况 

`sync_committee_signature` - 同步委员会为该区块/epoch负责而创建的签名。

执行部分--ETH区块的有效载荷(payload)。包括所有交易数据。

所附图片显示了所有的执行字段。

![[Pasted image 20220915101503.png]]

![[Pasted image 20220915101515.png]]

出于许多目的（特别是向后兼容），PoS块的执行payload看起来与PoW块几乎相同。欲了解更多信息，请参见这个主题。[Haym 在 Twitter: "(1/18) @ethereum Fundamentals: PoW Blocks Every ~15 seconds a new $ETH block is born... ever wondered what's inside? A field-by-field guide to the building blocks that make up the blockchain. https://t.co/UGwy3suo6T" / Twitter](https://twitter.com/SalomonCrypto/status/1568233231748841474)

`difficulty` - 设置为0 (PoS没有难度)

`nonce` - 设置为0x000000000000 (PoS不使用nonce)

`mixHash` - 已被重新命名为`prev_randao`，现在存储前一个区块的`randao`值

![[Pasted image 20220915101739.png]]

`sha3Uncles`和`uncles` - 因为PoS不会自然产生uncle块，这些字段将被设置为`0`（或者在sha3Uncles的情况下是空列表的哈希值）。这些字段已经变得无关紧要。

以下是PoS块的全部字段

![[Pasted image 20220915101915.png]]

