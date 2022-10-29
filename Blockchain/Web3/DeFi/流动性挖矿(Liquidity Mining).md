
流动性挖矿其实本质上就是向某个DeFi平台存钱，用户赚取收益。有时候流动性挖矿也叫收益耕作(Yield Farming)。

## 怎样设计一个简单的流动性挖矿

[How to: Develop A Liquidity Mine. Liquidity mining is one of he hottest… | by Buns | Medium (learn-solidity.com)](https://learn-solidity.com/how-to-develop-your-liquidity-mine-9d47656fe678)

流动性挖矿是去中心化金融中最热门的话题之一。流动性通常指的是生态系统中的资产。在治理协议的情况下，流动性也代表了你的那块治理蛋糕，可以这么说。在本教程中，我将回顾我是如何为 Governor DAO https://governordao.org/  加密货币项目构建流动性挖矿的 [GovTreasurer | Address 0x4dac3e07316d2a31baabb252d89663dee8f76f09 | Etherscan](https://etherscan.io/address/0x4dac3e07316d2a31baabb252d89663dee8f76f09#code) ，以及你可以如何复制。我设法将其缩小到任何人都可以遵循的3个简单步骤。

### Step 0 计划阶段

提供流动性挖矿的好处是，这放弃了对ICO的需求，即直接从项目中购买代币，这将受到监管部门的干预。最终，你应该有理由利用这种方法来分配代币，在你跳入智能合约本身的编码之前，你应该考虑以下问题。你可能有的一些考虑如下。

- 分发方法(Distribution Method)：考虑你是否要铸造代币，或者是否会有其他功能你利用来分发代币。
- 通货膨胀率(Inflation Rate)：你的代币将以何种速度被引入生态系统，并持续多久？
- 实用性(Utility)：虽然这在合约本身并不明显，但你应该问自己，你所创造的代币的目的是什么？

为了建立一个流动性的挖矿，你需要首先确定你的挖矿是否会铸造或转移代币作为对矿工的奖励。在 Governor DAO 的案例中，我们希望确保消除铸造额外代币的可能性，因此我必须想出一种方法，将代币从合约本身转移到 dApp 的用户。通常情况下，就像Sushi的情况一样，人们会铸造代币，因为这是代币提供给生态系统的方式。鉴于Governor DAO的历史根源是对CBDAO不幸的拉锯战的回应，我们想完全消除所有的疑问，这就是为什么我们决定从GDAO ERC20合约 [$0.19 | Governor (GDAO) Token Tracker | Etherscan](https://etherscan.io/token/0x515d7E9D75E2b76DB60F8a051Cd890eBa23286Bc?a=0xfd63bf84471bc55dd9a83fdfa293ccbd27e1f4c8#writeContract) 中取消铸币(mint)功能。让我们来了解一下我实现Governor DAO智能合约的步骤，它将为2020年12月22日推出的Governor Treasury Fund奠定基础。

### Step 1 导入依赖库

![[Pasted image 20221027092524.png]]

> 始终指定solidity的许可证和版本，以确保新版本不会妨碍性能。

IERC20.sol返回智能合约的关键方面，例如，我们使用transferFrom函数来转移（address sender，address recipient，uint256 amount）。SafeERC20.sol函数包裹了ERC20原生函数，在失败时（即当返回为假时）抛出。你的合约可以继承这个功能，只需包括这一行。"为IERC20使用SafeERC20"。EnumerableSet.sol使你能够存储唯一的值，这在solidity中是一个很方便的工具，你将在本文的后面看到。SafeMath.sol使Solidity能够在溢出的情况下抛出一个错误，消除了一整类的错误。Ownable.sol 是一个合约模块，它提供了一个基本的访问控制机制，所有者可以通过修改器 "onlyOwner "被授予对一套其他人无法访问的函数的访问权。

### Step 2 声明关键变量

![[Pasted image 20221027092824.png]]

> 这些变量将在后面具体说明，但请注意一些需要考虑的变量。

欢迎你声明与你的项目有关的多少个变量。在Governor DAO的案例中，我们利用Liquidity Mine作为分发Governor DAO治理代币的手段（也就是存钱，就获得该DAO的代币），同时也作为Governor DAO财政基金(treasury fund)的启动平台。因此，我们包括uint256 taxRate，以表示对资金池的征税率。请注意，由于Solidity的性质，我不得不对这个数字进行创造性的处理，所以它不像税率=0.02那么简单，因为Solidity不与小数合作。因此，taxRate = 50，我们只需将存入的价值除以50，就可以获得2%。这对任何整数n都是如此。

![[Pasted image 20221027093116.png]]


由于每一个流动性挖矿操作都是独一无二的，我省略了几行，因为它们包括声明无数的变量，这些变量可能与你的项目有关，也可能无关。完整的源代码将在文章末尾提供，但为了简单起见，本教程不涉及这些。

### Step 3 设计关键函数(功能)

![[Pasted image 20221027093228.png]]

是时候进入合约的功能了，它被存储在，你猜对了--你的函数中。让我们使用Governor DAO Liquidity Mining Smart Contract中的示例代码片段：GovTreasurer.sol，来探讨几个选项供你考虑。第175行声明了一个函数，并将其命名为 "存款(deposit)"，它存储了两个输入变量。矿池ID（Pool ID）和金额。Pool ID对应的是用户选择的矿池，而Amount对应的是用户选择存入的代币数量。不要被愚弄了，这个合约与你的典型存款功能不同，因为它的设计是为了获得2%的存款税，正如之前讨论的那样。因此，在第179行声明了一个新的变量：uint256 taxedAmount，它对应于被征税的金额，如taxedAmount=2%的存款金额。这个新变量在函数中被使用了4次，从存入智能合约的金额中扣除（第189行），作为发送到Governor DAO Treasury的金额（第190行），作为用户提款(withdraw)时被欠的金额的更新（第191行），以及作为从存款事件中发出的金额的扣除（第195行）。总的来说，这个函数可以达到预期的效果。发挥创意，设计一个你自己的函数吧。  

### Governor流动性挖矿源码

这个流动性挖矿的合约在地址0x4DaC3e07316D2A31baABb252D89663deE8F76f09上可以从Etherscan上看到 [GovTreasurer | Address 0x4dac3e07316d2a31baabb252d89663dee8f76f09 | Etherscan](https://etherscan.io/address/0x4dac3e07316d2a31baabb252d89663dee8f76f09#code) 

关于更多的Governor合约的地址 [Blockchain Biometric Authentication & Defi Crypto Governance (governordao.org)](https://governordao.org/contracts)  https://etherscan.io/address/0x5ab8e3a7bc8be9efdd0943ab65221bdf240518c3#code [GovernorDAO: GDAO Token | Address 0x515d7E9D75E2b76DB60F8a051Cd890eBa23286Bc | Etherscan](https://etherscan.io/address/0x515d7E9D75E2b76DB60F8a051Cd890eBa23286Bc#code) 

还有空投的合约地址 [MerkleDistributor | Address 0x7ea0f8bb2f01c197985c285e193dd5b8a69836c0 | Etherscan](https://etherscan.io/address/0x7ea0f8bb2f01c197985c285e193dd5b8a69836c0#code) 

## Uniswap的流动性挖矿

### 介绍

作为DeFi项目、代币创造者或其他相关方，你可能想激励Uniswap V3池的范围内流动性供应。本指南高度描述了uniswap-v3-staker中实现的一个特殊的激励方案。

### 设置

让我们先来定义一些术语。我们把激励流动性的项目称为 "激励"；它们的特点是有以下参数。

- rewardToken。也许是最重要的参数，潜在的激励者必须选择他们想分配的ERC20代币作为提供流动性的奖励。
- pool。必须提供流动性的Uniswap V3 pool的地址。
- startTime：开始分发奖励的UNIX时间戳。
- endTime: 奖励开始衰减的UNIX时间戳。
- refundee。有权在Incentive结束后收回任何剩余奖励的地址。

最后，每个Incentive都有一个相关的reward，即在程序的生命周期内分配的奖励代币的总量。

### Reward Math


### 参考

[Uniswap-V3的流动性挖矿原理介绍 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/541010213)