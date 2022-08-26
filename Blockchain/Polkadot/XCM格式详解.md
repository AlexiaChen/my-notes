
### 格式概览

XCM 提供了有关如何编写、发送和解释跨链消息的指令。XCM 消息中包含的指令由 XCVM（跨共识虚拟机）执行，XCVM 是集成在发送链和接收链中的高级非图灵完备计算机。

XCM 消息包括了指令、资产和任意类型的数据：

![[b431f9de285c788c5e17202c24805326.png]]


如上图中，由平行链 A 发送至平行链 B 的 XCM 消息中包含了消息目的地、任意数据、指令、资产、受益人的信息。

**XCM 是消息格式，而不是跨链协议。XCM 消息可以经由多种路径和通道发送：**

![[b611d09e65dfebbd908c8b61ca909188.png]]

-   XCMP（跨链消息传递协议）：在平行链之间传递消息 
-   VMP（垂直消息传递协议）：在中继链和某条平行链之间传递消息
-   桥：在波卡和外部区块链网络之间，或非波卡的两条单链之间传递消息

> 目前平行链间的消息传递实际使用的是 HRMP（水平中继路由消息传递协议），它是 XCMP 的过渡协议，同样用于在平行链之间传递消息。它与 XCMP 的区别在于其将消息存在中继链存储中，因此对资源的消耗更大。

所以你看到了，XCM只是一种消息传递格式，类似于json，XML这样，它不是消息传输协议，消息传输协议是HRMP，XCMP,VMP这样的，类似于TCP/IP这样。它不能用来在系统之间实际 "发送" 任何消息；它的作用只是表达接收方应该做什么。

![[3acb8a69b044476dcebf534dbb3c2113.png]]
**除了在链之间发送消息外，XCM 在其他情况下也很有用，** 比如与一个你不一定提前知道交易格式的链进行交易。对于业务逻辑变化不大的链（比如，比特币），交易格式 -- 或者说钱包用来向链发送指令的格式 -- 往往会无限期地保持完全相同，或者至少是兼容。有了高度可进化的基于元协议的链，如 Polkadot 及其组成的平行链，商业逻辑可以通过一个交易在整个网络中升级。这可以改变任何东西，包括交易格式，为钱包维护者带来了潜在的问题，特别是对于那些需要保持离线的钱包（如 Parity Signer）。由于 XCM 是良好版本管理的，抽象的和通用的，它可以作为一种手段，为钱包提供一个持久的交易格式，用来创建许多常见的交易。

### 格式详解

XCM 格式的核心是 XCVM。与一些人的看法相反，这不是一个（有效的）罗马数字（尽管如果是的话，它可能意味着 905）。**事实上，这代表了跨共识虚拟机。它是一种超高层的非图灵完备计算机，其指令被设计成与交易大致处于同一水平。**

XCM 中的 "消息" 实际上只是一个在 XCVM 上运行的程序。它是一条或多条 XCM 指令。该程序一直运行到最后或遇到错误为止，这时它就会结束（我暂时故意不解释）并停止。

XCVM 包括一些寄存器，以及对承载它的共识系统整体状态的访问。指令可能会改变一个寄存器，也可能会改变共识系统的状态，或者两者都改变。

这种指令的一个例子是 `TransferAsset`，它被用来将资产转移到远程系统的一些其他地址。它需要被告知要转移哪些资产，以及资产要被转移到谁/哪里。在 Rust 中，它是这样声明的：

```rust
enum Instruction {
	TransferAsset {
		assets: MultiAssets,
		beneficiary: MultiLocation,
	}
	/*snip*/
｝
```

正如你可能猜到的那样，`assets` 是表达哪些资产将被转移的参数，而`beneficiary`则说明这些资产将被放到谁/哪里。当然，我们还缺少另外一个信息，即资产将从谁/哪里获得。这一点可以从 Origin Register 中自动推断出来。当程序开始时，这个寄存器一般根据传输系统（桥接器、XCMP 或其他什么）进行设置，以反映信息的实际来源，它与受益人的信息类型相同。Origin Register 作为一个受保护的寄存器运行 -- 程序不能任意设置它，尽管有两条指令可以用来以某些方式改变它。

