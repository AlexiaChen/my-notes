# Substrate学习笔记

一些资料:

- [A guide to building Blockchain in Rust & Substrate - BLOCKGENI](https://blockgeni.com/a-guide-to-building-blockchain-in-rust-substrate/#:~:text=Substrate%20is%20Rust%20based%20and%20compiles%20to%20a,and%20various%20consensus%20mechanisms%20we%20can%20choose%20from.)
- [First impressions of Rust smart contracts with Substrate and Ink, part 1 (brson.github.io)](https://brson.github.io/2020/12/03/substrate-and-ink-part-1)
- [Building A Blockchain in Rust & Substrate: [A Step-by-Step Guide for Developers] | HackerNoon](https://hackernoon.com/building-a-blockchain-in-rust-and-substrate-a-step-by-step-guide-for-developers-kc223ybp)


## 快速开始

所有的Substrate教程和指南都要求你在你的开发环境中构建和运行一个Substrate节点。为了帮助你快速建立一个工作环境，[Substrate开发者中心]([Substrate Developer Hub (github.com)](https://github.com/substrate-developer-hub/))维护了几个模板供你使用。

[Substrate-node-template](https://github.com/substrate-developer-hub/substrate-node-template)的开发者中心快照包括了你开始使用一组核心功能所需的一切。

本快速入门假设你是第一次建立一个开发环境，并想尝试在本地计算机上运行一个单一的区块链节点。

为了保持简单，你将使用网络浏览器连接到本地节点，并查询预定义样本账户的余额。

### 构建node template


```bash
sudo apt install build-essential
sudo apt install --assume-yes git clang curl libssl-dev llvm libudev-dev make protobuf-compiler
rustup default stable
rustup update
rustup update nightly
rustup target add wasm32-unknown-unknown --toolchain nightly
rustup show
rustup +nightly show
```

```bash
git clone --branch polkadot-v0.9.25 https://github.com/substrate-developer-hub/substrate-node-template
cd substrate-node-template
cargo build  --release
```

### 启动节点

```bash
./target/release/node-template --help
```

以上帮助给你了一个启动节点，与account工作，修改节点操作的一个帮助提示，

```bash
./target/release/node-template key inspect //Alice
```

以上是查看预定义好的Alice开发账号的account信息，显示如下:

```txt
Secret Key URI `//Alice` is account:
Network ID:        substrate 
Secret seed:       0xe5be9a5092b81bca64be81d212e7f2f9eba183bb7a90954f7b76361f6edb5c0a
Public key (hex):  0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d
Account ID:        0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d
Public key (SS58): 5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
SS58 Address:      5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
```

可以以开发模式启动节点:

```bash
./target/release/node-template --dev
```

在开发模式下，链不需要任何P2P对端的计算机来最终完成区块。当节点启动时，终端会显示关于所执行操作的输出。如果你看到正在提议和确定区块的信息，你就有一个正在运行的节点。

### 连接节点

用JavaScript和Polkadot-JS API创建一个简单的HTML文件，与区块链互动。比如，创建一个index.html文件
      - 可以输入一个账号地址
      - 监听onClick事件来查询账号余额
      - 显示余额
html样例在这里 [substrate-docs/index.html at main · substrate-developer-hub/substrate-docs (github.com)](https://github.com/substrate-developer-hub/substrate-docs/blob/main/static/assets/quickstart/index.html)

### 停止节点

一般`ctrl + c`就可以了


## 区块链基础

区块链是一个去中心化的账本，以区块的序列记录信息。区块中包含的信息是一组有序的指令，可能导致状态的改变。

在区块链网络中，各个计算机--即节点--相互沟通，形成一个去中心化的点对点（P2P）网络。没有控制网络的中央机构，通常情况下，每个参与区块生产的节点都会储存一份构成canonical chain的区块副本。canonical chain就是最终被确定的那条主链，是没有歧义的主链。

在大多数情况下，用户通过提交可能导致状态改变的请求与区块链互动，例如，请求改变文件的所有者或将资金从一个账户转移到另一个账户。这些交易请求被广播到网络上的其他节点，并由区块打包者组装成一个区块。为了确保链上数据的安全和链的持续进展，节点使用某种形式的共识来商定每个区块中的数据状态和交易执行的顺序。

### 什么是区块链节点

在高层次上，所有区块链节点都需要以下核心组件。  
  
- 数据存储，用于记录作为交易结果的状态变化。  
- 点对点网络，用于节点之间的去中心化通信。  
- 共识方法，以防止恶意活动，确保链的持续进展。  
- 用于排序和处理传入交易的逻辑。  
- 密码学用于生成区块的哈希摘要，并用于签署和验证与交易相关的签名。  

由于构建区块链所需的核心组件的复杂性，大多数区块链项目从现有区块链代码库的完整clone开始，以便开发人员可以修改现有代码以添加新功能，而不是从头开始编写一切。例如，比特币的代码仓库被Fork以创建Litecoin、ZCash、Namecoin和Bitcoin Cash。同样，以太坊代码仓库也被fork以创建Quorum、POA Network、KodakCoin和Musicoin。  
  
然而，大多数区块链平台的设计都不允许修改或定制，都是从头开始，不是模块化的。因此，通过Fork别人的源码构建一个新的区块链有严重的局限性，包括原区块链代码中固有的可扩展性等限制。在你探索Substrate如何缓解与其他区块链项目相关的许多限制之前，了解大多数区块链共享的一些共同属性是很重要的。通过了解大多数区块链的运作方式，你将更好地了解Substrate如何为构建最适合你的需求的区块链提供替代方案和能力。  

### 状态转移和冲突

区块链本质上是一个状态机。在任何时间点上，区块链都有一个当前的内部状态。随着incoming的交易的执行，它们会导致状态的改变，所以区块链必须从其当前状态转移到新的状态。然而，可能有多个有效的转移会导致不同的未来状态(也就是分叉)，区块链必须选择一个可以被同意的单一状态转移。为了就转移后的状态达成一致，区块链内的所有操作必须是确定的。为了使链的高度成功向前推进，大多数节点必须就所有的状态转换达成一致，包括。  
  
- 链的初始状态，称为创世状态或创世块。  
- 由执行的交易产生的一系列状态转换，记录在每个区块中。  
- 将被纳入链中的区块的最终状态。  

在集中式网络中，一个中央机构可以在相互排斥的状态转换中进行选择。例如，一个被配置为主要权威的服务器可能会按照它看到的顺序记录状态转换的变化，或者在出现冲突时使用一个加权过程在竞争的替代方案中进行选择。在一个去中心化的网络中，节点以不同的顺序看到交易，所以它们必须使用更复杂的方法来选择交易，并在冲突的状态转换中进行选择。  
  
区块链用来将一批交易打包在区块中并选择哪个节点可以向链上提交区块的方法，被称为区块链的共识模型或共识算法。最常用的共识模型被称为PoW。在PoW共识模式下，首先完成一个计算问题的节点有权向链上提交一个区块。  
  
为了使区块链具有容错性，并提供一致的状态视图，即使一些节点被恶意行为者破坏或网络中断，一些共识模型要求至少2/3的节点在任何时候都对状态达成一致。这个2/3的多数确保了网络的容错性，并能承受一些网络参与者的不良行为，无论该行为是故意的还是意外的。  
  
### 区块链经济学

所有区块链都需要资源--处理器、内存、存储和网络带宽来执行操作。参与网络的计算机--生产区块的节点--向区块链用户提供这些资源。这些节点创建了一个分布式的、去中心化的网络，为参与者的社区的需求服务。

为了支持社区并使区块链可持续发展，大多数区块链要求用户以交易费的形式为他们使用的网络资源付费。交易费的支付要求用户身份与持有某种类型资产的账户相关联。区块链通常使用代币来代表账户中资产的价值，网络参与者通过交易所在链外购买代币。然后，网络参与者可以存入代币，使他们能够支付交易费用。

### 区块链治理

一些区块链允许网络参与者提交影响网络运行或区块链社区的提案并进行投票。通过提交提案和投票，区块链社区可以决定区块链在本质上的民主进程中如何发展。然而，链上治理相对罕见，为了参与，区块链可能要求用户在账户中保持大量的代币股份，或被选为其他用户的代表。

### 运行在链上的应用

在区块链上运行的应用程序--通常被称为去中心化的应用程序或Dapp--通常是使用前端框架编写的Web应用程序，但有后端智能合约来改变区块链状态。

智能合约是一个在区块链上运行的程序，在特定条件下代替用户执行交易并作出状态变更。开发人员可以编写智能合约，以确保以程序执行的交易结果被记录下来，并且不能被篡改。然而，仅凭智能合约，开发者无法使用一些底层区块链功能--如共识、存储或交易层--而是要遵守一条链的固定规则和限制。智能合约的开发者通常接受这些限制，作为一种权衡，以较少的核心设计决策来实现更快的开发时间。

## 为什么选择substrate

区块链底层的开发是复杂的。它涉及复杂的技术--包括先进的密码学和分布式网络通信--你必须正确地提供一个安全的平台，让应用程序在上面运行，让用户信任。还有围绕规模、治理、互操作性和可升级性的难题需要解决。这种复杂性为开发者创造了一个需要克服的高门槛。考虑到这一点，要回答的第一个问题是：你想建立什么？

Substrate并不完全适合每一个用例、应用程序或项目。然而，如果你想构建一个区块链，那么Substrate可能是一个完美的选择。

- 它可以为一个非常具体的用例定制化
- 能够与其他区块链连接和沟通
- 可通过预定义的可组合模块组件进行定制
- 能够随着时间的推移升级而发展和变化

Substrate是一个软件开发工具包（SDK），专门为您提供区块链所需的所有基本组件，因此您可以专注于制作使您的链独特和创新的逻辑。与其他分布式账本平台不同，Substrate是。

- 灵活
- 开放的
- 可互操作(跨链)
- 面向未来

### 灵活性

大多数区块链平台都有非常高的耦合，低内聚的子系统，相当难以解耦。在基于另一个区块链的源码作出Fork修改的链上也有风险，这些高耦合会从根本上破坏区块链系统本身。

Substrate是一个完全模块化的区块链框架，让你通过选择适合你的项目的网络堆栈、共识模型或治理方法，或通过创建你自己的组件，组成一个有明确解耦组件的链。

有了Substrate，你可以部署一个为你自定义规格设计和建造的区块链，但它也可以随着你不断变化的需求而发展。

### 开放性

所有的Substrate架构和工具都在开源许可下提供。Substrate框架的核心组件使用开放协议，如libp2p和jsonRPC，同时授权你决定你想定制多少区块链架构都可以。Substrate还有一个庞大的、活跃的、有帮助的建设者社区，为生态系统做出贡献。来自社区的贡献增强了可用的能力，随着区块链的发展，您可以将其纳入自己的区块链中。

### 互操作性

大多数区块链平台提供的与其他区块链网络互动的能力有限。所有基于Substrate的区块链都可以通过跨共识信息传递（XCM）与其他区块链进行互操作。Substrate可用于创建作为独立网络的链（独立链），或与中继链紧密接入，以分享其安全性，作为一个parachain。


### 面向未来

Substrate是为可升级、可组合和可适应而建立的。状态转换逻辑--Substrate runtime--是一个独立的WebAssembly对象。你的节点可以被赋予在特定条件下完全改变runtime本身的能力，在整个网络范围内诱发runtime升级。因此，"forkless "升级是可能的，因为在大多数情况下，节点不需要采取任何行动就可以使用这个新的runtime。随着时间的推移，你的网络的运行时协议可以无缝地，也许是彻底地，随着用户的需求而发展。

## Substrate架构

### 高层次下的概览

在一个去中心化的网络中，所有节点既是请求数据的客户端，也是响应数据请求的服务器。在概念上和程序上，Substrate架构按照类似的思路划分了操作职责。下图以简化的形式说明了这种责任划分，以帮助你直观地了解该架构以及Substrate如何为构建区块链提供一个模块化框架。

![[Pasted image 20220825154220.png]]

在高层次上，一个底层节点提供了一个具有两个主要元素的分层环境。

- 一个外层节点(类似于之前学习Polkadot提到的host部分)，处理网络活动，如对等体发现，管理交易请求，与对等体达成共识，并响应RPC调用。
- 一个runtime，包含执行区块链状态转换功能的所有业务逻辑。

#### 外层节点

外层节点负责在runtime之外进行的活动。例如，外部节点负责处理P2P的发现，管理交易池，与其他节点通信以达成共识，并回答来自外部世界的RPC调用或浏览器请求。

由外层节点处理的一些最重要的活动涉及以下组件。

- 存储： 外层节点使用一个简单而高效的KV存储层来保持底层区块链的演化状态。[State transitions and storage | Substrate_ Docs](https://docs.substrate.io/fundamentals/state-transitions-and-storage/)
- P2P网络： 用Rust编写的libp2p来与网络上的其他节点通信 [Networks and blockchains | Substrate_ Docs](https://docs.substrate.io/fundamentals/node-and-network-types/)
- 共识： 外层节点与其他网络参与者进行沟通，以确保他们对区块链的状态达成一致。[Consensus | Substrate_ Docs](https://docs.substrate.io/fundamentals/consensus/)
- RPC API： 外层节点接受incoming的HTTP和WebSocket请求，允许区块链用户与区块链网络互动。[Remote procedure calls | Substrate_ Docs](https://docs.substrate.io/build/custom-rpc/)
- 遥测: 外层节点通过内嵌的Prometheus服务器收集并提供对节点指标(metrics)的访问。这样的就方便运维人员监控了。
- 执行环境: 外层节点负责选择执行环境--WebAssembly或Native Rust--用于runtime，然后将调用分派给所选的runtime。也就是你的runtime代码可以被编译成WASM或者Native的Rust二进制。[Build process | Substrate_ Docs](https://docs.substrate.io/build/build-process/)

执行以上这些任务往往需要外层节点查询runtime的信息或向runtime提供信息。这种通信是通过调用专门的runtime API来处理的。[Runtime APIs | Substrate_ Docs](https://docs.substrate.io/reference/runtime-apis/)

这里翻译成外层节点可能有点不方便理解，应该翻译成节点的外层更好理解。

#### Runtime

runtime决定交易是有效还是无效，并负责处理区块链的状态转换功能的变化。

因为runtime执行它收到的函数，所以它控制交易如何被打包在区块中，以及区块如何返回到外层节点以进行广播或进入到其他节点中。从本质上讲，runtime负责处理链上发生的一切。它也是构建Substrate区块链的节点的核心组件。

Substrate runtime被设计为编译为WebAssembly（Wasm）字节代码。这一设计决定使:

- 支持无叉式升级。
- 多平台兼容性。
- runtime的有效性检查。
- 中继链共识机制的验证证明。

类似于外层节点有办法向runtime提供信息，runtime使用专门的host functions与外层节点或外部世界通信。[sp_io - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_io/index.html)

### 轻客户端节点

轻客户端或轻节点是Substrate节点的简化版本，只提供runtime和当前最新状态。轻节点可以让用户能够使用浏览器、浏览器扩展、移动设备或桌面计算机直接连接到Substrate runtime。通过轻型客户端节点，你可以使用用Rust、JavaScript或其他语言编写的RPC endpoints来连接到WebAssembly runtime，以读取区块头，提交交易，并查看交易的结果。 ^0b135b


## 区块链网络

在考虑建立区块链时，考虑到边界是定义网络的因素是很有用的。例如，连接到一个路由器的一组计算机可以被认为是一个家庭网络。防火墙可能是定义一个企业网络的边界。较小的、孤立的网络可以通过一个共同的通信协议连接到更广泛的区域网络。同样，你可以认为区块链网络是由其边界和与其他区块链的隔离或通信来定义的。

作为一个区块链构建者的工具包(SDK)，Substrate使你能够开发你能想象到的任何类型的区块链，并根据你的特定应用要求来定义其边界。考虑到这种灵活性，你需要做出的决定之一是你想建立的网络类型以及不同节点在该网络中可能扮演的角色。

### 网络类型

基于substrate的区块链可用于不同类型的网络架构。例如，substrate区块链被用来构建以下网络类型。

- 私有网络，限制对一组受限制的节点的访问。
- 单独的链，实施自己的安全协议，不与任何其他链连接或沟通。比特币和以太坊是不基于substrate的单机链的例子。
- 中继链，为连接到它们的其他链提供去中心化的安全和通信。Kusama和Polkadot是中继链的例子。
- Parachains是为连接到中继链而建立的，并有能力与使用同一中继链的其他链进行通信。因为parachains依赖于中继链来最终确定产生的区块，所以parachains必须实现与它们所连接的中继链相同的共识协议。

### 节点类型

区块链要求网络节点同步，以呈现区块链状态的一致和最新的视图。每个同步的节点都存储一份区块链的副本，并跟踪传入的交易。然而，保存整个区块链的完整副本需要大量的存储和计算资源，而且下载从创世块到最近的所有区块对大多数实际情况来说并不实际。为了更容易保持链的安全性和完整性，但减少想要访问区块链数据的客户的资源要求，有不同类型的节点可以与链交互:

- 全节点
- 归档节点
- 轻客户端节点 ^f85233

#### 全节点(full node)

全节点是区块链网络基础设施的重要组成部分，是最常见的节点类型。全节点存储区块链数据，通常参与常见的区块链操作，如生产和验证区块，接收和验证交易，以及响应用户请求提供数据。

![[Pasted image 20220825161340.png]]

默认情况下，全节点被配置为只存储最近的256个块，并丢弃比这更早的状态--创世块除外--以防止全节点无限制地增长并消耗所有可用的磁盘空间。你可以配置全节点保留的块的数量。

虽然旧的块被丢弃，但全节点保留了从创世块到最近的块的所有区块头，以验证状态是否正确。因为全节点可以访问所有的区块头，它可以通过执行从创世区块开始的所有区块来重建整个区块链的状态。因此，如果它需要更多的计算来检索以前的一些状态的信息，一般应使用归档节点来代替。

全节点允许你读取链的当前状态，并直接在网络上提交和验证交易。通过丢弃旧区块的状态，全节点需要的磁盘空间比归档节点少得多。然而，一个全节点需要更多的计算资源来查询和检索一些以前状态的信息。如果你需要查询历史区块，你应该purge全节点，然后把它作为一个归档节点重新启动。

#### 归档节点(Archive node)

归档节点与全节点类似，只是它们存储所有过去的区块，每个区块都有完整的状态。归档节点最常被需要访问历史信息的实用程序使用，如区块浏览器、钱包、讨论论坛和类似的应用程序。

![[Pasted image 20220825161846.png]]


因为归档节点保留了历史状态，它们需要大量的磁盘空间。由于运行它们所需的磁盘空间，归档节点不如全节点常见。然而，归档节点可以方便地在任何时间点查询链的过去状态。例如，你可以查询一个归档节点，以查询某个区块的账户余额，或查看导致特定状态变化的交易细节（查看所有交易转账）。当你在归档节点的数据上运行这些类型的查询时，速度更快，效率更高。

#### 轻客户端节点

轻型客户节点使你能够以最小的硬件要求连接到一个基于substrate的网络。

![[Pasted image 20220825162059.png]]

由于轻客户端节点需要最少的系统资源，它们可以被嵌入到基于网络的应用程序、浏览器扩展、移动设备应用程序或物联网（IoT）设备中。轻客户端节点通过RPC端点提供runtime和对当前状态的访问。轻客户端节点的RPC endpoints可以用Rust、JavaScript或其他语言编写，用于读取块头、提交交易和查看交易的结果。

轻客户端节点不参与区块链或网络操作。例如，轻客户端节点不负责区块生产或验证，广播交易或达成共识。轻客户端节点不存储任何过去的区块，所以如果不向拥有历史数据的节点请求，它就无法读取历史数据。

### 节点的角色

根据你在启动节点时指定的命令行选项，节点可以在链的发展中扮演不同的角色，并可以提供对链上状态的不同访问级别。例如，你可以限制被授权生产新区块的节点，以及哪些节点可以与P2P通信。没有被授权为区块生产者的对等节点可以同步新区块，接收交易，并向其他节点发送和接收关于新交易的广播消息。节点也可以被阻止连接到更广泛的网络，并限制与特定的节点进行通信。

## Runtime开发

正如在架构中讨论的那样，Substrate节点的runtime包含执行交易、保存状态转换和与外层节点交互的所有业务逻辑。Substrate提供了构建普通区块链组件所需的所有工具，因此你可以专注于开发定义区块链行为的runtime逻辑。

### 状态转移和runtime

在最基本的层面上，每个区块链本质上都是一个账本，或者说是对链上发生的每个变化的记录。在基于底层的链中，这些对状态的改变被记录在runtme中。由于runtime处理这一操作，runtime有时被描述为提供状态转换功能。

因为状态转换发生在runtime，所以runtime是你定义代表区块链状态的存储项和允许区块链用户对该状态进行修改的交易的地方(处理交易)。

![[Pasted image 20220825163107.png]]

Substrate runtime决定哪些交易是有效的，哪些是无效的，以及如何响应交易而改变链的状态。

### runtime接口

正如你在架构那一章节中所学到的，外层节点负责处理P2P发现、交易池、块和交易广播、共识，以及回答来自外部世界的RPC调用。这些任务经常要求外层节点向runtime查询信息或向runtime提供信息。runtime API促进了外层节点和runtime之间的这种通信。

在 Substrate 中，`sp_api` crate 提供了一个接口来实现runtime API。它旨在让你灵活地使用` impl_runtime_apis` 宏来定义自己的自定义接口。然而，每个runtime都必须实现`Core`和`Metadata`接口。除了这些必要的接口外，大多数Substrate节点--如`节点模板(node template)`--还实现了下列runtime接口:

[node template](https://github.com/substrate-developer-hub/substrate-node-template)

- BlockBuilder用于建立区块所需的功能。[BlockBuilder in sp_block_builder - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_block_builder/trait.BlockBuilder.html)
- TaggedTransactionQueue用于验证交易。[TaggedTransactionQueue in sp_transaction_pool::runtime_api - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_transaction_pool/runtime_api/trait.TaggedTransactionQueue.html)
- OffchainWorkerApi用于启用链外操作。[OffchainWorkerApi in sp_offchain - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_offchain/trait.OffchainWorkerApi.html)
- AuraApi用于区块的生产和验证，使用round-robin的共识方法。[AuraApi in sp_consensus_aura - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_consensus_aura/trait.AuraApi.html)
- SessionKeys用于生成和解码会话keys。[SessionKeys in sp_session - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_session/trait.SessionKeys.html)
- GrandpaApi，用于将区块最终确定到runtime。[GrandpaApi in sp_finality_grandpa - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_finality_grandpa/trait.GrandpaApi.html)
- AccountNonceApi 用于查询交易的索引。[AccountNonceApi in frame_system_rpc_runtime_api - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_system_rpc_runtime_api/trait.AccountNonceApi.html)
- TransactionPaymentApi，用于查询交易信息。[TransactionPaymentApi in pallet_transaction_payment_rpc_runtime_api - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_transaction_payment_rpc_runtime_api/trait.TransactionPaymentApi.html)
- Benchmark用于估计和测量完成交易所需的执行时间。[Benchmark in frame_benchmarking - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_benchmarking/trait.Benchmark.html)


### 核心原语

Substrate还定义了runtime必须实现的核心原语。Substrate框架对你的rungtime必须向Substrate的其他层提供的东西做了最小的假设。然而，有一些数据类型必须被定义，并且必须满足特定的接口才能在Substrate框架内工作。  
  
这些核心原语是:
  
- Hash。一种对某些数据的加密摘要进行编码的类型。通常只是一个256位的数量。  
- DigestItem。一种类型，它必须能够编码与共识和变化跟踪有关的一些 "硬接线 "替代方案，以及与runtime中的特定模块有关的任何数量的 "软编码 "变体之一。  
- Digest。一系列的DigestItems。它编码所有与轻客户端有关的信息，使其在区块内拥有。  
- Extrinsic：一种表示区块链外部的单一数据的类型，它被区块链所认可。这通常涉及一个或多个签名，以及某种编码指令（例如，用于转移资金所有权或调用智能合约）。  本质上就是交易了。
- Header。一种代表（加密或其他）与Block有关的所有信息的类型。它包括父哈希值、storage root和extrinsic trie root 、摘要和区块编号。  
- Block。本质上只是一个Header和一系列Extrinsic的组合，再加上要使用的散列算法的说明。  
- BlockNumber。一种编码任何有效块的祖先总数的类型。通常是一个32位的数量。  
  
### FRAME

FRAME[Glossary | Substrate_ Docs](https://docs.substrate.io/reference/glossary/#frame)是你作为一个runtime开发者可以使用的最强大的工具之一。正如在Substrate赋予开发者权力中所提到的，FRAME是Framework for Runtime Aggregation of Modularized Entities的缩写，它包含了大量的模块和支持库，以简化runtime开发。在Substrate中，这些被称为pallets的模块为不同的用例和你可能想包含在runtime中的功能提供可定制的业务逻辑。例如，有一些pallet为抵押、共识、治理和其他常见活动提供了业务逻辑框架。  
  
关于可用的pallets的摘要，见FRAME pallets [FRAME pallets | Substrate_ Docs](https://docs.substrate.io/reference/frame-pallets/)。  
  
除了pallets，FRAME还提供services，通过以下库和模块与runtime互动 : 
  
- `FRAME system crate frame_system` 为runtime提供低级类型、存储和函数。  [frame_system - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_system/index.html)
- `FRAME support crate frame_support` 是一个Rust宏、类型、traits和模块的集合，简化了Substrate pallets的开发。  [frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/index.html)
- `FRAME executive pallet frame_executive` 协调了runtime中对各自pallet的传入函数调用的执行。  [frame_executive - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_executive/index.html)

下图说明了FRAME及其system、support和exctive模块如何runtime环境提供服务。  
  
![[Pasted image 20220826085455.png]]



#### 用Pallets组合成一个runtime

你可以在不使用FRAME的情况下构建一个基于Substrate的区块链。然而，FRAME pallets使你能够使用预定义的组件作为起点，组成自定义的runtime逻辑。每个pallet都定义了特定的类型、存储项和功能，以实现runtime的一组特定特征或功能。

下图说明了你如何选择和组合FRAME pallets来组成一个runtime。

![[Pasted image 20220826085814.png]]


#### 构建自定义的Pallets

除了预先构建(pre-built)的FRAME pallets库，你可以使用FRAME库和services来建立你自己的自定义pallet。通过自定义pallet，你可以灵活地定义最适合你的目标的runtime行为。因为每个pallet都有自己的独立分散逻辑，你可以结合预先构建和自定义pallet来控制你的区块链提供的特性和功能，并实现你想要的结果。

例如，你可以在你的runtime中包含 Balances pallet [substrate/frame/balances at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/balances)，以使用其与加密货币相关的存储项(storage item)和管理代币的功能，添加自定义逻辑，在账户余额变化时调用你编写的pallet。

大多数pallets是由以下部分的某种组合组成的。

- imports和dependencies
- pallet type declaration
- runtime configuration trait
- runtime storage
- runtime events
- 应该在特定context下执行的逻辑的Hooks
- 可用于执行交易的函数调用

例如，如果你想定义一个自定义的pallet，你可以从一个类似于以下的pallet框架结构开始:

```rust
// Add required imports and dependencies
pub use pallet::*;
#[frame_support::pallet]
pub mod pallet {
 use frame_support::pallet_prelude::*;
 use frame_system::pallet_prelude::*;

 // Declare the pallet type
 // This is a placeholder to implement traits and methods.
    #[pallet::pallet]
    #[pallet::generate_store(pub(super) trait Store)]
    pub struct Pallet<T>(_);

 // Add the runtime configuration trait
 // All types and constants go here.
 #[pallet::config]
 pub trait Config: frame_system::Config { ... }

 // Add runtime storage to declare storage items.
 #[pallet::storage]
 #[pallet::getter(fn something)]
 pub type MyStorage<T: Config> = StorageValue<_, u32>;

 // Add runtime events
 #[pallet::event]
 #[pallet::generate_deposit(pub(super) fn deposit_event)]
 pub enum Event<T: Config> { ... }

 //  Add hooks to define some logic that should be executed
 //  in a specific context, for example on_initialize.
 #[pallet::hooks]
 impl<T: Config> Hooks<BlockNumberFor<T>> for Pallet<T> { ... }

 // Add functions that are callable from outside the runtime.
 #[pallet::call]
 impl<T:Config> Pallet<T> { ... }

}
```

根据以上框架代码，你可以根据需要用部分或全部section组成pallets。当你开始设计和构建你的自定义runtime，你会了解到更多关于FRAME库和runtime原语的信息，用于定义configuration trait、storage items、events和errors，以及如何编写派发到runtime执行的函数调用。

cumulus的pallets在这里 [cumulus/pallets at master · paritytech/cumulus (github.com)](https://github.com/paritytech/cumulus/tree/master/pallets) 也属于FRAME pallets的一部分。FRAME pallets包含system，functional，cumulus的pallets。

## 共识

所有的区块链都需要某种类型的共识机制来就区块链状态达成一致。因为Substrate为构建区块链提供了一个模块化的框架，它支持几个不同的模式，让节点达成共识。一般来说，不同的共识模型有不同的权衡，所以选择你想为你的链使用的共识类型是一个重要的考虑。Substrate默认支持的共识模型需要最小的配置，但如果需要，也可以建立一个自定义的共识模型。

### 共识的两个阶段步骤

与一些区块链不同，Substrate将达成共识的要求分成两个独立的阶段。

- block authoring是节点用来创建新区块的过程。
- block finalization是用于处理分叉和选择canonical chain的过程。

### Block authoring

在达成共识之前，区块链网络中的一些节点必须能够产生新的区块。区块链如何决定被授权编写区块的节点，取决于你使用的是哪种共识模式。例如，在一个中心化的网络中，一个单一的节点可能授权编写生成所有的区块。在一个完全去中心化的网络中，没有任何受信任的节点，一个算法必须在每个区块高度选择区块作者。

对于基于substrate的区块链，你可以选择以下区块生成算法之一，或者创建你自己的算法:

- Authority-based round-robin scheduling(Aura) [Glossary | Substrate_ Docs](https://docs.substrate.io/reference/glossary/#authority-round-aura
- Blind assignment of blockchain extension(BABE) slot-based scheduling [Glossary | Substrate_ Docs](https://docs.substrate.io/reference/glossary/#blind-assignment-of-blockchain-extension-babe)
- PoW computation-based scheduling.

Aura和BABE共识模型要求你有一组已知的允许产生块的Validator nodes。在这两种共识模型中，时间被划分为不连续的slots。在每个slot中，只有一些Validator可以产生一个区块。在Aura共识模型中，可以制作区块的Validator以Round-Robin的方式进行轮换。在BABE共识模型中，Validator的选择是基于可验证的随机函数（VRF），而不是Round-Robin选择的方法。

在PoW共识模型中，任何节点都可以在任何时候产生一个区块，如果该节点已经解决了一个计算密集型的问题。解决该问题需要CPU时间，因此节点只能按照其计算资源的比例生产区块。Substrate提供了一个PoW的区块生产引擎。

### 确定和分叉

作为一个基本原语，一个区块包含一个header和一堆交易。每个区块header都包含对其前一个区块的引用，因此你可以追溯到其起源的创世块。当两个区块引用同一个父块时，就会发生分叉。区块确定是一种解决分叉的机制，使其只存在canonical chain。

分叉选择规则是一种算法，用于选择应该被扩展的最佳链。Substrate通过`SelectChain trait`暴露了这个分叉选择规则。你可以使用该trait来编写你的自定义分叉选择规则，或者也可以直接使用GRANDPA [consensus/grandpa.pdf at master · w3f/consensus (github.com)](https://github.com/w3f/consensus/blob/master/pdf/grandpa.pdf)，即Polkadot和类似链中使用的最终确认机制。

在GRANDPA协议中，最长链规则简单地说，best chain就是最长的链。Substrate用`LongestChain struct`提供了这个链选择规则。GRANDPA在其投票机制中使用最长链规则。

![[Pasted image 20220826093230.png]]


贪婪的最重观察子树（GHOST，The Greedy Heaviest Observed SubTree）规则说，从创世块开始，每个分叉都是通过选择有最多块的分支递归来解决的。

![[Pasted image 20220826093427.png]]


### 确定性finality

用户很自然地想知道交易何时完成，并通过一些事件（如 receipt deliverred或papers signed）发出信号。然而，在迄今为止描述的区块授权生产和分叉选择规则下，交易永远不会完全最终完成。总是有一个偶然的机会，一个更长或更重的链可能会回滚交易。然而，越多的区块被建立在一个特定的区块之上，它就越不可能被回滚。这样一来，区块授权生产和适当的分叉选择规则提供了概率上的最终性。  
  
如果你的区块链需要确定的finality，你可以在区块链逻辑中加入finality机制。例如，你可以让一个固定的权威集的成员投出最终投票。当某一区块有足够多的投票时，该区块就被视为finalited。在大多数区块链中，这个门槛是三分之二。如果没有外部协调，如硬分叉，已经最终确定的区块不能被回滚。  
  
在一些共识模型中，区块生产和区块finality是结合在一起的，在区块N被finalited之前，新的区块N+1不能被生产出来。正如你所看到的，在Substrate中，这两个过程是相互隔离的。通过将区块授权编写与区块fniality分开，Substrate使你能够使用任何具有概率最终性的区块编写算法，或将其与finality机制相结合，以实现确定性的finality。  
  
如果你的区块链使用finality机制，你必须修改分叉选择规则以考虑最终性投票的结果。例如，一个节点将不采取最长的链，而是采取包含最近最终确定的区块的最长的链。  

### 默认的共识模型

虽然你可以实现你自己的共识机制，但Substrate node template [substrate-developer-hub/substrate-node-template: A new FRAME-based Substrate node, ready for hacking. (github.com)](https://github.com/substrate-developer-hub/substrate-node-template) 默认包括Aura，用于区块创作和GRANDPA最终确定。Substrate还提供了BABE和工作证明共识模型的实现。

#### Aura

Aura提供了一个slot-bsaed的区块编写机制。在Aura中，一组已知的权威(Validators)轮流制作区块。[sc_consensus_aura - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_consensus_aura/index.html)

#### BABE

BABE提供slot-bsaed的区块创作，有一组已知的Validators，通常用于PoS的区块链。与Aura不同，slot的分配是基于可验证的随机函数（VRF）的评估。每个Validator都被分配了一个epoch的权重。这个epoch被分成几个slots，Validator在每个slot评估其VRF。对于Validator的VRF输出低于其权重的每个slot，它被允许编写一个区块。

由于多个validators可能在同一个slot中产生一个区块，因此在BABE中，即使在良好的网络条件下，分叉也比在Aura中更常见。

Substrate对BABE的实现也有一个后备机制，当在一个给定的slot中没有验证者被选中时。这些secondray slot分配使BABE能够实现恒定的block time。也就是块的间隔时间是恒定的。 [sc_consensus_babe - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_consensus_babe/index.html)

#### PoW

PoW区块授权生成不是slot-based的，不需要一个已知的权威集(validators)。在PoW中，任何人都可以在任何时候产生一个区块，只要他们能够解决一个具有计算难度的问题。这个问题的难度可以被调整，以提供一个统计上的目标区块时间。

#### GRANDPA

GRANDPA提供了区块的最终确定（finalization）。它有一个像BABE一样的已知加权权威集。然而，GRANDPA并不编写区块。它只是监听由区块授权生产节点产生的区块的P2P广播。GRANDPA validator对链而不是区块进行投票。GRANDPA validator对他们认为最好的区块进行投票，他们的投票将被应用于所有先前的区块。在三分之二的GRANDPA validator对某一特定区块投票后，该区块被认为是最终的。

所有确定性的最终算法，包括GRANDPA，都要求至少有`2f+1`个`非故障`节点，其中f是故障或恶意节点的数量。了解更多关于这个阈值的来源以及为什么它是理想的，请看开创性的论文《Reaching Agreement in the Presence of Faults》[reaching.pdf (lamport.azurewebsites.net)](https://lamport.azurewebsites.net/pubs/reaching.pdf) 或在维基百科，拜占庭错误 [Byzantine fault - Wikipedia](https://en.wikipedia.org/wiki/Byzantine_fault)。

并非所有的共识协议都定义了一个单一的cononical chain。一些协议验证了有向无环图（DAG）[Directed acyclic graph - Wikipedia](https://en.wikipedia.org/wiki/Directed_acyclic_graph) ，当两个具有相同父本的块没有冲突的状态变化时。这样的例子见AlephBFT [aleph-zero-foundation/aleph-node: Mirror of development repo: https://github.com/Cardinal-Cryptography/aleph-node](https://github.com/aleph-zero-foundation/aleph-node)。


## 交易和区块的基础

在这篇文章中，你将了解到你可以创建的不同类型的交易，以及你如何在runtime中使用它们。广义上讲，交易决定了进入你的区块链中的数据的方式。通过学习不同的交易类型是如何使用的，你将更好地准备为你的需求选择合适的类型。

### 什么是交易

一般来说，交易提供了一种对状态进行改变的机制，这些改变可以被包含在一个区块中。在Substrate中有三种不同的交易类型。

- 已签名的交易
- 未签名交易
- 内在交易

在Substrate中，所有这三种交易类型通常被更广泛地称为`extrinsics`。术语extrinsic通常被用来指任何源自于runtime之外的信息。

然而，出于实际目的，独立考虑每种交易型并确定每种类型最适用的场景更为有用。

#### 已签名交易

签名交易必须包括一个账户的签名，该账户发送inbound请求以执行一些runtime调用。通常情况下，请求是使用提交请求的账户的私钥签署的。在大多数情况下，提交请求的账户也支付交易费。然而，交易费用和交易处理的其他要素取决于runtime逻辑的定义方式。

签名交易是最常见的交易类型。作为一个例子，假设你有一个拥有一定数量代币的账户。如果你想把代币转给Alice，你可以调用Balances pallet中的pallet_balances::Call::transfer函数。因为你的账户被用来做这个调用，你的账户私钥被用来签署交易。作为请求者，你通常会负责支付一定的费用来处理你的请求。另外，你也可以给区块作者一些小费，让他给予你的交易更高的优先权。

#### 未签名交易

无签名的交易不需要签名，也不包括任何关于谁提交交易的信息。

对于无签名的交易，没有经济威慑力（比如交易费）来防止垃圾邮件或重放攻击。你必须定义验证无签名交易的条件以及保护网络不被滥用和攻击所需的逻辑。因为无签名交易需要自定义验证，这种交易类型比签名交易消耗更多的资源。

`pallet_im_online::Call::heartbeat`函数使用无签名交易，使Validator节点向网络发送信号，表示节点在线。这个函数只能由在网络中注册为Validator的节点调用。该函数包括内部逻辑来验证节点是一个Validator，允许节点使用无签名交易调用该函数，以避免支付任何费用。

#### 内在交易(inherent tx)

内在交易--有时被称为`inherents` --是一种特殊类型的无签名交易。通过这种类型的交易，区块编写生产节点可以直接向区块中添加信息。内在交易只能由调用它们的区块编写节点插入到区块中。通常情况下，这种类型的交易不会被传给其他节点或存储在交易队列中。使用内在交易插入的数据被认为是有效的，不需要特定的验证。

例如，如果一个区块编写节点在一个区块中插入一个时间戳，没有办法证明一个时间戳是准确的。相反，验证者可能会根据时间戳是否在他们自己的系统时钟的某个可接受范围内来接受或拒绝该块。

作为一个例子，pallet_timestamp::Call::now函数使一个区块编写节点在该节点产生的每个区块中插入一个当前的时间戳。同样，paras_inherent::Call::enter函数使一个parachain collator节点向其中继链发送中继链所期望的验证数据。

### 什么是区块

在Substrate中，一个块由一个Header’和一个交易数组组成。头部包含以下属性:

- 区块高度
- 父哈希值
- 交易 root
- state root
- 摘要

所有的交易被捆绑在一起作为一个系列，按照runtime的定义执行。交易root是这个系列的一个加密摘要。这个加密摘要有两个目的。

- 它可以防止在Header被建立和分发后对该系列交易的任何改动。(类似于Merkle tree) 
- 它使轻型客户端节点能够简洁地验证任何给定的交易是否存在于一个区块中，只需给出Header的知识。

## 交易的生命周期

在Substrate中，交易的数据被打包在块中。由于交易中的数据来自于runtime之外，交易有时被更广泛地称为`extrinsics`。然而，最常见的`extrinsics`是签名交易。因此，对交易生命周期的讨论主要集中在如何验证和执行签名交易。

你已经了解到，签名交易包括发送请求的账户的签名，以执行一些runtime调用。通常情况下，请求是使用提交请求的账户的私钥签署的。在大多数情况下，提交请求的账户也会支付一笔交易费。然而，交易费和交易处理的其他要素取决于runtime逻辑的定义方式。


### 交易在哪里被定义

正如在Runtime开发中所讨论的，Substrate runtime包含了定义交易属性的业务逻辑，包括:

- 什么构成有效的交易。
- 交易是以签名还是未签名的方式发送。
- 交易如何改变链的状态。

通常，你使用pallets来组成runtime功能，并实现你希望你的链支持的交易。在你编译runtime后，用户与区块链互动，提交作为交易处理的请求。例如，一个用户可能会提交一个请求，将资金从一个账户转移到另一个账户。该请求成为一个有签名的交易，其中包含该用户账户的签名，如果用户的账户中有足够的资金来支付该交易，该交易就会成功执行，并进行转账。

### 交易在块的授权生产节点上是怎样被处理的

根据你的网络配置，你可能有被授权制作区块的节点和未被授权制作区块的节点的组合。如果一个substrate节点被授权生产区块，它可以处理它收到的有签名和无签名的交易。下图说明了一个提交给网络并由编写区块的节点处理的交易的生命周期。

![[Pasted image 20220826102302.png]]

任何发送到[非区块编写节点](https://docs.substrate.io/fundamentals/transaction-lifecycle/)的有签名或无签名的交易都会被P2P广播到网络中的其他节点，并进入他们的交易池，直到被区块编写节点收到。


### 校验和排队(queuing)交易

正如在共识中所讨论的，网络中的大多数节点必须就区块中的交易顺序达成一致，以确认区块链的状态，并继续安全地添加下一个区块。为了达成共识，三分之二的节点必须确定同意所执行的交易的顺序和由此产生的状态变化。为了准备达成共识，交易首先在本地节点的交易池中进行验证和排队。

#### 在交易池中校验交易

使用在runtime定义的规则，交易池检查每个交易的有效性。这些检查确保只有符合特定条件的有效交易才会被打包在一个区块中。例如，交易池可以执行以下检查来确定一个交易是否有效。

- 交易index--也被称为交易nonce--是否正确？这个nonce，ETH也有
- 用于签署交易的账户是否有足够的资金来支付相关费用？
- 用于签署交易的签名是否有效？

在最初的有效性检查之后，交易池定期检查池中现有的交易是否仍然有效。如果发现一个交易是无效的或已经过期，它就会从交易池中删除。

交易池只处理交易的有效性和放在交易队列中的有效交易的排序问题。关于验证机制如何工作的具体细节--包括对费用、账户或签名的处理，可以在`validate_transaction`方法中找到。

#### 加入有效交易到交易队列中

如果一个交易被确认为有效，交易池就会将该交易移到交易队列中。有效交易有两个交易队列:

- ready queue 包含可以包含在一个新的待处理块中的交易。如果runtime是用FRAME构建的，交易必须遵循它们在就ready queue的确切顺序。
- future queue，包含可能在未来变得有效的交易。例如，如果一个交易的nonce对其账户来说太高，它可以在future queue中等待，直到该账户的适当数量的交易被纳入链中。

#### 无效交易的处理

如果一个交易是无效的--例如，因为它太大或不包含有效的签名--它将被拒绝，不会被添加到一个区块中。一个交易可能因为以下任何原因被拒绝。

- 该交易已经被包含在一个区块中，所以它被从验证队列中删除。
- 该交易的签名无效，所以它立即被拒绝。
- 该交易太大，不适合当前区块（当前块的size满了），所以它被放回队列中进行新一轮验证。

### 根据优先级(priority)对交易进行排序

如果一个节点是下一个区块的作者，该节点使用一个优先级系统来为下一个区块的交易排序。交易从高到低的优先级排序，直到区块达到最大权重或长度（这里指size长度）。

交易的优先级是在runtime中计算的，并作为交易的标签提供给外层节点。在FRAME runtime中，一个特殊的pallet被用来根据与交易相关的权重和费用来计算优先级。这种优先级计算适用于所有类型的交易，但内在交易(inherents)除外。使用`EnsureInherentsAreFirst trait`，Inherents总是被放在第一位。[EnsureInherentsAreFirst in frame_support::traits - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/traits/trait.EnsureInherentsAreFirst.html)

#### 基于账户的交易排序

如果你的runtime是用FRAME构建的，每个签名的交易都包含一个nonce，这个nonce在特定账户进行新交易时被递增(这里与ETH一样)。例如，一个新账户的第一笔交易有nonce = 0，同一账户的第二笔交易有nonce = 1。区块制作生产节点可以在排序交易时使用nonce，以打包在一个区块中。

对于有依赖关系的交易，排序时考虑到交易支付的费用和它所包含的其他交易的任何依赖关系。比如说。

- 如果有一个具有`TransactionPriority::max_value()`的无签名交易和另一个签名交易，无签名交易会被放在队列的第一位。
- 如果有两个来自不同发送方的交易，`priority`决定了哪个交易更重要，应该首先被包含在区块中。
- 如果有两个来自同一发送方的交易，并且具有相同的nonce：只有一个交易可以被包含在区块中，所以只有具有较高费用的交易被包含在队列中。

### 执行交易和生产区块

在有效的交易被放入交易队列后，一个单独的excutive 模块协调交易如何被执行以产生一个块。excutive 模块调用runtime模块中的函数，并按特定顺序执行这些函数。

作为一个runtime的开发者，了解excutive 模块如何与system pallet和其他构成区块链业务逻辑的pallet互动是很重要的，因为你可以插入excutive 模块的逻辑，作为以下操作的一部分来执行。

- 初始化一个区块
- 执行区块中包含的交易
- 最终完成区块构建

#### 初始化区块

为了初始化一个块，excutive 模块首先在system pallet中调用`on_initialize`函数，然后在所有其他runtime pallet中调用。`on_initialize`函数使你能够定义在交易执行前应该完成的业务逻辑。system pallet的`on_initialize`函数总是首先被执行。其余的pallet按照在 `construct_runtime!` 宏中定义的顺序被调用。

在所有的`on_initialize`函数被执行后，excutive 模块会检查区块header中的父哈希值和trie root，以验证信息是否正确。

#### 执行交易

在区块被初始化后，每个有效的交易都按照交易的priority的顺序被执行。重要的是要记住，在执行之前，状态不会被缓存。相反，在执行过程中，状态的变化被直接写入存储。如果一个交易在执行过程中失败了，那么在失败之前发生的任何状态变化都不会被回滚，从而使块处于不可回滚的状态。在向存储提交任何状态变化之前，runtime逻辑应该执行所有必要的检查，以确保extrinsic会成功。

请注意，events [Events and errors | Substrate_ Docs](https://docs.substrate.io/build/events-and-errors/) 也会被写入存储中。因此，runtime逻辑不应该在执行补充动作之前发出事件。如果在事件发出后交易失败，该事件将不会被回滚。

#### 确认区块

在所有排队的交易被执行后，excutive 模块调用每个pallet的`on_idle`和`on_finalize`函数来执行任何应该在区块结束时发生的最终业务逻辑。这些模块再次按照它们在`construct_runtime！`宏中定义的顺序执行，但在这种情况下，system pallet中的`on_finalize`函数被最后执行。这个就像栈结构一样的原理。

在所有的`on_finalize`函数被执行后，excutive 模块检查区块header中的摘要和storage root是否与区块初始化时的计算结果一致。

`on_idle`函数还通过区块的剩余权重，以便根据区块链的使用情况来执行。

### 区块授权编写和区块导入(插入)

到目前为止，你已经看到了交易是如何被包含在本地节点产生的区块中的。如果本地节点被授权生产区块，交易的生命周期就会遵循这样的路径。  
  
- 本地节点监听网络上的交易。  
- 每个交易都被验证。  
- 有效的交易被放置在交易池中。  
- 交易池将有效的交易放在适当的交易队列中，excutive 模块调用runtime开始下一个块。  
- 交易被执行，状态变化被存储在本地存储引擎中。  
- 构建的区块被发布到网络上。  

在区块被发布到网络上后，它可以供其他节点插入。区块插入队列(import queue)是每个Substrate节点的外层节点的一部分。区块导入队列监听传入的区块和共识相关的信息，并将它们添加到一个池中。在这个池子里，传入的信息被检查是否有效，如果无效就被丢弃。在验证了区块或消息的有效性后，区块导入队列将传入的信息导入到本地节点的状态中，并将其添加到该节点所知道的区块数据库中。  
  
在大多数情况下，你不需要知道关于交易如何P2P广播或区块如何被网络上的其他节点导入的细节。但是，如果你打算写任何自定义的共识逻辑，或者想知道更多关于区块导入队列的实现，你可以在Rust API文档中找到细节。  
  
- [ImportQueue in sc_consensus::import_queue - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_consensus/import_queue/trait.ImportQueue.html)
- [Link in sc_consensus::import_queue - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_consensus/import_queue/trait.Link.html)
- [BasicQueue in sc_consensus::import_queue - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_consensus/import_queue/struct.BasicQueue.html)
- [Verifier in sc_consensus::import_queue - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_consensus/import_queue/trait.Verifier.html)
- [BlockImport in sc_consensus::block_import - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_consensus/block_import/trait.BlockImport.html)


## 状态转移和存储

Substrate使用一个简单的KV数据存储，它被实现为一个以数据库为基础的、经过修改的Merkle树。Substrate的所有高层存储抽象 [Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/) 都是建立在这个简单的KV存储之上的。

### KV数据库

Substrate用RocksDB实现了它的存储数据库，这是一个用于快速存储环境的持久性KV存储。它还支持一个实验性的Parity DB。

该数据库被用于Substrate的所有需要持久化存储的组件，例如。

- Substrate客户端
- Substrate轻型客户端
- 链外工作者

### trie的抽象

使用一个简单的KV存储的一个好处是，你能够很容易地在它上面抽象出存储结构。  
  
Substrate使用来自`paritytech/trie` [paritytech/trie: Base-16 Modified Patricia Merkle Tree (aka Trie) (github.com)](https://github.com/paritytech/trie) 的Base-16 Modified Merkle Patricia tree（"trie"）来提供一个trie结构，其内容可以被修改，其root hash被有效地重新计算。  
  
trie允许高效地存储和共享历史区块状态。trie root是trie内数据的代表；也就是说，两个具有不同数据的tries将总是有不同的root。因此，两个区块链节点可以通过简单地比较它们的trie root来轻松验证它们是否有相同的状态。  
  
访问 trie 数据的成本很高。每个读操作需要O(log N)时间，其中N是存储在trie中的元素数量。为了缓解这个问题，我们使用一个KV缓存。  
  
所有的 trie 节点都存储在数据库中，部分 trie 状态可以得到修剪，也就是说，当一个KV pair超出非归档节点(non-archive node)的修剪范围时，可以从存储中删除。由于性能原因，我们不使用引用计数。  

#### state trie

基于substrate的链有一个单一的主 trie，称为 state trie，其root hash被放在每个区块header中。这被用来轻松验证区块链的状态，并为轻型客户节点验证证明提供基础。

这个 trie 只存cononical chain的内容，而不是分叉。有一个单独的state_db [sc_state_db - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_state_db/index.html) 层用引用计数在内存中来维护non-canonical chain 的 trie的状态

#### child trie

Substrate还提供了一个API来生成具有自己的root hash的新的子tries，可以在runtime中使用。

子 trie 与主state trie 相同，只是子 trie 的root被作为主 trie 中的一个节点而不是区块header来存储和更新。由于它们的块header是主状态 trie 的一部分，当它包括子trie时，仍然很容易验证完整的节点状态。

当你想要自己的独立trie有一个单独的root hash时，子tries很有用，你可以用它来验证该trie中的特定内容。state trie中的subsections 没有 root-hash-like 的表示，不能自动满足这些需求；因此使用子child trie来代替。

### 查询存储

用Substrate构建的区块链暴露了一个远程程序调用（RPC）服务器，可用于查询runtime存储。当你使用Substrate RPC来访问一个存储项(storage item)时，你只需要提供与该项目相关的Keys（这里的key是指KV存储的Key）。Substrate的runtime storage API [Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/) 暴露了许多存储项的类型；继续阅读以了解如何为不同类型的存储项计算存储Keys。

#### Storage value keys

要计算一个简单的存储值[Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/#storage-value)的Key，需要取包含存储值的pallet名称的TwoX 128[Cyan4973/xxHash: Extremely fast non-cryptographic hash algorithm (github.com)](https://github.com/Cyan4973/xxHash) 哈希值，并将Storage Value本身的Key名称的TwoX 128哈希值附加到它上面。例如，Sudo pallet暴露了一个名为Key的storage value项:

```rust
twox_128("Sudo")                   = "0x5c0d1176a568c1f92944340dbfed9e9c"
twox_128("Key")                    = "0x530ebca703c85910e7164cb7d1c9e47b"
twox_128("Sudo") + twox_128("Key") = "0x5c0d1176a568c1f92944340dbfed9e9c530ebca703c85910e7164cb7d1c9e47b"
```

根据以上就可以看出来 `0x5c0d1176a568c1f92944340dbfed9e9c530ebca703c85910e7164cb7d1c9e47b` 就是在Rocks‘DB中的Key的值，是被编码后的。

如果熟悉的Alice账户是sudo用户，读取Sudo pallet的Key存储值的RPC请求和响应可以表示为：

```rust
state_getStorage("0x5c0d1176a568c1f92944340dbfed9e9c530ebca703c85910e7164cb7d1c9e47b") = "0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d"
```
’
根据以上返回的`0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d` 就是Alice的经过SCALE编码后 [Type encoding (SCALE) | Substrate_ Docs](https://docs.substrate.io/reference/scale-codec/) 的account ID(`5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY`)

你可能已经注意到，非密码学的TwoX 128哈希算法被用来生成存储值的Key。这是因为没有必要支付与密码学哈希函数相关的性能成本，因为哈希函数的输入（pallet和strage item的名称）是由runtime开发人员决定的，而不是由你的区块链潜在的恶意用户决定的。

#### Storage map keys

和storage value一样，storage map [Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/#storage-map) 的key等于包含map的pallet名称的2X128哈希值，预加到storage map本身名称的2X128哈希值。要从一个map中检索一个元素，将所需的map key的哈希值附加到storage map的storage Key上。对于有两个Key的map（Storage double maps），将第一个map key的哈希值和第二个map key的哈希值追加到storage double maps的storage key上。

像storage value一样，Substrate对pallet和storage map名称使用TwoX 128散列算法，但你需要确保在确定map中元素的散列键时使用正确的散列算法 [Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/#hashing-algorithms)（即在#\[pallet::storage\]  [Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/#declaring-storage-items) 宏中声明的算法）。

下面是一个例子，说明从一个名为Balances的pallet中查询名为FreeBalance的storage map，以获得Alice账户的余额。在这个例子中，FreeBalance map使用的是透明的Blake2 128 Concat散列算法。[Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/#transparent-hashing-algorithms)

```rust
twox_128("Balances")                                             = "0xc2261276cc9d1f8598ea4b6a74b15c2f"
twox_128("FreeBalance")                                          = "0x6482b9ade7bc6657aaca787ba1add3b4"
scale_encode("5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY") = "0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d"

blake2_128_concat("0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d") = "0xde1e86a9a8c739864cf3cc5ec2bea59fd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d"

state_getStorage("0xc2261276cc9d1f8598ea4b6a74b15c2f6482b9ade7bc6657aaca787ba1add3b4de1e86a9a8c739864cf3cc5ec2bea59fd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d") = "0x0000a0dec5adc9353600000000000000"
```

从以上代码可以看出，storage map的storage key是把hash和scale编码后的值的Hex字符串作追加，而得到key。再去state storage里面get。



### Runtime存储API

Substrate的FRAME support crate [frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/index.html) 提供了为你的runtime的Storage item生成唯一的、确定的key的实用工具。这些storage item被放置在state trie 中，并且可以通过key来查询 trie 来访问。

## 账户，地址和Keys

一个账户代表一个身份，通常是一个人或一个组织，能够进行交易或持有资金。虽然账户最常被用来代表一个人，但情况不一定是这样。一个账户可以用来代表一个用户或一个实体进行操作，或者自主地进行操作。此外，任何一个人或实体都可以创建并拥有多个账户。例如，Polkadot是一个基于Substrate的区块链，它有专门的账户用于持有资金，与用于进行交易的账户分开。如何实现和使用账户，完全取决于你作为区块链或parachain的开发者。

### 公私钥

一般来说，每个账户都有一个拥有公钥和私钥的所有者。私钥是一个加密安全的随机生成的数字序列。为了便于人类阅读，私钥会生成一个随机的文字序列，称为秘密种子短语(**secret seed phrase**)或助记符(**mnemonic**)。如果私钥丢失，该秘密种子短语可由账户所有者用于恢复对账户的访问。

对于大多数网络来说，与一个账户相关的公钥是该账户在网络上被识别的方式，并被用作交易的目标地址。然而，基于substrate的链使用底层公钥来推导一个或多个公有地址(public address)。与直接使用公钥不同，Substrate允许你为一个账户生成多个地址和地址格式。

### 地址编码和chain-specific地址

Substrate使你能够使用一个公钥来推导出多个地址，因此你可以与多个链进行交互，而无需为每个网络创建单独的公钥和私钥对。默认情况下，与一个账户的公钥相关的地址使用Substrate SS58地址格式 [Glossary | Substrate_ Docs](https://docs.substrate.io/reference/glossary/#ss58-address-format)。这种地址格式是基于base-58编码的, 除了允许你从同一个公钥衍生出多个地址外，base-58编码还有以下好处。  
  
- 编码后的地址由58个字母数字字符组成。  
- 字母数字字符串省略了字符--如0、O、I和l--这些字符在字符串中很难相互区分。  
- 网络信息--例如，网络特定的前缀--可以在地址中进行编码。  
- 可以使用校验和检测输入错误，以确保地址输入正确。  

因为一个公钥可以用来为不同的substrate链的推导地址，所以一个账户可以有多个特定链的地址。例如，如果你检查alice账户公钥`0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d`所表示的地址，那么取决于链特定的地址类型。  

| 链地址格式   |   地址       |
| -----------     | ----------  |
|     Polkadot (SS58)  |  15oF4uVJwmo4TdGW7VfQxNLavjCXviqxT9S1MgbjMNHr6Sp5           |
|      Kusama (SS58)  |  HNZata7iMYWmk5RvZRTiAsSDhV8366zq2YGb3tLH5Upf74F           |
|Generic Substrate chain (SS58)|5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY|

每个Substrate区块链都可以注册一个自定义前缀，以创建一个特定的链上地址类型。例如，所有Polkadot地址以1开头，所有Kusama地址以大写字母开头。所有未注册的Substrate链以5开头。

你可以使用`subkey inspect`命令和`--network`选项或使用Subscan来查询公钥的链特定地址。[Subscan | Polkadot Explorer - Blockchain Explorer - Aggregate Substrate ecological network High-precision Web3 explorer](https://polkadot.subscan.io/tools/format_transform)

关于生成公钥和私钥对以及检查地址的信息，见subkey [subkey | Substrate_ Docs](https://docs.substrate.io/reference/command-line-tools/subkey/)。关于链式特定地址的信息，见SS58注册库中的说明。[paritytech/ss58-registry: Registry for SS58 account types (github.com)](https://github.com/paritytech/ss58-registry)


### 在FRAME里面的账户信息

从概念上讲，账户代表身份，有一个公共/私人密钥对，有一个或多个公共地址。然而，在用FRAME构建的runtime中，账户被定义为一个具有32字节地址标识的storage map和相应的账户信息，如账户的交易数量，依赖账户的模块数量，以及账户余额。

帐户属性--如`AccountId`--可以在`frame_system`模块中通用地定义。然后，通用类型在runtime实现中被解析为一个特定的类型，并最终分配一个特定的值。例如，FRAME中的`Account`类型依赖于一个相关的`AccountId`类型。`AccountId`类型仍然是一个通用类型，直到它在runtime实现中为需要此信息的pallet分配一个类型。

关于如何在`frame_system` pallet中定义账户以及`Account` storage map中的账户属性，请参见账户数据结构 [Account data structures | Substrate_ Docs](https://docs.substrate.io/reference/account-data-structures/)。更多关于使用通用类型的信息，请参见Rust for Substrate [Rust for Substrate | Substrate_ Docs](https://docs.substrate.io/fundamentals/rust-basics/#generic-types)。

### 专门的账户

尽管大多数账户用于代表控制资金或执行交易的公钥/私钥对，但Substrate支持一些专门的账户来控制特定的密钥对如何使用。例如，你可能有需要定制加密方案的账户，只能用于执行特定功能，或只能访问特定的pallet。

#### Staking账户和Keys

在大多数情况下，专门账户是在特定的FRAME pallet背景下实现的。例如，提名的赌注证明（NPoS）可能需要节点Validator和提名人持有大量的代币。为了保持这些账户中的余额安全，Staking pallet提供了一些账户抽象，将执行特定操作所需的密钥对分开。

![[Pasted image 20220826145532.png]]

（TO DO：翻译上表）

#### Keyless代理账户

在某些情况下，你可能想创建一个与任何所有者分离的账户，这样它就可以被用来执行自主交易。例如，你可能会创建一个账户，然后将权限委托给该账户，这样它就可以在没有你的干预和没有访问你的秘钥的情况下分发函数调用。在创建了具有委托权限的新账户后，该账户可以作为一个接收者来燃烧资金或持有代币等待执行或转移。

你可以使用proxy pallet来创建一个有权限的账户，使用被委托的账户作为签名的原点来分发某些类型的调用。

## Rust for Substrate

为什么用Rust就不说了，正如它所说的安全，高性能，对WASM支持也友好，可移植性也高。

### Substrate中的Rust

在架构部分，你将了解到Substrate是由两个不同的架构组件组成的：外层节点(节点外层)和runtime。虽然Rust中更复杂的功能，如多线程和异步Rust在外层节点的代码中使用，但它们并不直接暴露给开发runtime的开发者，这使得runtime工程师更容易专注于其节点的业务逻辑。

一般来说，根据他们的重点，开发人员应该期望知道:

- 基本的Rust范式 [Idioms - Rust Design Patterns (rust-unofficial.github.io)](https://rust-unofficial.github.io/patterns/idioms/index.html)，与no_std一起工作[no_std - The Embedded Rust Book (rust-embedded.org)](https://docs.rust-embedded.org/book/intro/no-std.html) ，以及使用什么宏以及为什么使用宏（对于runtime开发而言）。
- 异步Rust（对于更高级的开发人员来说，他们需要处理外层节点（客户端）代码）。[Getting Started - Asynchronous Programming in Rust (rust-lang.github.io)](https://rust-lang.github.io/async-book/01_getting_started/01_chapter.html)

尽管在进入substrate之前，对Rust的一般熟悉是必不可少的，而且有许多资源可用于学习Rust，包括Rust语言编程书 [The Rust Programming Language - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/) 和Rust by Example [Introduction - Rust By Example (rust-lang.org)](https://doc.rust-lang.org/rust-by-example/)，但本节的其余部分强调了Substrate使用Rust的一些核心功能的方式，供开发人员在runtime开发入手。

### 编译的target

在构建Substrate节点时，我们使用`wasm32-unknown-unknown`编译target，这意味着Substrate runtime工程师被限制在编写必须编译为Wasm的runtime。这意味着你不能依赖一些典型的标准库类型和函数，而且必须在大多数runtime代码中只使用no_std兼容的crates。Substrate有很多自己的原始类型和相关的traits，使其有可能绕过no_std的要求。

相当于一旦开发runtime，依赖std的crates统统不能用了。

### Macros

当你学习编写FRAME pallets时，你会很快接触到各种不同类型的宏，旨在抽象和执行任何runtime的具体要求。有了它们，你就能专注于编写Rust style的Rust，尽量减少编写额外代码的开销，否则你就需要编写与runtime正确交互的代码。  
  
Rust宏是一个强大的工具，可以帮助确保满足某些要求（无需重写代码），如逻辑要以特定的方式格式化，进行特定的检查，或一些逻辑由特定的数据结构组成。这对于帮助开发者编写能够与Substrate runtime的复杂性相结合的代码特别有用。例如，`#\[frame_system::pallet\]`宏在所有FRAME pallets中都是必需的，以帮助你正确实现某些必需的属性--如存储项或外部可调用的函数--并使其与`construct_runtime`的构建过程兼容。  
  
开发Substrate runtime需要大量使用Rust的属性宏，它有两种类型：派生属性和自定义属性。当你开始使用Substrate时，知道它们是如何工作的并不重要，重要的是要知道它们的存在，使你能够编写正确的runtime代码。  
  
派生属性对于需要满足某些traits的自定义runtime类型很有用，例如，让类型在runtime执行时可被节点解码。  
  
其他类似宏的属性在整个Substrate的代码库中也很常见，用于: 
  
- 告诉编译器代码片段是否要编译为no_std或是否可以访问std库。  
- 自定义FRAME support 宏，用于编写pallets。  
- 指定runtime的构建方式。  

### 泛型和configuration traits

通常与Java等语言中的interface相比，Rust中的traits提供了为一个类型赋予高级功能的方法。  
  
如果你读过关于pallet的文章，你可能已经注意到每个pallet都有一个Config trait，它允许你定义一个pallet所依赖的类型和接口。  
  
这个traits本身从`frame_system::pallet::Config` trait中继承了一些核心runtime类型，使得在编写runtime逻辑时可以很容易地访问通用类型。此外，在任何FRAME pallet中，`Config` trait在T上是通用的（下一节将详细介绍通用性）。这些核心runtime类型的一些常见例子可以是`T::AccountId`，用于识别runtime的用户账户的通用类型，或者`T::BlockNumber`，runtime使用的块编号类型。  
  
关于Rust中泛型和特质的更多信息，请参见Rust书中的泛型 [Generic Data Types - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/ch10-01-syntax.html) 、traits [Traits: Defining Shared Behavior - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/ch10-02-traits.html) 和高级traits部分 [Advanced Traits - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/ch19-03-advanced-traits.html)。
  
通过 Rust 泛型，Substrate runtime开发人员可以编写完全与具体实现细节无关的pallet，从而充分利用 Substrate 的灵活性、可扩展性和模块化。  
  
`Config` trait 中的所有类型都是使用trait 约束进行泛型定义的，并在runtime实现中变得具体。这不仅意味着你可以编写支持同一类型的不同规格的pallet（例如，Substrate和Ethereum链的地址），而且你还可以根据你的需求定制通用的实现，并将开销降到最低（例如，将块数改为`u32`）。  
  
这使开发人员能够灵活地编写代码，不对你所做的核心区块链架构决定进行假设。  
  
Substrate最大限度地使用了泛型，以提供最大的灵活性。你定义如何解决通用类型以适应你的目的。  
  
关于Rust中的泛型和特征的更多信息，请参见Rust书中关于泛型的章节。  [Generic Data Types - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/ch10-01-syntax.html)
  

## 链下操作

在很多用例中，你可能想从链外源查询数据，或者在更新链上状态之前不使用链上资源处理数据。纳入链外数据的传统方式包括连接到Oracle [Glossary | Substrate_ Docs](https://docs.substrate.io/reference/glossary/#oracle)，从一些传统来源提供数据。虽然使用oracle是与链外数据源合作的一种方法，但Oracle所能提供的安全性、可扩展性和基础设施效率都有局限性。

为了使链外数据整合更加安全和高效，Substrate通过以下功能支持链外操作。

- 链外工作者(offchain workers)是一个组件子系统，能够执行长期运行的、可能是非确定性的任务，例如:
    - 网站服务请求
    - 数据的加密、解密和签名
    - 随机数生成
    - CPU密集型的计算
    - 枚举或聚合链上数据 链外工作者使你能够将那些可能需要更多时间执行的任务移出区块处理管道。任何可能需要超过允许的最大区块执行时间的任务，都是链下处理的合理候选。
- 链外存储(offchain storage)是Substrate节点本地的存储，可由链外工作者和链上逻辑访问。
     - 链外工作者对链外存储有读取和写入的权限。
     - 链上逻辑通过链下索引有写入权限，但没有读取权限。链外存储允许不同的工作线程相互通信，并存储不需要在整个网络上达成共识的用户特定或节点特定的数据。
- 链外索引（offchain indexing）是一种可选的服务，它允runtime独立于链外工作者直接向链外存储写入。链外索引为链上逻辑提供临时存储，并补充了链上状态。

### 链下的工作者

链外工作者在他们自己的Wasm执行环境中运行，在Substrate runtime之外。这种关注点的分离确保了区块生产不会受到长期运行的链外任务的影响。然而，由于链下工作者是在与runtime相同的代码中声明的，他们可以很容易地访问链上状态进行计算。

相当于offchain workder其实也是跑在substrate node里面的:

![[Pasted image 20220826152709.png]]

链外工作者可以访问扩展的API，与外部世界进行交流:

- 能够向链上提交交易--无论是签名还是不签名的交易，以发布计算结果。
- 一个功能齐全的HTTP客户端，允许工作者访问和获取外部服务的数据。
- 访问本地密钥库以签署和验证statement或交易。
- 一个额外的、在所有链外工作者之间共享的本地KV数据库。
- 一个安全的本地熵源，用于生成随机数。
- 访问节点的精确本地时间。
- 睡眠和恢复工作的能力。

请注意，链外工作者的结果不受常规交易验证的影响。因此，你应该确保offchain操作包括一个验证方法，以确定哪些信息进入链中。例如，你可以通过实现投票、平均或检查发送人签名的机制来验证链外交易。

链外工作者是在每个区块插入期间产生的。然而，在最初的区块链同步期间，它们不会被执行。

### 链下的存储

链外存储始终是Substrate节点的本地存储，不与任何其他区块链节点共享链上存储，也不受制于共识。你可以使用有读写权限的链下工作线程或通过链上逻辑使用链下索引来访问存储在链下存储的数据。

因为每个区块插入时都会产生一个链外工作线程，所以在任何时候都可以有多个链外工作线程在运行。与任何多线程编程环境一样，有一些实用工具可以在链外工作线程访问链外存储时对其进行互斥锁定，以确保数据一致性。

链外存储是链外工作线程之间相互通信的桥梁，也是链外和链上逻辑之间的通信。它也可以使用远程过程调用（RPC）进行读取，因此它符合存储无限增长的数据而不过度消耗链上存储的使用情况。

### 链下的索引

在区块链的背景下，存储最常关注的是链上状态。然而，链上状态是昂贵的，因为它必须被同意并填充到网络中的多个节点。因此，你不应该使用链上存储来存储历史或用户生成的数据--这些数据会随着时间的推移无限增长。

为了解决访问历史或用户生成的数据的需要，Substrate使用链外索引提供对链外存储的访问。链外索引允许runtime直接写入链外存储，而无需使用链外工作线程。你可以通过使用 `--enable-offchain-indexing` 命令行选项启动 Substrate 节点来启用这一功能以持久化数据。

与链外工作者不同，链下索引在每次处理一个区块时都会填充链下存储。通过在每个区块中填充数据，链下索引确保数据始终是一致的，并且对于启用索引的每个运行节点来说是完全相同的。也就是链下索引保证了链外存储的数据的一致性。


## Substrate的补充

这里可能主要是知乎或者其他地方看到的一些substrate的零散的知识片段。

### 项目文件命名的一些前缀缩写

- `sp-*` Substrate primitives
- `frame-*` FRAME
- `pallets-*`  pallets
- `/bin`  都是一些二进制和工具
- `/client/*` 和 `sc-`  Substrate client

### Runtime的简要补充

substrate的我们理解的普通的链的节点用FRAME框架构建runtime。FRAME允许runtime开发者在称为pallet的模块中声明自定义的逻辑，FRAME的核心是用宏写的，主要使创建pallet更容易，并且灵活地组合这些pallets，所以各种pallets组成了runtime。

- 每个runtime配置了多个pallet来包含到runtime中。每个pallet配置由一个代码块决定，该代码块以 `impl $PALLET_NAME::Config for Runtime` 开头
- 不同的pallet通过`construct_runtime!`宏来阻止这些pallets到一个runtime中。

### Pallet

Substrate项目的runtime是由许多FRAME pallet组成的

一个FRAME pallet由许多区块链原语组成:

-   Storage: 存储，FRAME 框架定义了一套丰富的强大的存储抽象，使得使用 Substrate 的高效键值数据库来管理区块链的不断变化的状态变得容易。
-   Dispatchable: 可调用方法，FRAME pallet 定义了特殊类型的函数，可以从运行时外部调用 (dispatched)，以更新其状态。
-   Event: 事件，Substrate 使用事件来通知用户在运行时中的重要变化。
-   Errors: 错误，当一个 dispatchable 函数失败时，它会返回一个错误。
-   Config: 配置，配置接口用于定义 FRAME pallet 所依赖的类型和参数。

以上的原语，不一定每个pallet都需要实现。可能是部分组合。


[(9 封私信 / 83 条消息) 金晓 - 知乎 (zhihu.com)](https://www.zhihu.com/people/jin-xiao-94-7/posts)

[使用 Rust 和 Substrate 构建自己的区块链平台 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/498774892)

