### Cosmos的核心概念

#### 一个区块链的应用架构

##### 什么是Tendermint

Tendermint [Tendermint](https://tendermint.com/) 创建于2014年，通过现成的网络和共识解决方案加速了不同区块链的发展，因此开发者不必为每个新案例重新创建这些功能。你可能已经在不知不觉中使用了Tendermint，因为其他区块链如Hyperledger Burrow  [Hyperledger Burrow](https://hyperledger.github.io/burrow/#/) 和Binance Chain https://www.binance.org/en/smartChain  使用了Tendermint。

Tendermint模块关注共识和网络，这是任何区块链的重要组成部分。这使得开发者可以专注于应用层面，而不至于沦落到低级别的区块链问题，如P2P发现、区块广播、共识和交易最终确定。如果没有Tendermint，开发者将被迫建立软件来解决这些问题，这将为他们的应用程序的开发增加额外的时间、复杂性和成本。

![[Pasted image 20221115144544.png]]

以应用为中心的Cosmos区块链节点由一个状态机组成，它由Cosmos SDK构建，而共识和网络层则由Tendermint Core [Tendermint Core | Tendermint](https://tendermint.com/core/)  处理。

> Tendermint Core是一个区块链应用平台，支持任何语言的状态机。与语言无关的Tendermint Core帮助开发者安全、稳定地复制确定性的有限状态机。[Overview | Tendermint Core](https://docs.tendermint.com/v0.34/tendermint-core/#) 
> 
> Tendermint BFT通过提供两个组件，即使有1/3的机器失败，也能维持。
> 
> - 一个区块链共识引擎。
> - 一个通用的应用程序接口。

##### Tendermint core和comos中的共识

Tendermint核心是一个高性能、一致、灵活和安全的共识模块，具有严格的分叉问责制。它依靠的是具有委托和实用拜占庭容错功能的权益证明（PoS）[[1807.04938] The latest gossip on BFT consensus (arxiv.org)](https://arxiv.org/abs/1807.04938) 。参与者对创建和确认区块的行为良好、可靠的节点表示支持。用户通过押注ATOM或相关链的原生代币来表示支持。stake有可能获得网络交易费用的份额，但也有可能在被支持的节点变得不可靠时减少回报甚至损失。  
  
网络参与者被激励将他们的ATOM stake给最有可能提供可靠服务的节点，并在这些条件改变时撤回他们的支持。预计Cosmos区块链会调整validator配置，即使在不利条件下也会继续。  
  
只有前150名的节点（按stake的ATOM排名）作为validators参与交易的最终确定过程。创建区块的特权是按照validators拥有的投票权的比例授予的。投票权的计算方法是一个validator及其代表所押的所有ATOM代币。例如，如果一个特定的validator的投票权是所有validators总投票权的15%，那么该validator可以期望在15%的时间内获得创建区块的特权。  
  
创建的区块会被广播给其他validators，他们应该及时和正确地做出反应。  
  
- 验证者确认候选区块。  
- 验证者如果不这样做，就会受到惩罚。  
- 验证者可以而且必须拒绝无效的区块。  
- 验证者通过返回他们的签名来接受该区块。  

当区块创建者收集到足够的签名时，区块就会被最终确定并广播给更广泛的网络。在这个过程中没有任何模糊之处：一个区块要么有必要的签名，要么没有。如果有，则没有足够的签名者来推翻该区块，因此该区块可以被理解为最终确定--没有任何区块链会被重组的过程。当涉及到交易的最终结果时，这提供了一个确定性的水平，而像工作证明（PoW）这样的概率系统是无法比拟的。

激进的区块时间是可行的，因为该过程旨在实现高性能，并基于具有良好网络连接的专用validators。这与PoW截然不同，PoW倾向于包容，必须适应延迟较大、可靠性较差的慢速节点。一个Cosmos区块链每秒可以处理成千上万的交易，确认需要7秒。

尽管验证被委托给所有网络节点的一个子集，但该过程避免了权力的集中。用户社区通过押注ATOM和参与承诺的回报和风险来选举validators，以提供可靠、负责的区块验证服务。

##### 链的可升级性

在任何已知的区块链中，实施方式的改变需要对每个节点上运行的节点软件进行升级。在自愿参与的无序过程中，这可能会导致硬分叉：在这种情况下，一个选区按照旧规则前进，而另一个选区采用新规则 （也就是substrate中的提到的无分叉升级，solo chain是用高度隔离规则  [[Substrate开发构建笔记#升级Runtime]] ）。虽然这种安排有积极的方面和支持者，但在对确定性有严格要求的环境中，它也有明显的缺点。例如，在关注权威注册机构和大额资产的环境中，交易最终性的不确定性（无论不确定性的程度如何）可能是不可接受的。

在Tendermint区块链中，交易在区块创建时被不可逆转地最终确定，而升级本身受区块创建和验证过程的制约。这没有留下任何不确定的空间：要么节点同意同时升级他们的协议，要么升级建议失败。

validators和委托者是对提案进行投票的一方，其权重与各自的利益成正比。如果一个委托人没有对提案进行投票，那么该委托人的投票就被视为其委托的validator的投票。这意味着，代表者在行动时应该有很高的要求，因为他们也会把默认的一票借给validator。

##### 应用区块链接口(ABCI)

Tendermint core [Tendermint Core | Tendermint](https://tendermint.com/core/) 将区块链的网络和共识层打包，并向应用层提供一个接口，即应用区块链接口（ABCI）。开发者可以专注于更高阶的关注点，并将P2P的发现、validator的选择、质押、升级和共识委托给Tendermint core。共识引擎在一个进程中运行并控制状态机，而应用程序在另一个进程中运行。该架构同样适用于私有或公共区块链。

Tendermint BFT引擎通过一个Socket协议连接到应用程序。ABCI为用其他语言编写的应用程序提供一个Socket。如果应用程序是用与Tendermint实现相同的语言编写的， 则不使用该socket。也就是说用golang写这个应用，就不需要socket了。

![[Pasted image 20221115160823.png]]

Tendermint BFT提供安全保障，包括以下内容。

- 只要至少一半的validators是诚实的，分叉就不会被创建。
- 严格的分叉创建问责制允许确定责任。
- 交易在区块创建后就被最终确定。

Tendermint BFT不关心交易的解释。这发生在应用层。Tendermint以匿名方式呈现已确认的、格式良好的交易和交易区块。Tendermint对任何交易的意义都不发表意见。

区块时间约为7秒，区块可能包含成千上万的交易。交易一旦出现在区块中，就被最终确定，不能被推翻。

> 这里有两个深入共识和tendermint的资料   [Consensus Systems with Ethan Buchman - Software Engineering Daily](https://softwareengineeringdaily.com/2018/03/26/consensus-systems-with-ethan-buchman/)  [What is Tendermint | Tendermint Core](https://docs.tendermint.com/v0.34/introduction/what-is-tendermint.html#consensus-overview) 

对于使用区块链的应用级关注，至少有两种广泛的方法。

- 创建一个特定于应用的区块链，其中可以做的所有事情都在协议中定义。
- 创建一个可编程的状态机，并将应用关注点推到更高的层次，例如由解释更高级语言的编译器创建的字节码。

类似以太坊的区块链属于第二类：在链上协议中只定义了状态机，它定义了合约创建、互动、执行的规则，其他的就很少了。

这种方法并非没有局限性。

- 几乎没有什么是普遍定义的：诸如代币等基本问题的标准是通过自愿参与而有机出现的。
- 合约可能而且确实包含了重复的代码，可能会也可能不会正确实现开发者的意图。
- 这种固有的灵活性使得推理什么是正确的，甚至什么是友好的成为挑战。
- 操作的复杂性有实际的限制，与其他环境中可能出现的情况相比，这些限制是非常低的。

这些限制使其特别难以进行分析或重组数据，而开发者被迫适应这些限制。

一个专门建造的或特定应用的区块链是不同的。没有必要提出一种 "图灵完备 "的语言或通用的、可编程的状态机，因为应用程序的关注点是由开发人员创建的协议解决的。

曾与基于以太坊虚拟机（EVM）的区块链合作的开发者将认识到关注点解决方式的转变。授权和访问控制，存储和状态的布局，以及应用程序的治理都不是作为状态机上的合约来实现的。相反，它们成为一个独特的区块链的属性，该区块链是为一个单一的目的而建立的。

> 这里我理解的就是，把一些应用上的业务，不通过合约的方式直接编写在链上。把业务做在链上，就叫应用链，而不是以太坊这样的通用链。 也就是跟substrate是大同小异的，但是App chain的概念，感觉是cosmos提出来的？而不是substrate

##### Cosmos SDK

开发人员使用Cosmos SDK创建应用层。Cosmos SDK提供了:

- Cosmos SDK为新的应用提供了一个开发的起点和一个预先建立的框架。
- Cosmos SDK提供了一套丰富的模块，解决常见的问题，如治理、代币、其他标准，以及通过区块链间通信协议（IBC）与其他区块链的互动。

使用Cosmos SDK创建特定应用的区块链，主要是一个选择、配置和整合解决好的模块的过程，也被称为 "组合模块"。这大大减少了所需的原始开发范围，因为开发主要集中在应用程序的真正新颖方面。

> 这里的模块估计跟substrate的pallet一样，都是通过组合模块来实现自定义的区块链

> 区块链间通信协议（IBC）是一个在区块链之间交换信息的通用框架。就目前而言，知道它的存在就足够了，它能使想要转移价值（代币转移）和交换信息的区块链之间进行无缝互动。它使运行在独立的特定应用区块链上的应用程序之间能够进行通信。

> 这里的应用之间的通信，是不是模块的通信，而不是链上的合约通信？

应用层、共识层和网络层都包含在自定义区块链节点中，构成了自定义区块链的基础。

Tendermint通过应用区块链接口（ABCI）将确认的交易传递给应用层。应用层必须实现ABCI，它是一个socket协议。Tendermint不关心交易的解释，应用层可以不关心传播、广播、确认、网络形成和其他Tendermint关注的低层次问题（除非它想检查这种属性）。

开发者可以自由地用任何支持socket的语言创建区块链，因为ABCI是一个socket协议，只要他们的应用程序实现了ABCI。ABCI定义了复制关注点和应用程序之间的边界，后者是一个状态机。

这本身就是一个相当大的进步，简化了独特区块链的创建。

> 关于更多的IBC信息，请看:
> [abci/specification.md at master · tendermint/abci (github.com)](https://github.com/tendermint/abci/blob/master/specification.md)
> [abci/types.proto at master · tendermint/abci (github.com)](https://github.com/tendermint/abci/blob/master/types/types.proto)
> [abci/application.go at master · tendermint/abci (github.com)](https://github.com/tendermint/abci/blob/master/types/application.go)
> [Tendermint Core Documentation | Tendermint Core](https://docs.tendermint.com/)

##### 状态机

区块链的核心是一个复制的状态机。状态机是一个计算机科学的概念，一个机器可以有多个状态，但每次只有一个状态。它遵循一个状态转换过程（或一组定义的过程），这是状态从旧的或初始状态（S）到新的状态（S'）的唯一变化方式。

![[Pasted image 20221115162338.png]]

区块链中的状态转换功能与交易是同义的。给定一个初始状态，一个确认的交易，以及一套解释该交易的规则，机器就会过渡到一个新的状态。解释的规则是在应用层定义的。

> 这里也跟substrate的本质是一样的，runtime就是STF，runtime与pallet提供一个类似ABCI的接口，只是这个接口不是基于socket这样的进程分离的模型

区块链是确定性的。对交易的唯一正确解释是新的状态，在前一张图片中显示为S-prime（S'）。

区块链是分布式的，交易分批到达，称为区块。机器状态在对区块中的每笔交易进行正确解释后存在。每笔交易都是在前面每笔交易产生的状态机的背景下执行的。所有交易执行完毕后的机器状态是一个有用的检查点，特别是对于历史状态。

初始化区块链的状态，其中 "尚未发生任何事情"，被称为创世状态（S）。区块链的当前状态（S'）总是可以通过将所有执行的交易应用于创世状态来实现。

![[Pasted image 20221115162552.png]]

开发人员可以使用Cosmos SDK创建状态机。这包括

- 存储组织：也被称为状态。
- 状态转换功能(STF)：这些功能决定了什么是允许的，以及对状态的调整是否是交易的结果。

在这种情况下，"共识 "建立了一个包含秩序良好的交易的典范（canonical）块集。所有节点都同意，典范集是所有最终完成的交易的唯一相关集。由于状态机的确定性，在任何给定的交易执行或任何区块高度，对规范交易集只有一个正确的解释。

这个状态机的定义对确认和传播交易的过程没有提及。Tendermint对于它所组织的区块的解释是不可知的。Tendermint的共识建立了有序的交易集。然后，各节点就应用程序的状态达成共识。

##### 额外的细节

交易和区块利用几个关键方法和消息类型。

###### CheckTx

许多可以被广播的交易不应该被广播。这方面的例子包括畸形的交易和类似垃圾邮件(spam-like)的人工制品。然而，Tendermint无法确定交易的解释，因为它是不可知的。为了解决这个问题，应用区块链接口包括一个`CheckTx`方法。Tendermint使用这个方法来询问应用层一个交易是否有效。应用程序实现这个功能。

###### DeliverTx

Tendermint调用`DeliverTx`方法，将区块信息传递给应用层进行解释和可能的状态机转换。

###### BeginBlock和EndBlock

`BeginBlock`和`EndBlock`消息通过ABCI发送，即使区块不包含任何交易。这提供了对基本连接的积极确认，并有助于识别没有操作的时间段。这些方法有助于执行应该一直运行的计划进程，因为它们调用了应用程序级别的方法，在那里开发人员定义了进程。

> 在每个区块的开始或完成时，谨慎地增加过多的计算量是明智的，因为区块大约以七秒的间隔到达。太多的工作可能会使你的区块链变慢。

任何使用Tendermint达成共识的应用程序都必须实现ABCI。你不必手动操作，因为Cosmos SDK提供了一个被称为`BaseApp`的模板来让你开始。

> 这里类似于substrate的node template

在下面的建议练习中，你将用Cosmos SDK创建一个最小的分布式状态机，并看到逐步实现概念的代码样本。你的状态机将依靠Tendermint来达成共识。

> 创造一个跳棋区块链
> 
   为什么要开发一个跳棋游戏？
>  
>  这个设计项目将随着你对Cosmos SDK的进一步了解而分阶段发展。随着你在各部分的进展和探索模板的作用，你将更好地理解和体验Cosmos SDK如何通过处理模板来提高你的生产力。
>  
>  ## 设置
>  
>  你要设计一个区块链，让人们互相玩跳棋。跳棋有很多版本，所以为这个练习选择这些简单的规则  [Kid's Games: Rules of Checkers (ducksters.com)](https://www.ducksters.com/games/checkers_rules.php) 。这个练习的目的是了解ABCI和学习更多关于使用Cosmos SDK的工作，而不是迷失在棋盘状态或游戏规则的正确实现中。
>  
>  使用并改编这个现成的实现 [checkers-go/checkers.go at a09daeb1548dd4cc0145d87c8da3ed2ea33a62e3 · batkinson/checkers-go (github.com)](https://github.com/batkinson/checkers-go/blob/a09daeb/checkers/checkers.go) ，包括棋盘是8x8并在黑色单元格上下棋的额外规则。随着你的进展，代码很可能需要改编。不要担心实现一个可销售的图形用户界面，这本身就是一个独立的设计项目。你必须首先创建一个基础，以确保图形用户界面的实现。
>  
>  除了执行游戏规则之外，你还需要做很多事情，所以要尽可能地简化。检查这些ABCI规格 [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md) ，看看应用程序需要什么来遵守ABCI。试着确定你会用哪些基本资源来制作第一个不完美的跳棋游戏区块链。
> 
    ## "制造 "状态机
   >
   >你想拥有一个最小的可行的ABCI状态机。Tendermint并不关心提议的交易是否有效，也不关心每个交易后的状态如何变化。它将此委托给状态机，状态机将事务解释为游戏动作和状态。
   >
   >以下是应用程序必须采取行动的重要时刻。
   >
   >### 启动应用程序
   >
   >这个状态机是一个应用程序，它必须在接收来自Tendermint的任何请求之前启动。它必须将代表可接受的一般动作的静态元素加载到内存中。
   >
   >这段代码已经存在，并在模块加载时自动运行。
   
```go
func init()
```

> ### 查询游戏状态
> 
> 为了简化，开始时只有一个游戏和一个棋盘。通过使用这个函数，你可以立即决定如何对棋盘进行序列化，而不需要做任何努力。

```go
func (game *Game) String() string
```

>  将棋盘存储在/store/board，当通过Query [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#query-1) 命令请求时，在响应的Value中返回它，path="/store/board"。如果你需要把棋盘的状态从序列化的形式中重新实体化，可以调用。

```go
func Parse(s string) (*Game, error)
```

> 	String()函数并不保存`.Turn`字段。你必须自己存储轮到谁来玩。在/store/turn选择一个字符串作为玩家的颜色。
> 
> 你的应用程序需要自己的数据库来存储状态。应用程序必须在某个Merkle根值处存储一个状态，以便以后能够调用过去的状态。这是你在创建应用程序时必须解决的另一个实现细节。
> 
> InitChain - 初始链状态
> 
> 这是 [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#initchain) 你的唯一游戏被初始化的地方。Tendermint向你的应用程序发送`app_state_bytes: bytes`，包含区块链的初始（创世）状态。你已经知道它代表一个游戏的样子了
> 
> 你的应用程序。
> 
> - 接收初始状态。
> - 将其保存在其数据库中，同时需要黑方 [checkers-go/checkers.go at a09daeb1548dd4cc0145d87c8da3ed2ea33a62e3 · batkinson/checkers-go (github.com)](https://github.com/batkinson/checkers-go/blob/a09daeb/checkers/checkers.go#L124) 来进行下一步游戏。
> - 返回对应于创世状态的Merkle根hash在`app_hash：bytes`中。
> 
> 你的应用程序还必须处理Tendermint发送的validators列表。Cosmos SDK的BaseApp将做到这一点。
> 
> ### 一个序列化的交易
> 
> 接下来你必须决定如何表示一个棋子。在现成的实现中，一个位置Pos由两个int [checkers-go/checkers.go at a09daeb1548dd4cc0145d87c8da3ed2ea33a62e3 · batkinson/checkers-go (github.com)](https://github.com/batkinson/checkers-go/blob/a09daeb/checkers/checkers.go#L42-L45) 表示，而一个移动由两个Pos  表示  [checkers-go/checkers.go at a09daeb1548dd4cc0145d87c8da3ed2ea33a62e3 · batkinson/checkers-go (github.com)](https://github.com/batkinson/checkers-go/blob/a09daeb/checkers/checkers.go#L168) 。你可以决定用四个int来表示一个序列化的移动：前两个表示原始位置（src），后两个表示目标位置（dst）。
> 
> ### BeginBlock 一个将要被创建的新区块
>
>BeginBlock  [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#beginblock) 指示应用程序在正确位置加载其状态。在 `header: Headner`（根据其详细定义  [spec/data_structures.md at master · tendermint/spec (github.com)](https://github.com/tendermint/spec/blob/master/spec/core/data_structures.md#header) ）你会发现。  
>
>  `AppHash：[] byte`：应用程序在执行和提交前一个块后返回的一个任意字节数组。这作为验证来自ABCI应用程序的任何Merkle证明的基础，并代表实际应用程序的状态，而不是区块链本身的状态。第一个区块的 `block.Header.AppHash`是由`ResponseInitChain.app_hash`给出。  
>
>在指示应用程序从其数据库加载应用程序的正确状态之前，这个实现细节被跳过了，其中包括正确的/store/board。重要的是，应用程序能够在任何时间点加载一个已知的状态。可能有一次崩溃或某种恢复，使Tendermint和应用程序失去了同步性。  
>
>应用程序应该在它最后到达的状态下工作，以防标题省略了`AppHash`（这不应该发生）。  
>
>应用程序现在已经准备好响应即将到来的`CheckTx`和`DeliverTx`，并加载状态。 
>
>### CheckTx 一个新交易出现在交易池中
>
>Tendermint询问  [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#checktx-1) 你的应用程序，该交易是否值得保留。为了最大限度地简化，你只关心交易中是否有一个有效的动作。你要检查在序列化的信息中是否有四个int，以确定这一点。你也可以检查int本身是否在棋盘的边界内，例如在0到7之间。
>
>根据应用程序的规则，最好不要检查这步棋是否有效。

```go
func (game *Game) ValidMove(src, dst Pos) bool
```

> 检查一个动作对棋盘来说是否有效，需要了解交易被纳入区块时的棋盘状态。棋盘只更新到交易被发送的那一刻。你可能会遇到这样的情况：两个交易被发送，一个在另一个之后，而且都是有效的。如果你用第一个未确认的棋子之前的棋盘状态来测试第二个交易中的棋子，就会发现第二个棋子是无效的。应该避免在`CheckTx`时间测试棋盘上的棋步。
> 
> 在`CheckTx`中检查交易有效性的可能性：如果交易是畸形的，包含无效的输入，等等，因此不可能被接受，则拒绝该交易；但要避免根据取决于上下文的关注点来确认它是否会成功。
> 
> ### DeliverTx 一个交易被加入并且需要被处理
> 
  当预选交易被交付时 [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#delivertx-1) ，它必须被应用到最新的棋盘状态。你调用

```go
func (game *Game) Move(src, dst Pos) (captured Pos, err error)
```

> 如有必要，解决任何错误。
> 
> 你需要看看通过ABCI将交易送回是否有意义。如果交易成功了，你就在内存中保留新的棋盘状态，为下一个交付的交易做好准备。在这一点上，你并没有保存到存储器中。
> 
> 你也可以选择通过事件定义哪些信息应该被索引：在响应中重复事件。返回的值是为了返回以其他方式收集的可能很繁琐的信息。这允许跨区块快速搜索相关的值，如果被索引的话。关于ABCI的事件，更多请查看[tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#events) 
> 
> 如果你来自以太坊世界，你会认识到这些是带有索引字段的Solidity事件，是交易接收日志中的主题。但与以太坊不同，事件不是共识（区块）的一部分，而是纯粹在节点上处理。
> 
> 通知Tendermint关于`GasUsed（int64）` 是明智的。每次移动的成本是一样的，所以你可以返回`1`。
> 
> ### EndBlock  区块正在被完成
> 
> 暂时忽略validators的问题，这是用来   [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#endblock) 让Tendermint知道应该发射什么事件（类似于DeliverTx的处理方式）。你只有跳棋的动作，所以不是很清楚以后会有什么有趣的事情。
> 
> 假设你想统计该区块中发生的事情。你返回这个聚合事件。

```json
[
    { key: "name", value: "aggregateAction", index: true },
    { key: "black-captured-count", value: uint32, index: false },
    { key: "black-king-captured-count", value: uint32, index: false },
    { key: "red-captured-count", value: uint32, index: false },
    { key: "red-king-captured-count", value: uint32, index: false }
]
```

>  Commit - 你在这里的工作已经完成
>  
>  该区块现在已被确认 [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#commit) 。应用程序需要将其状态保存到存储，即保存到其数据库。包括`/store/board`在内的状态是由其Merkle根哈希值唯一识别的。根据ABCI，这个哈希值必须以`[]byte`的形式返回。为了使共识能够出现，哈希值需要在文档中提到的相同的`BeginBlock`、相同的`DeliverTx`方法的顺序和相同的`EndBlock`的序列之后是确定的  [tendermint/abci.md at master · tendermint/tendermint (github.com)](https://github.com/tendermint/tendermint/blob/master/spec/abci/abci.md#determinism) 。
>  
>  应用程序也可以在其数据库中保留一个关于哪个状态是最新的指针，这样它就可以在返回和保存之后从其内存中清除棋盘。下一个`BeginBlock`将通知应用程序要加载哪个状态。如果下一个`BeginBlock`没有提到`AppHash`或者提到以前计算过的相同的`AppHash`，应用程序应该在内存中保留该状态，以便迅速建立在它上面。
>  
>  ### 如果你喜欢极端的序列化呢？
>  
>  这个数据严格来说不需要是Merkle根哈希值。它完全可以是任何字节，只要。
>  
>  结果是确定的和抗碰撞的。
>  应用程序可以从中恢复出状态。
>  如果你把你的单板以不同的方式进行序列化，你可以这样返回单板的状态。
>  
>  你有64个单元，其中只有32个被使用。每个格子要么什么都没有，要么是一个黑卒，一个黑王，一个红卒，或者一个红王。有五种可能性，可以很容易地装入一个字节，所以你需要32个字节来描述棋盘。也许你可以用第一个比特来表示该轮到谁下棋，因为在数到5的时候，字节的第一比特是不会被使用的。
>  
>  你现在有了。
>  
>  一个确定性的区块链状态。
>  防碰撞，因为相同的值表示一个相同的状态。
>  没有外部数据库需要处理。
>  完整的状态总是存储在区块头中。

#### 账户

一个账户是一对钥匙。

- PubKey：一个公钥
- PrivKey: 一个私人钥匙

公钥是一个用户或实体的唯一标识符，可以安全地披露。私钥是用户需要保密管理的敏感信息。私钥被用来签署信息，以向他人证明信息是由某人使用与特定公钥对应的私钥签署的。这是在不透露私钥本身的情况下完成的。

##### 公钥密码学

现代密码系统利用计算机能力使某些数学函数的威力得以发挥。公钥密码学也被称为非对称密码学，是一种采用成对密钥的密码系统。每一对都由一个公钥和一个私钥组成。只要私钥不被披露，系统的安全性就不会受到威胁。与对称钥匙算法相比，非对称算法不要求各方使用安全通道来交换加密和解密的钥匙。

非对称密码学有两个主要应用。

> 认证，公钥作为私钥对的验证工具。
> 
> 加密，只有私钥可以解密用公钥加密的信息。

这节主要关注公钥密码学的认证部分。

> 公钥密码学保证了保密性、真实性和不可抵赖性。应用的例子包括S/MIME [S/MIME - Wikipedia](https://en.wikipedia.org/wiki/S/MIME) 和GPG [GNU Privacy Guard - Wikipedia](https://en.wikipedia.org/wiki/GNU_Privacy_Guard) ，并且它是一些互联网标准的基础，如SSL [What is SSL? - SSL.com](https://www.ssl.com/faqs/faq-what-is-ssl/) 和TLS https://en.wikipedia.org/wiki/Transport_Layer_Security 。

非对称密码学由于其计算的复杂性，通常适用于小数据块。对称和非对称密码学可以在一个混合系统中结合起来。例如，可以采用非对称加密法来传输对称加密，然后作为信息的加密密钥。混合系统的一个例子是PGP [Pretty Good Privacy - Wikipedia](https://en.wikipedia.org/wiki/Pretty_Good_Privacy) 。

密钥的长度是至关重要的。非对称加密密钥通常非常长。人们可以记住一个一般的原则：密钥越长，破解密码就越困难。破解非对称密钥只能通过暴力攻击来完成，在这种情况下，攻击者需要尝试每一个可能的密钥来找到一个匹配。

##### 公钥和私钥

非对称钥匙总是成对出现，为其所有者提供各种能力。这些能力是基于密码学数学的。公钥是为了分发给相关的人，而私钥则是为了保持秘密。这类似于分享你房子的地址，但对你前门的钥匙保密。

###### 签名和验签

让我们看一下用公钥和私钥进行签名和验证的例子。

Alice和Bob正在进行通信。Alice想确定Bob的公开声明确实是来自Bob，所以。

- Bob给了Alice他的公钥。
- Bob用他的私钥签署他的公告。
- Bob向Alice发送他的公告及其签名。
- Alice用Bob的公钥验证了签名。

Alice可以通过检查签名是否是用与Bob的公钥（已经知道代表Bob）相对应的私钥来验证公告的来源。

> 私钥被用来证明信息来自于由其公钥知道的账户所有者：签名证明信息是由知道对应于特定公钥的私钥的人签署的。这是区块链中用户认证的基础，也是私钥被严格保护的秘密的原因。

> 怎样用分层确定性钱包来管理多个区块链上的多个密钥对？
> 
> ### 分层确定钱包
> 
> 区块链通常维护用户账户的分类账，并依靠公钥加密技术进行用户认证。对一个人的公钥和私钥的了解是执行交易的要求。被称为钱包的客户端软件应用程序提供了生成新密钥对和存储它们的方法，以及基本服务，如创建交易、签署消息、与应用程序互动以及与区块链通信。
> 
> 尽管在一个钱包中生成和存储多个密钥对在技术上是可行的，但对用户来说，密钥管理很快就变得繁琐和容易出错。鉴于所有的钥匙将只存在于一个地方，用户将需要设计出在不利情况下恢复他们的钥匙的方法，如计算机的丢失或破坏。账户越多，需要备份的钥匙就越多。
> 
> ### 我需要很多的地址吗？
> 
> 使用多个地址可以帮助你改善隐私。你可能是一个单独的个人或实体，但你可能想用不同的别名与其他人进行交易。此外，你可能会与cosmos生态系统中的多个区块链互动。方便的是，你在不同区块链上不可避免的不同地址都可以源于一个种子。
> 
> 一个分层确定的钱包使用一个种子短语来生成许多密钥对，以减少这种复杂性。只有种子短语需要被备份。
> 
> ### 密码学标准
> 
> Cosmos SDK使用BIP32  [BIP 0032 - Bitcoin Wiki](https://en.bitcoin.it/wiki/BIP_0032) ，它允许用户从初始秘密和包含一些输入数据（如区块链标识符和账户索引）的衍生路径中生成一组账户。自BIP39以来，这个初始秘密大多是用12或24个单词生成的，被称为助记符，取自标准化的字典。密钥对总是可以从助记符和推导路径中以数学方式再现，这解释了钱包的确定性。
> 
> 要看BIP32的运行情况，请访问https://www.bip32.net/  [BIP32/BIP39/BIP44 - Mnemonic Code](https://www.bip32.net/) 。
> 
> 点击`show entroy`的细节，在`Entropy`字段中输入随机数据。这揭示了初始种子生成过程的一个重要方面：随机性的来源是必不可少的。当你提供`Entropy`的时候，BIP39助记符字段开始弹出单词。进一步向下滚动，选择`BIP32 Derivation Path` 。在Derived Addresses下，你会看到公钥和私钥对。
> 
> 像大多数区块链实现一样，Cosmos从公钥中得出地址。
> 
> ![[Pasted image 20221116100610.png]]
> 
> 当使用BIP39或其变体之一时，用户只需要以安全和保密的方式存储他们的BIP39助记符。所有的密钥对都可以从记忆法中重建，因为它是确定性的。从一个记忆体中可以生成的密钥对的数量没有实际上限。从BIP44  [bips/bip-0044.mediawiki at master · bitcoin/bips (github.com)](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki) Derivation Path中获取的输入被用来为每个区块链使用单一记忆法生成一个密钥对。因此被称为 "分层确定性"来描述这种密钥生成方法。
> 
> ### 你可以在不同的地址被追踪吗？
> 
> 分层确定性的钱包还能保护隐私，因为下一个公钥或地址不能从以前的公钥或地址中推导出来。从一个记忆体发出的两个地址或从两个不同的记忆体创建的两个地址是无法区分的。

#### 钥匙环，地址和地址类型

在Cosmos SDK中，钥匙被存储和管理在一个叫做钥匙环的对象中。

认证是作为签名验证来实现的。

- 用户产生交易，签署交易，并将签署的交易发送到区块链上。
- 交易的格式是将公钥作为消息的一部分。签名是通过确认签名的公钥与发件人相关的公钥匹配来验证的。如果消息是由声称的发件人以外的任何人签署的，则签名无效，交易被拒绝。

如果上述内容不清楚，请考虑下面的伪信息。

```json
"Message": {
  "Payload": {
    "Sender": "0x1234",
    "Data": "Hello World"
  },
  "Signature": "0xabcd"
}
```

将`Payload`和`Signature`传入签名验证函数，会返回一个sender。 derived sender必须与Payload本身的sender相匹配。这证实了该payload只能来自于知道对应于sender的私钥的人，"0x1234".

##### 签名方案

之前描述的公钥签名过程有不止一种实现方式。Cosmos SDK支持以下数字密钥方案来创建数字签名。

不同的数字密钥方案在不同的SDK包中实现。

- secp256k1 [SEC 2, ver. 2.0 (secg.org)](https://www.secg.org/sec2-v2.pdf)  在SDK的crypto/keys/secp256k1包中实现：是最常用的，也是用于比特币的。
- secp256r1 [SEC 2, ver. 2.0 (secg.org)](https://www.secg.org/sec2-v2.pdf) , 在SDK的crypto/keys/secp256r1包中实现。
- tm-ed25519  https://ed25519.cr.yp.to/ed25519-20110926.pdf , 在SDK的crypto/keys/ed25519包中实现：只支持共识验证。

##### 账户

BaseAccount  [cosmos-sdk/auth.pb.go at bf11b1bf1fa0c52fb2cd51e4f4ab0c90a4dd38a0 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/bf11b1bf1fa0c52fb2cd51e4f4ab0c90a4dd38a0/x/auth/types/auth.pb.go#L31-L36) 对象提供了存储认证信息的基本账户实现。

##### 公钥

公钥一般不用于引用账户。公钥确实存在，它们可以通过cryptoTypes.PubKey [cosmos-sdk/types.go at 9fd866e3820b3510010ae172b682d71594cd8c14 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/9fd866e3820b3510010ae172b682d71594cd8c14/crypto/types/types.go#L9) 接口访问。这有利于开发人员的操作，如签署链外信息或使用公钥进行其他out of band操作。

##### 地址

地址是公共信息，通常用于引用一个账户。地址是使用ADR-28 [cosmos-sdk/adr-028-public-key-addresses.md at main · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/main/docs/architecture/adr-028-public-key-addresses.md) 从公钥中导出的。三种类型的地址在使用账户时指定一个上下文。

- AccAddress  [cosmos-sdk/address.go at 1dba6735739e9b4556267339f0b67eaec9c609ef · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/1dba6735739e9b4556267339f0b67eaec9c609ef/types/address.go#L129) 识别用户，是信息的sender。
- ValAddress  [cosmos-sdk/address.go at 23e864bc987e61af84763d9a3e531707f9dfbc84 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/23e864bc987e61af84763d9a3e531707f9dfbc84/types/address.go#L298) 识别validator operators。
- ConsAddress  [cosmos-sdk/address.go at 23e864bc987e61af84763d9a3e531707f9dfbc84 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/23e864bc987e61af84763d9a3e531707f9dfbc84/types/address.go#L448) 识别参与共识的validator nodes。validator nodes是使用ed25519 [Safe curves for elliptic cryptography (cryptosys.net)](https://www.cryptosys.net/pki/manpki/pki_eccsafecurves.html) 曲线得出的。

##### 钥匙环

钥匙环对象存储和管理多个账户。Keyring对象实现了Cosmos SDK中的Keyring [cosmos-sdk/keyring.go at bf11b1bf1fa0c52fb2cd51e4f4ab0c90a4dd38a0 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/bf11b1bf1fa0c52fb2cd51e4f4ab0c90a4dd38a0/crypto/keyring/keyring.go#L55) 接口。

#### 交易

交易是由终端用户创建的对象，用于触发应用程序中的状态变化。它们由定义上下文的元数据和一个或多个`sdk.Msg` [cosmos-sdk/tx_msg.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx_msg.go#L11-L22) 组成，通过模块的Protobuf消息服务触发模块内的状态变化。

##### 从终端用户的角度来看交易的处理

- Decide

决定放入到交易中的信息。这通常是在钱包或应用程序和用户界面的协助下完成的。

- Generate

使用Cosmos SDK的`TxBuilder`生成交易。`TxBuilder`是生成交易的首选方式。

- Sign

签署交易。交易必须在validator上将其纳入区块之前被签署。

- Broadcast

使用其中一个可用的接口广播已签名的交易。

Decide和Sign是用户的主要互动。Generate和Broadcast则由用户界面和其他自动化装置来处理。

##### 消息

> 交易消息不能与ABCI消息相混淆，后者定义了Tendermint和应用层之间的交互。

消息或`sdk.Msg`  [cosmos-sdk/tx_msg.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx_msg.go#L11-L22) 是特定于模块的对象，在它们所属的模块范围内触发状态转换。模块开发者通过向Protobuf `Msg` 服务添加方法和定义`MsgServe`r来定义模块消息。每个`sdk.Msg`都与每个模块的`tx.proto`文件中定义的一个Protobuf `Msg`服务RPC有关。Cosmos SDK应用路由器会自动将每个`sdk.Msg`映射到相应的RPC服务，并将其路由到适当的方法。Protobuf为每个模块的Msg服务生成了一个`MsgServer`接口，模块开发者实现这个接口。

这种设计让模块开发者承担了更多的责任。它允许应用程序开发人员重复使用通用功能，而不必重复实现状态转换逻辑。虽然消息包含状态转换逻辑的信息，但一个交易的其他元数据和相关信息都存储在`TxBuilder`和`Context`中。

##### 签名交易

交易中的每条消息都必须由其`GetSigners`  [cosmos-sdk/tx_msg.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx_msg.go#L21) 指定的地址来签署。Cosmos SDK目前允许以两种不同的方式签署交易。  
  
- `SIGN_MODE_DIRECT`  [cosmos-sdk/signing.pb.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx/signing/signing.pb.go#L36) （首选）：Tx接口最常用的实现是Protobuf Tx  [cosmos-sdk/tx.pb.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx/tx.pb.go#L32-L42) 消息，它被用于`SIGN_MODE_DIRECT`。一旦被所有签名者签名，`BodyBytes`、`AuthInfoBytes`和`Signatures` [cosmos-sdk/tx.pb.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx/tx.pb.go#L113) 就会被聚集到`TxRaw`  [cosmos-sdk/tx.pb.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx/tx.pb.go#L103-L114) ，其序列化的字节  [cosmos-sdk/tx.pb.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx/tx.pb.go#L125-L136) 会在网络上广播。  
- `SIGN_MODE_LEGACY_AMINO_JSON`  [cosmos-sdk/signing.pb.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/types/tx/signing/signing.pb.go#L43) ：Tx接口的传统实现是`x/auth`的`StdTx`  [cosmos-sdk/stdtx.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/x/auth/legacy/legacytx/stdtx.go#L77-L83) 结构。所有签名者签署的文件是`StdSignDoc`[cosmos-sdk/stdsign.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/x/auth/legacy/legacytx/stdsign.go#L42-L50) ，它使用Amino JSON编码成字节 [cosmos-sdk/stdsign.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/x/auth/legacy/legacytx/stdsign.go#L53-L78) 。一旦所有签名都被收集到`StdTx`中，`StdTx`将使用Amino JSON进行序列化，这些字节将在网络上广播。这种方法正在被废弃。  

##### 生成交易

`TxBuilder`接口包含与交易的生成密切相关的元数据。最终用户可以为要生成的交易自由设置这些参数。  
  
- Msgs [cosmos-sdk/tx_config.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/client/tx_config.go#L39) ：交易中包含的消息数组。  
- GasLimit [cosmos-sdk/tx_config.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/client/tx_config.go#L43) ：用户选择的一个选项，用于计算他们愿意花费的gas量。  
- Memo [cosmos-sdk/tx_config.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/client/tx_config.go#L41) ：与交易一起发送的说明或评论。  
- FeeAmount [cosmos-sdk/tx_config.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/client/tx_config.go#L42) ：用户愿意支付的最大费用金额。  
- TimeoutHeight [cosmos-sdk/tx_config.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/client/tx_config.go#L44) : 块的高度，直到交易有效。  
- Signatures https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/client/tx_config.go#L40 ：交易中所有签名者的签名数组。  

由于目前有两种签署交易的模式，所以也有两种`TxBuilder`的实现。有一个`SIGN_MODE_DIRECT`的包装器和`SIGN_MODE_LEGACY_AMINO_JSON`的`StdTxBuilder` [cosmos-sdk/stdtx_builder.go at 8fc9f76329dd2433d9b258a867500de419522619 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/8fc9f76329dd2433d9b258a867500de419522619/x/auth/migrations/legacytx/stdtx_builder.go#L18-L21) 。这两种可能性通常应该被隐藏起来，因为终端用户应该更喜欢总体的`TxConfig` [cosmos-sdk/tx_config.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/client/tx_config.go#L24-L30) 接口。`TxConfig`是一个应用范围内的[cosmos-sdk/context.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/client/context.go#L50)配置，用于管理可从上下文访问的交易。最重要的是，它持有关于是否用`SIGN_MODE_DIRECT`或`SIGN_MODE_LEGACY_AMINO_JSON`签署每个交易的信息。  
  
通过调用`txBuilder := txConfig.NewTxBuilder()`，一个新的`TxBuilder`将以适当的签名模式被实例化。一旦`TxBuilder`被正确地填充了前面描述的字段的设置器，`TxConfig`将使用`SIGN_MODE_DIRECT或SIGN_MODE_LEGACY_AMINO_JSON`对字节进行正确编码。  
  
这是一个关于如何使用`TxEncoder()`方法生成和编码交易的伪代码片段。  

```go
txBuilder := txConfig.NewTxBuilder()
txBuilder.SetMsgs(...) // and other setters on txBuilder
```


##### 广播交易

一旦交易字节被生成和签署，有三种主要方式来广播交易。

- 使用命令行界面（CLI）。
- 使用gRPC。
- 使用REST端点。

应用程序开发人员通过创建通常在应用程序的`./cmd`文件夹、gRPC和/或REST接口中找到的命令行接口来创建应用程序的入口。这些接口允许用户与应用程序进行交互。

###### CLI

对于命令行界面（CLI）模块，开发者创建了子命令，作为子命令添加到应用程序顶级交易命令`TxCmd` [cosmos-sdk/tx.go at v0.45.4 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.45.4/x/bank/client/cli/tx.go#L29-L60) 。

CLI命令将交易处理的所有步骤捆绑在一个简单的命令中。

- 创建消息。
- 生成交易。
- 签名。
- 广播。

###### gRPC

gRPC的主要用途是在模块查询服务的背景下。Cosmos SDK还公开了其他与模块无关的gRPC服务。其中之一是`Tx`服务，它暴露了一些实用的功能，如模拟交易或查询交易，以及一个广播交易的方法 [cosmos-sdk/03-txs.md at main · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/main/docs/docs/run-node/03-txs.md#broadcasting-a-transaction-1) 。

###### REST

每个gRPC方法都有其相应的REST端点，使用gRPC-gateway生成。与其使用gRPC，你也可以使用HTTP在`POST /cosmos/tx/v1beta1/txs`端点上广播相同的交易。

##### Tendermint RPC

前面介绍的三种方法是对Tendermint RPC `/broadcast_tx_{async,sync,commit}` 端点的高级抽象。如果你愿意，你可以使用Tendermint RPC端点 [RPC | Tendermint Core](https://docs.tendermint.com/v0.34/tendermint-core/rpc.html) 来直接通过Tendermint广播交易。

> Tendermint支持以下RPC协议。
> 
> 通过HTTP的URI。
> 通过HTTP的JSONRPC。
> 通过WebSockets的JSONRPC。
> 
> 关于用Tendermint RPC广播的更多信息，请看Tendermint RPC交易广播API的文档 [RPC | Tendermint Core](https://docs.tendermint.com/v0.34/tendermint-core/rpc.html) 

#### 消息

这里的消息跟账户章节和交易章节提到的消息这个概念是一样的。

消息是Cosmos SDK中由模块处理的两个主要对象之一。另一个由模块处理的主要对象是查询。消息通知状态并有可能改变它，而查询则检查模块的状态并总是只读的。

在Cosmos SDK中，一个交易包含一个或多个消息。在交易被共识层纳入一个区块后，模块会处理这些消息。

> 对一个消息签名
> 
> 请记住上一节关于交易的内容，交易必须在validator将其纳入区块之前进行签名。交易中的每个消息都必须由`GetSigners`指定的地址签名。
> 
> Cosmos SDK目前允许用`SIGN_MODE_DIRECT`或`SIGN_MODE_LEGACY_AMINO_JSON`方法签署交易。
> 
> 当一个账户签署一个消息时，它签署一个字节数组。这个字节数组是对消息进行序列化的结果。为了使签名在以后可以被验证，这种转换需要是确定性的。出于这个原因，你定义了一个典型的字节表示的消息，通常参数按字母顺序排列。

##### 消息和交易的生命周期

包含一个或多个有效消息的交易被序列化并由Tendermint共识引擎确认。正如你可能记得的那样，Tendermint对交易的解释是不可知的，并且具有绝对的最终性。当一个交易被包含在一个区块中时，它被确认并最终完成，没有重新组织(回滚)或取消的可能性。

被确认的交易被转发到Cosmos SDK应用程序进行解释。每个消息都通过`BaseApp`使用`MsgServiceRouter`路由到适当的模块。`BaseApp`对交易中包含的每个消息进行解码。每个模块都有自己的`MsgService`，处理每个收到的消息。

##### `MsgService`

尽管继续创建一个新的`MsgService`在技术上是可行的，但推荐的方法是定义一个`Protobuf Msg`服务。每个模块在`tx.proto`中正好有一个`Protobuf Msg`服务的定义，而且模块中每个消息类型都有一个RPC服务方法。Protobuf消息服务隐含地定义了状态的接口层，模块内包含的突变过程。

所有这些是如何转化为代码的呢？下面是一个来自银行模块的MsgService例子 [Bank Overview | Cosmos SDK](https://docs.cosmos.network/v0.46/modules/bank/) 。

```protobuf
// Msg defines the bank Msg service.
service Msg {
  // Send defines a method for sending coins from one account to another account.
  rpc Send(MsgSend) returns (MsgSendResponse);

  // MultiSend defines a method for sending coins from some accounts to other accounts.
  rpc MultiSend(MsgMultiSend) returns (MsgMultiSendResponse);
}
```

在这个例子中。

- 每个`Msg`方法正好有一个参数，例如`MsgSend`，它必须实现`sdk.Msg`接口和一个Protobuf响应。
- 标准的命名惯例是将RPC参数称为`Msg<service-rpc-name>`，RPC响应称为`Msg<service-rpc-name>Response`。

##### 客户端和服务端的代码生成

Cosmos SDK使用Protobuf定义来生成客户端和服务器代码。

- `MsgServer`接口定义了`Msg`服务的服务器API。它的实现在`Msg`服务文档中描述 [`Msg` Services | Cosmos SDK](https://docs.cosmos.network/main/building-modules/msg-services.html) 。
- 所有的RPC请求和响应类型都会生成结构。

> 如果你想更深入地了解消息、Msg服务和模块，请参阅。
> 
> Cosmos SDK关于Msg服务的文档 [`Msg` Services | Cosmos SDK](https://docs.cosmos.network/main/building-modules/msg-services.html) .
> 
> Cosmos SDK关于消息和查询的文档，涉及如何使用Msg服务定义消息 - Amino LegacyMsg [Messages and Queries | Cosmos SDK](https://docs.cosmos.network/main/building-modules/messages-and-queries.html#legacy-amino-legacymsgs) 。

#### 模块(Modules)

> 模块是解决应用层面问题的功能组件，如token管理或治理。Cosmos SDK包括几个现成的模块，以便应用程序开发人员可以专注于他们应用程序的真正独特方面。

每条Cosmos链都是一个专门建造的区块链。Cosmos SDK模块定义了每个链的独特属性。模块可以被认为是更大的状态机中的状态机。它们包含存储布局，或状态，以及状态转换功能（STF），也就是消息方法。

模块定义了Cosmos SDK应用程序的大部分逻辑。

> 这里与substrate的pallets是一致的，都是更强大的状态机，都是业务STF。

![[Pasted image 20221116150124.png]]

当一个交易从底层的Tendermint共识引擎中转时，BaseApp会分解交易中包含的消息，并将消息传送到适当的模块进行处理。当适当的模块消息处理程序收到消息时，就会进行解释和执行。

开发人员使用Cosmos SDK将模块组合在一起，以建立定制的特定应用区块链。

##### 模块范围

模块包括每个区块链节点需要的核心功能。

- 一个应用区块链接口（ABCI）的模板实现，与底层的Tendermint共识引擎进行通信。
- 一个通用的数据存储，用于持久保存模块状态，称为multistore。
- 一个服务器和接口，促进与节点的互动。

模块实现了大部分的应用逻辑，而核心关注的是布线和基础设施问题，并使模块能够被组成高阶模块。

一个模块定义了整体状态的一个子集，使用:

- 一个或多个键或值存储，被称为KVStore。
- 一个应用程序需要的、还不存在的消息类型的子集。

模块还定义了与其他已经存在的模块的交互。

> 对于参与构建Cosmos SDK应用的开发者来说，大部分工作包括构建其应用所需的、尚不存在的自定义模块，并将其与已经存在的模块整合为一个连贯的应用。现有的模块可以来自Cosmos SDK本身，也可以来自第三方开发者。你可以从在线模块库中下载这些模块。

##### 模块组件

最好的做法是在`x/moduleName`文件夹中定义一个模块。例如，名为`Checkers`的模块将放在`x/checkers`中。如果你看一下Cosmos SDK的基础代码，它也在`x/`文件夹中定义了它的模块 [cosmos-sdk/x at main · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/tree/main/x) 。

模块实现了几个元素:

- 接口

接口促进了模块之间的通信，并将多个模块组成连贯的应用程序。

- Protobuf

Protobuf提供了一个`Msg`服务来处理消息，一个gRPC `Query`服务来处理查询。

- Keeper

Keeper是一个控制器，它定义了状态并提出了更新和检查状态的方法。

##### 接口

一个模块必须实现三个应用模块接口，以便与应用程序的其他部分集成。

- `AppModuleBasic`：实现模块的非依赖性元素。
- `AppModule`：相互依赖的，模块的专门元素，对应用程序来说是独一无二的。
- `AppModuleGenesis`：相互依赖的，模块的创世/初始化元素，在开始时建立区块链的初始状态。

你在你的模块的`x/moduleName/module.go`文件中定义`AppModule`和`AppModuleBasic`，以及它们的功能。

##### Protobuf服务

每个模块都定义了两个Protobuf服务。

- `Msg`: 一组与Protobuf请求类型一对一相关的RPC方法，用于处理消息。
- `Query`：gRPC查询服务，用于处理查询。

了解portobuf请看 [Protobuf: Explanation, Benefits + Tutorial - IONOS](https://www.ionos.com/digitalguide/websites/web-development/protocol-buffers-explained/) 

##### `Msg`服务

关于`Msg`服务，请牢记:

- 最佳做法是在`tx.proto`文件中定义`Msg` Protobuf服务。
- 每个模块都应该实现`RegisterServices`方法，作为`AppModule`接口的一部分。这让应用程序知道该模块可以处理哪些消息和查询。
- 服务方法应该使用一个Keeper，它封装了关于storage layout的知识，并提出了更新状态的方法。

##### gRPC `Query` 服务

对于gRPC `Query`服务，请牢记：

- 最佳做法是在`query.proto`文件中定义`Query` Protobuf服务。
- 它允许用户使用gRPC来查询状态。
- 每个gRPC端点都对应着一个服务方法，在gRPC `Query`服务里面用`rpc`前缀命名。
- 它可以在`app.toml`的`grpc.enable`和`grpc.address`字段下进行配置。

Protobuf生成一个`QueryServer`接口，包含每个模块的所有服务方法。模块通过在单独的文件中提供每个服务方法的具体实现来实现这个 `QueryServer` 接口。这些实现方法是相应的gRPC查询端点的处理程序。这种跨越不同文件的关注点划分使得设置不会被Protobuf重新生成的文件所影响。

> gRPC [gRPC](https://grpc.io/) 是一个现代的、开源的、高性能的框架，支持多种语言。它是外部客户（如钱包、浏览器和后台服务）与节点互动的推荐标准。

gRPC-Gateway REST端点支持可能不希望使用gRPC的外部客户端。Cosmos SDK为每个gRPC服务提供一个gRPC-gateway REST端点。关于Gateway请看 [gRPC-Gateway | gRPC-Gateway Documentation Website (grpc-ecosystem.github.io)](https://grpc-ecosystem.github.io/grpc-gateway/) 

##### 命令行命令

每个模块都为一个命令行界面（CLI）定义了命令。与一个模块有关的命令被定义在一个叫做`client/cli`的文件夹中。CLI将命令分为两类：交易和查询。这些命令与你在`tx.go`和`query.go`中分别定义的命令相同。

##### Keeper

Keeper是模块中任何stores的守门人。要访问一个存储空间，必须通过一个模块的Keeper。Keeper封装了关于存储空间布局的知识，并包含更新和检查(inspect)的方法。如果你来自一个模块-视图-控制器（MVC）的世界，那么把Keeper看作是控制器会有帮助。

![[Pasted image 20221116154015.png]]

其他模块可能需要访问store，但其他模块也有可能是恶意的或写得不好的。出于这个原因，开发者需要考虑谁和什么应该访问他们的模块storage。为了防止一个模块在运行时随机访问另一个模块，一个模块需要在构造时声明其使用另一个模块的意图。在这一点上，这样的模块被授予一个runtime key，让它访问另一个模块。只有持有这个key的模块才能访问storage空间。这就是所谓的对象能力模型的一部分。

keeper是在 `keeper.go` 中定义的。一个Keeper的类型定义通常由模块在`multistore`中自己的存储的键(keys)组成，对其他模块keeper的引用，以及对应用程序的编解码器的引用。

##### 核心模块

Cosmos SDK包括一组核心模块，通过良好的解决方案和标准化的实现来解决常见的问题。核心模块解决了应用需求，如代币、staking和治理。

与临时解决方案相比，核心模块有几个优势。

- 早期建立标准化，这有助于确保与钱包、分析、其他模块和其他Cosmos SDK应用程序的良好互操作性。
- 重复工作大大减少，因为应用程序开发人员专注于他们的应用程序的独特之处。
- 核心模块是Cosmos SDK模块的工作实例，它提供了关于建议结构、风格和最佳实践的有力提示。

开发人员通过首先选择和组合核心模块，然后实现自定义逻辑来创建连贯的应用程序。

模块列表，请看 [cosmos-sdk/x at main · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/tree/main/x) 

##### 推荐的目录结构

###### 结构

一个典型的Cosmos SDK模块可以有如下结构。

1. 可序列化的数据类型和Protobuf接口。

```txt
proto
└── {project_name}
    └── {module_name}
        └── {proto_version}
            ├── {module_name}.proto
            ├── event.proto
            ├── genesis.proto
            ├── query.proto
            └── tx.proto
```

- `{module_name}.proto`: 该模块的普通消息类型定义。
- `event.proto`：模块中与事件有关的消息类型定义。
- `genesis.proto`：模块中与genesis状态相关的消息类型定义。
- `query.proto`: 该模块的`Query`服务和相关的消息类型定义。
- `tx.proto`：该模块的`Msg`服务和相关的消息类型定义。

2. 剩下的一些元素

```txt
x/{module_name}
├── client
│   ├── cli
│   │   ├── query.go
│   │   └── tx.go
│   └── testutil
│       ├── cli_test.go
│       └── suite.go
├── exported
│   └── exported.go
├── keeper
│   ├── genesis.go
│   ├── grpc_query.go
│   ├── hooks.go
│   ├── invariants.go
│   ├── keeper.go
│   ├── keys.go
│   ├── msg_server.go
│   └── querier.go
├── module
│   └── module.go
├── simulation
│   ├── decoder.go
│   ├── genesis.go
│   ├── operations.go
│   └── params.go
├── spec
│   ├── 01_concepts.md
│   ├── 02_state.md
│   ├── 03_messages.md
│   └── 04_events.md
├── {module_name}.pb.go
├── abci.go
├── codec.go
├── errors.go
├── events.go
├── events.pb.go
├── expected_keepers.go
├── genesis.go
├── genesis.pb.go
├── keys.go
├── msgs.go
├── params.go
├── query.pb.go
└── tx.pb.go
```

- `client/`：模块的CLI客户端功能实现和模块的集成测试套件。
- `exported/`：模块的导出类型--通常是接口类型（也见以下说明）。
- `keeper`：模块的`Keeper`和`MsgServer`的实现。
- `module/`：模块的`AppModule`和`AppModuleBasic`的实现。
- `simulation/`：该模块的模拟包定义了区块链模拟器应用程序（`simapp`）所使用的功能。
- `spec/`：模块的规范文件，概述了重要的概念，状态存储结构，以及消息和事件类型定义。
- 根目录包括消息、事件和创世状态的类型定义，包括由Protobuf生成的类型定义。
    - `abci.go`：模块的`BeginBlocker`和`EndBlocker`实现。只有在需要定义`BeginBlocker`和/或`EndBlocker`时才需要这个文件。
    -  `codec.go`：模块的接口类型的注册方法。
    - `errors.go`：该模块的哨兵错误。
    - `events.go`：模块的事件类型和构造函数。
    - `expected_keepers.go`：模块的预期其他Keeper接口。
    - `genesis.go`：该模块的genesis状态方法和辅助函数。
    - `keys.go`: 该模块的存储key和相关的辅助函数。
    - `msgs.go`：该模块的消息类型定义和相关方法。
    - `params.go`：该模块的参数类型定义和相关方法。
    - `\*.pb.go`：该模块的类型定义，由protobuf生成，定义在各自的\*.proto文件中。

> 如果一个模块依赖于另一个模块的keeper，那么`exported/`代码元素希望以接口契约的形式接收保keeper，以避免对实现keeper的模块的直接依赖。然而，这些接口契约可以定义对实现keeper的模块进行操作（或返回特定类型）的方法。
> 
> 在`exported/`中定义的接口类型使用规范的类型，允许模块通过`expected_keepers.go`文件接收接口契约。这种模式允许代码保持DRY [Don't repeat yourself - Wikipedia](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) ，也减轻了导入周期的混乱。

##### 错误

我们鼓励模块定义和注册自己的错误，以便为失败的消息或处理程序的执行提供更好的背景。错误应该是常见的或一般的错误，可以进一步包装以提供额外的具体执行环境。[Errors | Cosmos SDK](https://docs.cosmos.network/main/building-modules/errors.html) 

##### 注册

模块应该在`x/{module}/errors.go`中定义和注册他们的自定义错误。错误的注册是通过`types/errors`包处理的。

每个自定义模块错误必须提供代码空间，通常是模块名称（例如，"distribution"），每个模块都是唯一的，以及一个`uint32`代码。代码空间和代码一起提供一个全局唯一的Cosmos SDK错误。

对错误代码的唯一限制是以下几点。

- 它们必须大于1，因为代码值为1是为内部错误保留的。
- 它们在模块内必须是唯一的。

> Cosmos SDK提供了一套核心的常见错误。https://github.com/cosmos/cosmos-sdk/blob/master/types/errors/errors.go 

##### 包装(Wrapping)

自定义模块错误可以作为其具体类型返回，因为它们已经满足错误接口。模块错误可以被包装起来，以便为失败的执行提供进一步的背景和意义。

不管一个错误是否被包装，Cosmos SDK的错误包提供了一个API来确定一个错误是否是通过 `Is` 的特定类型。

##### ABCI

如果一个模块错误被注册，Cosmos SDK错误包允许通过`ABCIInfo` API提取ABCI信息。该包还提供`ResponseCheckTx`和`ResponseDeliverTx`作为辅助API，以自动从错误中获得`CheckTx`和`DeliverTx`响应。

#### Protobuf

Protobuf是Google出品的高性能序列化库，开源，可扩展，跨平台，可支持多语言。主要用于通信，存储的序列化。有自己的一个数据结构描述语言保存在`.proto`文件中，经过编译，生成各种语言的编码解码的程序。一些RPC是用它来做底层的序列化的。gRPC框架也直接支持Protobuf。

怎样用protobuf呢？ 首先你必须在一个.proto文件中定义一个数据结构。这是一个具有描述性语法的普通文本文件。数据被表示为包含称为字段的name-value对的消息。

接下来，编译你的Protobuf schema。`.protoc`生成数据访问类，根据命令行选项，用你喜欢的语言为每个字段设置访问器。访问器包括序列化、反序列化和解析。

##### Go中的Protobuf基础

Go的gobs https://golang.org/pkg/encoding/gob/ 包是一个用于Go环境的综合包。然而，如果你需要与用其他语言编写的应用程序共享信息，它就不能很好地工作。如何应对那些本身可能包含需要解析或编码的信息的字段是另一个挑战。

例如，一个JSON或XML对象可能包含不连续的字段，这些字段被存储在一个字符串字段中。在另一个例子中，一个时间可能被存储为代表小时和分钟的两个整数。Protobuf封装了两个方向的必要转换。生成的类为字段提供了getters和setters，并处理了作为一个单元的读写消息的细节。

Protobuf格式支持随着时间的推移扩展格式，这样代码仍然可以读取以旧格式编码的数据。

Go开发人员通过Go Protobuf API访问生成的源代码中的设置器和获取器。

[Cosmos SDK](https://docs.cosmos.network/main/core/encoding.html)

[Protobuf Documentation | Cosmos SDK](https://docs.cosmos.network/main/core/proto-docs.html) 

##### gRPC

gRPC可以使用Protobuf作为其接口定义语言和底层消息交换格式。客户端可以通过gRPC直接调用不同机器上的服务器应用程序的方法，就像调用一个本地对象一样。

gRPC是基于定义一个服务并指定可以远程调用的方法及其参数和返回类型的想法。关于gRPC的客户端和服务器端，请记住以下几点。

- 服务器端：服务器实现这个接口并运行gRPC服务器来处理客户端的调用。
- 客户端：客户端有一个stub（在某些语言中被称为只是一个客户端），提供与服务器相同的方法。

gRPC客户端和服务器可以在各种环境中运行并相互交谈，从谷歌内部的服务器到你自己的桌面，并且可以用gRPC支持的任何语言编写。例如，你可以很容易地用Java创建一个gRPC服务器，客户端用Go、Python或Ruby。

最新的Google API将有gRPC版本的接口，让你轻松地在你的应用程序中建立谷歌功能。

##### 类型

Cosmos SDK应用程序的核心主要由类型定义和构造函数组成。在`app.go`中定义，一个自定义应用程序的类型定义只是一个由以下内容组成的结构。  
  
- 对`BaseApp`的引用：对`BaseApp`的引用为你的应用程序定义了一个嵌入`BaseApp`的自定义应用程序类型。对`BaseApp`的引用允许自定义应用程序继承`BaseApp`的大部分核心逻辑，如ABCI方法和路由逻辑。  
- store keys列表：Cosmos SDK中的每个模块都使用一个mutistore来保存他们的部分状态。访问这种存储需要一个在应用程序的类型定义中声明的keys列表。  
- 每个模块的keepers列表：keeper是每个模块中的一个抽象部分，用于处理模块与存储的交互，指定对其他模块keeper的引用，并实现模块的其他核心功能。为了实现跨模块的交互，Cosmos SDK中的所有模块都需要在应用程序的类型定义中声明它们的keeper，并作为接口输出给其他模块，这样一个模块的keeper的方法就可以在其他模块的授权下被调用和访问。  
- 对编解码器的引用：默认为`go-amino`，你的Cosmos SDK应用程序中的编解码器可以用其他合适的编码框架代替，只要它们以byte slices的形式持久化数据存储，并且是确定性的。  
- 对模块管理器的引用：对一个包含应用模块列表的对象的引用，称为模块管理器。  
  
#### Mutistore和Keepers

> Keepers负责管理对模块所定义的状态的访问。因为状态是通过Keepers来访问的，所以它们是确保不变性被强制执行和安全原则始终被应用的理想场所。

在特定应用的区块链上的Cosmos SDK应用通常由几个模块组成，分别处理不同的问题。每个模块都有一个状态，是整个应用状态的一个子集。  
  
Cosmos SDK采用了一种基于对象能力的方法，以帮助开发者更好地保护他们的应用程序免受不必要的模块间互动。Keepers是这种方法的核心。  
  
  
> Keeper是Cosmos SDK的一个抽象，它管理对一个模块所定义的状态子集的访问。所有对模块数据的访问必须通过模块的Keeper。  
  
Keeper可以被认为是一个模块的存储(stores)的字面看门人。在一个模块中定义的每个存储空间（each store）（通常是IAVL store）都有一个`storeKey`，它授予无限的访问权。模块的keeper持有这个`storeKey`，否则它就不会被暴露出来，并定义了对任何存储store的读写方法。  
  
当一个模块需要与另一个模块中定义的状态进行交互时，它通过与其他模块的keeper的方法进行交互。开发者通过定义方法和控制访问来控制他们的模块与其他模块的交互。  
  
##### 格式

Keepers一般被定义在位于模块文件夹中的`/keeper/keeper.go`文件中。一个模块的Keeper类型按惯例简单命名为`keeper.go`。它通常遵循以下结构。

```go
type Keeper struct {
    // Expected external keepers, if any

    // Store key(s)

    // codec
}
```

##### 参数

以下参数对于模块中keeper的类型定义非常重要。  
  
- 一个预期的`keeper`是一个模块的外部keeper，它被该模块的内部keeper所要求。外部keeper作为接口被列在内部keeper的类型定义中。这些接口本身被定义在模块文件夹根目录下的 `expected_keepers.go` 文件中。接口的使用是为了减少依赖关系的数量，并在此背景下方便模块的维护。  
- `storeKeys`授予对模块所管理的multistore的任何存储的访问。它们应该始终保持对外部模块的不暴露。  
- `cdc`是用于对`[]byte`的结构进行`marshal`和`unmarshal`的编解码器。`cdc` 可以是 `codec.BinaryCodec、codec.JSONCodec` 或 `codec.Codec`，具体取决于您的需求。请注意，`code.Codec`包括其他接口。它可以是proto或amino codec，只要它们实现了这些接口。  

每个keeper都有自己的构造函数，从应用程序的构造函数中调用。这是keeper被实例化的地方，也是开发者确保将模块的keeper的正确实例传递给需要它们的其他模块的地方。  

##### 范围和最佳实践

keepers主要为其模块所管理的store暴露getter和setter方法。方法应该保持简单，并严格限于获取或设置请求的值。在调用keeper的方法之前，应该已经用消息和`Msg`服务器的`ValidateBasic()`方法进行了有效性检查。

getter方法通常有以下签名。

```go
func (k Keeper) Get(ctx sdk.Context, key string) (value returnType, found bool)
```

setter 方法通常有以下签名

```go
func (k Keeper) Set(ctx sdk.Context, key string, value valueType)
```

Keeper也应该在适当的时候实现一个具有以下签名的迭代器方法。

```go
func (k Keeper) GetAll(ctx sdk.Context) (list []returnType)
```


##### store 类型

Cosmos SDK提供了不同的store类型供人们使用。对可用于开发的不同store类型有一个良好的概述是很重要的。

> 这里与substrate中的存储是一致的，有单map和双map

###### cosmos中的KVStore和Multistore

每个Cosmos SDK应用程序在其根部包含一个状态，即`Multistore`。它被细分为独立的隔间（compartments），由应用程序中的每个模块管理。`Multistore`是一个遵循`Multistore`接口  [cosmos-sdk/store.go at v0.40.0-rc6 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L104-L133)  的`KVStores`的store。

基本的`KVStore`和`Multistore`的实现被包裹在提供专门行为的扩展中。一个`CommitMultistore`  [cosmos-sdk/store.go at v0.40.0-rc6 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/types/store.go#L141-L184) 是一个有commiter的`Multistore`。这是在Cosmos SDK中使用的主要的multistore类型。底层的`KVStore`主要用于限制对committer的访问。

`rootMulti.Store`  [cosmos-sdk/store.go at v0.40.0-rc6 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/rootmulti/store.go#L43-L61) 是`CommitMultiStore`接口的首要实现。它是一个围绕着数据库建立的底层multistore，在其上可以挂载多个`KVStores`。它是`BaseApp`中使用的默认multistore的store。

###### CacheMultiStore

`cachemulti.Store` [cosmos-sdk/store.go at v0.40.0-rc6 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc6/store/cachemulti/store.go#L17-L28) 在`rootMulti.Store`需要分支的时候被使用。`cachemulti.Store`将所有子substore分支，在其构造函数中为每个substore创建一个虚拟store，并将它们保存在`Store.store`中。这主要用于创建一个孤立的store，通常是在需要突变状态的时候，但以后可能会被还原。

`CacheMultistore`缓存了所有的读取查询。`Store.GetKVStore()`从`Store. stores`中返回store，`Store.Write()`在所有substores上递归调用`CacheWrap.Write()`。

###### Transient store

顾名思义，瞬时store，`Transient.Store`是一个`KVStore`，在每个区块结束时自动被丢弃。`Transient.Store`是一个带有`dbm.NewMemDB（）`的`dbadapter.Store`。所有的`KVStore`方法都被重复使用。当`Store.Commit()`被调用时，一个新的`dbadapter.Store`被分配，丢弃之前的引用。垃圾收集被自动关注。

> 如果要进一步了解IAVL Store，就请看 [iavl/overview.md at v0.15.0-rc5 · cosmos/iavl (github.com)](https://github.com/cosmos/iavl/blob/v0.15.0-rc5/docs/overview.md) cosmos的存储底层使用的是AVL+Tree，而不是以太坊和polkadot使用的Merkle Patricia Trie。

> `KVStore`和`CommitKVStore`的默认实现是`IAVL.Store`。`IAVL.Store`是一个自平衡的二进制搜索树，确保获取和设置操作是`O(log n)`，其中`n`是树中元素的数量。

##### 额外的KVStore包装器

除了前面的store类型外，还有一些其他更具体的用途。

###### GasKv store

Cosmos SDK应用程序使用gas来跟踪资源使用情况并防止垃圾邮件（spam）。`GasKv.Store`是一个`KVStore`包装器，它可以在每次对存储进行读或写时自动消耗gas。它是在Cosmos SDK应用程序中跟踪storage使用的首选解决方案。

当父`KVStore`的方法被调用时，`GasKv.Store`会根据`Store.gasConfig`自动消耗适当的gas量。所有的`KVStores`在被检索时默认被包裹在`GasKv.Store`中。这是在上下文的`KVStore()`方法中完成的。在这种情况下会使用默认的gas配置。

###### TraceKv Store

`tracekv.Store`是一个`KVStore`包装器，它在底层`KVStore`上提供操作跟踪功能。如果追踪功能在父级`MultiStore`上被启用，它将被Cosmos SDK自动应用于所有`KVStores`。当每个`KVStore`方法被调用时，`tracekv.Store`会自动将`traceOperation`记录到`Store.writer`中。当`traceOperation.Metadata`不是`nil`时，它被填充为`Store.context`。`TraceContext`是一个`map[string] interfacen{}`。

###### Prefix store

`prefix.Store`是一个`KVStore`包装器，它在底层的`KVStore`上提供自动的key前缀功能。

- 当`Store.{Get, Set}()`被调用时，商店会将调用转发到它的父级，并以`Store.prefix`为key的前缀。
- 当`Store.Iterator()`被调用时，它并不简单地以`Store.prefix`为前缀，因为它并不像预期那样工作。在这种情况下，一些元素即使不是以前缀开始的，也会被遍历。

##### `AnteHandler`

`AnteHandler`是一个实现`AnteHandler`接口的特殊处理器。它用于在交易的内部消息被处理之前对交易进行认证。

`AnteHandler`理论上是可选的，但仍然是公共区块链网络的一个非常重要的组成部分。它有三个主要目的。

- 它是防止垃圾邮件的第一道防线，也是防止交易重放的第二道防线（在mempool之后），有费用扣除和序列检查。
- 它执行初步的状态有效性检查，如确保签名有效，或发件人有足够的资金来支付费用。
- 它在通过收取交易费来激励利益相关者方面发挥了作用。

`BaseApp`持有一个`AnteHandler`作为参数，在应用程序的构造函数中被初始化。最广泛使用的`AnteHandler`是`auth`模块。

> cosmos SDK 的gas和费用 包括AnteHandler 更多的请看  [cosmos-sdk/04-gas-fees.md at main · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/main/docs/docs/basics/04-gas-fees.md)  [cosmos-sdk/04-gas-fees.md at main · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/main/docs/docs/basics/04-gas-fees.md#antehandler) 

#### BaseApp

> 在这一节中，你将发现如何定义应用程序状态机和服务路由器，如何创建自定义交易处理，以及如何创建在每个块的开始或结束时执行的周期性进程。

`BaseApp`是一个Cosmos SDK应用程序的模板实现。这个抽象实现了每个Cosmos应用程序需要的功能，首先是Tendermint应用区块链接口（ABCI）的实现。

> Tendermint共识是与应用程序无关的。它建立了规范的交易列表，并将确认的交易发送给Cosmos SDK应用程序进行解释，反过来，它也接收来自Cosmos SDK应用程序的交易，并将其提交给validator进行确认。

依靠Tendermint共识的应用程序必须实现支持ABCI接口的具体函数。`BaseApp`包括一个ABCI的实现，所以开发者不需要构建一个。

ABCI本身包括一些方法，如`DeliverTx`，它交付一个交易。交易的解释是应用程序层面的责任。由于一个典型的应用程序支持不止一种类型的交易，解释意味着需要一个服务路由器，它将根据交易类型将交易发送到不同的解释器。`BaseApp`包括一个服务路由器的实现。

除了ABCI的实现之外，`BaseApp`还提供了一个状态机的实现。状态机的实现是一个应用层面的问题，因为Tendermint的共识是与应用无关的。Cosmos SDK的状态机实现包含一个整体状态，该状态被细分为各种子状态。子状态包括模块状态、持久状态和瞬态状态。这些都是在`BaseApp`中实现的。

`BaseApp`在应用程序、区块链和状态机之间提供了一个安全接口，同时对状态机的定义尽可能少。

##### 定义一个应用

开发人员通常通过引用`BaseApp`并声明store keys、keepers和模块管理器，为他们的应用程序创建一个自定义类型，像这样。

```go
type App struct {
  // reference to a BaseApp
  *baseapp.BaseApp

  // list of application store keys

  // list of application keepers

  // module manager
}
```

用`BaseApp`扩展应用程序，使前者可以访问`BaseApp`的所有方法。开发人员用他们想要的模块组成他们的自定义应用程序，而不必关心实现ABCI、服务路由器和状态管理逻辑的艰苦工作。

###### 类型定义

对于任何基于Cosmos SDK的应用，`BaseApp`类型拥有许多重要的参数[cosmos-sdk/baseapp.go at v0.40.0-rc3 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/v0.40.0-rc3/baseapp/baseapp.go#L46-L131) 。

###### Bootstrapping

在应用程序的启动过程中，被初始化的重要参数有。  
  
- `CommitMultiStore`：这是应用程序的主要存储空间，它持有在每个块结束时提交的cononical状态。这个store没有缓存，也就是说，它不用于更新应用程序的易失性（未提交的）状态。  
  
`CommitMultiStore`是一个stores的store。应用程序的每个模块使用multistore中的一个或多个`KVStores`来持久化它们的状态子集。  
  
- `Database`：数据库被CommitMultiStore用来处理数据持久性。  
  
- `Msg`服务路由器：`msgServiceRouter`便于将`sdk.Msg`请求路由到适当的模块`Msg`服务进行处理。  
  
这里的`sdk.Msg`指的是需要被服务处理以更新应用状态的交易组件，而不是ABCI消息，它实现了应用和底层共识引擎之间的接口。  
  
- `gRPC Query Router`：`grpcQueryRouter`便于将gRPC Query路由到将处理这些查询的适当模块。这些查询本身不是ABCI消息。它们被转发到相关模块的gRPC查询服务。  
  
- `TxDecoder`: 这是用来解码由底层Tendermint引擎转发的原始交易字节的。  
  
- `ParamStore`：这是用于获取和设置应用程序共识参数的参数存储。  
  
- `AnteHandler`：这是用来处理签名验证，费用支付，以及其他在收到交易时的消息执行前检查。它在`CheckTx/RecheckTx`和`DeliverTx`期间执行。  
  
- `InitChainer`、`BeginBlocker`和`EndBlocker`：这些是当应用程序收到来自底层Tendermint引擎的`InitChain`、`BeginBlock`和`EndBlock` ABCI消息时执行的函数。  
  
###### Volatile 状态

定义易失性状态、缓存状态的参数包括。

- `checkState`：这个状态在`CheckTx`期间被更新，在`Commit`时被重置。
- `deliverState`：这个状态在`DeliverTx`期间被更新，在`Commit`时被设置为`nil`。它在`BeginBlock`中被重新初始化。

###### 共识参数

共识参数定义了整体的共识状态。

- `voteInfos`：这个参数包含了pre-commit缺失的validators名单，要么是因为他们没有投票，要么是因为提议者（proposer）没有包括他们的投票。这个信息由上下文携带，可以被应用程序用来做各种事情，比如惩罚缺席的validator。
- `minGasPrices`：这个参数定义了节点所接受的最低gas价格。这是一个本地参数，意味着每个全节点可以设置不同的`minGasPrices`。在`CheckTx`期间，它被用于`AnteHandler`，主要是作为一种垃圾邮件保护机制。只有当交易的gas价格大于`minGasPrices`中的最低gas价格之一时，交易才会进入`mempool`。如果`minGasPrices == 1uatom,1photon`，交易的gas价格必须大于`1uatom OR 1photon`。
- `appVersion`：在应用程序的构造函数中设置的应用程序的版本。

###### 构造函数

```go
func NewBaseApp(
  name string, logger log.Logger, db dbm.DB, txDecoder sdk.TxDecoder, options ...func(*BaseApp),
) *BaseApp {

  // ...
}
```

`BaseApp`的构造函数是非常直接的。注意到向`BaseApp`提供额外`options`的可能性，`BaseApp`会按顺序执行这些选项。这些选项通常是重要参数的设置函数，如`SetPruning()`设置修剪选项，或`SetMinGasPrices()`设置节点的最小gas价格。

开发者可以根据他们的应用程序的需要添加额外的选项。

##### 状态

BaseApp提供了三种主要状态。两个是易失(volatile)的，一个是持久的。

- 持久的主状态是应用程序的canonical状态。
- 易失的状态`checkState`和`deliverState`用于处理提交过程中主状态之间的转换。

有一个单一的`CommitMultiStor`e，被称为主状态或根状态。`BaseApp`使用一种叫做分支的机制从这个主状态派生出两个易失状态，这个分支由`CacheWrap`函数执行。

> 我的理解就是，持久状态就是整个链的固化的DB里面的世界状态，易失的状态就是变化的世界状态，还没被commit到DB中的临时状态。

###### InitChain状态更新



###### CheckTx 状态更新

###### BeginBlock 状态更新

###### DeliverTx状态更新

###### Commit状态更新

###### ParamStore

##### 服务路由器

###### Msg服务路由器


#### 查询

#### 事件

#### 上下文(Context)

#### 迁移

#### 桥