在 XCM 中，所使用的类型是相当基本的想法：资产，由 `MultiAsset` 代表，以及共识内的地址，由 `MultiLocation` 代表。Origin Register 是一个可选的 `MultiLocation`（可选的，因为如果需要的话，它可以被完全清除）。

##### XCM中，如何表达位置（地点）

`MultiLocation` 类型标识了存在于共识世界中的任何单一位置。它是一个相当抽象的概念，可以代表存在于共识中的各种事物，从可扩展的多分片区块链，如 Polkadot，一直到平行链上的低级 ERC-20 资产账户。在计算机科学术语中，它实际上只是一个全局性的单例数据结构，无论其大小或复杂性如何。

`MultiLocation` 总是表达一个相对于当前位置的位置。你可以认为它有点像文件系统路径，但没有办法直接表达文件系统树的"根"。这是因为一个简单的原因：在 Polkadot 的世界里，区块链可以被合并到其他区块链中，也可以从其他区块链中分离出来。一个区块链可以开始非常孤独的生活，并最终被提升到成为一个更大的共识中的一个平行链。如果它这样做了，那么，"根"的含义将在一夜之间改变，这可能会给 XCM 信息和其他使用 `MultiLocation` 的东西带来混乱。为了使事情简单化，我们完全排除了这种可能性。

XCM 中的位置是分层次的；共识中的一些地方完全被封装在共识中的其他地方。Polkadot 的一个平行链完全存在于整个 Polkadot 共识中，我们称它为内部位置。更严格地说，我们可以说，只要有一个共识系统，其中的任何变化都意味着另一个共识系统的变化，那么前一个系统就是在后者的内部。例如，一个 Canvas 智能合约对承载它的合约 pallet 来说是内部的。比特币中的 UTXO 是比特币区块链的内部。

这意味着 XCM 并不区分 "谁" 和 "哪里？"这两个问题。从 XCM 这种相当抽象的角度来看，这种区别其实并不重要 -- 这两个问题模糊不清，本质上是同一件事。

`MultiLocations` 用于识别发送 XCM 信息的地方，可以接收资产的地方，然后甚至可以帮助描述资产本身的类型，我们将看到非常有用的东西。

当把它们写成本文这样的文本时，它们被表达为一些`..`（或 "父级"，封装的共识系统）组件，后面是一些交叉点，都用/隔开。（当我们用 Rust 这样的语言表达它们时，一般不会出现这种情况，但在写作中是有意义的，因为它很像人们熟悉的目录路径，被广泛使用。）交叉点在其封装的共识系统中识别一个内部位置。如果根本就没有父节点/交叉点，那么我们就说这个位置是这里。

其实有点像URL路径。

以下是一些MuliLocation的例子:

- `../Parachain(1000)`: 在一个平行链中进行评估，这将确定我们的同级平行链的索引为 1000。(在 Rust 中我们会写 `ParentThen(Parachain(1000)).into()`)
- `../AccountId32(0x1234...cdef)`: 在平行链中进行评估，这将确定中继链上的 32 字节账户 `0x1234...cdef`。
- `Parachain(42)/AccountKey20(0x1234...abcd)`: 在中继链上评估，这将识别 42 号平行链上的 20 字节账户 `0x1234...abcd`（估计是像 Moonbeam 那样的东西，它承载着与 Ethereum 兼容的账户）。

##### XCM上表达资产

在 XCM 中工作时，经常需要提及某种资产。这是因为几乎所有现存的公链都依赖一些本地数字资产来为其内部经济和安全机制提供支柱。对于工作证明区块链，如比特币，原生资产（BTC）被用来奖励发展区块链的矿工，并防止重复消费。对于像 Polkadot 这样的基于权益证明的区块链，原生资产（DOT）被用作一种抵押品，网络维护者（被称为 stakers）必须冒着风险，以生成有效的区块并获得实物奖励。

