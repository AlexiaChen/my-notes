
### 白皮书

- [波卡Polkadot白皮书 (中文版) - CoderZjz - 博客园 (cnblogs.com)](https://www.cnblogs.com/coderzjz/p/15284943.html)

- https://polkadot.network/Polkadot-lightpaper.pdf

### 波卡

波卡提出来就是为了跨异构区块链的。而且是解决scabblity可伸缩性和隔离性。与ETH 2.0的目标还是不一样。波卡是一套许多独立异构链的集合(ETH.BTC等)，这个集合可以实现安全的跨链交易和pooled安全性。一句话就是，波卡上会有大量的平行链。


![[Pasted image 20220819160544.png]]

上图它展示了收集人收集并且广播用户的交易，也广播候选区块给钓鱼人和验证人。展示了用户提交一个交易，先转移到平行链外部，然后通过中继链再转移到另一个平行链，成为一个可以被那里的账户执行的交易。

### 共识

波卡的共识还是以PoS为基础。共识其实就是中继链上的Vlidators维护的，它会协调各个平行链的共识和状态。一个平行链对应一组Vlidators，因为不同平行链之间出块时间都不一样，那么对应的某组Vlidator出块也会和其所属的平行链出块时间一样短。

Validator组需要提供对应平行链的候选区块，还要保证区块是有效的，不然会损失押金token。相当于Validator组会执行对应平行链区块里面交易的状态变更。这些交易也可以被参与方（抵押token在Validator上的群体）手工下载提交验证区块。  如果Validator提交空块，Validator也会受到奖励惩罚。

Validator和收集人(提供候选区块的角色)在平行链的一个gossip上面工作，收集人把对应平行链的交易收集到区块中，并且提供一个NIZK，用来证明本区块的父区块(前一个区块)是有效的（这个操作，收集人可以收取手续费）。

为了防止垃圾交易大量攻击，波卡把这些预防机制留给了平行链他们自己，因为波卡的主链，也就是中继链上没有交易费这些东西，平行链这些安全机制，它们爱做不做。

波卡的中继链随着以后的发展，将来中继链本身可能还是会加上类似以太坊这样的账户和状态模型，比如虚拟机可能是EVM的衍生版本，因为中继链节点将需要做大量的其他计算，将会通过提高手续费尽量减少交易吞吐量，还有限制块的大小。

**注意**：波卡的平行链跟侧链完全不同，在sidechain里，sidechain是需要将所有的活动和主链同步的，侧链的状态是基于主链来的。而在平行链里，侧链的活动可以不基于主链，侧链应用主链relay chain（波卡的主链）的目的是为了和其他的平行链通讯。

### 跨链通信

这是波卡的核心了，各种异构平行链间可以存在某种信息通道，波卡是可伸缩的多链系统。在波卡中，通信是这样的： 平行链A中执行交易的时候(按照链A的逻辑执行交易)，可以给平行链B或者中继链转发一个交易，这个交易是一个外部交易（链外交易），不受平行链A约束，完全异步发送，它们并没有给它的发送方返回任何信息的原生能力。

现在主要就是XCM这个跨共识系统的消息格式，目前因为XCMP今年目前为止(2022.8)还没有完全出来，所以是用过渡协议HRMP来传递两条平行链之间的跨链消息，通信双方需要打开一个通道，来达成沟通。

今年5月份(2022.5)，首个基于XCM技术的跨链已经MoonBean和Acala之间上线，Moonbeam是波卡社区一条兼容以太坊EVM的一条平行链，把以太坊的生态复用到波卡生态带来了更大的可能性。

**XCM为波卡生态独有的跨链技术，全名为跨共识信息格式，为一种在两条区块链之间传递信息的表达语言**。实际上XCM并不局限于波卡当中，其最终的目标是成为区块链之间通用的沟通语言。简单而言，XCM技术能够让两条区块链进行资产的流动，如同国家与国家之间的贸易一般，但更像是一种整合。


> _新的HRMP通道已在Polkadot上的主要平行链之间开放，允许GLMR、ACA和aUSD Token在链之间本地移动。_


> 基于Polkadot的多链应用开发首选平台Moonbeam，已宣布使用双向HRMP通道与Acala网络建立跨链连接，无需桥梁便可允许两条链之间的本地通信和Token传输。

> 这是自上周XCM发布以来在Polkadot上使用XCM的第一个连接，未来会出现更多的连接遵循基于XCM的连接在Kusama上传播的相同方式。

#### XCM

##### XCM概览

波卡连接起了区块链，结束了网络之间各自孤立的时代。借助跨共识消息格式 [XCM](http://mp.weixin.qq.com/s?__biz=MzI3MzYxNzQ0Ng==&mid=2247495222&idx=1&sn=6879dd119bd4778fe08d6d15367b9734&chksm=eb22297fdc55a069d3ba11843dd77665fc3efcc3ed54ffad0b41083c021454fd5de979660bf5&scene=21#wechat_redirect)，波卡可以实现任何类型数据的安全跨链交换，解锁前所未有的新产品和服务。

XCM 已经上线到波卡网络，允许平行链之间通过开启 HRMP 通道进行通信，波卡的跨链互操作愿景也正式开启。

为了让区块链之间协同工作，形成 Web3 的基础，不同的链之间需要一种通用的通信语言。波卡用 XCM 设定了标准，XCM 是一种强大的格式，用于跨波卡原生区块链 “平行链” 和通过桥与外部网络进行安全消息传递。

**XCM 是波卡的跨共识消息格式，就像是一种跨共识沟通的统一 “语言”。**XCM 不仅支持区块链之间的跨链通信，还支持智能合约、dApp、Substrate pallet（模块）、SPREE 等共识系统之间的通信。通过 XCM 不仅可以传送 token，还可以传递任何类型的信息。

XCM 将释放出异步可组合性的潜力。在单个孤立的区块链上，消息是同步发送和执行的（在同一瞬间）。这适用于许多情况，但不适合需要不受时间限制的功能，例如链上调度。跨链 XCM 消息是异步的，允许开发者跨多个链触发函数，以便随着时间的推移发生变化。

XCM 为互操作时代做好了准备。XCM 是一种用于构建创新的新型跨链应用和服务的语言。赋予应用跨越多个区块链的能力，使它们能够利用每个区块链的特性和优势来实现新的互操作场景。

##### XCM如何运作

这里可以看 ![[XCM格式详解]]



##### XCM解锁的场景

- 以平行链之间的跨链为例子

Statemint 是波卡生态的一条公共利益平行链，它的主要作用是铸造和管理一些 token 和 NFT 资产，是外部资产（如 USDT）进入波卡生态的大本营。自 5 月 4 日开通资产创建功能以来，Statemint 上目前已有 16 种资产。Tether 已经和波卡达成合作，目前 USDT 已经上线到了 Statemint。

那么当平行链和 Statemint 的双向 HRMP 通道开通后，就可以使用 XCM 将 Statemint 上的 USDT 传送到各条平行链中，平行链上的应用就可以使用 USDT 了。USDT 作为目前规模最大的稳定币，和它集成会为波卡生态的应用带来更多的用户，并降低用户的使用门槛。

- Acala和moonbeam

平行链 Acala 是波卡生态中的一站式 DeFi 枢纽，提供稳定币、DOT 流动性质押、AMM DEX 等功能。Acala 上的资产包括 ACA（原生功能性 token）、aUSD（去中心化稳定币）、LDOT（流动性 DOT 衍生品）、lcDOT（Crowdloan DOT 衍生品）等。

平行链 Moonbeam 是波卡生态中兼容以太坊的智能合约平台，开发者可以将 Solidity 合约和 DApp 前端部署到 Moonbeam 上。Moonbeam 的原生 token 为 GLMR。为了更好地兼容以太坊，Moonbeam 提出了 XC-20 标准，让其他平行链的原生资产能够如同 ERC-20 token 一样在基于 Moonbeam 的 DApp 中使用，例如 ACA “存入” 到 Moonbeam 上后的 XC-20 token 就是 xcACA。

在 Acala <-> Moonbeam 双向 HRMP 通道开启后，ACA、GLMR、aUSD 就可以通过 XCM 在两条平行链之间自由地移动和使用。未来，如果经过了双方治理的批准，我们可以想象出很多可能的使用场景：

GLMR 跨到 Acala 上之后，可以作为抵押品铸造稳定币 aUSD，可以和 Acala 上的其他资产组成交易对并通过 Acala AMM DEX 进行 Swap，还可以在 Acala 进行流动性质押等；aUSD 等资产跨到 Moonbeam 上之后，基于 Moonbeam 的 DApp 可以集成 xcaUSD， 用户可以使用 aUSD 作为交易媒介和价值储存手段，以避免受 Crypto 大幅波动的影响。

##### XCM与之前的改进

XCM 是跨链技术的下一个重大飞跃，与之前的桥接解决方案相比提供了多项改进：

- 支持丰富的数据类型

以前的跨链技术主要涉及将 token 从一条链移动到另一条链。XCM 消息可以包含任何类型的数据，不仅可以实现跨链 token 传输，还可以实现功能丰富的跨链应用。这将带来传统网络上无法实现的创新服务。

- 可编程性

非图灵完备。与传统意义上的消息相比，XCM 消息实际上是从一个地方发送并在另一个地方执行的计算机程序。这实现了区块链技术中前所未有的跨链可编程性：使用 XCM，不同的区块链现在可以相互编程。有点像异步的

- 安全且无需信任

在 XCM 之前，区块链只能通过依赖受信任第三方的桥进行通信，从而产生 “最薄弱环节” 问题并导致数次臭名昭著的黑客攻击。在波卡上，平行链之间的消息与整个网络共享相同的高安全性，并且不需要将资金存放在中心化且易受攻击的第三方托管人处。

- 跨共识

XCM 不仅适用于不同的区块链之间，还适用于不同虚拟机上的智能合约之间、Substrate pallet 之间和桥之间。它甚至可以连接建立在不同共识机制上的网络。例如，XCM 可用于在比特币等工作量证明网络和波卡等权益证明网络之间进行通信。

不同于以往区块链网络的各自孤立，XCM 让波卡实现安全的跨链可组合性，这将真正释放出波卡的潜力 —— 不同的平行链可以专注发展自己的长处，再通过 XCM 进行 “链际贸易” ，组合出更强大、更具创新的应用场景。

XCM 也在不断迭代中，目前正在使用的是 V2 版本，在今后会不断根据社区的实际反馈和需求不断迭代和增添新的功能。在上月底的 Polkadot Decoded 波卡全球大会上，Gavin Wood 博士介绍了即将推出的 [XCM V3 版本](http://mp.weixin.qq.com/s?__biz=MzI3MzYxNzQ0Ng==&mid=2247495616&idx=1&sn=0674b0b537128a6ab96df6d2a387fe77&chksm=eb222889dc55a19f89e52b39d08699b27488f2fb992ee4d3fcb97b0cd53363d9e6e3e767cf8b&scene=21#wechat_redirect)，该版本将提升可编程性，支持将应用功能分解并跨链分布到不同的链上，支持跨网络桥接等。


#### Polkadot与Kusama

Kusama也是一个类似Polkadot的网络，代码几乎与Polkadot一样，但是毕竟它是山寨的，所以它一般作为Polkadot网络的测试网，也号称金丝雀网络。所以有些时候你会看到它们两个放在一起，好像同一个概念一样。

据说，现在所有要上波卡的项目基本都要经过Kusama网络的测试，以保证平行链在Polkadot上的运行会更平稳，以及确保整个网络的稳定。

Kusama的原生代币是KSM。

Polkadot真正的测试是Westend [Networks · Polkadot Wiki](https://wiki.polkadot.network/docs/maintain-networks#westend-test-network)



#### Polkadot与Statemint

Statemint是Polkadot网络上的一条平行链。主要是做通用资产的。这个平行链链将提供在Polkadot和Kusama网络中部署资产的功能--从代币化艺术品到稳定币和中央银行数字货币(CBDC)，并为其用户提供比现有解决方案更低的体验。

区块链的一个基本功能是它们能够跟踪一个基础层资产的所有权，即原生协议代币(比如比特币的BTC, 以太坊的ETH，这些都是L1的原生代币)。但区块链能追踪的不仅仅是他们的原生协议代币，所以追踪其他资产也是有意义的，不需要为每个资产启动一个新的网络。如果你了解ERC20，那么就知道Statemint的意义了。

相当于你用DOT代币抵押，你可以在Statemint上发币。

当然资产又分同质化代币(FT)和非同质化代币(NFT)。

FT在Statemint里面是用 Assets pallet 这个Pallet。

这里着重说一下NFT，statemint用 [pallet_uniques - Rust (parity.io)](https://crates.parity.io/pallet_uniques/index.html) crate来表达NFT资产，考虑到这一点，对NFTs最简单的解释是，NFTs是一排排任意的、特定项目的、不可互换的数据，可以通过密码学证明其 "属于 "某人。这种数据可以是任何东西--音乐会门票、入场证、简单的文字、头像、元宇宙中的地块、音频片段、房契、抵押贷款等等。

最广为人知的FT标准就是以太坊的ERC20，NFT标准是ERC721，然后是ERC1155。 ERC1155是它们的升级版，同时支持FT和NFT。

[Statemint · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-statemint)



#### Polkadot与NFT

由于NFT的大火，然后又找出了以太坊NTF的一些缺点。Polkadot搞出来了一个所谓的NTF2.0。

Unique network，一个专门针对NFT的区块链，提供创新，如赞助交易，将可替代的代币(FT)与非可替代的代币(NFT)捆绑在一起，并将NFT拆分为可替代的代币(FT)以获得部分所有权。

RMRK是一个 "hack"，是直接在Kusama中继链之上的强制标准。由于Kusama是为了轻量级地处理与之相连的各种parachains，所以它没有任何其他复杂的链逻辑，如原生NFT或智能合约来实现它们。然而，由于市场需求和Kusama的 "混乱 "性质，RMRK团队决定采取比特币的 "染色比 "方法，并在Kusama链上实现NFTs作为涂鸦。

Moonbeam和Polkadot/Kusama对应物Moonriver是具有Ethereum RPC endpoint的完整EVM部署。也就是RPC与ETH一致。

这意味着Moonriver/Moonbeam的用户和开发者可以使用提供给其他EVM链的整个工具包（如Hardhat、Remix、Truffle、Metamask等技术堆栈），使其在吸引现有用户群方面有一个明显的先发优势。Moonriver是Moonbeam的测试网。这个Moonriver是在kusama这个金丝雀网络上的平行链的。

然而，必须注意的是，Moonbeam是一个EVM链，因此在定制和NFT的功能丰富性方面会受到与任何其他EVM链相同的限制。

Uniques是一个FRAME pallet [substrate/frame/uniques at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/uniques) ，这个pallet部署在Statemint这条平行链上。它实现了最基本的一种NFT--一个引用一些元数据的数据记录。这种元数据引用在冻结之前是可变的，所以NFT和它们的类（派生的实体）是可变的，除非发行者特别说明是不可变(NF)的。

Uniques特意采取了一个非常简单的方法，以保持Statemint链是一个简单的可替代物(FT)和非可替代物(NFT)的平衡保持链。

#### 传送资产(teleporting assets)

Polkadot和Kusama为生态系统带来的主要属性之一是去中心化的区块链互操作性。这种互操作性允许资产远程传输：在平行链（parachains）之间移动资产的过程，如硬币、代币或NFT，以便像使用该链的任何其他资产一样使用它们。互操作性是通过XCM[Cross-Consensus Message Format (XCM) · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-crosschain)和SPREE模块[SPREE · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-spree)实现的，它们共同确保资产不会在多个链上丢失或重复。

##### 传送资产的工作原理

![[Pasted image 20220823170335.png]]

正如你在上图中所看到的，这个模型中只有2个角色：来源和目的地。我们在源地和目的地之间转移资产的方式在图上的数字标签中作了简要总结，并在下面作了更详细的解释。

- 初始化Teleport

源头（发送方）从发送账户中收集要传送的资产，并将其从流通供应(circulating supply)中取出，同时记录取出的资产总量。

- 接收已传送的资产

然后来源方（发送方）创建一个名为ReceiveTeleportedAssets的XCM指令，并将从流通中提取的资产数量和接收账户（接收方的账户）作为该指令的参数。然后，它将这一指令发送到目的地，在那里进行处理，并相应地将新的资产放回流通供应(circulating supply)中。

- 存入资产

然后，目的地（目标平行链）将资产存入资产的接收账户。  
  
上面强调了从流通供应中取出和放回流通供应中这两个短语，首先是为了说明XCM执行者在实现将资产从流通供应中取出和放回流通供应中的语义方面有多大的灵活性。直截了当的答案是烧掉资产以使其退出流通，但可以想象，确实有多种方法可以实现相同的目标，比如将资产转移到本地无法访问的账户，同样，对于将资产放回流通，接收共识系统可以自由选择通过从预先填充的、无法访问的转移资产库中释放资产来实现这种语义，或者对资产进行铸币。  
  
因此，上面也提示了这种模式的缺点--它要求源头和目的地都有高度的相互信任。目的地必须相信源头已经适当地从流通供应中移除被送过来的资产，而源头也必须相信目的地会将从流通中取出的资产重新投入流通。资产传送的结果应该导致资产的相同流通供应。如果不能坚持这两个条件中的任何一个，将导致资产的总发行量发生变化（在可替换代币的情况下），或者完全损失/复制一个NFT。  

在官网可以看到用Polkadot.js打造的App的界面来传送DOT代币的例子，这里是从Polkadot传输到Statemint上 [Teleporting Assets · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-teleport)

### 架构

Polkadot是一个具有共享安全和互操作性的异构多链。

#### 中继链

中继链是Polkadot的中心链。Polkadot的所有验证者都在DOT的中继链上被盯着，并为中继链进行验证。中继链由相对较少的交易类型组成，包括与治理机制的互动方式，parachain拍卖，以及参与NPoS。中继链特意将功能降到最低--例如，不支持智能合约。主要职责是协调整个系统，包括parachain。其他具体的工作被委托给parachain，它们有不同的实现方式和功能。

这个跟ETH 2.0的分片架构很像。主链就是信标链，主要拿来做共识的元数据，子链，也就是其他分片链，跑主要业务。安全性都依赖信标链。其实波卡和ETH 2.0，包括我以前做过的fnfn链，本质上都是分片的不同实现方式。

#### Parachain和ParaThread插槽

Polkadot可以支持许多执行槽。这些插槽就像计算机处理器上的核心（例如，现代笔记本电脑的处理器可能有八个核心）。这些核心中的每一个都可以同时运行一个进程。Polkadot允许这些插槽使用两种订阅模式：parachains和parathreads。Parachains有一个专门的插槽（核心）用于其链，就像一个不断运行的进程。Parathreads在一个组中共享插槽，因此更像是需要被唤醒的进程，运行频率较低。

大多数发生在Polkadot网络中的计算将被委托给特定的parachain或parathread实现，以处理各种用例。Polkadot对parachain可以做的事情没有任何限制，除了它们必须能够生成一个可以被分配给parachain的验证者验证的证明。这个证明验证了parachain的状态转换。一些parachain可能是特定的应用程序，其他的可能专注于特定的功能，如智能合约、隐私或可扩展性--仍然，其他的可能是实验性的架构，不一定是区块链的性质。  
  
Polkadot提供了许多方法来确保一个parachain在特定时间内的插槽。Parathreads是一个共享槽位的池子的一部分，对单个区块进行必赢的拍卖。Parathreads和parachains有相同的API，它们的区别在于经济性。Parachains将不得不在其槽位租赁期内保留DOT；parathreads将按区块付费。Parathreads可以成为parachains，反之亦然。  

#### 共享安全性

连接到Polkadot中继链的parachains都共享中继链的安全。Polkadot在中继链和所有连接的parachains之间有一个共享状态。如果中继链因任何原因必须回退revert，那么所有的parachains也将回退(revert)。这是为了确保整个系统的有效性能够持续下去，没有任何一个单独的部分是可以被破坏的。

共享状态确保在使用Polkadot parachains时，信任假设只有中继链Validators的信任，而没有其他。由于中继链上的Validators被认为是安全的，并有大量的抵押(stake)支持它，因此parachains应该受益于这种安全性。

#### 互操作性

互操作性也就是我们所说的跨链。

##### XCM

XCM是交叉共识信息的简称，是一种格式，而不是一种协议。该格式并不假设任何关于接收方或发送方的共识机制，它只关心消息必须以何种格式来结构化。XCM格式是准信使能够相互通信的方式。与XCMP不同，XCM是跨链消息协议的简称，XCMP是被传递的东西，而XCMP是传递机制。了解更多关于XCM的最好方法是阅读该规范。[paritytech/xcm-format: Polkadot Cross Consensus-system Message format. (github.com)](https://github.com/paritytech/xcm-format)

##### 桥

区块链桥是一种连接，允许任意的数据从一个网络传输到另一个网络。这些链通过桥接器可以互操作，但也可以作为独立的链存在，有不同的协议、规则和治理模式。在Polkadot中，桥连接到中继链，并通过Polkadot的共识机制来保证安全，由整理者维护。

Polkadot使用桥接器来连接Web 3.0的未来，因为桥接器是Polkadot的互操作架构的基础，它充当了孤立的链的安全和强大的通信渠道。

#### 主要角色

##### 验证人(Validators)

验证者，如果被选入验证者集合，则在中继链上产生区块。他们也接受收集者提供的有效状态转换证明。作为回报，他们将获得抵押(staking)奖励。

##### 提名者(Nominators)

提名人将他们的staking质押与特定的验证人结合起来，以帮助验证人进入活跃的验证人集合，从而为链上产生区块。作为回报，提名人通常会得到该验证人的部分赌注奖励。

##### 收集者(Collators)

Collators是parachain和中继链上的全节点。他们收集parachain的交易，并为中继链上的验证者产生状态转换证明。它们也可以使用XCMP发送和接收来自其他parachain的消息。

#### 收集者(Collator)

这里主要讲解收集者更加细节化的东西。

Collator通过从用户那里收集parachain交易并为中继链Validators产生状态转换证明来维护parachain。换句话说，Collator通过将parachain交易汇总成parachain区块候选，并根据这些区块为Validator制作状态转换证明来维护parachain。  
  
Collator为中继链维护一个全节点（Collator运行一个波卡中继链的全节点）同时为Validator所对应的特定的parachain维护一个全节点；这意味着Collator保留了所有必要的信息，能够接收生成新的区块并执行交易中的状态变更，其方式与矿工在当前PoW区块链上的方式基本相同。在正常情况下，他们将整理和执行交易，创建一个未验证和持久化的区块，并将其与状态转换证明一起提供给一个或多个负责提出准区块的Validator。  
  
与验证者不同，Collator节点不保证网络的安全。如果一个parachain块是无效的，它将被Validator拒绝。因此，拥有更多的Collator是更好或更安全的假设并不正确。相反，太多的Collator可能会使网络变慢。Collator唯一邪恶的权力是交易审查。为了防止审查，一个parachain只需要确保存在一些中立的Collators--但不一定是大多数。理论上，审查问题可以通过只有一个诚实的Collator而得到解决。  

##### XCM

Collators是XCM（跨共识消息传递格式）的一个关键要素。作为中继链的全节点，它们都知道平行链双方是对等的。这使得它们有可能将消息从parachain A发送到parachain B。

##### 以一个平行链来作为场景例子

一个新的候选区块的初始化是以区块创建时间开始的。collator在该过程结束时收集所有新的交易。当这样做的时候，collator会签署候选区块并产生状态转换证明，这是一个由候选区块中的交易引起的最终账户余额的总结证明。collator将候选区块和状态转换证明转发给中继链上的Validator。Validator对候选区块内的交易进行验证。在验证之后，如果一切顺利，验证者将与中继链分享候选区块。

平行链的候选区块被收集在一起，并产生一个中继链上的候选区块。

![[Pasted image 20220824093202.png]]

网络上的验证者们将试图就中继链的候选区块达成共识。达成共识后，经过验证的中继链候选区块将与validator和collator共享，相当于collator的中继链全节点就接收到了新的候选区块并持久化写入硬盘，这一过程将重复用于新的平行链的交易。在他们向中继链验证者提出的候选区块被验证之前，collator不能继续在上一个区块的后面接入并构建新区块。

![[Pasted image 20220824093806.png]]

中继链上的区块每6秒产生一个。

##### Collator与Substrate

使用Substrate构建的区块链无法自行挂上中继链。Parity团队建立了Cumulus库来解决这个问题[paritytech/cumulus: Write Parachains on Substrate (github.com)](https://github.com/paritytech/cumulus/)。Collators正在Rococo测试网中使用[Parachain Development · Polkadot Wiki](https://wiki.polkadot.network/docs/build-pdk##testing-a-parachains:-rococo-testnet)，你可以通过Cumulus库了解更多关于它们与Cumulus的使用。更多信息可以在build parachain页面上的Cumulus部分找到。

Rococo是一个专门为测试平行链所构建的一个测试网。

Rococo利用Cumulus和HRMP（Horizontal Relay-routed Message Passing）在parachains和Relay Chain之间发送传输和消息。每条信息都被发送到中继链，然后再从中继链发送到所需的parachain。

Rococo目前运行着四个测试系统的平行链(Statemint Tick, Trick)。以及几个外部开发的parachain。

##### 指南和工具

- Cumulus库和Collators的教程 [paritytech/cumulus: Write Parachains on Substrate (github.com)](https://github.com/paritytech/cumulus/)

- Rococo测试网指南 [Parachain Development · Polkadot Wiki](https://wiki.polkadot.network/docs/build-pdk##testing-a-parachains:-rococo-testnet)
- 一个cli工具，可以根据一些参数，如平行链的数量、collator的设置等，快速启动一个本地的Polkadot测试网。[paritytech/polkadot-launch: Simple CLI tool to launch a local Polkadot test network (github.com)](https://github.com/paritytech/polkadot-launch)

#### 波卡的共识

PoW和PoS这些就不说了，直接直奔主题。

##### Nominated Proof of Stake(NPOS)

在传统的PoS系统中，区块生产的参与取决于代币持有量，而不是计算能力。虽然PoS开发者通常有一个主张以去中心化的方式公平参与，但大多数项目最终都提出了某种程度的中心化操作，其中拥有完全参与权的Validator的数量是有限的。这些Validator通常被认为是最富有的，因此，影响PoS网络，因为他们的赌注最大。通常，拥有必要知识（和设备）来维护网络的候选人数量有限；这也会直接增加运营成本。拥有大量Validator的系统倾向于形成池子，以减少其收入的差异，并从规模经济中获利。这些池子往往是链外的。  
  
缓解这种情况的方法是在链上实现池的形成，并允许代币持有者用他们的股份对代表他们的验证人进行投票。  
  
Polkadot使用NPoS（Nominated Proof-of-Stake）作为其选择验证人组（Validator set）的机制。它被设计成验证人和提名人的角色，以最大限度地提高链的安全性。对维护网络感兴趣的行为者可以运行一个验证者节点。  
  
Validatorr承担着在BABE中生产新区块、验证平行链的区块和保证最终性(finality)的角色(BABE是一个区块生成机制)。提名人可以选择用他们的股份来支持选定的Validator。提名人可以批准他们信任的候选人，并用他们的代币支持他们，让他们成为Validator。

##### 概率性 VS 可证明的最终性(Finality)

一个运行PoW的纯中本共识区块链只能实现概率最终性的概念并达成最终共识。概率最终性是指在关于网络和参与者的一些假设下，如果我们看到有几个区块建立在一个给定的区块上，我们可以估计某个区块它的最终的概率。

最终共识是指在未来的某个时间点，所有节点都会对一组数据的真实性达成共识。这种最终共识可能需要很长的时间，并且将无法提前确定它需要多长时间。然而，最终性gadgets，如GRANDPA（基于GHOST的递归ANcestor衍生前缀协议）或以太坊的Casper FFG（友好的最终性Gadget），旨在对区块的最终性给予更强和更快的保证--具体来说，在发生了一些拜占庭协议的过程后，它们永远不能被逆转。不可逆转的共识的概念被称为可证明的最终性。

其实上面一段说白了就是PoW更容易分叉，导致大量交易回滚， 而PoS和DPoS出块更快，然后基本0概率分叉。最终共识其实就是最终一致性了。

详情请看Grandpa的论文 [consensus/grandpa.pdf at master · w3f/consensus (github.com)](https://github.com/w3f/consensus/blob/master/pdf/grandpa.pdf)

##### 混合共识

当我们谈论Polkadot的共识协议时，有两个协议，GRANDPA和BABE（Blind Assignment for Blockchain Extension）。我们谈论这两种协议是因为Polkadot使用的是所谓的混合共识。混合共识将最终结果的gadget从区块生产机制中分割出来。

这是一种在Polkadot中同时获得概率最终性（总是能够产生新的区块）和可证明最终性（在canonical chain上的一个终极协议，没有机会逆转回滚）的好处的方法。它还避免了每种机制的相应缺点（在概率最终性中有可能不自觉地跟随错误的分叉，也就是硬分叉，在可证明最终性中有可能 "停滞"--无法产生新区块）。通过结合这两种机制，Polkadot允许快速生产区块，而较慢的最终性机制则在一个单独的进程中运行，以最终确定区块，而不会有较慢的交易处理或停滞的风险。

混合共识在过去已经被提出。值得注意的是，在EIP 1011中[EIP-1011: Hybrid Casper FFG (ethereum.org)](https://eips.ethereum.org/EIPS/eip-1011)，它被作为以太坊过渡到PoS的一个步骤提出（现已失效），其中指定了Casper FFG。
  
##### 区块生成:BABE

BABE（Blockchain Extension的Blind Assignment）是在Validators节点之间运行的区块生产机制，决定了新区块的作者。BABE作为一种算法可以与Ouroboros Praos相媲美[573.pdf (iacr.org)](https://eprint.iacr.org/2017/573.pdf)，但在链的选择规则和槽的时间(slot)调整方面有一些关键的区别。BABE根据股权质押并使用Polkadot随机性周期[Randomness · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-randomness)将区块生产槽分配给验证者。

Polkadot中的Validator将参与每个时段(slot)的抽签，这将告诉他们是否是该时段的区块生产者候选人。插槽是不连续的时间单位，名义上是6秒的长度。由于这种随机性机制，多个验证者可能是同一槽位（slot）的候选人。其他时候，一个槽可能是空的，导致区块时间不一致。

###### 每个slot上有多个Validators

当多个验证者在一个给定的时段内成为区块生产者候选人时，所有验证者都会生产一个区块并将其广播到网络上。在这一点上，这是一场竞赛。谁的区块首先到达网络的大部分地区，谁就获胜。根据网络的拓扑结构和延迟，两个链将继续以某种方式建立，直到最终化确定了才终止切断一个无效分叉。请看下面的分叉选择，了解其运作方式。

###### 没有Validator的slot

当没有验证者在随机性抽签中掷出足够低的分数以获得生产区块的资格时，一个slot时间段内可能仍然看起来没有区块。我们通过在后台运行二次Round-Robin式的Validator选择算法来避免这种情况。通过这种算法选择的验证器总是产生区块，但是如果同一个槽也从VRF选择的验证器中产生一个primary区块，那么这些二次产生的区块就会被忽略。因此，一个槽可以有一个primary块，也可以有一个secondary块，而且没有槽被跳过。

关于BABE的更多细节，请参见BABE论文。[BABE — Research at W3F (web3.foundation)](https://research.web3.foundation/en/latest/polkadot/block-production/Babe.html)

##### 最终Gadget: Grandpa

GRANDPA（基于GHOST的递归ANcestor衍生前缀协议）是为Polkadot中继链实现的终结性Gadget。

只要有2/3的节点是诚实的，它就能在半同步的网络模型中工作，并能在异步的环境中应对1/5的拜占庭恶意节点。

一个值得注意的区别是，GRANDPA在链上而不是在块上达成协议，大大加快了最终确定的过程，即使在长期的网络分区或其他网络故障之后。

换句话说，只要有超过2/3的验证者证明某条链包含某个区块，那么导致该区块的所有前序区块就会被一次性确定。

[consensus/grandpa.pdf at master · w3f/consensus (github.com)](https://github.com/w3f/consensus/blob/master/pdf/grandpa.pdf)


Substrate中实现Grandpa在这里，它是Substrate Frame的一部分 [substrate/lib.rs at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/blob/master/frame/grandpa/src/lib.rs)


##### 分叉选择

将BABE和GRANDPA结合起来，Polkadot的分叉选择就变得很清楚了。BABE必须总是建立在已经被GRANDPA最终确定的链上。当在最终确定的头之后有分叉时，BABE通过建立在拥有最多primary区块的链上提供概率上的最终性。

![[Pasted image 20220824111910.png]]

在上图中，黑色区块是finality的，而黄色区块不是。标有 "1 "的区块是primary区块；标有 "2 "的区块是次级(secondary)区块。尽管最上面的链是最新完成的区块上最长的链，但它并不符合条件，因为在评估时它的primary块比下面的primary块少。

##### PBFT/Tendermint

这里可以去查Tendermint这样的PBFT共识与Grandpa + BABE有什么区别

##### Casper FFG

GRANDPA和Casper FFG之间的两个主要区别是。

- 在GRANDPA中，不同的投票者可以同时对不同高度的区块进行投票
- GRANDPA只依赖于最终确定的区块来影响底层区块生产机制的分叉选择规则

##### 资源

- grandpa协议实现 [paritytech/finality-grandpa: finality gadget for blockchains using common prefix agreement (github.com)](https://github.com/paritytech/finality-grandpa)
- 区块生成和区块确定：理解BABE和grandpa [Web3 Foundation Consensus Tutorial - Bill Laboon - #CryptoEconSys20 - YouTube](https://www.youtube.com/watch?v=1CuTSluL7v4&t=4s)


#### 治理（Governance）

[Governance · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-governance)

#### 身份(identity)

[Identity · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-identity)

#### 波卡网络的安全性

[Security of the network · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-security)

#### 提名者(Nominator)

[Nominator · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-nominator)


#### 波卡的Host


Polkadot的架构可以分为两个不同的部分，即Polkadot runtime和Polkadot host。Polkadot runtime是链条的核心状态转换逻辑，可以随着时间的推移升级，不需要硬分叉。相比之下，Polkadot host是runtime的执行环境，预计在Polkadot的生命周期内将保持稳定和大部分不改动。

Polkadot host与Polkadot runtime的交互方式是有限的，而且是明确规定的。由于这个原因，实施团队可以建立一个Polkadot host的替代实现，同时将Polkadot runtime视为一个黑盒子。关于host和runtime之间互动的更多细节，请参见规范。[w3f/polkadot-spec: The Polkadot Protocol Specification (github.com)](https://github.com/w3f/polkadot-spec/)

这里的原理大部分与Substrate的原理相似，因为substrate就是脱胎于波卡的。

##### 波卡host的组件

- 网络组件，如Libp2p，促进了网络交互。
- state存储和存储trie与数据库层一起。
- GRANDPA和BABE的共识引擎。
- Wasm解释器和虚拟机。
- 区块链的低级原语，如哈希函数等密码学原语。

编译后的Polkadot runtime，是一坨Wasm代码，可以上传到Polkadot host，并作为执行状态转换的逻辑。如果没有runtime，Polkadot host就无法进行状态转换或无法产生任何块。因为runtime是区块链的业务逻辑。

##### 波卡的图表

下面是一个显示Polkadot host围绕Polkadot runtime的图表。可以把rutime（白色）看作是一个可以插入、交换或完全移除的组件。而灰色的部分是稳定的，没有明确的硬分叉需求就不能改变。

![[Pasted image 20220824142947.png]]
因为runtime和host是分割独立的，所以Polkadot host有Go语言和C++语言实现的第三方代码：

- [ChainSafe/gossamer: 🕸️ Gossamer: A Go implementation of the Polkadot Host (github.com)](https://github.com/ChainSafe/gossamer)

- [soramitsu/kagome: Kagome - C++17 implementation of Polkadot Host (github.com)](https://github.com/soramitsu/kagome)

#### PolkadotJS

这个就是与波卡区块链交互的一系列工具了，有UI，有插件，有js的API [PolkadotJS · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-polkadotjs)

#### runtime升级

runtime升级允许Polkadot改变链的逻辑，而不需要进行硬分叉。

##### 无分叉升级

在区块链领域，你可能曾经遇到过 "硬分叉 "这个词。当一个区块链的逻辑发生变化，以至于不包括新变化的节点将无法与包括新变化的节点保持共识时，就会发生硬分叉。这种变化是向后不兼容的。由于升级的性质，硬分叉可能是政治性的，而且由于网络中需要升级其软件的节点的数量（可能是数千），在后勤上也很繁重。

Polkadot节点没有将runtime（链的 "业务逻辑"）编码在节点中，而是包含一个WebAssembly执行host(Polkadot host)。他们在一个非常低级和完善的指令集上保持共识。Polkadot的runtime存储在Polkadot区块链本身。

因此，Polkadot可以通过升级存储在链上的逻辑来升级其runtime，并消除了要求数以千计的节点操作员在给定的区块编号之前升级的协调挑战。相当于不同逻辑，要用高度隔离，不然会出错。Polkadot的利益相关者通过链上治理系统提出并批准升级，该系统也会自主地颁布升级。

##### 新的Client发布

这里的client就是波卡节点了。

现有的runtime逻辑被遵循，将存储在区块链上的Wasm runtime更新为新的版本。然后，该升级包含在区块链本身，这意味着网络上的所有节点都会执行它。一般来说，在runtime升级之前，不需要手动升级你的节点，因为它们会自动开始遵循链的新逻辑。只有当runtime需要新的host功能或者网络或共识发生变化时，才需要更新节点。host层的代码才需要用高度隔离。

所以，如果一个给定的runtime版本构建的交易将不能在以后的版本上运行，因为交易是属于runtime层的。因此，基于一个运行时版本构建的交易在以后的运行时版本中是无效的。如果你认为你不能在升级前提交一个交易，那么最好等待，在升级发生后再构建该交易。

虽然升级你的节点一般不需要跟随升级，但我们建议跟随Polkadot的发布，及时升级，尤其是高优先级或关键的发布。

##### 对于不同用户的角度来看待runtime升级

###### 基础设施提供者

- 验证者[How to Upgrade Your Validator · Polkadot Wiki](https://wiki.polkadot.network/docs/maintain-guides-how-to-upgrade)
- API服务
- 节点即服务(Node as a Service, NaaS)
- 通用的基础设施管理（区块链浏览器等）
- 钱包[Wallets · Polkadot Wiki](https://wiki.polkadot.network/docs/build-wallets)

对于验证者来说，与网络保持同步是关键。有时，升级会要求验证者在特定的时间范围内升级他们的客户，例如，如果一个版本包括对网络的突破性变化（host变了）。必须检查版本说明，从升级的优先级开始，并采取相应的行动。

一般的基础设施供应商，除了关注Polkadot的发布和及时升级外，还应该监测runtime事件和辅助工具的变化，如Substrate API Sidecar。

为runtime版本n构建的交易将无法在runtime 版本 >n 时工作。如果在广播先前构建的交易之前发生了运行时升级，你将需要用适当的runtime版本和相应的元数据重新构建它。

###### 提名者

runtime升级不需要提名者采取任何行动，不过我们一直鼓励提名者保持最新，参与最新的运行时升级动议和发布，同时关注网络上的节点对新升级的反应。

##### 监控这些变化

你可以监控链上即将进行的升级。客户端发布说明包括与任何链上升级有关的任何建议的哈希值，以方便匹配。监视链上的：

- democracy(Started)事件，并记录index和blockNumber。这个事件表明公投已经开始（尽管不意味着它是一个runtime的升级）。获取公投信息*；它应该有一个Ongoing的状态。找到结束的blockNumber（end）和颁布延迟（delay）。如果公投通过，它将在blockNumber为end+delay的时候执行。
- democracy(Passed)、democracy(NotPassed) 或者，democracy(Cancelled)事件引用了该索引。如果状态是Passed，你需要在同一区块中查看制定区块的scheduler(Scheduled)事件。
- democracy(PreimageNoted)事件，其hash值与ReferendumInfoOf(index)项相同。这可以到执行前的最后一个区块，但如果缺少这个，就不会有效果。
- democracy(Executed)事件用于实际执行。在runtime升级的情况下，也会有一个system(CodeUpdated)事件。

当然，你可以可以监控这里 [Polkassembly](https://polkadot.polkassembly.io/)。社区的讨论和公投。

#### 交易费用

其实就是为中继链的存储，计算压力付费，跟其他区块链的交易费没什么本质不同
[Transaction Fees · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-transaction-fees)

#### 国库(Treasury)

[Treasury · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-treasury)

财政部是一个通过部分区块生产奖励、交易费、砍价、staking效率低下等收集的资金罐。

#### 验证者(Validator)

[Validator · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-validator)

#### 抵押Staking

[Staking · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-staking)

#### 一些高级的概念
- [Availability and Validity · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-availability)
- XCM [Cross-Consensus Message Format (XCM) · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-crosschain)
- NPOS选举算法 [NPoS Election Algorithms · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-phragmen)
- 代理账户 [Proxy Accounts · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-proxies)
- 随机性 [Randomness · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-randomness)
- 简单赔付 [Simple Payouts · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-simple-payouts)
- SPREE [SPREE · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-spree)
- Staking Miner [Staking Miner · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-staking-miner)
- WASM [WebAssembly (Wasm) · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-wasm)
- 波卡的Keys [Polkadot Keys · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-keys)
- 波卡中使用的密码学算法 [Cryptography Explainer · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-cryptography)
- 波卡中是平行对象，平行链，ParaThreads和桥接 [Parachains · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-parachains)
- 波卡与其他链的对比 [What is the difference between Polkadot and Kusama? · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-kusama-vs-polkadot)
- 波卡的一些视频教程资料  [Video Tutorials · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-video-tutorials)
- 波卡的构建者指南 [Builder's Guides · Polkadot Wiki](https://wiki.polkadot.network/docs/build-index)
- 波卡的开发指南 [Builders Starter's Guide · Polkadot Wiki](https://wiki.polkadot.network/docs/build-build-with-polkadot)
- 波卡的集成指南 [Polkadot Integration Guide · Polkadot Wiki](https://wiki.polkadot.network/docs/build-integration)
- 波卡的运维指南 [Network Maintainers · Polkadot Wiki](https://wiki.polkadot.network/docs/maintain-index)



