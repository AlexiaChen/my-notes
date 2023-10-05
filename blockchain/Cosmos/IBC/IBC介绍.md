有没有想过跨链通信是如何实现的？快来了解一下区块链间通信协议（IBC）的世界。

了解更多关于IBC的运输、认证和订购层，并深入了解链间代币转移如何成为可能。最后，快速浏览一下链间账户和可用来可视化与IBC连接的链间网络的工具。

## 什么是IBC

区块链间通信协议（IBC）  [Inter-Blockchain Communication (ibcprotocol.org)](https://ibcprotocol.org/) 是一个处理两个区块链间的认证和数据传输的协议。IBC需要一套最小的功能，在inter区块链标准（ICS）  [ibc/spec/ics-001-ics-standard at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/ics-001-ics-standard) 中指定。请注意，这些规范并不限制网络拓扑结构或共识算法，因此IBC可用于广泛的区块链或状态机。IBC协议为区块链之间的数据包中继提供了一种无权限的方式，与大多数可信Bridge技术不同。IBC的安全性降低到参与链的安全性。  
  
IBC解决了一个普遍存在的问题：跨链通信。当交易所希望进行互换时，这个问题存在于公共区块链上。在特定应用区块链的情况下，这个问题很早就出现了，每个资产都有可能出现在自己的专用链上。在私有区块链的世界里，跨链通信也是一个挑战，在与公共链或其他私有链的通信是可取的情况下。已经有IBC实现的私有区块链，如Hyperledger Fabric和Corda  [Meet YUI, one of the new Hyperledger Labs taking on cross-chain and off-chain operations – Hyperledger Foundation](https://www.hyperledger.org/blog/2021/06/09/meet-yui-one-the-new-hyperledger-labs-projects-taking-on-cross-chain-and-off-chain-operations) 。  
  
Cosmos中特定应用区块链之间的跨链通信创造了具有交易最终性的高水平可扩展性的潜力。这些设计特点为困扰其他平台的众所周知的问题提供了令人信服的解决方案，如交易成本、网络容量和交易确认最终性。  

#### 区块链互联网

IBC对于像Cosmos网络中的特定应用区块链是必不可少的。它为两个不同链上的应用提供了一个标准的通信渠道，这些应用需要相互通信。

大多数Cosmos应用程序在自己的专用区块链上执行，运行自己的validators集（至少在引入Interchain Security [Interchain Security is Coming to the Cosmos Hub | by Billy Rennekamp | Cosmos Blog](https://blog.cosmos.network/interchain-security-is-coming-to-the-cosmos-hub-f144c45fb035?gi=85b48474596d) 之前）。这些是用Cosmos SDK构建的特定应用区块链。一个链上的应用可能需要与另一个区块链上的应用进行通信，例如，一个应用可以接受另一个区块链的代币作为支付形式。在这个层面上的互操作性需要一种方法来交换关于另一个区块链上的状态或交易的数据。

虽然这种区块链之间的bridge可以建立并且确实存在，但它们通常是临时构建的（也就是通常说的跨链桥）。IBC为区块链提供了一个通用的协议和框架来实现标准化的区块链间通信。对于用Cosmos SDK构建的链来说，这是开箱即用的，但IBC协议并不限于用Cosmos栈构建的链。

> 关于规范的更多细节将在下一节介绍，但请注意，IBC并不限于Cosmos区块链。甚至可以为最初没有满足某些要求的情况找到解决方案。这种情况的一个例子是将IBC用于工作证明（PoW）区块链，如Ethereum。由于PoW共识算法不能确保最终性，使用IBC的主要要求之一没有得到满足。不过，通过创建一个peg-zone，在给定的区块确认阈值之后，概率最终性被认为是确定的（不可逆的），从而实现与以太坊的兼容。  
  
尽管与通用区块链平台相比，特定应用的区块链提供了卓越的（横向）可扩展性，但通用链的智能合约开发和像以太坊中的通用虚拟机（VM）提供了它们自己的好处。IBC提供了一种方法，将通用区块链和特定应用区块链的优势纳入统一的整体设计。例如，它允许针对性能和可扩展性定制的Cosmos链使用源自以太坊的资金，并可能在Corda ledger中记录事件；或者，反过来说，Corda ledger启动Cosmos或以太坊中定义的基础资产的转移。  
  
通过IBC的跨链通信，一个由独立和可互操作的链组成的去中心化网络交换信息和资产是可能的。这种 "区块链的互联网 "带来了增加和无缝扩展的承诺。在Cosmos中，正在实施的愿景是拥有一个由独立链组成的宇宙，这些链都是用peg-zone作为Cosmos网络和它之外的链之间的桥梁，并通过hub连接所有链。所有这些都构成了区块链的互联网。

#### IBC的高层次概览

传输层（TAO，transport layer）提供必要的基础设施，以建立安全连接，并对链之间的数据包进行认证。应用层建立在传输层之上，确切地定义了数据包应如何被发送和接收链打包和解释。

IBC的巨大承诺是提供一个可靠的、无权限(permissionless)的和通用的基础层，同时通过将应用设计转移到更高层次的层，允许可组合性和模块化的关注点分离。

这种分离反映在一般ICS定义中的ICS类别中 [ibc/README.md at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/blob/main/spec/ics-001-ics-standard/README.md) 。

- IBC/TAO：定义数据包的传输、认证和排序的标准，即基础设施层。在ICS中，这是由Core、Client和Relayer等类别组成的。
- IBC/APP：定义数据包在传输层上传递的应用处理程序的标准。这些标准包括但不限于同质化代币传输（ICS-20）、NFT传输（ICS-721）和链间账户(interchain accounts)（ICS-27），并可在ICS的App类别中找到。

下图显示了IBC在高层次上的运作方式。

![[Pasted image 20221118105300.png]]

注意图中的三个关键因素。

- 链依赖中继器(relayers)进行通信。中继器算法（ICS-18  [ibc/spec/relayer/ics-018-relayer-algorithms at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/relayer/ics-018-relayer-algorithms) ）是IBC的 "物理 "连接层：链外进程负责在运行IBC协议的两个链之间中继数据，方法是扫描每个链的状态，构建适当的数据报，并在协议允许的情况下在对面的链上执行这些数据报。
- 许多中继器可以为一个或多个通道服务，在链之间发送消息。
- 中继器的每一方都使用另一条链的轻客户端来快速验证传入的消息。

#### IBC/TAO - 传输层

在图中，说明了TAO类别中ICS定义之间的关系--箭头说明了要求。  
  
简单地说，传输层包括。  
  
- Light Clients - ICS-2 [ibc/spec/core/ics-002-client-semantics at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-002-client-semantics) ，6 [ibc/spec/client/ics-006-solo-machine-client at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/client/ics-006-solo-machine-client) ，7 [ibc/spec/client/ics-007-tendermint-client at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/client/ics-007-tendermint-client) ，8 [ibc/spec/client/ics-008-wasm-client at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/client/ics-008-wasm-client) ，10 [ibc/spec/client/ics-010-grandpa-client at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/client/ics-010-grandpa-client) 。IBC客户端是由唯一的客户端ID识别的轻型客户端。IBC客户端跟踪其他区块链的共识状态和这些区块链的证明specs，这些区块链需要根据客户端的共识状态正确验证证明。一个客户端可以与对手链的任何数量的连接相关联。  
  
- Connections - ICS-3 [ibc/spec/core/ics-003-connection-semantics at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-003-connection-semantics) ：连接一旦建立，负责促进IBC状态的所有跨链验证。一个连接可以与任何数量的通道相关联。连接封装了两个独立区块链上的两个`ConnectionEnd`对象。每个`ConnectionEnd`都与另一个区块链的轻客户端相关联--例如，对手方的区块链。连接握手负责验证每条链上的轻客户端是其各自对应方的正确客户端。  
  
- Channels - ICS-4 [ibc/spec/core/ics-004-channel-and-packet-semantics at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-004-channel-and-packet-semantics) ：一个区块链上的模块可以通过channels发送、接收和确认数据包与其他区块链上的其他模块进行通信，这些通道由（channelID，portID）元组唯一识别。通道封装了与一个连接相关的两个`ChannelEnds`。通道提供了一种在链之间转发不同类型信息的方式，但并不增加总容量。就像连接一样，通道是通过握手建立的。  
  
一个信道可以是`ORDERED`，即发送模块的数据包必须由接收模块按照其发送的顺序进行处理；也可以是`UNORDERED`，即发送模块的数据包按照其到达的顺序进行处理（可能不同于其发送的顺序）。  
  
- Ports - ICS-5 [ibc/spec/core/ics-005-port-allocation at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-005-port-allocation) ：一个IBC模块可以与任何数量的端口绑定。每个端口必须由一个唯一的`portID`来识别。`portID`表示应用的类型，例如，在同质化代币转移中，`portID`是`transfer`。  
  
虽然这些背景信息很有用，但IBC模块完全不需要与这些低级别的抽象进行交互。

#### IBC/APP - 应用层

ICS还提供了IBC应用的定义。

- Fungible token transfer - ICS-20 [ibc/spec/app/ics-020-fungible-token-transfer at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/app/ics-020-fungible-token-transfer) 。IBC的第一个也是最明显的应用是同质化代币的跨链转移。通过ICS-20制定的标准，用户可以在支持IBC的链上发送代币。这是通过在源链上托管代币来实现的：证明与代币元数据一起被转发到目的地链，然后由源链的轻客户端验证证明，并存储在目的地链上。如果验证通过，目的链上的代币的凭证就会被铸造出来，并将确认函发回给源链。

数据包流程将在后面的章节中详细探讨，但你可以在Mintscan [Mintscan - cosmos chain explorer by COSMOSTATION](https://www.mintscan.io/cosmos) 上关注IBC代币转移的进展时的步骤。下面的例子显示了在源端提交的原始`Transfer`的交易，在目的地提交的`Receive`信息，以及在源端再次提交的`Acknowledgement`信息。

![[Pasted image 20221118112736.png]]

Interchain accounts - ICS-27 [ibc/spec/app/ics-027-interchain-accounts at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/app/ics-027-interchain-accounts) ：链间账户概述了建立在IBC上的跨链账户管理协议。启用ICS-27的链可以在其他启用ICS-27的链上以编程方式创建账户，并通过IBC交易控制这些账户，而不必用私钥签名。链间账户包含普通账户的所有功能（即stake、发送、投票），但由单独的链通过IBC管理，其方式是控制链上的所有者账户保留对其在主机链上注册的任何链间账户的完全控制。

> 这个列表可以并将随着时间的推移而被扩展。诸如链间账户的新概念将继续增加采用，并在Interchain生态系统中提供应用多样性。

#### 安全

伴随着协议的可扩展性、可靠性和无需信任第三方的安全性，IBC作为一个通用的互操作性标准的无许可性质是IBC与标准Bridge协议相比最宝贵的辨别特征之一。然而，由于创建IBC客户端、连接和通道是无许可的，或在链之间中继数据包，你可能会想知道。安全方面的影响如何？

IBC的安全设计是围绕着两个主要原则。

- 对你所连接的链的信任（共识）。
- 实施故障隔离机制，以便在这些链受到恶意行为的影响时限制任何损害。

IBC实施的安全考虑是值得探讨的。

#### IBC轻客户端

与许多可信桥解决方案不同，IBC不依赖中介来验证跨链交易的有效性。如前所述，数据包承诺证明的验证是由轻客户端提供的。轻客户端能够跟踪并有效地验证对手方区块链的状态，以检查源链和目的链上分别发送和接收数据包的承诺证明。  
  
  
> 在IBC中，区块链不直接在网络上相互传递信息。为了通信，区块链将状态提交给一个精确定义的路径，为特定的消息类型和特定的交易方保留。中继者监测这些路径的更新，并通过提交存储在路径下的数据以及该数据的证明给对手链来中继消息。  
  
  
> 所有IBC实施方案必须支持的提交IBC消息的路径在ICS-24主机状态机要求中定义 [ibc/spec/core/ics-024-host-requirements at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-024-host-requirements) 。  
> 所有实施方案必须产生和验证的证明格式在Confio的ICS-23实施文件中定义 [ibc/README.md at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/blob/main/spec/core/ics-023-vector-commitments/README.md) 。  
  
这一点很重要，因为它确保了IBC协议即使在中继器可能以恶意或错误的方式行事的拜占庭环境中也能保持安全。你不需要相信中继者；相反，你相信轻型客户端提供的证明验证。在最坏的情况下，所有中继者都以拜占庭的方式行事，发送的数据包会被拒绝，因为它们没有正确的证明。这将只影响Interchain网络中中继者是恶意的那一部分的有效性，而不是安全性。  
  
请注意，只有在所有中继者都是拜占庭的情况下，这种影响才会影响网络。由于中继是无权限的，一个简单的修复方法是让一个非恶意的中继者用正确的证明来中继数据包。这符合IBC和更广泛的Interchain生态系统所采用的安全性高于灵活性的理念。  
  

#### 相信链，而不是桥

> 这里就是对于L1的链的结算共识层来说，L1是最安全的，而桥是链下的机制

IBC客户和交易承担了他们所连接的链的信任模式。为了在通过Interchain的资产中准确表示这一点，资产所经过的路径信息（资产的安全保证）被存储在资产本身的面额中。如果终端用户或应用程序不信任特定的起源链，他们将能够验证他们的资产不是来自不受信任的链，只需查看资产的面额，而不是参考桥梁的验证器集或其他一些受信任的第三方验证器。


> 所有在特定通道上转移的代币将被分配到与在该通道上流动的其他代币相同的面额，但与同一链之间的相同资产在不同通道上发送时的面额不同。IBC面额看起来像`ibc/<hash of the channel-d & port-id>`。
> 
> 你可以在IBC denoms的教程中找到更详细的信息。[Understanding IBC Denoms | Developer Portal (cosmos.network)](https://tutorials.cosmos.network/tutorials/5-ibc-dev/) 

####  提交错误行为

在支持IBC的链上可能发生的一种拜占庭行为是validator对一个区块进行双签--意味着他们在同一高度上签署了两个不同的区块。这种情况被称为分叉。与PoW区块链（如比特币或以太坊）不同的是，在Tendermint中，分叉是偶尔会发生的，而在Tendermint中，链的快速终结性是人们所希望的（也是IBC的前提条件），因此分叉不应该发生。

通过分叉责任原则 [cosmos/WHITEPAPER.md at master · cosmos/cosmos (github.com)](https://github.com/cosmos/cosmos/blob/master/WHITEPAPER.md#fork-accountability) ，导致共识失败的过程可以被识别，并根据协议的规则进行惩罚。然而，如果这种情况发生在国外的链上，它将开始在对手链上为这个被破坏的链的轻客户意识到分叉而进行竞赛。

IBC协议提供了提交不当行为证明的功能，这可以由中继者提供，在此基础上，轻客户被冻结，以避免因分叉造成的后果。当攻击被消除后，资金可以通过治理建议解冻轻型客户机来恢复。因此，提交不当行为的功能使中继者能够增强IBC的安全性，尽管中继者本身本质上是不可信的。

#### 动态容量

IBC的目的是在模块之间不一定相互信任的执行环境中工作。因此，IBC必须验证模块在端口和通道上的操作，以便只有具有适当权限的模块才能使用它们。这种模块认证是通过一个动态能力存储来完成的。在为一个模块绑定到一个端口或创建一个通道时，IBC返回一个动态能力，该模块必须要求使用该端口或通道。动态能力模块防止其他模块使用该端口或通道，因为他们不拥有相应的能力。

值得一提的是，在IBC采取的特殊安全考虑之上，Cosmos SDK的安全考虑和Cosmos白皮书的特定应用链模型仍然有效。参考前面的章节，请记住，虽然模块上的迭代可能比合约上的迭代慢，但特定应用的链所暴露的攻击载体比部署在链上的智能合同设置要少得多。链上的人必须通过治理故意采用恶意模块。

#### 开发路线图

如前所述，尽管IBC起源于Cosmos技术栈，但它允许非Cosmos SDK构建的链采用IBC，甚至是那些与Tendermint完全不同的共识。然而，取决于你想为哪条链实现IBC或在其上构建IBC应用程序，它可能需要事先开发，以确保IBC工作所需的所有不同组件都可用于你选择的共识类型和区块链框架。

一般来说，你将需要以下东西。

- 一个IBC传输层的实现。
- 你的链上的轻客户端实现，以跟踪你想连接的对手方链。
- 你的共识类型的轻客户端实现，被纳入你想连接的对手链上。

##### 有IBC功能的链的路线图

下面的决策树有助于可视化实现IBC-enabled链的路线图。为了简单起见，它假设了连接到Cosmos SDK链的意图。  
  
你有机会接触到现有的链吗？  
  
- 没有。你必须建立一个链。  
    - Cosmos SDK链：见前面的章节。[Chapter Overview - First steps to run your own chain | Developer Portal (cosmos.network)](https://tutorials.cosmos.network/hands-on-exercise/1-ignite-cli/)   
    - 另一条链。  
        - 是否有一个Tendermint轻客户端的实现可用于你的链？  
            - 是的。继续。  
            - 没有。建立一个自定义的Tendermint轻客户端实现。  
        - 在SDK的IBC模块中是否有适用于你的链的共识的轻客户端实现？  
            - 是的。继续。  
            - 没有。为您的共识建立一个自定义的轻客户端，用于Cosmos SDK的链上（使用IBC模块）。  
         - 实现IBC核心。  
            - 源代码。  
            - 基于智能合约。  
- 是的。它是一个Cosmos SDK链吗？  
    - 是的。转入应用开发。  
    - 不是。  
        - 你的链支持Tendermint轻型客户端吗？  
            - 是的。继续。  
            - 没有。为你的链提供一个Tendermint轻客户端的实现。  
        - 你的链的共识是否有Cosmos SDK轻客户端实现？  
            - 是的。继续。  
            - 没有。构建一个你的链的共识的自定义轻客户端实现+治理建议，在你想连接到的Cosmos SDK链的IBC客户端上实现它。  
        - 你的链有可用的核心IBC模块吗（连接、通道、端口）？源代码或基于智能合约？  
            - 是的。继续进行应用开发。  
            - 没有。建立IBC核心实现（源代码或智能合约）。  
  

使用IBC最直接的方法是用Cosmos SDK建立一个链，它已经包括了IBC模块--正如你在检查IBC-Go资源库时看到的那样 [cosmos/ibc-go: Interblockchain Communication Protocol (IBC) implementation in Golang. (github.com)](https://github.com/cosmos/ibc-go) 。IBC模块支持一个开箱即用的Tendermint轻客户端。其他的实现是可能的，但可能需要进一步开发必要的组件；去IBC网站 [Inter-Blockchain Communication (ibcprotocol.org)](https://ibcprotocol.org/implementations/) 看看哪些实现在生产中可用或正在开发中。


## IBC-TAO 连接

TAO层(Transport, Authentication, Ordering Layer)。

现在你已经涵盖了介绍，并对不同的区块链间通信协议（IBC）组件和interchain标准（ICS）之间的关系有了更好的了解，请深入了解IBC/TAO和IBC模块 [cosmos/ibc-go: Interblockchain Communication Protocol (IBC) implementation in Golang. (github.com)](https://github.com/cosmos/ibc-go) 。

##### 连接

如果你想用IBC连接两个区块链，你将需要建立一个IBC连接。连接，通过四次握手建立，负责。

- 建立交易方链的身份。
- 防止恶意实体通过假装是交易方链来伪造不正确的信息。IBC连接是由链上ledger code建立的，因此不需要与链外（受信任的）第三方进程互动。

> 连接的语义在ICS-3中描述 [ibc/spec/core/ics-003-connection-semantics at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-003-connection-semantics) 。

在IBC堆栈中，连接是建立在客户端之上的，所以从技术上讲，如果客户端与IBC协议的多个版本进行交互，每个客户端可以有多个连接。目前，该设置应该意味着每个客户端有一个连接。

###### 版本协商

请注意，这里的版本是指IBC协议规范，而不是ibc-go模块。目前没有计划进行向后不兼容的更新。  
  
```go
// 版本定义了用于在连接握手中协商IBC版本的版本划分方案。  
// 连接握手时用于协商IBC版本。  
type Version struct {  
// 唯一的版本标识符  
Identifier string `protobuf: "bytes,1,opt,name=identifier,proto3" json: "identifier,omitempty"`.  
// 与指定标识符兼容的功能列表  
Features []string `protobuf: "bytes,2,opt,name=features,proto3" json: "features, omitempty"`.  
}
```

协议版本的建立很重要，因为不同的协议版本可能不兼容，例如，由于证明被存储在不同的路径上。有三种类型的协议版本协商。  
  
- 默认，不选择：只支持一个协议版本。这是默认提出的。  
- 有选择：可以提出两个协议版本，这样，启动`OpenInit`或`OpenTry`的链可以选择使用哪个版本。  
- 不可能的通信：一个向后不兼容的IBC协议版本。例如，如果一个IBC模块改变了它存储证明的位置（证明路径），就会产生错误。目前没有计划升级到一个向后不兼容的IBC协议版本。  

如前所述，开放握手协议允许每个链验证用于引用其他链上的连接的标识符，使每个链上的模块能够推理出其他链的引用。

![[Pasted image 20221118151332.png]]

关于另一边的连接，连接的protobufs[ibc-go/connection.proto at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/proto/ibc/core/connection/v1/connection.proto) 包含了`Counterparty`对手方的定义。

```protobuf
// Counterparty defines the counterparty chain associated with a connection end.
message Counterparty {
    option (gogoproto.goproto_getters) = false;

    // identifies the client on the counterparty chain associated with a given
    // connection.
    string client_id = 1 [(gogoproto.moretags) = "yaml:\"client_id\""];
    // identifies the connection end on the counterparty chain associated with a
    // given connection.
    string connection_id = 2 [(gogoproto.moretags) = "yaml:\"connection_id\""];
    // commitment merkle prefix of the counterparty chain.
    ibc.core.commitment.v1.MerklePrefix prefix = 3 [(gogoproto.nullable) = false];
}
```

在这个定义中，`connection-id`被用作一个key用于map和检索与商店中某个客户相关的连接。

`prefix`被客户用来构建Merkle prefix路径，然后用来验证证明。

##### 连接的握手和状态

建立一个IBC连接（例如，链A和链B之间）需要四次握手。

- OpenInit
- OpenTry
- OpenAck
- OpenConfirm

一个成功的四次握手的高层次概述如下。

###### 握手1 - OpenInit

`OpenInit`初始化任何可能发生的连接，同时仍然需要双方的同意。这就像IBC模块在链A上的识别公告，由中继者提交。在这次握手之前，中继者还应该提交一个以链A为源链的`UpdateClient`。`UpdateClient`用链B的最新共识状态更新初始化链A上的客户端。

![[Pasted image 20221118154529.png]]

从链A发起的这次握手将其连接状态更新为`INIT`。

`OpenInit`提出了一个用于IBC连接的协议版本。中继器提交的`OpenInit`如果包含一个链A不支持的协议版本，将被预期失败。

连接握手的参考实现可以在IBC模块库中找到 [ibc-go/handshake.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/03-connection/keeper/handshake.go) 。检查`ConnOpenInit`。

```go
func (k Keeper) ConnOpenInit(
    ctx sdk.Context,
    clientID string,
    counterparty types.Counterparty, // counterpartyPrefix, counterpartyClientIdentifier
    version *types.Version,
    delayPeriod uint64,
) (string, error) {
  ...

    // connection defines chain A's ConnectionEnd
    connectionID := k.GenerateConnectionIdentifier(ctx)
    connection := types.NewConnectionEnd(types.INIT, clientID, counterparty, types.ExportedVersionsToProto(versions), delayPeriod)
    k.SetConnection(ctx, connectionID, connection)

    if err := k.addConnectionToClient(ctx, clientID, connectionID); err != nil {
        return "", err
    }

    k.Logger(ctx).Info("connection state updated", "connection-id", connectionID, "previous-state", "NONE", "new-state", "INIT")

    defer func() {
        telemetry.IncrCounter(1, "ibc", "connection", "open-init")
        }()

    EmitConnectionOpenInitEvent(ctx, connectionID, clientID, counterparty)

    return connectionID, nil
}
```

这个函数创建一个唯一的`connectionID`。它将该连接添加到与特定客户相关的连接列表中。

它创建了一个新的`ConnectionEnd`。

```go
connection := types.NewConnectionEnd(types.INIT, clientID, counterparty, types.ExportedVersionsToProto(versions), delayPeriod)
k.SetConnection(ctx, connectionID, connection)

// ConnectionEnd defines a stateful object on a chain connected to another separate one.
// NOTE: there must only be 2 defined ConnectionEnds to establish
// a connection between two chains, so the connections are mapped and stored as `ConnectionEnd` on the respective chains.
message ConnectionEnd {
    option (gogoproto.goproto_getters) = false;
    // client associated with this connection.
    string client_id = 1 [(gogoproto.moretags) = "yaml:\"client_id\""];
    // IBC version which can be utilised to determine encodings or protocols for
    // channels or packets utilising this connection.
    repeated Version versions = 2;
    // current state of the connection end.
    State state = 3;
    // counterparty chain associated with this connection.
    Counterparty counterparty = 4 [(gogoproto.nullable) = false];
    // delay period that must pass before a consensus state can be used for
    // packet-verification NOTE: delay period logic is only implemented by some
    // clients.
    uint64 delay_period = 5 [(gogoproto.moretags) = "yaml:\"delay_period\""];
}
```

`ConnOpenInit`由中继器触发，中继器构建消息并将其发送到SDK，SDK使用之前看到的`msg_server.go` 
 [ibc-go/msg_server.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/keeper/msg_server.go) 来调用`ConnOpenInit`。

```go
// ConnectionOpenInit defines a rpc handler method for MsgConnectionOpenInit.
func (k Keeper) ConnectionOpenInit(goCtx context.Context, msg *connectiontypes.MsgConnectionOpenInit) (*connectiontypes.MsgConnectionOpenInitResponse, error) {
    ctx := sdk.UnwrapSDKContext(goCtx)

    if _, err := k.ConnectionKeeper.ConnOpenInit(ctx, msg.ClientId, msg.Counterparty, msg.Version, msg.DelayPeriod); err != nil {
        return nil, sdkerrors.Wrap(err, "connection handshake open init failed")
    }

    return &connectiontypes.MsgConnectionOpenInitResponse{}, nil
}
```

###### 握手2 - OpenTry

`OpenInit`之后是`OpenTry`响应，其中B链根据B链在其轻客户端中关于A链的信息（算法和共识状态的最后快照，包含最新高度的root hash以及下一个validators集）来验证A链的身份。它也会对链A的`OpenInit`公告中关于其自身身份的一些信息做出回应。

![[Pasted image 20221118155251.png]]

握手的这一步的目的是双重验证：不仅让链B验证链A是预期的对手方身份，而且还验证对手方拥有关于链B身份的准确信息。在这次握手之前，中继器还提交了两个以链A和链B为源链的`UpdateClients`。这些更新了链A和链B的轻客户端，以确保本步骤中的状态验证是成功的。

从链B发起的这次握手将其连接状态更新为`TRY`。

关于IBC协议版本，`OpenTry`要么接受`OpenInit`中提出的协议版本，要么从链A可用的版本中提出另一个协议版本，用于IBC连接。中继器提交的包含不支持的协议版本的`OpenTry`将被预期为失败。

OpenTry的实现如下。

```go
// ConnOpenTry relays notice of a connection attempt on chain A to chain B (this
// code is executed on chain B).
//
// NOTE:
//  - Here chain A acts as the counterparty
//  - Identifiers are checked on msg validation
func (k Keeper) ConnOpenTry(
    ctx sdk.Context,
    previousConnectionID string, // previousIdentifier
    counterparty types.Counterparty, // counterpartyConnectionIdentifier, counterpartyPrefix and counterpartyClientIdentifier
    delayPeriod uint64,
    clientID string, // clientID of chainA
    clientState exported.ClientState, // clientState that chain A has for chain B
    counterpartyVersions []exported.Version, // supported versions of chain A
    proofInit []byte, // proof that chain A stored connectionEnd in state (on ConnOpenInit)
    proofClient []byte, // proof that chain A stored a light client of chain B
    proofConsensus []byte, // proof that chain A stored chainB's consensus state at consensus height
    proofHeight exported.Height, // height at which relayer constructs proof of A storing connectionEnd in state
    consensusHeight exported.Height, // latest height of chain B which chain A has stored in its chain B client
) ...
```

###### 握手3 - OpenAck

`OpenAck`与`OpenInit`的功能非常相似，只是现在的信息验证是针对链A的。与`OpenTry`一样，中继器在这次握手前也提交了两个`UpdateClients`，链A和链B是源链。这些更新了链A和链B的轻客户端，以确保这一步的状态验证是成功的。

![[Pasted image 20221118155938.png]]

从链A发起的这次握手将其连接状态更新为`OPEN`。值得注意的是，对手链必须有`TRY`连接状态，才能使握手和连接状态更新成功。

关于版本协商，`OpenAck`必须确认已在`OpenTry`中提出的协议版本。如果版本不受欢迎或不被支持，它就会结束连接握手过程。

`OpenAck`的代码与`OpenTry`非常相似。

```go
func (k Keeper) ConnOpenAck(
    ctx sdk.Context,
    connectionID string,
    clientState exported.ClientState, // client state for chain A on chain B
    version *types.Version, // version that Chain B chose in ConnOpenTry
    counterpartyConnectionID string,
    proofTry []byte, // proof that connectionEnd was added to Chain B state in ConnOpenTry
    proofClient []byte, // proof of client state on chain B for chain A
    proofConsensus []byte, // proof that chain B has stored ConsensusState of chain A on its client
    proofHeight exported.Height, // height that relayer constructed proofTry
    consensusHeight exported.Height, // latest height of chain A that chain B has stored on its chain A client
) ...
```

这两个函数做同样的检查，只是`OpenTry`以`proofInit`为参数，而`OpenAck`以`proofTry`为参数。

```go
// This function verifies that the snapshot we have of the counter-party chain looks like the counter-party chain, verifies the light client we have of the counter-party chain
// Check that Chain A committed expectedConnectionEnd to its state
if err := k.VerifyConnectionState(
    ctx, connection, proofHeight, proofTry, counterparty.ConnectionId,
    expectedConnection,
); err != nil {
    return "", err
}

// This function verifies that the snapshot the counter-party chain has of us looks like us, verifies our light client on the counter-party chain
// Check that Chain A stored the clientState provided in the msg
if err := k.VerifyClientState(ctx, connection, proofHeight, proofClient, clientState); err != nil {
    return "", err
}

// This function verifies that the snapshot the counter-party chain has of us looks like us, verifies our light client on the counter-party chain
// Check that Chain A stored the correct ConsensusState of chain B at the given consensusHeight
if err := k.VerifyClientConsensusState(
    ctx, connection, proofHeight, consensusHeight, proofConsensus, expectedConsensusState,
); err != nil {
    return "", err
}
```

因此，每个链都会验证另一个链的`ConnectionState`、`ClientState`和`ConsensusState`。注意，在这一步之后，链A的连接状态从`INIT`更新为`OPEN`。

###### 握手4 - OpenConfirm

`OpenConfirm`是最后一次握手，其中链B确认自我识别和对手方识别都是成功的。

![[Pasted image 20221118160306.png]]

这次握手的结果是成功建立了一个IBC连接。

```go
func (k Keeper) ConnOpenConfirm(
    ctx sdk.Context,
    connectionID string,
    proofAck []byte, // proof that connection opened on Chain A during ConnOpenAck
    proofHeight exported.Height, // height that relayer constructed proofAck
)
```

来自B链的这次握手将其连接状态从`TRY`更新为`OPEN`。对方链必须有一个开放的连接状态，以便握手和连接状态的更新能够成功。

所描述的成功的四向握手在两个链之间建立了一个IBC连接。现在考虑两种相关的情况：两个链同时试图执行相同的握手，以及冒名顶替者试图进行干扰。


##### 同时问候（crossing hellos）

"crossing hellows "是指两个链同时尝试同一个握手步骤。

如果两条链同时提交`OpenInit`和`OpenTry`，应该不会出现错误。在这种情况下，双方仍然需要用一个成功的`OpenAck`来确认，但不需要`OpenConfirm`，因为两个`ConnectionEnds`都将更新为`OPEN`状态。

##### 一个冒牌货

事实上，这不是一个问题。任何来自冒名顶替者的`OpenInit`尝试都会在`OpenTry`上失败，因为它不包含Client/Connection/ConsensusState的有效证明。

## IBC-TAO 通道

连接和客户端构成了IBC中传输层的主要组成部分。然而，IBC中应用与应用之间的通信是通过信道(channel)进行的，信道在一个应用模块（如interchain标准（ICS）20代币传输的模块）和另一个相应的应用模块之间路由。这些应用由端口标识符命名，如ICS-20代币传输的 "transfer"。  
  
  
> 注意，在链间账户的情况下，host和controller模块有两个不同的端口ID。  
> 
>`icahost`是默认的端口ID，链间账户host子模块与之绑定，而`icacontroller-`是默认的端口前缀，链间账户controller子模块与之绑定。  
  
与IBC的核心传输层逻辑相反，它只处理验证、排序(ordering)和所有周围的基本数据包的正确性，通道上的应用层只处理特定的应用逻辑，它解释已经通过传输层发送的数据包。IBC中传输层和应用层之间的这种分割类似于Tendermint的共识层（共识、mempool、交易的排序）和ABCI层（这些交易字节的处理）之间的分割。  
  
一个连接可以有任何数量的相关通道。然而，每个通道只与一个连接ID相关联（一对多的关系），该ID表明它是由哪个轻客户端保证的，还有一个端口ID，表明它所连接的应用程序。  
  
如上所述，通道是不可知的payload。发送和接收IBC数据包的应用模块决定如何解释和处理传入的数据包，并使用他们自己的应用逻辑和处理程序来决定根据每个收到的数据包中包含的数据应用哪些状态转换。  
  
> 其实到这里来看，IBC本身就是一个TCP/UDP的协议。应用层来解释这些数据字节。

> 有序信道是一个信道，其中数据包完全按照它们被发送的顺序交付。（类似TCP）
> 无序信道是一个信道，数据包可以按照任何顺序交付，这可能与它们被发送的顺序不同。 (类似UDP)

##### 建立一个通道

与建立连接的方式类似，信道是通过四次握手建立的，其中每一步都是由中继者发起。

![[Pasted image 20221118162234.png]]

- `ChanOpenInit`: 将设置链A进入`INIT`状态。这将调用`OnChanOpenInit`，因此应用程序A可以应用它在`INIT`时设置的自定义回调，例如，检查端口是否被正确设置，通道是否确实如预期那样无序/有序，等等。在这个步骤中还提出了一个应用版本。
- `ChanOpenTry`：将设置链B进入`TRY`状态。它将调用`OnChanOpenConfirm`，所以应用B可以应用它的自定义`TRY`回调。应用程序的版本协商也发生在这个步骤中。
- `ChanOpenAck`: 将把链A设置为`Open`状态。这将调用`OnChanOpenAck`，它将由应用程序实现。应用程序的版本协商在此步骤中完成。
- `ChanOpenConfirm`: 将把链B设置为`Open`状态，这样应用程序B就可以应用其`CONFIRM`逻辑。

> 从上面看，Channel感觉还是需要应用模块参与的

##### 样例代码: ChannelOpenInit

你可以在`msg_server.go`  [ibc-go/msg_server.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/keeper/msg_server.go) 中找到`ChannelOpenInit`的实现。

在这个代码片段中，需要注意的重要部分是，一个应用模块对所请求的端口拥有能力。因此，只有当应用程序拥有一个端口的能力，并且试图打开一个通道的模块是我们在`app.go`中授予能力的模块时，应用程序模块才能使用该通道和端口。

```go
// ChannelOpenInit defines a rpc handler method for MsgChannelOpenInit.
// ChannelOpenInit will perform 04-channel checks, route to the application
// callback, and write an OpenInit channel into state upon successful execution.
func (k Keeper) ChannelOpenInit(goCtx context.Context, msg *channeltypes.MsgChannelOpenInit) (*channeltypes.MsgChannelOpenInitResponse, error) {
    ctx := sdk.UnwrapSDKContext(goCtx)

    // Lookup module by port capability
    module, portCap, err := k.PortKeeper.LookupModuleByPort(ctx, msg.PortId)
    if err != nil {
        return nil, sdkerrors.Wrap(err, "could not retrieve module from port-id")
    }

    ...

    // Perform 04-channel verification
    channelID, cap, err := k.ChannelKeeper.ChanOpenInit(
        ctx, msg.Channel.Ordering, msg.Channel.ConnectionHops, msg.PortId,
        portCap, msg.Channel.Counterparty, msg.Channel.Version,
    )

    ...

    // Perform application logic callback
    if err = cbs.OnChanOpenInit(ctx, msg.Channel.Ordering, msg.Channel.ConnectionHops, msg.PortId, channelID, cap, msg.Channel.Counterparty, msg.Channel.Version); err != nil {
        return nil, sdkerrors.Wrap(err, "channel open init callback failed")
    }

    // Write channel into state
    k.ChannelKeeper.WriteOpenInitChannel(ctx, msg.PortId, channelID, msg.Channel.Ordering, msg.Channel.ConnectionHops, msg.Channel.Counterparty, msg.Channel.Version)

    return &channeltypes.MsgChannelOpenInitResponse{
        ChannelId: channelID,
    }, nil
}
```

> 能力
> 
> IBC的目的是在模块之间不一定相互信任的执行环境中工作。这种安全性是通过一个动态的能力存储 [cosmos-sdk/adr-003-dynamic-capability-store.md at main · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-003-dynamic-capability-store.md) 来实现的。这种绑定策略可以防止其他模块使用特定的端口或通道，因为这些模块并不拥有相应的能力。
> 
> 虽然这些背景信息很有用，但IBC应用开发者除了在`app.go`中适当地设置能力外，不应该需要修改这个较低层次的抽象。

##### 应用数据报(packet)流

如前所述，应用模块通过IBC通道发送数据包(packets)来相互通信。然而，IBC模块并不直接通过网络将这些消息传递给对方。相反，模块将把反映交易执行的一些状态提交给为特定消息类型和特定对手方保留的精确定义的路径。例如，作为ICS-20代币转移的一部分，bank模块将托管要转移的那部分代币，并存储这个托管的证明。

中继器将监测通道，以了解当更新被提交到这些路径时发出的事件，并且（在首先提交`UpdateClient`以更新目的链上的发送链轻客户端之后）中继包含数据包的消息，同时证明消息中包含的状态转换已经被写入发送链的状态。然后，目的链根据轻客户端中包含的状态来验证这个数据包和数据包承诺证明。

看一下数据包定义 [ibc-go/packet.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/04-channel/types/packet.go) ，看看数据包结构。

```go
// NewPacket creates a new Packet instance. It panics if the provided
// packet data interface is not registered.
func NewPacket(
    data []byte,
    sequence uint64, sourcePort, sourceChannel,
    destinationPort, destinationChannel string,
    timeoutHeight clienttypes.Height, timeoutTimestamp uint64,
) Packet {
    return Packet{
        Data:               data,
        Sequence:           sequence,
        SourcePort:         sourcePort,
        SourceChannel:      sourceChannel,
        DestinationPort:    destinationPort,
        DestinationChannel: destinationChannel,
        TimeoutHeight:      timeoutHeight,
        TimeoutTimestamp:   timeoutTimestamp,
    }
}
```

`Sequence`表示通道中数据包的序列号。

`TimeoutTimestamp`和`TimeoutHeight`决定了接收模块必须处理一个数据包之前的时间。

![[Pasted image 20221118164017.png]]

##### 成功的情况

在一个成功的数据包流的第一步，应用程序A将发送一个数据包（调用`sendPacket`）给应用程序B。`SendPacket`可以由用户触发，但应用程序也可以作为一些其他应用逻辑的结果来触发。  
  
核心IBC A将把数据包提交给自己的状态，中继器可以查询这个数据包，并向核心IBC B发送`RecvPacket`消息。核心IBC处理一些验证，包括验证数据包确实是由链A发送的，如果数据包是通过有序通道发送的，那么数据包的顺序是正确的，状态承诺证明是有效的，等等。如果这个验证步骤是成功的，核心IBC就会把数据包路由到应用B。  
  
请注意，核心IBC对数据包的实际内容没有意见，因为这些数据在这一点上只是字节。两端的应用程序有责任将数据从两边的预期数据结构中提取和释放出来。这也是为什么之前在信道握手中讨论的应用程序版本协商是很重要的，因为不同版本的应用程序可能会导致信道和应用程序两端的预期数据结构不同。  

> 相当于这个数据结构是预先需要用户自定义的？那么这个数据包跟substrate中的XCM是一样了？只是XCM这个数据格式有限制
  
在收到核心IBC的数据包后，应用B将把数据包整合成预期的结构并应用相关的应用逻辑。例如，在ICS-20代币转移的情况下，这将需要在B链上将收到的代币铸币给指定的接收者用户账户。然后，应用程序B将发送一个确认消息给核心IBC B，它将再次把它提交给自己的状态，以便它可以被查询并由中继器发送到核心IBC A。  
  
> 同步和异步确认  
> 
> 确认既可以同步进行，也可以异步进行。这意味着`OnRecvPacket`回调有一个返回值`Acknowledgement`，这是可选的。  
> 
> 在同步确认的情况下，回调将在进程结束时返回一个确认，中继器可以查询这个`Acknowledgement`包，并在进程结束后立即中继。这在应用A期待一个`AckPacket`以启动一些应用逻辑`OnAcknowledgePacke`t的情况下很有用。例如，ICS-20 token传输的发送链在`AckPacket`成功的情况下将不做任何事情，但在返回错误的情况下，发送链将解除之前锁定的token。  
> 
> 在Interchain Security这样的应用中，有一个异步的`Acknowledgement`流程。这意味着确认不是作为`OnRecvPacket`返回值的一部分被发送，而是在以后的某个时间点被发送。IBC的设计是为了处理这种情况，允许异步提交或查询确认。  
> 
> 在这两种情况下，即使没有应用特定的逻辑要作为收到的确认的直接结果来启动，`OnAcknowledgePacket`至少也会从存储中移除承诺证明，以避免用旧数据扰乱存储。  
  
##### 超时的情况

如果一个数据包是时间敏感的，并且基于链B的时间在数据包参数中指定的超时块高度或超时时间戳已经过去，无论由于发送数据包而发生了什么状态转换，都应该被revert。

在这些情况下，初始流程是相同的，核心IBC A首先将数据包提交给它自己的状态。然而，中继器将提交一个`QueryNonReceipt`，以收到核心IBC B没有收到数据包的证明，而不是查询数据包，然后它可以向核心IBC A发送`TimeoutPacket`，这将触发相关的`OnTimeoutPacket`应用逻辑。例如，ICS-20 token转移应用将解除锁定的token，并将这些token送回原发件人`OnTimeoutPacket`。

## IBC-TAO 客户端

![[Pasted image 20221118165117.png]]

如前所述，IBC的结构是由几个抽象层组成的。在顶层，应用程序，如链间标准（ICS）20代币转移 
 [ibc/spec/app/ics-020-fungible-token-transfer at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/app/ics-020-fungible-token-transfer)  实现ICS-26 IBC标准 [ibc/README.md at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/blob/main/spec/core/ics-026-routing-module/README.md) ，描述了用于连接应用程序层和传输层的路由和回调功能。在应用程序下面是通道(channel)，每个应用程序都是唯一的（例如，一个通道允许A链上的传输应用程序与B链上的传输应用程序对话）。连接，可能有许多通道，用于连接两个客户端（例如，允许A链的整个IBC堆栈连接到B链的IBC堆栈）。这些可能有许多连接的客户端，构成了IBC的基础层。  
  
> IBC应用开发者将主要与IBC通道互动  [[#IBC-TAO 通道]] 。这一层是由握手和数据包回调组成的。  
  
在IBC设置中，每个链在自己的IBC堆栈中都会有一个其他链的客户端(轻客户端)。IBC客户端跟踪其他区块链的共识状态，以及这些区块链的证明specs，这些区块链需要针对客户端的共识状态正确验证证明。链外中继者(relayers)来回发送的数据包、确认和超时可以通过证明每个链上的这些客户端内部存在的数据包承诺来验证。  

> 虽然中继者不对数据包进行任何验证，因此不需要被信任，但中继者除了通过提交数据包来实现IBC网络的有效性外，在IBC设置中还具有特别重要的作用。他们负责提交初始消息以创建一个新的客户端，以及保持每个链上的客户端状态的更新，以便对提交的数据包进行验证成功。中继者还负责发送连接和通道握手，以建立链之间的连接和通道。此外，如果连接的另一端的链试图分叉或尝试其他类型的恶意行为，中继者可以提交不当行为的证据。

#### 创建一个客户端

从`msg_serve.go` [ibc-go/msg_server.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/keeper/msg_server.go) 开始，这就是消息的来历。这是CreateClient函数的第一次出现，它将由中继者通过中继软件提交，在消息所提交的链上创建一个IBC客户端。

```go
// CreateClient defines a rpc handler method for MsgCreateClient.
func (k Keeper) CreateClient(goCtx context.Context, msg *clienttypes.MsgCreateClient) (*clienttypes.MsgCreateClientResponse, error) {
    ctx := sdk.UnwrapSDKContext(goCtx)

    clientState, err := clienttypes.UnpackClientState(msg.ClientState)
    ...

    consensusState, err := clienttypes.UnpackConsensusState(msg.ConsensusState)

    ...

    ... = k.ClientKeeper.CreateClient(ctx, clientState, consensusState);

    ...
}
```

通过调用`ClientKeeper.CreateClient` [ibc-go/client.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/02-client/keeper/client.go) 创建一个客户端

```go
// CreateClient creates a new client state and populates it with a given consensus
// state as defined in https://github.com/cosmos/ibc/tree/master/spec/core/ics-002-client-semantics#create
func (k Keeper) CreateClient(
    ctx sdk.Context, clientState exported.ClientState, consensusState exported.ConsensusState,
)
    ...

    clientID := k.GenerateClientIdentifier(ctx, clientState.ClientType())

    ...

    k.SetClientState(ctx, clientID, clientState)

    ...

    // verifies initial consensus state against client state and initializes client store with any client-specific metadata
    // e.g. set ProcessedTime in Tendermint clients
    ... := clientState.Initialize(ctx, k.cdc, k.ClientStore(ctx, clientID), consensusState);

    ...

    EmitCreateClientEvent(ctx, clientID, clientState)

    return clientID, nil
}
```

为链上的每个客户生成一个本地的、唯一的标识符`clientID`。这与`chainID`没有关系，因为IBC实际上不使用`chainID`作为标识符。  
  
  
> IBC的安全模型是基于客户端而不是具体的链。这意味着IBC协议不需要知道连接两边的链是谁，只要IBC客户端与有效的更新保持同步，并且这些更新或其他类型的消息（即ICS-20 tokne传输）可以被验证为针对初始共识状态（root of trust）的Merkle Proof。这类似于IP地址和DNS，其中IP地址将是IBC `clientIDs`的推论，而DNS是`chainIDs`。  
> 
> 由于这种关注点的分离，IBC客户端可以为任何数量的机器类型创建，从完全成熟的区块链到基于密钥对的单机，并且对链的升级，增加`chainID`不会破坏底层的IBC客户端和连接。  
  
此外，你可以看到该函数期望有一个`ClientState`。这个`ClientState`将根据为IBC创建的客户类型而有所不同。在Cosmos-SDK链和ibc-go的相应实现中，Tendermint客户端 [ibc-go/client_state.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/light-clients/07-tendermint/client_state.go) 被开箱提供。  

```go
// NewClientState creates a new ClientState instance
func NewClientState(
    chainID string, trustLevel Fraction,
    trustingPeriod, ubdPeriod, maxClockDrift time.Duration,
    latestHeight clienttypes.Height, specs []*ics23.ProofSpec,
    upgradePath []string, allowUpdateAfterExpiry, allowUpdateAfterMisbehaviour bool,
) *ClientState {
    return &ClientState{
        ChainId:                      chainID,
        TrustLevel:                   trustLevel,
        TrustingPeriod:               trustingPeriod,
        UnbondingPeriod:              ubdPeriod,
        MaxClockDrift:                maxClockDrift,
        LatestHeight:                 latestHeight,
        FrozenHeight:                 clienttypes.ZeroHeight(),
        ProofSpecs:                   specs,
        UpgradePath:                  upgradePath,
        AllowUpdateAfterExpiry:       allowUpdateAfterExpiry,
        AllowUpdateAfterMisbehaviour: allowUpdateAfterMisbehaviour,
    }
}
```

Tendermint `ClientState`包含了验证一个头所需的所有信息。这包括适用于所有Tendermint客户端的属性，如相应的chainID，链的unbonding period，客户端的最新高度等。  
  
信任期（TrustingPeriod）决定了自最新的时间戳以来的持续时间，在此期间，提交的头信息对升级有效。如果一个客户端在TrustingPeriod内没有更新，该客户端将过期。这并不意味着该客户端是不可恢复的。然而，恢复一个过期的Tendermint客户端将需要对每个过期的客户端提出治理建议[How to recover an expired client with a governance proposal | IBC-Go (cosmos.network)](https://ibc.cosmos.network/main/ibc/proposals.html#preconditions) 。如果一个连接两边的客户端都过期了，那么每条链上都需要一个治理建议，以便恢复每个客户端。  
  
TrustLevel决定了你想要签署一个头的validators集的部分，以便它被认为是有效的。Tendermint将其定义为2/3，IBC Tendermint客户端从Tendermint继承这个属性。  
  
  
> 诸如TrustLevel和TrustingPeriod这样的属性可以被定制，这样同一链条上的不同客户端可以有不同的安全保证，并对处理更新的效率有不同的权衡。  
  
> 必须强调的是，IBC客户端的某些参数在客户端创建后不能更新，以保持每个客户端的安全保障，防止出现中继器单方面更新这些安全保障的情况。这些参数是 MaxClockDrift、TrustingPeriod和TrustLevel。  
> 
> 如前所述，TrustLevel是从Tendermint继承的，对所有Tendermint客户端来说，它将是2/3。然而，对于其他类型的客户，这可能会改变。  
> 
> 建议TrustingPeriod应该被设置为UnbondingPeriod的2/3。  
> 
> 还建议MaxClockDrift应该设置为至少5秒，最多15秒，这取决于连接中链之间的预期块大小差异。如果你没有手动设置，Hermes（Rust）中继器将为你计算这个值。  
  
CreateClient还期望有一个ConsensusState [ibc-go/consensus_state.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/light-clients/07-tendermint/consensus_state.go) 。在Tendermint客户端的情况下，初始root of trust（或共识状态）看起来像这样。  
  
```go
// NewConsensusState creates a new ConsensusState instance.
func NewConsensusState(
    timestamp time.Time, root commitmenttypes.MerkleRoot, nextValsHash tmbytes.HexBytes,
) *ConsensusState {
    return &ConsensusState{
        Timestamp:          timestamp,
        Root:               root,
        NextValidatorsHash: nextValsHash,
    }
}
```

Tendermint客户端的`ConsensusState`跟踪正在创建的区块的时间戳，为对手方区块链的下一个区块设置的validator的哈希值，以及对手方区块链的root。最初的`ConsensusState`不需要从对手方链的创世区块开始。  
  
> 下一个validators集用于验证随后提交的标题或对对手方ConsensusState的更新。关于validators集在区块之间发生变化时的更多信息，请参见下面关于更新客户端的部分 [[#更新一个客户端]] 。  
  
root是`AppHash`，或该客户端所代表的对手方区块链的应用状态的hash value。这个root hash特别重要，因为它是接收链在验证与通过IBC来的数据包相关的Merkle  [Merkle tree - Wikipedia](https://en.wikipedia.org/wiki/Merkle_tree) proof时使用的root hash，以确定相关交易是否已经在发送链上实际执行。如果与中继器交付的数据包承诺相关的Merkle proof成功哈希到这个`ConsensusState` root hash，就可以肯定该交易在发送链上实际执行，并包括在发送区块链的状态中。  
  
下面是Tendermint客户端如何处理这种Merkle proof验证的例子 [ibc-go/merkle.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/23-commitment/types/merkle.go) 。ICS-23规范  [ibc/spec/core/ics-023-vector-commitments at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-023-vector-commitments) 涉及如何构建membership proof，ICS-23实现  [cosmos/ics23: Building generic merkle proof format for IBC (github.com)](https://github.com/cosmos/ics23) 目前支持Tendermint IAVL和简单的Merkle proof开箱即用。请注意，非Tendermint客户类型可以选择以不同的方式处理证明验证。  
  
```go
// VerifyMembership verifies the membership of a merkle proof against the given root, path, and value.
func (proof MerkleProof) VerifyMembership(specs []*ics23.ProofSpec, root exported.Root, path exported.Path, value []byte) error {
    if err := proof.validateVerificationArgs(specs, root); err != nil {
        return err
    }

    // VerifyMembership specific argument validation
    mpath, ok := path.(MerklePath)
    if !ok {
        return sdkerrors.Wrapf(ErrInvalidProof, "path %v is not of type MerklePath", path)
    }
    if len(mpath.KeyPath) != len(specs) {
        return sdkerrors.Wrapf(ErrInvalidProof, "path length %d not same as proof %d",
            len(mpath.KeyPath), len(specs))
    }
    if len(value) == 0 {
        return sdkerrors.Wrap(ErrInvalidProof, "empty value in membership proof")
    }

    // Since every proof in chain is a membership proof we can use verifyChainedMembershipProof from index 0
    // to validate entire proof
    if err := verifyChainedMembershipProof(root.GetHash(), specs, proof.Proofs, mpath, value, 0); err != nil {
        return err
    }
    return nil
}
```

> IBC链上客户也可以称为轻客户。与追踪整个区块链状态并包含每一个tx/block的完整节点相比，这些链上IBC "轻客户端 "只追踪前面提到的关于对手方链的少数信息（时间戳、root hash、下一个validators集哈希）。这样可以节省空间，提高处理共识状态更新的效率。
> 
> 其目的是为了避免出现这样的情况：为了建立一个trustless的IBC连接，必须在链A上有一个链B的副本。然而，跟踪区块链整个状态的全节点对于IBC relayer operators来说是有用的，因为它是查询验证IBC数据包承诺所需的证明的一个终端。这整个过程保持了IBC的无信任、无许可和高度安全的设计。由于证明验证仍然发生在IBC客户端本身，因此不需要信任relayer operators，任何人都可以无权限地启动中继操作，只要他们能够访问一个全节点端点。

#### 更新一个客户端

假设最初的`ConsensusState`是在区块50创建的，但你想提交一个发生在区块100的交易证明。在这种情况下，你需要首先更新`ConsensusState`，以反映区块50和区块100之间发生的所有状态变化。

为了更新客户端上交易方的`ConsensusState`，必须由中继器提交一个包含要更新的链`Header`的`UpdateClient`消息。对于所有的IBC客户类型，不管是Tendermint还是其他，这个`Header`包含了更新`ConsensusState`所需的信息。然而，除了返回`ClientType`和`GetClientID`的基本方法之外，IBC并没有规定Header必须包含什么。每个客户端期望执行`ConsensusState`更新的重要信息的具体内容将在每个客户端实现中找到。

例如，Tendermint客户端的`Header`看起来像这样 [ibc-go/tendermint.pb.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/light-clients/07-tendermint/tendermint.pb.go) 。

```go
type Header struct {
    *types2.SignedHeader `protobuf:"bytes,1,opt,name=signed_header,json=signedHeader,proto3,embedded=signed_header" json:"signed_header,omitempty" yaml:"signed_header"`
    ValidatorSet         *types2.ValidatorSet `protobuf:"bytes,2,opt,name=validator_set,json=validatorSet,proto3" json:"validator_set,omitempty" yaml:"validator_set"`
    TrustedHeight        types.Height         `protobuf:"bytes,3,opt,name=trusted_height,json=trustedHeight,proto3" json:"trusted_height" yaml:"trusted_height"`
    TrustedValidators    *types2.ValidatorSet `protobuf:"bytes,4,opt,name=trusted_validators,json=trustedValidators,proto3" json:"trusted_validators,omitempty" yaml:"trusted_validators"`
}
```

Tendermint `SignedHeader`是交易方链创建的头和提交。在`UpdateClient`的例子中，这将是区块100的头，它将包含区块的时间戳，下一个validators集的哈希值，以及更新对手链记录中的`ConensusState`所需的root hash。提交将是对该header的至少2/3的validators集的签名，这作为Tendermint的BFT共识模型的一部分得到保证。  
  
`ValidatorSet`将是实际的validators集，而不是存储在`ConsensusState`上的下一个validators集的哈希。这对Tendermint `UpdateClient`很重要，因为为了维护Tendermint的安全模型，有必要能够证明至少有2/3的validators在块50处签署了初始Header，并签署了头，将`ConsensusState`更新到块100。这个`ValidatorSet`将由中继器提交，作为`UpdateClient`消息的一部分，因为中继器可以访问全节点，可以从中提取这些信息。  
  
`TrustedValidators`是与该高度相关的validators。注意，`TrustedValidators`必须哈希到`ConsensusState` `NextValidatorsHas`h，因为那是在`TrustedHeight`设置的最后一个受信任的validator。  
  
`TrustedHeight`是客户端上存储的`ConsensusState`的高度，它将被用来验证新的不可信的header。你可以在这里看到获取`TrustedHeight`处的`ConsensusState`并使用它来验证新header的代码 [ibc-go/update.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/light-clients/07-tendermint/update.go) 。这段代码证明了提交的header是有效的，并为提交的header创建了一个经过验证的`ConsensusState`，以及更新客户端状态以反映提交header的最新高度。这个经过验证的`ConsensusState`将被添加到客户端，作为`ClientConsensusStates`集合的一部分，随后可以作为其相应高度的可信状态。  
  
>如果你想看ConsensusState的存储位置，请看Interchain标准（ICS）24 [ibc/spec/core/ics-024-host-requirements at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-024-host-requirements) ，它还描述了其他密钥被IBC存储和使用的路径。  
  

#### 校验数据包(packet)的承诺(commitment)

如在channels的深入研究中所示，中继者将首先提交`UpdateClient`以更新目的链上的发送链客户端，然后再中继包含其他消息类型的数据包，如ICS-20 token传输。目的地链可以确定该数据包将包含在其`ConsensusState` root hash中，并成功地验证该数据包和数据包承诺证明与它的（更新的）IBC轻型客户端中包含的状态。

说明客户端如何验证传入数据包的代码片段 [ibc-go/client_state.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/light-clients/07-tendermint/client_state.go) 如下。

```go
// VerifyPacketCommitment verifies a proof of an outgoing packet commitment at
// the specified port, specified channel, and specified sequence.
func (cs ClientState) VerifyPacketCommitment(
    ctx sdk.Context,
    store sdk.KVStore,
    cdc codec.BinaryCodec,
    height exported.Height,
    delayTimePeriod uint64,
    delayBlockPeriod uint64,
    prefix exported.Prefix,
    proof []byte,
    portID,
    channelID string,
    sequence uint64,
    commitmentBytes []byte,
) error {
    merkleProof, consensusState, err := produceVerificationArgs(store, cdc, cs, height, prefix, proof)
    ...

    // check delay period has passed
    if err := verifyDelayPeriodPassed(ctx, store, height, delayTimePeriod, delayBlockPeriod);
    ...

    commitmentPath := commitmenttypes.NewMerklePath(host.PacketCommitmentPath(portID, channelID, sequence))
    path, err := commitmenttypes.ApplyPrefix(prefix, commitmentPath)
    ...

    if err := merkleProof.VerifyMembership(cs.ProofSpecs, consensusState.GetRoot(), path, commitmentBytes);
    ...

    return nil
}
```


## IBC token转移

看过了IBC的传输、认证和order层（IBC/TAO），现在可以看看ICS-20 [ibc/README.md at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/blob/main/spec/app/ics-020-fungible-token-transfer/README.md) 。ICS-20描述了同质化代币的传输。


> 同质化指的是一个代币可以与该代币的其他实例互换或不互换。同质化的代币可以被交换和替换。

在区块链上有许多涉及代币转移的用例，如持有价值的资产的代币化或为区块链项目融资的首次代币发行（ICOs）。IBC使代币和其他数字资产在（主权）链之间转移成为可能，包括同质化的和非同质化的代币。例如，同质化的代币转移允许你建立依赖于跨链支付和代币交换的应用程序。因此，IBC通过提供一个技术上可靠的跨链互操作协议，与多个网络上的数字资产兼容，为跨链去中心化金融（DeFi）应用释放了巨大潜力。

相应的实现  [ibc-go/modules/apps/transfer at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/tree/main/modules/apps/transfer) 是应用层面的一个模块。

![[Pasted image 20221121104438.png]]

请看上面的图片。你可以看到两个链，A和B。你还看到有一个通道连接这两个链。

代币如何在链和通道之间转移？

为了理解代币转移的应用逻辑，首先，你必须确定源链(source chain / origin chain)。

![[Pasted image 20221121104538.png]]

然后可以总结出应用逻辑。

![[Pasted image 20221121104601.png]]

很快你就会看到相应的代码。现在再来看看从source到sink的转移。

![[Pasted image 20221121104632.png]]

source上方是链A，源通道是`channel-2`，目的通道是`channel-40`。代币面额表示为{port}/{channel}/{denom}。前缀的port和channel对表示资金以前是通过哪个通道发送的。你看到`transfer/channel-...`是因为transfer模块将绑定到一个端口，这个端口被命名为transfer。如果链A发送了100个ATOM代币，链B将收到100个ATOM代币并附加目的地前缀`port/channel-id`。所以链B将把这100个ATOM令牌作为`transfer/channel-40/atoms`。在一个给定的连接上，每个通道的`channel-id`将被依次增加。

如果代币从收到的同一通道发回。

![[Pasted image 20221121105313.png]]

链A将 "解押 "100个ATOM代币，因此，前缀将被删除。链B将燃烧`transfer/channel-40/atoms`。

> 前缀决定了source链。如果模块从另一个通道发送token，那么B链就是source链，A链会mint带有前缀的新令牌，而不是未被丢弃的ATOM token。你可以在两个链之间有不同的通道，但你不能在不同的通道上来回传输同一个token。如果`{denom}`包含`/`，那么它也必须遵循ICS-20的形式，表示这个token有一个多跳（multi-hop）记录。这就要求在非IBC token称谓名称中禁止使用字符`/`。


![[Pasted image 20221121105544.png]]

你已经知道应用程序需要实现IBC模块接口 [ibc-go/module.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/05-port/types/module.go) ，所以请看一下token传输的实现 [ibc-go/ibc_module.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/transfer/ibc_module.go) ，例如`OnChanOpenInit`。

```go
// OnChanOpenInit implements the IBCModule interface
func (im IBCModule) OnChanOpenInit(
    ctx sdk.Context,
    order channeltypes.Order,
    connectionHops []string,
    portID string,
    channelID string,
    chanCap *capabilitytypes.Capability,
    counterparty channeltypes.Counterparty,
    version string,
) error {
    if err := ValidateTransferChannelParams(ctx, im.keeper, order, portID, channelID); err != nil {
        return err
    }

    if version != types.Version {
        return sdkerrors.Wrapf(types.ErrInvalidVersion, "got %s, expected %s", version, types.Version)
    }

    // Claim channel capability passed back by IBC module
    if err := im.keeper.ClaimCapability(ctx, chanCap, host.ChannelCapabilityPath(portID, channelID)); err != nil {
        return err
    }

    return nil
}
```

`OnChanOpenAck`、`OnChanOpenConfirm`、`OnChanCloseInit`和`OnChanCloseConfirm`将不做（几乎）检查。

通道建立后，模块可以开始发送和接收数据包。`OnRecvPacket`将解码一个数据包，并应用传输token应用逻辑。

```go
// OnRecvPacket implements the IBCModule interface. A successful acknowledgement
// is returned if the packet data is successfully decoded and the receive application
// logic returns without error.
func (im IBCModule) OnRecvPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
    relayer sdk.AccAddress,
) ibcexported.Acknowledgement {
    ack := channeltypes.NewResultAcknowledgement([]byte{byte(1)})

    var data types.FungibleTokenPacketData
    if err := types.ModuleCdc.UnmarshalJSON(packet.GetData(), &data); err != nil {
        ack = channeltypes.NewErrorAcknowledgement("cannot unmarshal ICS-20 transfer packet data")
    }

    // only attempt the application logic if the packet data
    // was successfully decoded
    if ack.Success() {
        err := im.keeper.OnRecvPacket(ctx, packet, data)
        if err != nil {
            ack = types.NewErrorAcknowledgement(err)
        }
    }

    ctx.EventManager().EmitEvent(
        sdk.NewEvent(
            types.EventTypePacket,
            sdk.NewAttribute(sdk.AttributeKeyModule, types.ModuleName),
            sdk.NewAttribute(types.AttributeKeyReceiver, data.Receiver),
            sdk.NewAttribute(types.AttributeKeyDenom, data.Denom),
            sdk.NewAttribute(types.AttributeKeyAmount, data.Amount),
            sdk.NewAttribute(types.AttributeKeyAckSuccess, fmt.Sprintf("%t", ack.Success())),
        ),
    )

    // NOTE: acknowledgment will be written synchronously during IBC handler execution.
    return ack
}
```

在进一步深入学习代码之前，请看一下token数据包的类型定义 [ibc-go/packet.proto at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/proto/ibc/applications/transfer/v2/packet.proto) 。

```go
syntax = "proto3";

package ibc.applications.transfer.v2;

option go_package = "github.com/cosmos/ibc-go/v3/modules/apps/transfer/types";

// FungibleTokenPacketData defines a struct for the packet payload
// See FungibleTokenPacketData spec:
// https://github.com/cosmos/ibc/tree/master/spec/app/ics-020-fungible-token-transfer#data-structures
message FungibleTokenPacketData {
    // the token denomination to be transferred
    string denom = 1;
    // the token amount to be transferred
    string amount = 2;
    // the sender address
    string sender = 3;
    // the recipient address on the destination chain
    string receiver = 4;
}
```

那么，该模块将token发送到哪里呢？请看一下代币传送模块的`msg_serve.go` [ibc-go/msg_server.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/transfer/keeper/msg_server.go) 。

```go
// Transfer defines a rpc handler method for MsgTransfer.
func (k Keeper) Transfer(goCtx context.Context, msg *types.MsgTransfer) (*types.MsgTransferResponse, error) {
    ...

    if err := k.SendTransfer(
        ctx, msg.SourcePort, msg.SourceChannel, msg.Token, sender, msg.Receiver, msg.TimeoutHeight, msg.TimeoutTimestamp,
        ); err != nil {
        return nil, err
    }
    ...
}
```

在那里你可以看到`SendTransfer`，它在检查了发送者是source链还是sink链后实现了应用逻辑 [ibc-go/coin.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/transfer/types/coin.go) 。

```go
func (k Keeper) SendTransfer(
    ctx sdk.Context,
    sourcePort,
    sourceChannel string,
    token sdk.Coin,
    sender sdk.AccAddress,
    receiver string,
    timeoutHeight clienttypes.Height,
    timeoutTimestamp uint64,
) {
  ...

    // deconstruct the token denomination into the denomination trace info
    // to determine if the sender is the source chain
    if strings.HasPrefix(token.Denom, "ibc/") {
        fullDenomPath, err = k.DenomPathFromHash(ctx, token.Denom)
        if err != nil {
            return err
        }
    }

    ...

    // NOTE: SendTransfer simply sends the denomination as it exists on its own
    // chain inside the packet data. The receiving chain will perform denom
    // prefixing as necessary.

    if types.SenderChainIsSource(sourcePort, sourceChannel, fullDenomPath) {
        ...

        // create the escrow address for the tokens
        escrowAddress := types.GetEscrowAddress(sourcePort, sourceChannel)

        // escrow source tokens. It fails if balance insufficient.
        if err := k.bankKeeper.SendCoins(...) {
        } else {
            ...

            if err := k.bankKeeper.SendCoinsFromAccountToModule(...);

            ...

            if err := k.bankKeeper.BurnCoins(...);

            ...
        }

        packetData := types.NewFungibleTokenPacketData(
            fullDenomPath, token.Amount.String(), sender.String(), receiver,
        )
        ...
    }
}
```

## Interchain 账户（链间账户）

在IBC-Go [ibc-go/docs/apps/interchain-accounts at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/tree/main/docs/apps/interchain-accounts) 存储库中实施的另一个应用模块是链间账户（ICS-27） [ibc/README.md at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/blob/main/spec/app/ics-027-interchain-accounts/README.md) 。

![[Pasted image 20221121113301.png]]

主机链（host chain）：注册interchain账户的链。主机链监听来自控制器链的IBC数据包，该数据包应包含指令（例如cosmos SDK消息），链间账户将执行该指令。

控制器链( controller chain)：在主机链上注册和控制一个账户的链。控制器链向主机链发送IBC数据包以控制账户。一个控制器链必须至少有一个链间账户认证模块，才能作为一个控制器链。


> 链间账户应用模块的结构是支持专门启用控制器或主机功能的能力。这可以通过简单地从链间账户`NewAppModule`构造函数中省略控制器或主机Keeper来实现，并通过`IBCRouter`仅安装所需的子模块。另外，可以使用链上参数动态地启用和禁用子模块。

认证模块：控制器链上的一个自定义IBC应用模块，使用链间账户模块API来建立创建和管理链间账户的自定义逻辑。一个控制器链需要一个认证模块来利用链间账户模块的功能。

链间账户（ICA）：一个主机链上的账户。一个链间账户具有普通账户的所有功能。然而，控制器链的认证模块不是用私钥签署交易，而是向主机链发送IBC数据包，该数据包表明链间账户应执行哪些交易。

ICA模块提供了一个用于注册账户和发送链间交易的API。开发者将通过实现ICA Auth Module（认证模块）来使用这个模块，并可以为应用程序或用户暴露gRPC端点。普通账户使用私钥来签署链上交易。链间账户则是由独立的链通过IBC交易进行程序化控制。链间账户作为链间账户模块账户的子账户来实现。

首先，看看一个账户是如何注册的。

![[Pasted image 20221121143031.png]]

为了在主机链上注册一个账户，在控制器链上调用函数`RegisterInterchainAccount` [ibc-go/account.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/27-interchain-accounts/controller/keeper/account.go) 

```go
// RegisterInterchainAccount is the entry point to registering an interchain account.
// It generates a new port identifier using the owner address. It will bind to the
// port identifier and call 04-channel 'ChanOpenInit'. An error is returned if the port
// identifier is already in use. Gaining access to interchain accounts whose channels
// have closed cannot be done with this function. A regular MsgChanOpenInit must be used.
func (k Keeper) RegisterInterchainAccount(ctx sdk.Context, connectionID, owner string) error {
    portID, err := icatypes.NewControllerPortID(owner)

    ...

    // if there is an active channel for this portID / connectionID return an error
    activeChannelID, found := k.GetOpenActiveChannel(ctx, connectionID, portID)
    if found {
        return sdkerrors.Wrapf(icatypes.ErrActiveChannelAlreadySet, "existing active channel %s for portID %s on connection %s for owner %s", activeChannelID, portID, connectionID, owner)
    }

    ...

    connectionEnd, err := k.channelKeeper.GetConnection(ctx, connectionID)

    ...

    msg := channeltypes.NewMsgChannelOpenInit(portID, string(versionBytes), channeltypes.ORDERED, []string{connectionID}, icatypes.PortID, icatypes.ModuleName)

    ...
}
```

`portID`将是`interchain account owner`的地址，前缀是链间账户控制器子模块`icacontroller-`的默认端口前缀。ICA模块假定在主机和控制器链之间已经建立了一个IBC连接。作为`RegisterInterchainAccount`的一部分，它将打开一个`ORDERED`通道。  
  
ICA模块使用`ORDERED`通道来维持从控制器链向主机链发送数据包时的交易顺序。使用`ORDERED`通道的一个限制是，当一个数据包超时时，通道将被关闭。在通道关闭的情况下，控制器链需要能够重新访问在这个通道上注册的链间账户。  
  
ICA提供活动通道（active chainnels），使用相同的控制器前缀链端口ID创建一个新通道。这意味着当使用`RegisterInterchainAccount` API注册一个链间账户时，一个新的通道会在一个特定的端口上被创建。在`OnChanOpenAck`和`OnChanOpenConfirm`步骤中（控制器和主机链），这个链间账户的活动通道被存储在状态中。  
  
使用存储在状态中的活动通道，就有可能使用相同的控制器链端口ID创建一个新的通道，即使之前设置的活动通道处于关闭状态。  
  
如果消息到了主控链（用前面显示的`NewMsgChannelOpenInit`调用），主控链上的ICA模块也会调用`RegisterInterchainAccount` [ibc-go/account.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/27-interchain-accounts/host/keeper/account.go) 。  

```go
// RegisterInterchainAccount attempts to create a new account using the provided address and
// stores it in state keyed by the provided connection and port identifiers
// If an account for the provided address already exists this function returns early (no-op)
func (k Keeper) RegisterInterchainAccount(ctx sdk.Context, connectionID, controllerPortID string, accAddress sdk.AccAddress) {
    if acc := k.accountKeeper.GetAccount(ctx, accAddress); acc != nil {
        return
    }

    interchainAccount := icatypes.NewInterchainAccount(
        authtypes.NewBaseAccountWithAddress(accAddress),
        controllerPortID,
    )

    k.accountKeeper.NewAccount(ctx, interchainAccount)
    k.accountKeeper.SetAccount(ctx, interchainAccount)

    k.SetInterchainAccountAddress(ctx, connectionID, controllerPortID, interchainAccount.Address)
}
```

这个调用会经过主机链上`handshake.go` [ibc-go/handshake.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/27-interchain-accounts/host/keeper/handshake.go) 中的`OnChanOpenTry`实现。

```go
// OnChanOpenTry performs basic validation of the ICA channel
// and registers a new interchain account (if it doesn't exist).
// The version returned will include the registered interchain
// account address.
func (k Keeper) OnChanOpenTry(
    ...
) (string, error) {
    ...

    // Register interchain account if it does not already exist
    k.RegisterInterchainAccount(ctx, metadata.HostConnectionId, counterparty.PortId, accAddress)

    ...
}
```

结果是，在主机链上创建了一个新的链间账户。你可以在`handshake.go`  [ibc-go/handshake.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/27-interchain-accounts/controller/keeper/handshake.go) 中找到`OnChanOpenInit`和`OnChanOpenAck`的实现。

```go
// OnChanOpenInit performs basic validation of channel initialization.
// The channel order must be ORDERED, the counterparty port identifier
// must be the host chain representation as defined in the types package,
// the channel version must be equal to the version in the types package,
// there must not be an active channel for the specfied port identifier,
// and the interchain accounts module must be able to claim the channel
// capability.
func (k Keeper) OnChanOpenInit(
    ctx sdk.Context,
    order channeltypes.Order,
    connectionHops []string,
    portID string,
    channelID string,
    chanCap *capabilitytypes.Capability,
    counterparty channeltypes.Counterparty,
    version string,
) error {
    ...

    activeChannelID, found := k.GetActiveChannelID(ctx, connectionHops[0], portID)
    if found {
        channel, found := k.channelKeeper.GetChannel(ctx, portID, activeChannelID)
        if !found {
            panic(fmt.Sprintf("active channel mapping set for %s but channel does not exist in channel store", activeChannelID))
        }

        if channel.State == channeltypes.OPEN {
            return sdkerrors.Wrapf(icatypes.ErrActiveChannelAlreadySet, "existing active channel %s for portID %s is already OPEN", activeChannelID, portID)
        }

        if !icatypes.IsPreviousMetadataEqual(channel.Version, metadata) {
            return sdkerrors.Wrap(icatypes.ErrInvalidVersion, "previous active channel metadata does not match provided version")
        }
    }

    return nil
}
```

> 在链间账户和通道之间有一个一对一的映射，以满足活跃通道的要求。

在`OnChanOpenAck`中，你可以看到通道ID和账户地址将被设置在状态。

```go
// OnChanOpenAck sets the active channel for the interchain account/owner pair
// and stores the associated interchain account address in state keyed by it's corresponding port identifier
func (k Keeper) OnChanOpenAck(
    ctx sdk.Context,
    portID,
    channelID string,
    counterpartyVersion string,
) error {
    ...

    k.SetActiveChannelID(ctx, metadata.ControllerConnectionId, portID, channelID)
    k.SetInterchainAccountAddress(ctx, metadata.ControllerConnectionId, portID, metadata.Address)

    return nil
}
```

注册后，注册账户可用于签署主机链上的交易。

![[Pasted image 20221121143913.png]]

你可以在`relay.go`  [ibc-go/relay.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/27-interchain-accounts/controller/keeper/relay.go) 中找到`SendTx`。

```go
// SendTx takes pre-built packet data containing messages to be executed on the host chain from an authentication module and attempts to send the packet.
// The packet sequence for the outgoing packet is returned as a result.
// If the base application has the capability to send on the provided portID. An appropriate
// absolute timeoutTimestamp must be provided. If the packet is timed out, the channel will be closed.
// In the case of channel closure, a new channel may be reopened to reconnect to the host chain.
func (k Keeper) SendTx(ctx sdk.Context, chanCap *capabilitytypes.Capability, connectionID, portID string, icaPacketData icatypes.InterchainAccountPacketData, timeoutTimestamp uint64) (uint64, error) {
    activeChannelID, found := k.GetOpenActiveChannel(ctx, connectionID, portID)

    ...

    sourceChannelEnd, found := k.channelKeeper.GetChannel(ctx, portID, activeChannelID)

    ...

    destinationPort := sourceChannelEnd.GetCounterparty().GetPortID()
    destinationChannel := sourceChannelEnd.GetCounterparty().GetChannelID()

    ...

    return k.createOutgoingPacket(ctx, portID, activeChannelID, destinationPort, destinationChannel, chanCap, icaPacketData, timeoutTimestamp)
}
```

这就验证了source和dest的ID，并使用`createOutgoingPacket`来发送交易，它调用IBC核心模块的`SendPacket`。

在主机链上，如果有数据包到达，就会调用`relay.go`  [ibc-go/relay.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/27-interchain-accounts/host/keeper/relay.go) 中的`OnRecvPacket`。

```go
// OnRecvPacket handles a given interchain accounts packet on a destination host chain.
// If the transaction is successfully executed, the transaction response bytes will be returned.
func (k Keeper) OnRecvPacket(ctx sdk.Context, packet channeltypes.Packet) ([]byte, error) {
    var data icatypes.InterchainAccountPacketData

    if err := icatypes.ModuleCdc.UnmarshalJSON(packet.GetData(), &data); err != nil {
        // UnmarshalJSON errors are indeterminate and therefore are not wrapped and included in failed acks
        return nil, sdkerrors.Wrapf(icatypes.ErrUnknownDataType, "cannot unmarshal ICS-27 interchain account packet data")
    }

    switch data.Type {
        case icatypes.EXECUTE_TX:
            msgs, err := icatypes.DeserializeCosmosTx(k.cdc, data.Data)
            if err != nil {
                return nil, err
            }

            txResponse, err := k.executeTx(ctx, packet.SourcePort, packet.DestinationPort, packet.DestinationChannel, msgs)
            if err != nil {
                return nil, err
            }

            return txResponse, nil
        default:
            return nil, icatypes.ErrUnknownDataType
    }
}
```

这将调用`executeTx`在主机链上应用收到的交易。

```go
// executeTx attempts to execute the provided transaction. It begins by authenticating the transaction signer.
// If authentication succeeds, it does basic validation of the messages before attempting to deliver each message
// into state. The state changes will only be committed if all messages in the transaction succeed. Thus the
// execution of the transaction is atomic, all state changes are reverted if a single message fails.
func (k Keeper) executeTx(ctx sdk.Context, sourcePort, destPort, destChannel string, msgs []sdk.Msg) ([]byte, error) {
    channel, found := k.channelKeeper.GetChannel(ctx, destPort, destChannel)

    ...

    if err := k.authenticateTx(ctx, msgs, channel.ConnectionHops[0], sourcePort);

    ...

    // CacheContext returns a new context with the multi-store branched into a cached storage object
    // writeCache is called only if all msgs succeed, performing state transitions atomically
    cacheCtx, writeCache := ctx.CacheContext()
    for i, msg := range msgs {
        if err := msg.ValidateBasic(); err != nil {
            return nil, err
        }

        msgResponse, err := k.executeMsg(cacheCtx, msg)
        if err != nil {
            return nil, err
        }

        txMsgData.Data[i] = &sdk.MsgData{
            MsgType: sdk.MsgTypeURL(msg),
            Data:    msgResponse,
        }
    }

    ...
}
```

`authenticateTx`确保提供的消息包含正确的interchain account owner地址。 `executeMsg`将调用消息处理程序，从而执行该消息。

#### SDK安全模型

一个链上的SDK模块被认为是值得信赖的。例如，没有任何检查可以阻止一个不值得信任的模块访问银行保管员。

ibc-go [ibc-go/modules/apps/27-interchain-accounts at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/tree/main/modules/apps/27-interchain-accounts) 上的ICS-27 [ibc/README.md at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/blob/main/spec/app/ics-027-interchain-accounts/README.md) 的实现在其安全考虑中使用了这个假设。该实施假设认证模块不会试图在它不控制的所有者地址上打开通道。

该实施方案假定其他 IBC 应用模块不会绑定到 ICS-27 [ibc/README.md at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/blob/main/spec/app/ics-027-interchain-accounts/README.md) 命名空间内的端口。

#### 认证模块

控制器子模块 [ibc-go/modules/apps/27-interchain-accounts/controller at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/tree/main/modules/apps/27-interchain-accounts/controller) 用于账户注册和数据包发送。它只执行链间账户的所有控制器所需的逻辑。用来管理链间账户的认证类型仍未指定。可能存在许多不同类型的认证，这些认证对不同的用例是可取的。因此，认证模块的目的是用自定义的认证逻辑来包装控制器模块。  
  
在ibc-go [cosmos/ibc-go: Interblockchain Communication Protocol (IBC) implementation in Golang. (github.com)](https://github.com/cosmos/ibc-go) 中，认证模块通过中间件 [ibc-go/ibc_middleware.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/apps/27-interchain-accounts/controller/ibc_middleware.go) 栈连接到控制器链。控制器模块作为中间件 [ibc/spec/app/ics-030-middleware at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/app/ics-030-middleware) 实现，认证模块作为中间件堆栈的基础应用连接到控制器模块。为了实现一个认证模块，必须满足`IBCModule`接口的要求。通过将控制器模块作为中间件来实现，可以创建任何数量的认证模块并连接到控制器模块，而不需要编写多余的代码。  
  
认证模块必须  
  
- 对链间账户所有者进行认证  
- 跟踪所有者的相关链间账户地址  
- 在`OnChanOpenInit`中申请通道能力  
- 代表一个所有者发送数据包（在认证之后）  

以下`IBCModule`  [ibc-go/module.go at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/main/modules/core/05-port/types/module.go) 的回调必须用适当的自定义逻辑来实现。  

```go
// OnChanOpenInit implements the IBCModule interface
func (im IBCModule) OnChanOpenInit(
    ctx sdk.Context,
    order channeltypes.Order,
    connectionHops []string,
    portID string,
    channelID string,
    chanCap *capabilitytypes.Capability,
    counterparty channeltypes.Counterparty,
    version string,
) error {
    // the authentication module *must* claim the channel capability on OnChanOpenInit
    if err := im.keeper.ClaimCapability(ctx, chanCap, host.ChannelCapabilityPath(portID, channelID)); err != nil {
        return err
    }

    // perform custom logic

    return nil
}

// OnChanOpenAck implements the IBCModule interface
func (im IBCModule) OnChanOpenAck(
    ctx sdk.Context,
    portID,
    channelID string,
    counterpartyVersion string,
) error {
    // perform custom logic

    return nil
}

// OnChanCloseConfirm implements the IBCModule interface
func (im IBCModule) OnChanCloseConfirm(
    ctx sdk.Context,
    portID,
    channelID string,
) error {
    // perform custom logic

    return nil
}

// OnAcknowledgementPacket implements the IBCModule interface
func (im IBCModule) OnAcknowledgementPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
    acknowledgement []byte,
    relayer sdk.AccAddress,
) error {
    // perform custom logic

    return nil
}

// OnTimeoutPacket implements the IBCModule interface.
func (im IBCModule) OnTimeoutPacket(
    ctx sdk.Context,
    packet channeltypes.Packet,
    relayer sdk.AccAddress,
) error {
    // perform custom logic

    return nil
}
```

必须定义`OnChanOpenTry`、`OnChanOpenConfirm`、`OnChanCloseInit`和`OnRecvPacket`等函数来完成`IBCModule`接口，但它们永远不会被控制器模块调用，所以它们可能会出错或恐慌。

认证模块可以通过调用`RegisterInterchainAccount`开始注册链间账户。

```go
if err := keeper.icaControllerKeeper.RegisterInterchainAccount(ctx, connectionID, owner.String()); err != nil {
    return err
}

return nil
```

认证模块可以通过调用`SendTx`来尝试发送一个数据包。

```go

// Authenticate owner
// perform custom logic

// Construct controller portID based on interchain account owner address
portID, err := icatypes.NewControllerPortID(owner.String())
if err != nil {
    return err
}

channelID, found := keeper.icaControllerKeeper.GetActiveChannelID(ctx, portID)
if !found {
    return sdkerrors.Wrapf(icatypes.ErrActiveChannelNotFound, "failed to retrieve active channel for port %s", portID)
}

// Obtain the channel capability, claimed in OnChanOpenInit
chanCap, found := keeper.scopedKeeper.GetCapability(ctx, host.ChannelCapabilityPath(portID, channelID))
if !found {
    return sdkerrors.Wrap(channeltypes.ErrChannelCapabilityNotFound, "module does not own channel capability")
}

// Obtain data to be sent to the host chain.
// In this example, the owner of the interchain account would like to send a bank MsgSend to the host chain.
// The appropriate serialization function should be called. The host chain must be able to deserialize the transaction.
// If the host chain is using the ibc-go host module, `SerializeCosmosTx` should be used.
msg := &banktypes.MsgSend{FromAddress: fromAddr, ToAddress: toAddr, Amount: amt}
data, err := icatypes.SerializeCosmosTx(keeper.cdc, []sdk.Msg{msg})
if err != nil {
    return err
}

// Construct packet data
packetData := icatypes.InterchainAccountPacketData{
    Type: icatypes.EXECUTE_TX,
    Data: data,
}

// Obtain timeout timestamp
// An appropriate timeout timestamp must be determined based on the usage of the interchain account.
// If the packet times out, the channel will be closed requiring a new channel to be created
timeoutTimestamp := obtainTimeoutTimestamp()

// Send the interchain accounts packet, returning the packet sequence
seq, err = keeper.icaControllerKeeper.SendTx(ctx, chanCap, portID, packetData, timeoutTimestamp)
```

`InterchainAccountPacketData`中的数据必须使用主机链支持的格式进行序列化。如果主机链使用IBC-Go主机链子模块 [ibc-go/modules/apps/27-interchain-accounts/host at main · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/tree/main/modules/apps/27-interchain-accounts/host) ，应该使用`SerializeCosmosTx`。如果`InterchainAccountPacketData.Data`使用主机链不支持的格式进行序列化，数据包将不会被成功接收。

一旦中继器转发了确认，控制器链将能够访问写进主机链状态的确认。确认字节将通过`OnAcknowledgementPacke`t回调传递给认证模块。认证模块应该知道如何对确认进行解码。

如果控制器链使用ibc-go上的主机模块连接到主机链，它可以如下解释确认字节。

```go
var ack channeltypes.Acknowledgement
if err := channeltypes.SubModuleCdc.UnmarshalJSON(acknowledgement, &ack); err != nil {
    return err
}

txMsgData := &sdk.TxMsgData{}
if err := proto.Unmarshal(ack.GetResult(), txMsgData); err != nil {
    return err
}
```

如果`txMsgData.Data`为空，说明主机链使用的SDK版本大于v0.45。auth模块应按以下方式解释`txMsgData.Responses`。

```go
...
// switch statement from above
case 0:
    for _, any := range txMsgData.MsgResponses {
        if err := handleAny(any); err != nil {
            return err
        }
    }
}
```

需要一个处理程序来解释根据Any的类型url来执行什么行动。可以使用一个路由器，或者更简单的一个开关语句。

```go
func handleAny(any *codectypes.Any) error {
    switch any.TypeURL {
        case banktypes.MsgSend:
            msgResponse, err := unpackBankMsgSendResponse(any)
            if err != nil {
                return err
            }

            handleBankSendMsg(msgResponse)

        case stakingtypes.MsgDelegate:
            msgResponse, err := unpackStakingDelegateResponse(any)
            if err != nil {
                return err
            }

            handleStakingDelegateMsg(msgResponse)

            case transfertypes.MsgTransfer:
            msgResponse, err := unpackIBCTransferMsgResponse(any)
            if err != nil {
                return err
            }

            handleIBCTransferMsg(msgResponse)

        default:
            return
    }
}
```

#### 样例集成

这里 [cosmos/interchain-accounts-demo: ICA (github.com)](https://github.com/cosmos/interchain-accounts-demo) 是一个端到端的链间账户的工作演示。你会注意到，它包含一个类似的应用程序。去看看下面这个通用的应用程序。

```go
// app.go

// Register the AppModule for the interchain accounts module and the authentication module
// Note: No `icaauth` exists, this must be substituted with an actual interchain accounts authentication module
ModuleBasics = module.NewBasicManager(
    ...
    ica.AppModuleBasic{},
    icaauth.AppModuleBasic{},
    ...
)

...

// Add module account permissions for the interchain accounts module
// Only necessary for host chain functionality
// Each interchain account created on the host chain is derived from the module account created
maccPerms = map[string][]string{
    ...
    icatypes.ModuleName:            nil,
}

...

// Add interchain account keepers for each submodule used and the authentication module
// If a submodule is being statically disabled, the associated Keeper does not need to be added.

type App struct {
    ...

    ICAControllerKeeper icacontrollerkeeper.Keeper
    ICAHostKeeper       icahostkeeper.Keeper
    ICAAuthKeeper       icaauthkeeper.Keeper

    ...
}

...

// Create store keys for each submodule Keeper and the authentication module
keys := sdk.NewKVStoreKeys(
    ...
    icacontrollertypes.StoreKey,
    icahosttypes.StoreKey,
    icaauthtypes.StoreKey,
    ...
)

...

// Create the scoped keepers for each submodule keeper and authentication keeper
scopedICAControllerKeeper := app.CapabilityKeeper.ScopeToModule(icacontrollertypes.SubModuleName)
scopedICAHostKeeper := app.CapabilityKeeper.ScopeToModule(icahosttypes.SubModuleName)
scopedICAAuthKeeper := app.CapabilityKeeper.ScopeToModule(icaauthtypes.ModuleName)

...

// Create the Keeper for each submodule
app.ICAControllerKeeper = icacontrollerkeeper.NewKeeper(
    appCodec, keys[icacontrollertypes.StoreKey], app.GetSubspace(icacontrollertypes.SubModuleName),
    app.IBCKeeper.ChannelKeeper, // may be replaced with middleware such as ics29 fee
    app.IBCKeeper.ChannelKeeper, &app.IBCKeeper.PortKeeper,
    app.AccountKeeper, scopedICAControllerKeeper, app.MsgServiceRouter(),
)
app.ICAHostKeeper = icahostkeeper.NewKeeper(
    appCodec, keys[icahosttypes.StoreKey], app.GetSubspace(icahosttypes.SubModuleName),
    app.IBCKeeper.ChannelKeeper, &app.IBCKeeper.PortKeeper,
    app.AccountKeeper, scopedICAHostKeeper, app.MsgServiceRouter(),
)

// Create interchain accounts AppModule
icaModule := ica.NewAppModule(&app.ICAControllerKeeper, &app.ICAHostKeeper)

// Create your interchain accounts authentication module
app.ICAAuthKeeper = icaauthkeeper.NewKeeper(appCodec, keys[icaauthtypes.StoreKey], app.ICAControllerKeeper, scopedICAAuthKeeper)

// ICA auth AppModule
icaAuthModule := icaauth.NewAppModule(appCodec, app.ICAAuthKeeper)

// ICA auth IBC Module
icaAuthIBCModule := icaauth.NewIBCModule(app.ICAAuthKeeper)

// Create host and controller IBC Modules as desired
icaControllerIBCModule := icacontroller.NewIBCModule(app.ICAControllerKeeper, icaAuthIBCModule)
icaHostIBCModule := icahost.NewIBCModule(app.ICAHostKeeper)

// Register host and authentication routes
ibcRouter.AddRoute(icacontrollertypes.SubModuleName, icaControllerIBCModule).
    AddRoute(icahosttypes.SubModuleName, icaHostIBCModule).
    AddRoute(icaauthtypes.ModuleName, icaControllerIBCModule) // Note, the authentication module is routed to the top level of the middleware stack

...

// Register interchain accounts and authentication module AppModule's
app.moduleManager = module.NewManager(
    ...
    icaModule,
    icaAuthModule,
)

...

// Add interchain accounts module InitGenesis logic
app.mm.SetOrderInitGenesis(
    ...
    icatypes.ModuleName,
    ...
)
```


## IBC的相关工具

主要是这三个工具，MapOfZones， Mintscan，IOBScan。

你将看到三个非常有用的IBC网络的可视化工具。它们包括网络中的链（hubs和zones）、连接、通道和交易等信息。

在浏览概述时，建议尝试一下所有可以发现的东西：只要点击一下，看看会发生什么。

这些类型的工具有助于保持对整个IBC网络的概述，但也可以帮助诸如中继器的选择，因为它们提供了一个关于中继的基本指标的概述。

#### MapOfZones

[Map of zones - Cosmos network explorer](https://mapofzones.com/home?columnKey=ibcVolume&period=24h)

#### Mintscan

https://hub.mintscan.io/   与MapOfZones定位一样，都是cosmos网络浏览器。

#### IOBScan

[IOBScan - IOB - Home](https://ibc.iobscan.io/home)  也是一个浏览器，可以看到转账，代币，网络，通道(channels)， 中继者(relayers)

## IBC的协议规范以及实现文档

[cosmos/ibc: Interchain Standards (ICS) for the Cosmos network & interchain ecosystem. (github.com)](https://github.com/cosmos/ibc)

IBC的各类ICS规范，以及相应的实现，可以在这里追踪最新的进展。反正是目前写这篇文章截止，IBC并没有实现ERC721的代币转移的ICS，但是ICS 721的规范已经有了，只是官方没有实现。所以Gravity Bridge也没有支持ERC721的跨链 https://github.com/Gravity-Bridge/Gravity-Bridge/issues/234 