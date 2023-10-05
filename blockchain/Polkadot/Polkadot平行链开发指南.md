## 前言

这个指南主要是一份概览overview，开发波卡Parachain或parathread的。工具，步骤，测试和最后怎样在波卡上启动你的网络。

## 为什么要创建一个平行链

Parachains与中继链相连，并受到中继链的保护。它们受益于Pooled Security、深思熟虑的治理以及网络的异构分片方法的整体可扩展性。创建parachain可以被看作是创建一个L1的区块链，它有自己的逻辑，并在Polkadot生态系统中平行运行。

开发人员可以专注于创建最先进的链，利用Polkadot的下一代方法。parachain可能作为一些例子是:

- DeFi
- 数字钱包
- IoT应用
- 游戏
- Web3.0的基础设施

Polkadot旨在成为对区块链最牛逼的东西，Polkadot的异构多链方法的成功将在Web 3.0和去中心化系统的整体推进中发挥关键作用。因此，Polkadot的parachain模式在设计时相信，未来的互联网将有许多不同类型的区块链一起工作。

##  部署Parachain的好处是什么

如Polkadot白皮书所述，parachain模型试图缓解目前技术堆栈的五个关键的弱势:

- 可扩展性。在资源上花了多少成本，网络是否会受到瓶颈的影响？
- 隔离性。许多人不同人的需求是否可以在同一框架下得到考虑？
- 可开发性。系统工具、系统支持和整个系统的完整性是否可靠？
- 治理。网络能否保持灵活的发展，并随着时间的推移进行调整？决策能否具有足够的包容性、合法性和透明度，以便对一个去中心化的系统进行有效的领导？
- 适用性。该技术是否能解决自身的迫切需求？是否需要其他 "中间件 "来弥补与实际应用的差距？

### 共享安全(Shared security or Pooled security)

Parachains可以通过绑定DOT来租赁Polkadot网络的安全，以获得一个parachain名额。这意味着围绕你的项目建立一个社区并说服Validator参与你的网络安全的社会成本降低了。Polkadot有很强的安全性，希望从这种安全性中获益的Dapp会希望成为parachain来分享这种集合的安全性。

### 链上治理

区块链中的大多数治理系统使用的是链外治理机制。Polkadot的链上治理鼓励代币持有人的最大参与，并且是无摩擦和透明的。它还可以实现无分叉升级。

### 可扩展性

分片式多链网络方法允许本质上的并行计算（处理能力），可以并行处理几个交易。孤立的区块链往往面临着依次处理交易的网络约束，造成瓶颈。

### 互操作性

任何想要向已经连接到Polkadot的其他parachains实现Trustless的信息传递的Dapp或链，都希望成为parachain。主权链之间的互操作性涉及到某些限制和复杂的协议，以在广泛的链中实现。

有了Polkadot，如果你把你的应用程序构建为一个parachain，你将会得到这个功能的开箱即用。XCM格式允许任何parachain通过在它们之间传递消息来进行通信。此外，随着连接到其他链的桥梁（如比特币或以太坊的桥梁），Polkadot的parachains也将能够与这些链通信。

> 注意： 尽管成为parachain有那么多好处，但是开发者应该意识到成为parachain的挑战，以及建立一个以成为parachain为最终目标的区块链对他们的项目来说是否可行。也就是要评估复杂度和工作量。

在Polkadot上，你需要能够把你的区块链的最新区块头放到中继链上。作为一个parachain，你提交的区块由具有Wasm runtime的Validator进行验证，可以存储在中继链上。你还可以使用XCM格式与其他平行链进行通信：一个抽象的消息传递系统。消息传递在中继链上被跟踪--因此，你可以证明消息的传递，并促进trustless的互动。

由于你可以放置你的区块链的最新区块头，你可以为你的链实现确定性的最终确定。对区块链来说，达到最终确定的困难部分往往是共识，在parachain模型中，区块链可以将共识卸载给整个共享的波卡网络，并专注于区块生产。由于Validator拥有的Wasm运行时间是服务于所有parachain的，你的parachain与中继链上的所有人共享验证者池的安全性。

验证者池中的任何验证者都可以帮助验证你的区块链。

## 需要考虑的事情

### 平行经济学(Para-nomics)

#### 数字化联邦