一些区块链管理多种资产，例如，以太坊的 ERC-20 框架允许在链上管理许多不同的资产。有些管理的资产不是可替代资产 (如以太坊的 ETH)，而是不可替代资产 (NFT) -- 独一无二的实例；Crypto-kitties 是这种不可替代的代币或 NFT 的早期例子。

**XCM 的设计是为了能够处理所有这类资产而不费力。** 为此，有一个数据类型 `MultiAsset` 及其相关类型 `MultiAssets`、`WildMultiAsset` 和 `MultiAssetFilter`。让我们来看看 Rust 中的 `MultiAsset`:

```rust
struct MultiAsset {
	id: AssetId,
	fun: Fungibility,
}
```

因此，有两个字段定义了我们的资产：`id` 和 `fun`，这很好地说明了 XCM 是如何处理资产的。首先，必须提供一个整体的资产身份。对于可替代的资产来说，这只是确定了资产的身份。对于 NFT 来说，这确定了整个资产的 `class`--不同的资产实例可能在这个`class`中。

```rust
enum AssetId {
	Concrete(MultiLocation),
	Abstract(BinaryBlob),
}
```

**资产身份以两种方式之一表达；具体或抽象。** Abstract的方式并没有真正使用，但它允许资产 ID 通过名称来指定。这很方便，但依赖于接收者以发送者期望的方式解释名称，这可能并不总是那么容易。Concrete的方式被普遍使用，它使用一个位置来明确地识别资产。对于本地资产（如 DOT），资产往往被识别为铸造该资产的链（本例中的 Polkadot 中继链，这将是其平行链中的一个位置）。主要在一个链的 pallet 内管理的资产可以通过一个位置来识别，包括其在该 pallet 内的索引。例如，Karura 平行链可能指的是 Statemine 链上的资产，位置为：`../Parachain(1000)/PalletInstance(50)/GeneralIndex(42)`。

```rust
enum Fungibility {
	Fungible(NonZeroAmount),
	NonFungible(AssetInstance),
}
```

**其次，它们必须是可替代（Fungible）的或不可替代（NonFungible）的。** 如果它们是Fungible资产，那么就应该有一些相关的非零金额。如果它们是NonFungible资产，那么就应该有一些指示来说明它们是哪一个实例，而不是一个数额。这通常用一个索引来表示，但 XCM 也允许使用其他各种数据类型，如数组和二进制 blobs。

这涵盖了 `MultiAsset`，但我们有时也会使用其他三种相关类型。`MultiAssets` 是其中之一，实际上只是指一组 `MultiAsset` 的条目。然后我们有 `WildMultiAsset`；这是一个通配符，可以用来与一个或多个 `MultiAsset` 项目相匹配。实际上，它只支持两种通配符。`All`（与所有资产匹配）和 `AllOf`（与具有特定身份（`AssetId`）和Fungible的所有资产匹配）。值得注意的是，对于后者，不需要指定金额（对于Fungible资产）或实例（对于NonFungible资产），所有资产都被匹配。

**最后，还有 `MultiAssetFilter`。** 这是最常用的，它实际上只是 `MultiAssets` 和 `WildMultiAsset` 的组合，允许指定通配符或明确的（即非通配符）资产列表。

在 Rust XCM API 中，我们提供了大量的转换功能，以使这些数据类型的工作尽可能不费力。例如，当我们在 Polkadot 中继链上时，要指定相当于 100 个不可分割的 DOT 资产单位（Planck，对于那些知道的人来说）的Fungible的 `MultiAsset`，那么我们将使用`(Here, 100).into()`。

##### XCM指令之原始寄存器(Origin Register)和保留寄存器(Holding Register)

