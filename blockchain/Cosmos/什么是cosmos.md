### 什么是cosmos

#### cosmos概览

可互操作的区块链的Cosmos网络如何融入区块链技术的整体发展中？

深入了解区块链技术的历史，发现Cosmos生态系统--它是一个由钱包、服务、代币和dApps组成的星系。然后，获得你的第一个Cosmos Hub原生代币，了解如何将你的ATOM stake。

#### 区块链技术和cosmos

> 什么是区块链？

>  区块链协议定义了持有一种状态的程序，并描述了如何根据收到的输入修改该状态。这些输入被称为交易。
>  
>  共识机制确保区块链有一个规范的交易历史。区块链交易必须是确定性的，意味着只有一种正确的解释。区块链状态也是确定的。如果你从相同的创世状态开始，并复制所有的变化，你总是能达到相同的状态。
>  
>  一个区块链架构可以分成三层。
>  
>  网络层：负责发现节点，在单个节点之间传播交易和共识相关的信息。
>  共识层：在点对点（P2P）网络的单个节点之间运行共识协议。
>  应用层：运行一个定义应用状态的状态机，并随着实现网络共识的交易的处理而更新。
>  
>  这种分层模型可以普遍适用于区块链。

##### 区块链的世界，从公有通用链到特定的App应用链

你可以在20世纪80年代和90年代找到区块链技术的基石，当时计算机科学和密码学的突破奠定了必要的基础。

这些必要的突破包括：使用内置错误检查的只附加正确的交易日志；使用公钥的强认证和加密；成熟的容错系统理论；对点对点（P2P）系统的广泛理解；互联网的出现和无处不在的连接；以及强大的客户端计算机。