Parachains可以被看作是自主代理；作为分散的数字化联邦的网络。parachain有自己的社区、规则、经济、治理、财政以及与外部链的关系。因此，parachain生态系统内的经济政策受制于该parachain生态系统的开发者和整体社区；不一定有一个parachain应该遵循的经济模式。

此外，成为parachain有一个机会成本要考虑。理想情况下，你可以通过参与parachain的选举过程来增加网络的价值，而这应该作为一个良好的投资回报。

#### 连接数字经济

Collator作为波卡网络维护者，维护一个平行链的全节点。他们的激励是由原生代币(DOT)支付的, 这些费用来自于:

- 收集的交易费用
- parathread的代币的赞助
    - 当一个parathread的出价低于原生代币的报酬时，它会自发会产生区块。

### 平行对象(Para-objects)

> 中继链可以承载任意的状态机，而不仅仅是区块链。
Polkadot网络将鼓励不同的para-objects之间的连接和互操作性。
这里，para-objects指的是网络上平行运行的对象，一般来说，是可并行的对象。

平行对象可能由以下组成:

- 系统级链（永久链）：租借槽 [Parachain Slots Auction · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-auction)，parathread池 [Parathreads · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-parathreads)
- 桥Hubs [Bridges · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-bridges)
- 嵌套式的中继链，Polkadot 2.0 [Polkadot Launch Phases · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-launch##polkadot2.0)

### 移植(Migration)

已经作为 "solochain "或在孤立环境中运作的项目(就是一条独立的L1区块链)可能对迁移到Polkadot作为Paraoject感兴趣。虽然parachain模式有其好处，但它可能不是某些项目的首选策略。

作为迁移到Polkadot上的路径，迁移到某个保留槽中的链上可能更可行。

例如，目前通过在最新的插槽拍卖中获得插槽的网络，可以选择在Kusama上部署智能合约。[Smart Contracts · Polkadot Wiki](https://wiki.polkadot.network/docs/build-smart-contracts)


## 实现一条平行链

这个文档可能一直在更新，所以最新请考虑 [polkadot/roadmap/implementers-guide at master · paritytech/polkadot (github.com)](https://github.com/paritytech/polkadot/tree/master/roadmap/implementers-guide)  而不是以下的文档

### 平行链开发工具包(Parachain dev Kit, PDK)

PDK是一套工具，可以让开发人员轻松地创建一个parachain。在实践中，PDK将由以下关键组件组成。

- 状态转换函数：你的应用程序从一个状态转移到另一个状态的方法。
- collator节点：Polkadot网络中的一种点对点节点，在parachain方面负有一定的责任。

#### 关键组件

状态转换函数（STF）可以是一个应用程序从一个状态到另一个状态的抽象方式。Polkadot对这个STF的唯一约束是，它必须容易验证--通常是通过我们所说的证人或证明。它必须如此，因为中继链Validator将需要检查它从Collator节点收到的每个状态是否正确，而不需要实际运行整个计算过程。这些证明的一些例子包括有效性证明（Proof-of-Validity）块或zk-SNARKs，这些证明的验证过程需要的计算资源比它们生成证明时的要少。STF的证明生成中的验证不对称性是允许Polkadot在保持高安全保障的同时进行扩展的整体见解之一。  
  
collator节点是该协议中网络维护者的一种类型。他们负责保持parachain的状态和从状态转换函数的迭代中返回的新状态的可用性。它们必须保持在线，以跟踪状态，也跟踪它将在自己和其他parachain之间路由的XCMP消息。collator节点负责将简洁的证明传递给中继链的Validator，并跟踪中继链的最新块。实质上，collator节点也充当了中继链的轻型客户端。


#### PDKs的存在是什么

目前，唯一的PDK是Parity Substrate[paritytech/substrate: Substrate: The platform for blockchain innovators (github.com)](https://github.com/paritytech/substrate)和Cumulus[paritytech/cumulus: Write Parachains on Substrate (github.com)](https://github.com/paritytech/cumulus)。Substrate是一个区块链框架，它提供了区块链的基本构件（如网络层、共识、Wasm解释器），同时提供了一种直观的方式来构建你的runtime。Substrate是为了简化创建新链的过程，但它并没有直接提供对Polkadot兼容性的支持。出于这个原因，Cumulus，这一个附加的库包含了所有的Polkadot兼容的胶水代码。

学习substrate的资料 [Home | Substrate_ Docs](https://docs.substrate.io/)

#### Cumulus

Cumulus是Substrate的一个扩展，它使任何Substrate构建的runtime都可以很容易地成为一个与Polkadot兼容的parachain。

Cumulus Consensus是Substrate的一个共识引擎，它遵循Polkadot中继链。它在内部运行一个Polkadot节点，并向客户端和同步算法发出指令，要求遵循哪条链，最终确定，并将其视为正确的。

关于Cumulus的更详细描述，请参见Cumulus概述。[cumulus/overview.md at master · paritytech/cumulus (github.com)](https://github.com/paritytech/cumulus/blob/master/docs/overview.md)


Cumulus仍在开发中，但我们的想法是，通过导入crates(Rust的概念)并添加一行代码，应该就可以简单地让一个substrate的链变成一个parachain了。

> Substrate和Cumulus从区块链格式的抽象中提供了一个PDK，但一个parachain甚至不需要是一个区块链。例如，一个parachain只需要满足上面列出的两个约束：状态转换函数(STF)和collator节点。

> 其他一切都由PDK的实现者决定。

Cumulus处理任何parachain需要实现的网络兼容开销，以连接到Polkadot。这包括:

- 跨链消息传递(XCMP)
- 开箱即用的Collator节点设置
- 中继链的一个嵌入式完整客户端
- 区块授权的兼容性

### 怎样建立你的平行链

在用Substrate创建了你的链的runtime逻辑后，你将能够把它编译成Wasm可执行文件。这个Wasm的code blocb将包含你的链的整个状态转换功能(STF)，也是你将项目部署到Polkadot作为parachain或parathread所需要的。  
  
Polkadot上的Validator将使用提交的Wasm代码来验证你的链或线程的状态转换，但这样做需要一些额外的基础设施。一个Validtor需要一些方法来保持最新的状态转换，因为Polkadot的节点不需要成为你开发的平行链的节点。  
  
这就是Collator节点发挥作用的地方。Collator是你的parachain的维护者，执行关键的行动，为你的链产生新的区块候选者，并将它们传递给Polkadot中级链的Validator，以纳入Polkadot中继链。  
  
Substrate内置了自己的网络层，但不幸的是，它只支持单体独立的链（即不连接到中继链的链）。然而，有一个Cumulus扩展，它包括一个Collator节点，并允许你的Substrate构建的逻辑与Polkadot兼容，作为一个parachain或parathread。  

### 未来的PDKs

这里主要是一些规划，可能会随时过时，请关注最新的消息。

W3F有兴趣支持的PDK的一个例子是一个roll-up的工具包[Roll_up / roll_back snark side chain ~17000 tps - zk-s[nt]arks - Ethereum Research (ethresear.ch)](https://ethresear.ch/t/roll-up-roll-back-snark-side-chain-17000-tps/3675)，它允许开发者创建基于zkSNARK的parachains。如果我们回顾一下rollup的写法，我们会发现系统使用了两个角色：更新状态的user和将状态更新聚合成单一链上更新的operator。我们应该可以直接看到如何将其转化为parachain的术语。类似rollup的parachain的状态转换函数(STF)将从用户的输入中更新状态（在实践中，很可能是Merkle树，这将很容易被验证）。operator将充当collator节点，它将聚合的状态并创建zk-SNARK证明，它将交给中继链的Validator进行验证。

## 测试一条平行链

### Rococo测试网

Rococo是一个测试网[paritytech/cumulus: Write Parachains on Substrate (github.com)](https://github.com/paritytech/cumulus#rococo-) ，它为了测试平行链而构建。

Rococo利用Cumulus和HRMP（Horizontal Relay-routed Message Passing）在parachains和Relay Chain之间发送传输和消息。每条信息都被发送到中继链，然后再从中继链发送到所需的parachain。

Rococo目前运行着四个测试系统的parachain(Statemint Tick, Trick, Track [Polkadot/Substrate Portal](https://polkadot.js.org/apps/?rpc=wss://track-rpc.polkadot.io#/explorer),)  以及几个外部开发的parachain。



### Rococo上目前有哪些平行链

[Polkadot/Substrate Portal](https://polkadot.js.org/apps/?rpc=wss%3A%2F%2Frococo-rpc.polkadot.io#/parachains)

### 获取ROC

ROC是Rococo的token代币，ROC可以在Matrix app上的Rococo Faucet频道[Element](https://app.element.io/#/room/#rococo-faucet:matrix.org)获得。要接收ROC代币，请使用以下命令。

```txt
!drip <your_rococo_addr>
```

### 构建和注册一个Rococo的ParaThread

Rococo的parachain都使用相同的runtime代码。它们之间唯一的区别是用于在中继链上注册的parachain ID.x。

你将需要运行一个collator节点。要做到这一点，你需要编译以下二进制文件。

```bash
cargo build --release --locked -p polkadot-collator
```

一旦可执行文件构建完成，你就可以为你的平行链启动一个对应的collator节点了:

```bash
./target/release/polkadot-collator --chain $CHAIN --validator
```

如果你对运行和启动你自己的parathread或parachain感兴趣。在这一过程中遇到困难或需要支持？加入Parachain技术矩阵聊天频道，在那里与其他建设者联系。[You're invited to talk on Matrix](https://matrix.to/#/#parachain-technical:matrix.parity.io)

### 怎样创建一个跨链转账

要在parachains之间发送转账，请在Polkadot-JS应用程序上导航到 "Accounts">"Transfer"。从这里，你需要选择你正在运行的parachain节点。接下来，输入你想发送至另一个parachain的金额。请确保选择你想发送金额的正确parachain。一旦你点击 "提交 "按钮，你应该看到一个绿色的通知，表明转账成功。

#### 向下转账

向下转账是指中继链上的一个账户向其在不同Parachain上的账户发送转账。这种类型的转账使用了存管(deposit)和铸币(mint)的模式，也就是说，当DOT离开中继链上的发送者的账户并被转入一个parachain上的账户时，parachain就会在parachain它自己上铸造(mint)相应数量的代币。

例如，我们可以从Alice在中继链上的账户发送代币到她在parachain 200上的账户。要做到这一点，我们需要前往 "Network">"parachains "标签，并点击 "Transfer to chain "按钮。

![[Pasted image 20220825110925.png]]

注意这里，我们可以选择发送资金给哪个parachain，指定要发送的金额，并为转账添加任何评论或备忘录。


####  向上转账

向上转移转账在从一个parachain到中继链上的一个账户。要进行这种转移，我们需要连接到网络上的一个parachain节点，并在 "Network">"Parachains "标签上。点击 "Transfer to Chain "按钮。

![[Pasted image 20220825111119.png]]

注意，图上那个切换开关应该设置为关闭，确保资金流向中继链，而不是另一个parachain。

#### 横向转账

也就是平行链到平行链的转账。

横向转账只有在至少有两个不同的已注册的parachain的情况下才能实现。在真正的XCMP中，横向转账将允许消息直接从一个parachain发送至另一个parachain。然而，这还没有实现，所以中继链暂时在帮助我们传递消息。横向转账是通过存管模式进行的，这意味着为了将代币从200号链转移到300号链，代币必须已经被200号链拥有并存放在300号链上。横向转账被称为HRMP，即横向中继-链式消息传递。

在我们从一个parachain向另一个parachain发送资金之前，我们必须确保该链在接收链上的账户里有一些资金（因为要接收的parachain要运行XCM指令，存在开销）。在这个例子中，Alice将从她在parachain 200上的账户发送一些资金到她在parachain 300上的账户。

我们可以从parachain 300的终端获得该发送方的parachain账户地址。

```txt
2020-08-26 14:46:34 Parachain Account: 5Ec4AhNv5ArwGxtngtW8qcVgzpCAu8nokvnh6vhtvvFkJtpq
```

从Alice在中继链上的账户，她可以向parachain 200的存钱罐（deposit）发送一些金额。

![[Pasted image 20220825111802.png]]

Alice现在能够从她在parachain 200的账户向她在parachain 300的账户汇款。

![[Pasted image 20220825111853.png]]



### 怎样连接到一个平行链

如果你想通过Polkadot-JS Apps[Polkadot/Substrate Portal](https://polkadot.js.org/apps/#/explorer)连接到一个parachain，你可以通过点击导航左上角的网络选择，选择任何parachain。

![[Pasted image 20220825112013.png]]

[Connect to a relay chain | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/parachains/connect-to-a-relay-chain/)



### 平行链的游乐场(playground)

你也可以利用Polkadot-JS应用程序提供的账户功能来测试整个Parachain的注册登录流程（例如，众筹、拍卖、注册）。

在Westend[Networks · Polkadot Wiki](https://wiki.polkadot.network/docs/maintain-networks###westend-test-network)上启动一个本地节点，运行:

```bash
polkadot --chain=westend-dev --alice
```

然后用Polkadot-JS Apps连接你的本地节点。

![[Pasted image 20220825112436.png]]


## 部署

基于Subtrate的链，这些链包括Polkadot和Kusama Relay链，也就是Polkadot和kusama的中继链都是基于substrate的，使用SS58编码作为其地址格式。本页[ss58-registry/ss58-registry.json at main · paritytech/ss58-registry (github.com)](https://github.com/paritytech/ss58-registry/blob/main/ss58-registry.json)作为典型的注册表，供团队查看哪条链对应于某个前缀，以及哪些前缀是可用的。

### 平行链

要将你的parachain纳入Polkadot网络，你将需要获得一个parachain的插槽(slot)。

parachain插槽将在公开拍卖中出售，其机制可在维基的parachain的拍卖页面中找到。[Parachain Slots Auction · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-auction)

### Parathread

Parathreads将不需要parachain插槽，所以你将不需要参与parachain那样的蜡烛拍卖机制。相反，你将能够把你的parathread代码注册到一个中继链上（需要点费用），并从那时起能够开始参与每个块的拍卖，以将你的状态转移到中继链上。

[Parathreads · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-parathreads)

## Westend与Rococo有什么不同

这两个都是polkadot的测试网。

Rococo是生态系统中所有Parachain的测试网，所有想尝试成为Parachain的团队，或在Polkadot上建立的团队，都应该连接到Rococo。


Polkadot拥有与传统区块链提供的不同的基础设施。建设者可以建立基础设施、pallets、Dapp和parachain，所以Rococo的目标是成为所有这些建设者来测试他们建设的地方。为此，它拥有Kusama和Polkadot上的所有功能（如HRMP和XCM），甚至还有一些实验性功能，如Beefy。

Westend是一个在生产网络上发布前进行最终测试的网络。它以前被用来在Kusama/Polkadot上注册为parachains或parathread前需要测试的网络，但随着Rococo的出现，它不再被使用。


## 资源

- Common Good Parachains [Common Good Parachains: An Introduction to Governance-Allocated Parachain Slots (polkadot.network)](https://polkadot.network/blog/common-good-parachains-an-introduction-to-governance-allocated-parachain-slots/)
- 平行链的启动 [The Launch of Parachains (polkadot.network)](https://polkadot.network/blog/the-launch-of-parachains/)
- Parathreads: 现收现付的parachain [Parathreads: Pay-as-you-go Parachains | by Joe Petrowski | Polkadot Network | Medium](https://medium.com/polkadot-network/parathreads-pay-as-you-go-parachains-7440d23dde06)
- 波卡的桥 [Polkadot Bridges — Connecting the Polkadot Ecosystem with External Networks | by Polkadot | Polkadot Network | Medium](https://medium.com/polkadot-network/polkadot-bridges-connecting-the-polkadot-ecosystem-with-external-networks-1118916392e3)
- 平行链的区块的数据流路径 [Polkadot Bridges — Connecting the Polkadot Ecosystem with External Networks | by Polkadot | Polkadot Network | Medium](https://medium.com/polkadot-network/polkadot-bridges-connecting-the-polkadot-ecosystem-with-external-networks-1118916392e3) [The Path of a Parachain Block - Crowdcast](https://www.crowdcast.io/e/polkadot-path-of-a-parachain-block?utm_source=profile&utm_medium=profile_web&utm_campaign=profile)
- 波卡平行链插槽 [Polkadot Parachain Slots](https://polkadot.network/blog/polkadot-parachain-slots/)
- 怎样成为波卡上的一条平行链 [Sub0.1: How to become a parachain on Polkadot - Shawn Tabrizi, Developer experience @ Parity - YouTube](https://www.youtube.com/watch?v=fYc1yolanoE)
- ‘’可信执行环境(TEE)与波卡的生态 [Trusted Execution Environments and the Polkadot Ecosystem](https://polkadot.network/blog/trusted-execution-environments-and-the-polkadot-ecosystem/)