让我们来看看另一条 XCM 指令：`WithdrawAsset`。从表面上看，这有点像 `TransferAsset` 的前半部分：它从 Origin Register 指定的地方的账户中提取一些资产。但它是如何处理这些资产的呢？- 如果它们没有被存放在任何地方，那么这肯定是一个非常无用的操作。让我们看一下它的 Rust 声明：

```rust
WithdrawAsset(MultiAssets)
```

所以，这次只有一个参数（`MultiAssets`类型，决定了哪些资产必须从 Origin Register 的所有权中撤出）。但并没有指定把资产提取出来以后放在什么地方。

已提取和未使用的资产被暂时保存在所谓的`Holding Register`(保留寄存器) 中（"holding"是因为它们处于临时位置，不能无限期地存在）。有许多指令在`Holding Register`上操作。其中一个非常简单的指令是 `DepositAsset` 指令。让我们来看看它：

```rust
enum Instruction {
	DepositAsset {
		asset: MultiAssetFilter,
		max_assets: u32,
		beneficiary: MultilLocation,
	},
	/* snip */
}
```

精明的读者会发现，这看起来很像 `TransferAsset` 指令中缺少的那后一半。我们有 `assets` 参数，指定哪些资产应该从`Holding Register`中取出来存入链上。`max_assets` 让 XCM 作者通知接收者打算存入多少个独特的资产。(这在事先了解 c 的内容而计算费用时很有帮助，因为存入资产可能是一项昂贵的操作）。最后是`beneficiary`(受益人)，也就是我们之前在 `TransferAsset` 操作中遇到的那个参数。

有许多指令表达了对 beneficiary 的操作，`DepositAsset` 是其中最简单的一个。其他的一些指令则比较复杂。

到此为止，更多的详情请看 
- [跨共识信息格式（XCM） | Moonbeam Docs](https://docs.moonbeam.network/cn/builders/xcm/overview/)
- [xcm-format/README.md at master · paritytech/xcm-format (github.com)](https://github.com/paritytech/xcm-format/blob/master/README.md)

##### XCM中支付手续费

**XCM 中的手续费支付是一个相当重要的用例。** Polkadot 社区中的大多数链会要求他们的对话者为他们想要进行的任何操作支付手续费，否则他们会让自己受到"交易垃圾邮件"和DoS攻击。当链有充分的理由相信他们的对话者会表现良好时，就会出现例外情况 -- 当 Polkadot 中继链与 Polkadot Statemint 共同利益链相对应时就是这种情况。然而对于一般情况，收费是确保 XCM 消息及其传输协议不能被过度使用的好方法。让我们来看看当 XCM 消息到达 Polkadot 时如何支付手续费。

正如已经提到的，XCM 并没有把手续费和手续费支付的想法作为第一等公民纳入其中。与以太坊的交易模式不同，手续费支付并不是协议中的内容，没有必要的用例必须明显地规避。就像 Rust 的零成本抽象一样，在 XCM 中，手续费支付没有很大的设计开销。

不过对于那些需要支付手续费的系统，**XCM 提供了用资产购买执行资源的能力。** 概括地说，这样做包括三个部分。

- 首先，需要提供一些资产。
- 其次，必须就用资产换取计算时间（用 Substrate 的说法就是权重）进行协商。
- 最后，XCM 的操作将按照指示进行。

第一部分是由若干提供资产的 XCM 指令之一管理的。我们已经知道其中一条指令（`WithdrawAsset`），但还有其他几条，我们将在后面看到。当然，持有寄存器中的资产将用于支付与执行 XCM 相关的手续费。任何未用于支付费用的资产，我们都将存入某个目标账户中。在我们的例子中，我们假设 XCM 发生在 Polkadot 中继链上，而且是 1 DOT（即 10,000,000,000 不可分割的单位）。

到目前为止，我们的 XCM 指令看起来像：

```rust
WithdrawAsset((Here, 10_000_000_000).into()),
```

这给我们带来了第二部分，用这些资产的（部分）换取计算时间来支付我们的 XCM 的费用。为此，我们有一个 XCM 指令 `BuyExecution`。让我们来看看它。

```rust
enum Instruction {
    /*snip*/
    BuyExecution {
       fees: MultiAsset,
       weight: u64,
    },
}
```

第一项 `fees` 是应从 Holding Register 中提取并用于支付手续费的金额。从技术上讲，这只是最高限额，因为任何未使用的余额都会立即退回。

最终花费的金额由解释系统决定 -- `fees` 只是限制它，如果解释系统需要为所需的执行支付更多的费用，那么 `BuyExecution` 指令将导致错误。第二项是指定要购买的执行时间的数量。这一般应不低于 XCM 程序 (programme) 的总权重。

在我们的例子中，我们假设所有的 XCM 指令都需要一百万的权重，所以到目前为止，我们的两个项目（`WithdrawAsset` 和 `BuyExecution`）是两百万，还有一个是接下来的内容。我们将使用所有我们拥有的 DOT 来支付这些费用（这只是一个好主意，如果我们相信目的地链不会有疯狂的费用 -- 我们将假设我们有）。让我们看看到目前为止我们的 XCM:

```rust
WithdrawAsset((Here, 10_000_000_000).into),
BuyExecution {
	fees: (Here, 10_000_000_000).into(),
	weight: 3_000_000
},
```

我们的 XCM 的第三部分是将剩余的资金存入 Holding Register。为此，我们将使用 `DepositAsset` 指令。我们实际上不知道持有 Holding Register 中还有多少资金，但这并不重要，因为我们可以为应该存入的资产指定一个通配符。我们将把它们存入 Statemint 的主权账户（该账户被识别为 `Parachain(1000)`）。

因此，我们最终的 XCM 指令看起来像这样：

```rust
WithdrawAsset((Here, 10_000_000_000).into),
BuyExecution {
	fees: (Here, 10_000_000_000).into(),
	weight: 3_000_000
},

DepositAsset {
	assets: All.into(),
	max_assets: 1,
	beneficiary: Parachain(1000).into(),
}
```

##### 在不同链之间移动资产

向另一条链发送资产可能是链间信息传递的最常见的使用情况。允许一条链管理另一条链的原生资产，可以实现各种衍生用途（不是双关语），最简单的是去中心化的交易所，但通常被归类为去中心化的金融或 DeFi。

一般来说，有两种方式可以让资产在链之间移动，**这取决于链之间是否信任对方的安全和逻辑。**

##### Teleporting（远程传送）

对于相互信任的链（如同一整体共识和安全保护伞下的同质分片），我们可以使用 Polkadot 称为远程传送 (Teleporting) 的框架，这基本上只是意味着在发送方销毁资产，在接收方铸币。这很简单，也很高效 -- 它只需要两个链的协调，而且只涉及任何一方的一个动作。不幸的是，如果接收链不能 100% 相信发送链会真正销毁它正在铸造的资产（并且确实不会在商定的资产规则之外铸造资产），那么发送链就真的没有依据在消息的背后铸造资产了。

让我们看看从 Polkadot 中继链上传送（大部分）1 DOT 到 Statemint 上的主权账户的 XCM 会是什么样子。我们将假设 Polkadot 方面已经支付了手续费。

```rust
WithdrawAsset((Here, 10_000_000_000).into()),
InitiateTeleport {
	assets: All.into(),
	dest: Parachain(1000).into(),
	xcm: Xcm(vec![
		BuyExecution {
			fees: (Parent, 10_000_000_000).into(),
			weight: 3_000_000,
		},
		DepositAsset {
			assets: All.into(),
			max_assets: 1,
			beneficiary: Parent>into(),
		}
	]),
}
```

正如你所看到的，这看起来与我们上次看到的直接`提款 - 购买 - 存款`模式相当相似。不同的是 `InitiateTeleport` 指令，它被插入到最后两条指令（`BuyExecution` 和 `DepositAsset`）的周围。在幕后，发送方（Polkadot Relay）链在执行 `InitiateTeleport` 指令时，正在创建一个全新的消息；它采用 xcm 字段，并将其置于一个新的 XCM，`ReceiveTeleportedAsset`，并将这个 XCM 发送给接收方（Statemint）链。Statemint 相信 Polkadot 中继链在发送消息之前已经销毁了它那边的 1 DOT。(它确实如此！）。

`beneficiary`(受益人) 被表述为 `Parent.into()`，精明的读者可能会想，在 Polkadot 中继链的上下文中，这可能指的是什么。答案是 "没有什么"，但这里没有错。xcm 参数中的所有内容都是从接收方的角度编写的，所以尽管这是整个 XCM 的一部分，被送入 Polkadot 中继链，但它实际上只在 Statemint 上执行，因此它是在 Statemint 的上下文中编写的。

当 Statemint 最终得到消息时，它看起来像这样：

```rust
ReceiveTeleportedAsset((Parent, 10_000_000_000).into()),
BuyExecution {
	fees: (Parent, 10_000_000_000).into(),
	weight: 3_000_000,
},
DepositAsset {
	assets: All.into(),
	max_assets: 1,
	beneficiary: Parent.into(),
}
```

你可能会注意到，这看起来与之前的 `WithdrawAsset` XCM 相当相似。唯一的区别是，不是通过从本地账户取款来资助费用和存款，而是通过相信 DOT 在发送方（Polkadot 中继链）被忠实地销毁，并遵守 `ReceiveTeleportedAsset` 消息而 "神奇" 地存在。

值得注意的是，我们在 Polkadot 中继链上发送的 1 个 DOT 的资产标识符（这里指的是中继链本身是 DOT 的原生家园）已经自动变异为 Statemint 上的表示。`Parent.into()`，这是 Statemint 上下文中的中继链的位置。

`beneficiary`(受益人) 也被指定为 Polkadot 中继链，因此它的主权账户（在 Statemint 上）被记入新造的 1 DOT 减去手续费。XCM 可能只是很容易地为受益人指定一个账户或其他地方。就像现在这样，后来从中继链发送的 `TransferAsset` 可以用来转移这 1 DOT。

##### 储备金

https://mp.weixin.qq.com/s/KR9-tETKZRjCurb0nmPgEQ


### 一些资料
- 深入了解波卡跨共识消息 XCM 1 https://mp.weixin.qq.com/s/dbgR9xJNNT1zyb_3iyDb3A
- 深入了解波卡跨共识消息 XCM 2 https://mp.weixin.qq.com/s/9_aAN2mlRo9yy10Y-RNbIQ
- 深入了解波卡跨共识消息 XCM 3 https://mp.weixin.qq.com/s/7GuBD_3b0Opxui3AKpWoGA
-  悄悄颠覆你跨链习惯的XCM是什么 https://mp.weixin.qq.com/s/ylZdlJp53BEl7lBmMHNdXg
- Gavin Wood: XCM 1 https://mp.weixin.qq.com/s/TLgXYLMedzpopvtlhFDWyA
- Gavin Wood: XCM 2  版本控制和兼容性 https://mp.weixin.qq.com/s/uDRKfVrx2E8-uiTpZJF9iQ
- Gavin Wood: XCM 3  执行和错误管理 https://mp.weixin.qq.com/s/VANLIM9WS6vzh3jcZL6rnQ
- 波卡XCM从基础到实践 https://mp.weixin.qq.com/s/vy0PpP8fN_wYhyifhuwHMw
- 波卡跨链转账教程 [Parachain Development · Polkadot Wiki](https://wiki.polkadot.network/docs/build-pdk#testing-a-parachain)
- 波卡XCM 官方 [Cross-Consensus Message Format (XCM) · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-crosschain)
