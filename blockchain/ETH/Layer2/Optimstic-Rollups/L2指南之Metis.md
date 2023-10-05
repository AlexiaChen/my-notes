![[Pasted image 20221007113448.png]]
随着以太坊基金会在合并前不断传出积极的消息，第2层的夏天开始升温。本指南已经探讨了3个Optimistic rollup Optimism、Arbitrum、Boba，但今天的重点是有争议的Optimistic rollup的 "四大 "中的第四个，也是目前最便宜的rollup，Metis。

Metis开发了一个Optimistic rollup解决方案，在第2层对交易进行捆绑和排序，然后将这些数据作为一个交易送回以太坊第1层作为其共识层。通过这种方式，Metis能够提供极快的交易，只需要几秒钟，只需要几分钱，同时仍然保持以太坊第1层的安全性。Optimistic rollup的工作方式是假设，除非受到挑战，否则交易是有效的。如果受到挑战，交易可以被检查，此后通过使用计算进行验证。  
  
Metis是Optimism的一个hard fork，即把Optimism虚拟机（OVM）分叉成一个它称之为Metis虚拟机（MVM）的虚拟机。这意味着Metis团队使用了Optimism的底层代码库，同时实现了与Optimism不同的额外功能和变化。MVM在功能和代码执行方面反映了OVM。交易被发送到sequencer，然后发布。由sequencer发布的交易被认为在主网上直接有效，并从那里被命名为乐观主义(optimistic)。Arbitrum和Optimism用中心化的sequencer处理交易，本质上是作为中心化的运营商，尽管必须说Arbitrum和Optimism都已经口头计划将其sequencer去中心化，作为每个协议路线图的一部分。此外，Optimism和Arbitrum都允许用户运行验证节点(verification)并自由挑战中央sequencer。  
  
然而，与Arbitrum和Optimism不同的是，Metis有多个sequencer，它们将被汇集到称为去中心化自治公司（DACs）的链上实体。每个Metis区块，协议将从sequencer DAC中随机选择一个新的sequencer，将任何状态变化推送到以太坊。Metis的sequencer池和选择过程与Arbitrum和Optimism使用的单定序器方法形成对比。  
  
该网络允许用户作为 "游侠（ranger） "参与协议的监管。rangers分析一系列的区块，并验证state roots，以换取代币奖励。一个成功的挑战可以通过提交欺诈证明来实现，这将导致对sequencer的削减（slash），对sequencer进行惩罚(penalize)，并从sequencer中获取抵押的资产。这些被砍掉（slashed）的资产将被转移到发起争议过程的ranger。ranger反复挑战失败，会导致他们被踢出网络。 

![[LC0234 - Metis - TechnicalWhitepaper.pdf]]
  
挑战过程对于Optimistic rollup的功能至关重要。Optimistic rollup在设计上是 "乐观的"，假设没有发布的交易是欺诈性的。他们基本上使用了一种 "在证明有罪之前是无罪的 "方法，依靠 "检查和平衡 "系统来阻止欺诈，并确保以太坊仍然是真理的仲裁者。Metis上的所有欺诈检测都是通过Rangers监视sequencer的活动和挑战任何不当行为来进行的。  
  
使用Optimistic rollup的主要缺点是确认撤回到以太坊第一层（甚至是另一个第二层）的等待时间很长。这些等待时间对欺诈检测过程至关重要，但它们也可能降低用户的灵活性，并在试图将资金转移到以太坊或不同的L2之间产生摩擦。Metis的多方欺诈检测系统使其能够将状态变化确认的时间从几天缩短到几小时。sequencer在经济上受到激励，以避免发布欺诈性的状态变化（被抓到会导致削减METIS的stake），而ranger因正确报告sequencer欺诈企图而获得经济奖励。  
  
##### Metis有什么不同

虽然Metis使用的是Optimism的OVM的分叉，但它有一些旨在改进原始设计的特点。这些特点包括。

###### 去中心化自治公司(DACs)

Metis DACs的治理结构与DAO不同，因为DAO只专注于投票和治理。相比之下，Metis DACs被设计为完全运作的公司，具有商业相关的任务，如工资、营销和其他，实现所有你在 "现实世界 "公司中看到的基本功能。

Metis DACs还实现了特殊权限的可分配性或角色授权给不同的参与者，允许每个人有不同的影响确定的角色和权限。这种角色授权补充了另一个DAC功能，允许其他DAC参与者评估生态系统中其他成员的角色。

###### IPFS整合

由于区块空间有限，以太坊和比特币上的数据存储成本很高。Metis通过允许每个DAC运行一个单独的MVM实例，其内置的存储层使用星际文件服务（IPFS），从而节约链上空间。用户和开发者可以通过MVM的IPFS解析器访问Metis的内容。

以这种方式存储和访问数据有两个潜在的好处。首先，数据可以在Metis协议内进行加密，允许用户存储机密信息。其次，IPFS解析器使Metis应用程序能够直接访问和链接到存储的数据，如NFT的基础艺术品或音乐。

IPFS中的内容是唯一识别的，而且如果任何内容中的数据发生变化，其CID也会随之变化，这意味着IPFS网络中的数据是不可改变的。这些CID只有几个字节长，可以存储在链上，并在智能合约中使用，以指向存储在IPFS网络中的数据。有了这个功能，就不再需要在链上存储大块的数据了。在链上存储和管理的唯一东西成为相应数据的CID。