2008年10月31日，一个自称中本聪的个人或团体提出了一个数字货币的P2P网络，称之为比特币。比特币引入了一种新颖的共识机制，现在被称为中本聪共识，它使用工作证明（PoW）来使节点在一个分散的网络中达成协议。在独立于金融机构和可信第三方的各方之间直接发送在线支付成为可能。比特币成为第一个公共的、去中心化的支付应用。[bitcoin.pdf](https://bitcoin.org/bitcoin.pdf) 

> 关于PoW

> 比特币使用PoW来实现拜占庭容错（BFT），这使得去中心化、无信任的网络即使在有故障或恶意节点存在的情况下也能运作。它最好被描述为一个由称为 "矿工 "的网络节点解决的加密难题。该谜题是一个预先定义的、任意难度的任务。在目前使用PoW的网络规模下，结果类似于抽奖，只有一张中奖彩票，即成功的矿工节点。  
> 
> 在大多数PoW系统中，任务包括搜索一个未知的随机数--"nonce"。要成为一个成功的nonce，当它与区块中的有序交易相结合时，其结果是一个符合预先定义的标准的哈希值。找到nonce是在搜索中投入大量努力或工作的证据。每个节点都在竞赛中使用其计算能力，以首先解决这个难题，并赢得编写最新区块的权利。  
> 
> 经济激励也在发挥作用：首先公布解决方案的节点会得到奖励。通过这些奖励，网络的本地货币被发行，并鼓励节点将计算能力投入到解决这个任务中。网络安全与计算能力成正比；更多的投资导致更安全的PoW网络。  
  
建立在区块链网络上的去中心化应用程序的发展在比特币首次亮相后不久就开始了。在早期，开发dApps只能通过分叉或在比特币代码库上构建来完成。然而，比特币的单体代码库和有限的脚本语言使得dApp的开发对开发者来说是一个繁琐而复杂的过程。

在引入比特币后，几个所谓的 "公链 "应运而生，其中第一个是2013年的以太坊。这些通用的区块链旨在提供一个去中心化的网络，允许实施各种用例。

![[Pasted image 20221115105158.png]]

以太坊是一个具有智能合约功能的公链，能够实现基于自我执行、自我强化和自我验证的账户持有对象的应用。它可以被看作是对在比特币上开发应用的困难的回应。  
  
有了以太坊，链的应用层采取了虚拟机的形式，称为以太坊虚拟机（EVM）。EVM运行智能合约，提供一个单一的链，在上面部署各种程序。尽管EVM有很多好处，但它是一个沙盒，划定了可实施的用例范围。简单的（有时是复杂的）用例可以用它来实现，但在设计和效率方面还是受到了沙盒的限制。此外，开发人员被限制在为EVM定制的编程语言上。  
  
尽管以太坊及其EVM的推出是一个很大的进步，但公共的、通用的区块链的一些问题仍然存在：开发人员的灵活性低，在速度、吞吐量、可扩展性、状态最终性和主权方面存在困难。  
  
在区块链的世界里，"速度 "意味着交易速度。你可以把交易速度理解为确认一笔交易所需的时间。速度自然受到区块之间的目标延迟的影响，在比特币是10分钟，在以太坊是15秒。速度也会受到积压的同样有价值的pending交易的影响，这些交易都争相被纳入新区块。  
  
吞吐量描述了网络在每单位时间内可以处理多少交易。吞吐量可能会因为物理网络带宽、计算机资源，甚至是协议中的决定而受到限制。并非所有的dApps都有相同的吞吐量要求，但如果它们是在通用区块链上实现的，它们都必须与平台本身的平均结果吞吐量相适应。这影响了dApps的可扩展性。  
  
状态最终性是一个额外的问题。最终性描述了是否以及何时承诺的交易区块不能再被恢复/撤销。区分概率最终性和绝对最终性是很重要的。  
  
> 概率最终性

> 概率最终性描述了交易的最终性取决于恢复区块的可能性有多大，即删除交易的概率。在包含特定交易的区块之后出现的区块越多，交易被恢复的可能性就越小，因为在分叉的情况下适用最长或最重的链规则。

> 绝对最终性

>  绝对的终结性是基于权益证明（PoS）的协议的一个特征。一旦交易和区块被验证，就会出现最终结果。在任何情况下，都不存在交易被最终确定后可以撤销的情况。

> PoW和PoS网络的最终性

> 权益证明（PoS）网络可以有绝对的终结性，因为总的押注金额在任何时候都是已知的。需要一个公开的交易来押注，另一个交易来解除押注。如果大多数人同意一个区块，那么这个区块可以被认为是 "最终的"，因为没有任何程序可以推翻这个共识。
> 
> 这与PoW网络不同，PoW网络的总Hash能力是未知的；它只能通过谜题的难度和新区块的发行速度来估计。Hash能力可以通过打开或关闭机器来增加或删除。当Hash能力被突然删除时，会导致网络交易吞吐量的下降，因为区块突然无法在目标时间段内被发行。

在以太坊上开发时，开发者需要面对两层治理。链的治理和应用程序的治理。独立于dApp的治理需求，开发者必须与底层链的治理达成协议。

鉴于现有公链项目的特点和某些行业对隐私的要求，随之而来的是对私有链或管理链的推动。私有分布式账本是具有访问障碍和复杂权限管理的区块链。例子包括许可网络的平台，如R3的Corda和Linux基金会的Hyperledger项目的Hyperledger Fabric。

更复杂的应用的最终发展需要一个更灵活的环境。这导致了多个特定目的/特定应用的区块链的推出，每个区块链都提供了一个为用例和应用的需要而定制的平台。这些区块链中的每一个都作为独立的环境，受到它们所设想的用例的限制。

通用链仅限于简单化的用例应用，而特定应用链只适合某些用例。这引发了一个问题：是否有可能为所有用例建立一个平台，摆脱通用链的限制？

##### Cosmos是怎样来做到区块链技术的通用开发的

2016年，Jae Kwon和Ethan Buchman创立了Cosmos网络，其共识算法为Tendermint [Tendermint](https://tendermint.com/) 

> 请参阅2016年cosmos白皮书 [Whitepaper - Resources - Cosmos Network](https://v1.cosmos.network/resources/whitepaper) ，了解更多关于cosmos的起源。

Kwon在2014年发明了最初的Tendermint机制。Buchman和Kwon于2015年开始合作，并在当年年底共同成立了Tendermint Inc公司。

> Tendermint:  你需要知道什么

> Tendermint是一种具有拜占庭容错（BFT）的共识算法和一个共识引擎。它使应用程序可以在许多机器上同步复制。区块链网络需要BFT来确保正常的功能，即使有故障或恶意的节点存在。其结果被称为具有拜占庭容错的复制状态机。它保证了分布式系统及其应用的BFT属性。
> 
> 它做到了这一点。
> 
> 安全地--即使多达1/3的机器失败或行为不端，Tendermint也会继续工作。
> 一致性--每台机器计算相同的状态并访问相同的交易日志。
> 
> Tendermint在整个行业中被广泛使用，是最成熟的BFT共识引擎，用于权益证明（PoS）区块链。
> 
> 这里有更多的关于tendermint的核心原理介绍 [What is Tendermint | Tendermint Core](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html) 

最初，Cosmos是一个由Tendermint团队建立的开源社区项目。从那时起，Interchain基金会（ICF）就协助开发和推出了该网络。ICF是一个瑞士的非营利组织，在2017年筹集资金，资助建立在Cosmos网络上的开源项目的发展。  
  
Cosmos的创始愿景是，为区块链技术提供一个简单的开发环境。Cosmos希望解决以前区块链项目的主要问题，并提供链之间的互操作性，以促进区块链的互联网。  
  
Cosmos是如何成为区块链的互联网的？Cosmos是一个可互操作的区块链网络，每个区块链都以适合其个人使用情况的不同属性实现。Cosmos让开发者创建的区块链保持主权，不受任何 "主链 "治理的影响，具有快速的交易处理能力，并且是可互操作的。有了Cosmos，众多的用例就变得可行了。  
  
为了实现这一愿景和网络类型，该生态系统依赖于一个开源工具包，包括区块链间通信协议（IBC）[Inter-Blockchain Communication (ibcprotocol.org)](https://ibcprotocol.org/) ，IBC在Cosmos SDK中实现 [Cosmos SDK - Cosmos Network](https://v1.cosmos.network/sdk) ，以及Tendermint[Tendermint](https://tendermint.com/) 作为提供分布式状态最终性的基础层。一套模块化的、可适应的、可互换的工具不仅有助于快速创建区块链，而且有利于安全和可扩展链的定制。  
  
Cosmos上可互操作的应用区块链是用Cosmos SDK构建的。Cosmos SDK包括使创建的区块链有可能使用区块链间通信协议（IBC）参与区块链间通信的先决条件。用Cosmos SDK构建的链使用Tendermint共识。在接下来的章节中，将对这些主题中的每一个进行更详细的探讨。  
  
##### cosmos是怎样解决可扩展问题的

可扩展性是区块链技术的一个核心挑战。Cosmos允许应用程序扩展到数以百万计的用户。这种程度的可扩展性是可能的，因为Cosmos解决了两种类型的可扩展性。  
  
- 横向可扩展性：通过向网络添加类似的机器进行扩展。当 "扩展 "时，网络可以接受更多的节点参与状态复制、共识观察和任何查询状态的活动。  
- 垂直可扩展性：通过改进网络的组件来增加其计算能力来进行扩展。当 "扩大规模 "时，网络可以接受更多的交易和任何修改状态的活动。  

在区块链背景下，垂直可扩展性通常是通过优化共识机制和在链上运行的应用程序来实现的。在共识方面，Cosmos在Tendermint BFT的帮助下实现了垂直可扩展性。目前Cosmos Hub在7秒内进行交易。那么剩下的唯一瓶颈就是应用程序了。  
  
你的区块链的共识机制和应用优化只能带你走到这里。为了克服垂直可扩展性的限制，Cosmos的多链架构允许一个应用程序在不同但IBC协调(IBC-coordinated)的链上并行运行，无论是否由相同的validators组操作。这种链间的横向可扩展性在理论上允许无限的纵向可扩展性，减去协调的开销。  
  
> 在区块链中，validator是一个或多个合作的计算机，通过创建区块等方式参与共识。


##### Cosmos是如何促进主权的

部署在通用区块链上的应用程序都共享相同的基础环境。当需要对应用程序进行更改时，不仅取决于应用程序的治理结构，也取决于环境的治理结构。实施变化的可行性取决于应用程序所建立的协议所设定的治理机制。链的治理限制了应用程序的主权。由于这个原因，它通常被称为两层治理。

例如，典型的区块链上的应用程序有其治理结构，但它存在于区块链治理之上，链上的升级有可能破坏应用程序。因此，应用程序的主权在双层治理环境中被削弱了。

Cosmos解决了这个问题，因为开发者可以建立一个为应用量身定做的区块链（也就是最近提到很多的App chain这个概念）。当每条链都由自己的validator维护时，对应用的治理没有任何限制。Cosmos遵循的是单层治理设计。

##### cosmos是如何提高用户体验的

在传统的通用区块链世界中，应用程序的设计和效率对区块链开发者来说是有限的。在Cosmos宇宙中，架构组件的标准化与定制机会相结合，提供了不受约束、无缝和直观的用户体验的可能性。

用户在不同的区块链和应用程序之间的导航变得更加容易，因为由于组件的标准化，同样的基础规则适用。Cosmos让开发者的世界变得更容易，同时让dApps变得更友好。Cosmos实现了主权与互操作性的统一!

#### Cosmos生态系统

##### 整个生态宇宙-代币，钱包，apps和服务

Cosmos是一个不断扩大的代币、钱包和工具的生态系统，以及相互连接的应用程序和服务，都是为去中心化的未来而建立的。

在2022年，近1000亿美元的数字资产由Cosmos管理和担保。Cosmos上的数字资产包括同质代币和非同质化代币（NFT）。你可以在应用中发行代币来进行结算，定制发行，处理通货膨胀/通货紧缩，以及更多。

在Cosmos担保的同质化代币中，有Binance Coin[BNB - What is BNB and What Is It Used For? (binance.com)](https://www.binance.com/en/bnb) 、Terra [Terra](https://www.terra.money/) 和ATOM[Cosmos: The Internet of Blockchains](https://cosmos.network/learn/faq/what-is-the-atom) 。请记住，由于这些代币是在特定应用的区块链上定义的，它们的开发者可以不受假设的通用区块链的限制。

更多的代币请看 [Market Capitalization - Ecosystem - Cosmos: The Internet of Blockchains](https://cosmos.network/ecosystem/tokens/)

除了大量的代币之外，各种应用、服务、钱包和区块链浏览器都是基于Cosmos的。

数以百计的应用和服务建立在Cosmos之上。目前，大多数应用和项目涉及金融，紧随其后的是基础设施相关的应用。隐私、市场和社会影响等领域的应用和项目也正在出现。

最新的生态请看 [Apps & Services - Ecosystem - Cosmos: The Internet of Blockchains](https://cosmos.network/ecosystem/apps/?ref=cosmonautsworld) 

虽然大多数应用程序和项目都部署在主网上，但有些目前要么处于概念验证阶段，要么仍在开发中，要么只部署在测试网上。

此外，33个钱包 https://cosmos.network/ecosystem/wallets 和cosmos的区块浏览器是生态系统的一部分。大多数是安卓系统的，如Atomic钱包 [Atomic Wallet - Buy, Stake and Exchange in One Crypto Wallet](https://atomicwallet.io/) 和Coinex [CoinEx - The Global Cryptocurrency Exchange](https://www.coinex.com/) ，或iOS系统的，如AirGap [AirGap - Self custody made simple and secure for all your crypto assets](https://airgap.it/) 和Wallet.io[wallet.io (walletio.io)](https://walletio.io/) ，但你也可以找到一些web钱包，如Exodus  [Best Crypto Wallet for Desktop & Mobile: Altcoin & Bitcoin | Exodus](https://www.exodus.com/)  和Keplr [Keplr Dashboard](https://wallet.keplr.app/) 。

查看更多钱包 https://cosmos.network/ecosystem/wallets

##### cosmos SDK- 模块化和可定制化

cosmos网络专注于一个易于区块链开发的生态系统，提供可定制性和互操作性。它建立了一个稳定的宇宙，由平等适用于整个生态系统的规则决定。

在Cosmos出现之前，开发一个全新的链比构建一个智能合约要困难得多，也昂贵得多。现在，有了Cosmos SDK，就可以开发出完全灵活、安全、高性能和主权应用的区块链。为了实现这一点，建立模块化、可适应、可互换的开源开发工具是Cosmos的中心任务。

Cosmos网络的主要目的是为基于Tendermint BFT和区块链间通信协议（IBC）的所谓Cosmos SDK[Cosmos SDK - Cosmos Network](https://v1.cosmos.network/sdk) 提供一个易于区块链开发的生态系统。

Cosmos生态系统中的每一条链都依赖于Tendermint的快速最终性的BFT共识算法。这确保了在网络的所有链上有一个共同的共识机制。除了在Cosmos中使用外，Tendermint共识机制还被用于Binance Chain https://www.binance.org/en 、Terra  [Terra](https://www.terra.money/) 、Kava [Kava Network](https://www.kava.io/) 等。

Cosmos SDK是一个通用的框架，用于在Golang的Tendermint BFT上构建安全的区块链应用。它是一个针对特定应用的区块链的模块化框架。该设计基于两个主要原则：模块化和基于能力的安全。Cosmos SDK被设想为一个类似npm的框架，用于Tendermint之上的安全应用。随着时间的推移，它已经成为一个先进的框架，用于定制特定的应用程序区块链。

Cosmos SDK的现成模块易于导入、调整和使用。开发人员可以创建自己的模块，以引入特定的功能。随着生态系统的发展，模块的数量会越来越多，有利于开发更复杂的应用程序。

> 建立在模块化组件上，其中许多组件不是你自己编写的--这是否增加了攻击的可能性，以及有问题或恶意的节点运行不被发现？不需要担心。
> 
> Cosmos SDK是建立在对象能力模型上的 [Object-Capability Model | Cosmos SDK](https://docs.cosmos.network/main/core/ocap.html) 。它不仅有利于模块化，而且还封装了代码的实现。对象能力模型确保。
> 
> - 内存中的对象没有办法仅仅通过别人的组成对象来发现。
> - 拥有对对象的引用或访问服务的唯一方法是已经被传递了相关的对象引用。

> 当使用SDK来开发时， 默认可用的共识机制是 tendermint core [Overview | Tendermint Core](https://docs.tendermint.com/v0.34/tendermint-core/) 

##### IBC协议

区块链间通信协议（IBC） [Inter-Blockchain Communication (ibcprotocol.org)](https://ibcprotocol.org/) 是Cosmos中互操作性的基础。它利用Tendermint的即时终结性，允许价值转移（代币转移）和异构链之间的通信。具有不同应用和架构规范的区块链，无论是否共享validators集，都可以实现互操作。

如果没有IBC，异构链的互操作性就难以实现，因为它们可能以不同的方式实现共识、网络和应用层。只要一个区块链与IBC兼容，它就可以与其他区块链互通。

Cosmos实现了一个具有两个区块链类别的模块化架构：枢纽(hubs)和区域(zones)。

![[Pasted image 20221115113427.png]]

Zones是异构的区块链，进行账户和交易的认证，代币的创建和分配，以及执行对链的更改。Hub是旨在连接所谓Zones的区块链。一旦一个Zone通过IBC连接到一个Hub，它就可以自动访问连接到该Hub的其他Zones。在这一点上，数据和价值可以在各Zones之间发送和接收，而不会有例如重复花费代币的风险。这有助于减少为实现互操作性而需要建立的链对链连接的数量。

没有执行实际的拓扑结构。一个Hub可以被理解为一个有许多连接到其他Zones的Zone。应用Zones可以被期望加入生态系统中的Hub，但它们可以自由地凝聚成开发者认为合适的任何拓扑结构。

关于查看Hub和Zones的测试网和主网的拓扑 请看 [Map of zones - Cosmos network explorer](https://mapofzones.com/) 

consmos hub [Introduction | Cosmos Hub](https://hub.cosmos.network/main/hub-overview/overview.html) 是第一个创建的Hub。它是一个公共的权益证明（PoS）区块链，有一个native代币ATOM。cosmos hub可以被理解为一个路由器，促进与它相连的链之间的交易。例如，Cosmos Hub允许以不同的代币支付交易费用，只要该Zone信任Cosmos Hub和与之相连的其他Zones。  
  
我们如何将我们的链连接到非Tendermint链？IBC连接不限于基于Tendermint的链。如果另一个非Tendermint的区块链使用快速确定性的共识算法，可以通过调整IBC以与非Tendermint的共识机制一起工作来建立连接。  
  
如果另一个链是一个概率最终性的链(PoW like)，那么简单地适应IBC是不够的。一个叫做Peg-zone的代理链有助于建立互操作性。Peg-zone是快速最终性区块链，它跟踪链的状态以建立最终性。peg-zone链本身是与IBC兼容的，并作为IBC网络的其他链和概率最终性链之间的桥梁。  
  
> 以太坊存在一个peg-zone的实现，被命名为Gravity桥 [cosmos/gravity-bridge: A CosmosSDK application for moving assets on and off of EVM based, POW chains (github.com)](https://github.com/cosmos/gravity-bridge) 。关于桥和Gravity桥的更多信息，见桥一节。[Bridges | Developer Portal (cosmos.network)](https://tutorials.cosmos.network/academy/2-cosmos-concepts/13-bridges.html) 

##### Ignite CLI - 一键构建应用区块链

Ignite CLI [Innovation Platform For Blockchain Development | Ignite CLI](https://ignite.com/) 是一个方便开发者的命令行界面（CLI）工具，用于特定应用的区块链，它建立在Tendermint和Cosmos SDK之上。该CLI工具提供了开发人员构建、测试和启动链所需的一切。它通过搭建脚手架和组装生产就绪的区块链所需的所有组件来加速区块链开发。Ignite CLI使从最初的想法到生产的过程快了95%。这让开发者可以在几分钟内构建一个区块链，并让他们更专注于应用程序的业务逻辑。

通过Ignite CLI，开发者可以。

- 用一个命令创建一个用Go编写的模块化区块链。
- 启动一个开发服务器，试验代币的创建和分配，以及模块配置。
- 通过使用内置的IBC中继器将价值和数据发送到不同的链上，允许链间代币转移。
- 受益于快速开发的前端，自动生成JavaScript、TypeScript和Vue的API和网页。

更多的Ignite CLI的信息请看 [Ignite CLI | Ignite Cli Docs](https://docs.ignite.com/) [Ignite CLI | Developer Portal (cosmos.network)](https://tutorials.cosmos.network/hands-on-exercise/1-ignite-cli/1-ignitecli.html) 

当你用Ignite CLI搭建脚手架时，像密钥管理、创建Validator和转移token等事情都可以通过CLI完成。

#####  CosmWASM - 多链智能合约

CosmWasm [CosmWasm](https://cosmwasm.com/) 是Cosmos上智能合约的多链解决方案--这就是Cosm部分。它是一种在Cosmos 中使用WebAssembly的方式--这就是Wasm部分。  
  
通过CosmWasm，你可以使用智能合约并在Tendermint和Cosmos SDK的基础上为Cosmos创建强大的去中心化应用程序（dApps）。它的主要特点是。  
  
- 用于开发和测试智能合约的成熟的工具  
- 与Cosmos SDK和生态系统紧密结合  
- 一个安全的架构以避免攻击载体  

CosmWasm的设计使你的代码与底层链的细节无关。它只需要一个Cosmos SDK应用程序嵌入Wasm模块。  
  
有了CosmWasm，智能合约可以在IBC协议的帮助下运行在多个链上。它为开发者增加了进一步的灵活性，并使智能合约的开发更快。CosmWasm被写成一个可以插入Cosmos SDK的模块，并利用Wasm的速度和Rust的力量。CosmWasm是寻找具有快速交易终结性的区块链平台的Rust开发者的理想选择。  更多请看 [Introduction | CosmWasm Documentation](https://docs.cosmwasm.com/docs/1.0/)

##### 使用其他区块链框架和SDK的可能性

由于Cosmos SDK是模块化的，开发者可以在SDK的基础上移植现有的Go代码库。这使得开发者可以在Cosmos上进行构建，而不必在所使用的工具集和环境上做出过多的妥协。

例如，通过Ethermint https://github.com/tharsis/ethermint ，开发者可以将Go以太坊主客户端的以太坊虚拟机（EVM）作为Cosmos SDK模块使用，这与现有的模块是兼容和可组合的。
  
> Ethermint是一个为将EVM移植到Cosmos模块而开发的软件。它使可扩展、高吞吐量、PoS区块链成为可能。这些都与Ethereum和Cosmos SDK完全兼容。
> 
> Ethermint兼容Web3，并通过Tendermint实现高吞吐量，通过IBC实现水平扩展。它提供了一个Web3、JSON-RPC层来与以太坊客户端和工具进行交互。

更多关于ethermint请看 [Evmos Documentation | Evmos Documentation](https://docs.evmos.org/) 

所有以太坊工具（如Truffle和Metamask）都与Ethermint兼容。开发者甚至可以将他们的Solidity智能合约移植到Cosmos Ecosystem上进行互动。开发与Cosmos兼容的智能合约并不需要建立一个链，这一切都可以通过Ethermint完成。然而，虽然Ethermint允许运行vanilla Ethereum作为Cosmos特定应用的区块链，但开发者可以从Tendermint BFT中获益。

#### 获得ATOM和抵押它

[Getting ATOM and Staking It | Developer Portal (cosmos.network)](https://tutorials.cosmos.network/academy/1-what-is-cosmos/3-atom-staking.html) 

#### cosmos VS polkadot

因为cosmos和polkadot大体上的哲学目标一致，都是连接各个区块链，形成可互操作性，所以这里有一个对比 [Polkadot vs Cosmos - Miscellaneous - Cosmos Hub Forum](https://forum.cosmos.network/t/polkadot-vs-cosmos/1397/4) 