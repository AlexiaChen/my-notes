
#### 什么是ibTokens？  


计息代币是指余额因计息而随时间变化的代币，其汇率与基础资产挂钩。  
利息代币表示为借贷金库中的资产份额。  
  
当提供流动性时，例如BNB或BUSD，用户会收到ib（计息）代币--ibBNB或ibBUSD。ib是一种特殊的代币，可以累积利息。也就是说，随着存款资金的出借，百分比的累积，增加了ib代币的价值。估计ib代币的成本增长只是预测性的landing 的年百分比收益率(APY, Annual Percentage Yield)。  
  
比如说: 
  
在一个借贷协议中，在借出USDC这样的代币后，你会收到等量的intUSDC来代表你的借贷位置。随着时间的推移，intUSDC的数量会增加，并且可以兑换等量的USDC。  
  
#### ib-tokens如何赚取利息？  

- 每个存款库(deposit vault)都会赚取利息。但是，利息并不分配。相反，仅仅通过持有ibTokens，你就可以赚取利息。  
- ibToken通过他们的汇率来积累利息；随着时间的推移，每个ibToken的价值增加，每一个区块都可以转换为更多的相关资产，即使你钱包里的ibToken数量保持不变。  
- 每个存款池（deposit pool）都有自己的利用率，也会反映出相应的ibTokens随着时间的推移会升值多少。  
- 用户持有ibTokens的时间越长，这些代币的价值升值就越高。这就是利息的积累。  
- 你不需要用ibTokens做stake来享受这种价格升值，尽管在你的代币被stake的时候，你仍然会赚取它。  (也就是还可以用ibTokens来做stake，赚取更多的利息)


####  我需要计算ibTokens的汇率吗

在登陆池首次推出时，ib-Token汇率（即一个ibBNB值多少BNB）开始为1：1。虽然自推出杠杆收益农庄以来，它一直以等于复利市场利率的速度持续增加。这代表了借贷费用对贷款人代币的累积。   ^43f258
  
例如，如果一年的借贷年利率平均为50%，ibToken在年底的价值将是~1.5，就是1个ibBNB等于1.5个BNB，1:1.5  
  
每个用户都有相同的ibToken汇率。  
  
#### 例子

假设你在我们的金库中存入1000个BNB，当每个ibBNB价值1.05个BNB时，你会收到952.38个ibBNB (1000 / 1.05)  
  
几个月后，你决定是时候从金库中提取你的ibBNB了，此时汇率为1.10。以下情况将发生。  
  
你的952.38 ibBNB现在等于1,047.618 BNB (952.38 * 1.10)  
你可以提取1,047.618 BNB，这将兑换所有952.38 ibBNB  
或者，你可以提取一部分，比如你原来的1,000 BNB，这将兑换909.09 ibBNB（在你的钱包里保留43.29 ibBNB）。  ibTokens的实现与Compound协议的cToken类似。  
  

我看上面的例子与Compound的cTokens的实现和文档，例子都几乎一模一样，所以直接学习cTokens的文档 [[Compound的cTokens]]


# 一个使用ib-Tokens的DeFi项目

[Introduction - Abracadabra](https://docs.abracadabra.money/) 

Abracadabra.money是一个借贷平台，使用有利息的代币（ibTKNs）作为抵押，借入一个与美元挂钩的稳定币（Magic Internet Money - MIM），可以像其他传统稳定币一样使用。
目前，很多资产，如yVaults已经锁定了资本，无法进一步投入使用。Abracadabra提供了一个使用它的机会。

## 借贷

这个项目是使用Kashi借贷技术提供独立的借贷市场，允许用户根据他们决定使用的抵押品调整其风险承受能力。要阅读更多关于隔离式借贷市场和kashi的工作原理，以及为什么它们与传统借贷协议不同，请参考 [Kashi Lending | SushiSwap](https://docs.sushi.com/docs/Products/Kashi%20Lending)  [[Kashi借贷]]

### 存放抵押品和借出MIM稳定币

https://docs.abracadabra.money/intro/lending-markets#depositing-collateral-and-borrowing-mim

定位到这个项目的Borrow界面。
![[Pasted image 20221019104148.png]]

一旦进入借款页面，你将能够选择你能够借贷的抵押品(collateral assets)，只需点击突出显示的框架即可。

![[Pasted image 20221019104307.png]]

一旦选择了它，你将能够看到你的钱包和我们的市场中所有不同的代币。这些在同一个市场中的不同代币简称大熔炉(Cauldron)

![[Pasted image 20221019104541.png]]

点击抵押品将自动选择适当的市场（这个抵押品，好像是Cauldron里面的其中一个代币）, 可以看到下图右侧出现了新的Frame。

![[Pasted image 20221019104751.png]]

在这个例子中，我们将使用xSUSHI代币作为抵押品(collateral)。

首先，输入决定有多少xSUSHI将被用作抵押品。然后通过输入数字或使用下面的百分比按钮，决定将借入多少MIM。这里相当于是用xSUSHI来借钱，这个钱是MIM稳定币。

![[Pasted image 20221019105136.png]]

一旦你选择了你的参数，在交易确认后/如果交易确认后， 右边的方框将显示你的posistions是什么样子。

- 存放的抵押品(Collateral Deposited)。投入position的抵押品数量, 这里就是xSUSHI抵押进去的数量
- 抵押品价值(Collateral Value)。存入的抵押品的当前价值。单位美元
- 借入的MIM。开仓(opened position)后将借入的MIM总金额。
- 清算价格（Liquidation Price）。清算价格将显示我们可以被标记为清算的抵押品价格。

> 屏幕左下角的小方框显示用户钱包中的可用代币数量。

上面可以理解为，用xSUSHI这个ib-token作为抵押，借入MIM。

### 清算价格

0.71美元的清算价格意味着，如果你的计息代币跌至或低于0.71美元( <= 0.71美元)，你的position就有被清算的风险。显示的清算价格总是与用于借入MIM的ib-token相关，根据上一节，这里的ib-token就是xSUSHI。你可以在本维基的专门章节中阅读更多关于清算的信息。

### 市场参数

![[Pasted image 20221019110304.png]]

这两个按钮下面的方框描述了借贷市场参数。请注意，这个参数中的每一个都是针对你所选择的抵押品的的参数。这里是xSUSHI作为抵押。

- 最大抵押品比率 - 最大抵押品比率表示用户用选定的抵押品代币可以借到的最大债务数额。
- 清算费 - 这是清算人在购买标记为清算的抵押品时得到的折扣。
- 借款费 - 这个费用在你每次借入MIM时都会加到你的债务上。例如，如果你借了1000MIMs，你的债务将增加到1000.5MIMs，但你将实际收到1000MIM。这0.5个MIM将被分配给sSPELL的持有人。
- 利息(Interest) - 这是你的债务每年将增加的年化百分比。该利息费用随后将分配给sSPELL持有人。
- 价格 - 所选抵押品的当前价格。也就是一个xSUSHI代币等于1.93美元。

https://docs.abracadabra.money/intro/lending-markets#market-parameters

### 打开一个position

一旦你决定了你要使用的价值，只需点击清算价格下面的两个按钮（Approve和Add collateral and borrow），就可以开仓（open your position）了!

请注意，如果这是你第一次使用Abracadabra，在你加入position之前可能需要几个批准(approvals)

这将移除您钱包中的抵押品并将其放入合约中，并将相应的借入的MIM稳定代币放入我们的钱包。
一旦交易被确认，用户将收到一个弹出窗口，将他重定向到 "positions "页面，在那里他可以编辑、偿还或关闭他的position。

> 所以这个我感觉的position其实代表你某一次的借贷行为的通道，每次借贷，你的position都不一样。所以有打开和关闭position。

### 关闭一个position

为了关闭你的position，请前往position页面，并点击偿还(repay)。

## 杠杆(Leverage)

使用Kashi作为我们的借贷引擎的一个特点是，它允许用户利用他们的生息代币positions。在Abracadabra，我们已经开发了一个一键式UI，允许你自动这样做。让我们深入了解一下吧! 

杠杆的工作流程

### 在Abracadabra上Leveraging一个position

#### 交易

### 去杠杠化(Deleverage)Positions


### 在Abracadabra上去杠杠化(Deleverage)Positions


## Positions

### 使用Positions页面

![[Pasted image 20221019112006.png]]

这个页面主要是管理借贷通道的页面。

### 选择链

选择链允许用户改变他们在Abracadabra上使用的链。请注意，如果你使用Fantom，以太坊上的位置将不会弹出，反之亦然。相当于是选择在哪个链上的这个Abracadabra应用。因为DeFi可以跑在不同的链上。

### MIM在Bentobox/Degenbox上的余额

这个Frame描述了用户在Bentobox上拥有的MIM数量。请记住，由于Abracadabra使用的是Kashi技术 [[Kashi借贷]]  ，它与Bentobox互动。在这里阅读更多关于什么是Bentobox的信息。[sushiswap/bentobox (github.com)](https://github.com/sushiswap/bentobox) bentobox是类似Compound那样在sushi上的借贷平台。

![[Pasted image 20221019112628.png]]

在对您的position进行去杠杆化后，请考虑在此查看您的余额，一些MIM可能会被发送到您的Bentobox余额中，您可以通过点击WITHDRAW按钮进行提款。

### 总结

这个Frame为用户提供了一个关于他们当前位置总数的快速摘要。

![[Pasted image 20221019113043.png]]

- 借入的MIM--显示借用的MIM代币的总金额。
- 抵押存款 - 显示用户提供给所有positions的所有ib-tokens的美元价值。
- 已耕作的SPELL - 在我们的农场中中耕作的SPELL数量。

### 打开Positions

页面底部将显示用户的所有的positions摘要以及他们目前的健康状况。（健康状况就是指，离清算远不远）

![[Pasted image 20221019142607.png]]

通过点击屏幕上方的小 "+"，你会看到更多关于你的positions的信息。

点击偿还(repay)MIMs或去杠杆化，你将能够关闭你的position。
如果你想了解更多关于如何关闭或闪电式偿还你的position，请到借贷和杠杆页面。[[#借贷]] [[#去杠杠化(Deleverage)Positions]]

### Farming(耕作)

在页面的末尾，用户还可以看到他们是否有任何耕作positions 开放。他们可以收获奖励，检查金额或提取LP代币。

![[Pasted image 20221019143025.png]]

> 看上图，感觉SPELL是LP代币？


## Stake(抵押)

### 费用分担(Fee sharing)

stake页面允许用户参与Abracadabra的费用分担机制!
我们目前支持两种押注方式。

- sSPELL - 质押SPELL代币，赚取更多的SPELL。
- mSPELL - 质押SPELL代币并通过MIM赚取稳定币收入

### mSPELL

在AIP#8通过后，mSPELL的stake允许用户将他们的SPELL代币作为stake，并从协议收入中赚取稳定币MIM收入！mSPELL的stake已被实施，该协议可以找到 [AIP #8 - An update on Abracadabra fee distribution, introducing mSpell staking pools - Governance Proposals / Voted Proposals - Abracadabra Forum](https://forum.abracadabra.money/t/aip-8-an-update-on-abracadabra-fee-distribution-introducing-mspell-staking-pools/3218) 

#### 它是怎样工作的

- 用户能够选择他们最喜欢的stake方法。
- 对于mSPELL，来自借贷市场的费用留在MIM中，并按比例在不同的赌池中分享。换句话说，MIM将按比例分配给池中的SPELL赌注金额（与sSPELL池的过程相同）。
- mSPELL的staker将能够在任何时候用claim按钮取他们的MIM。

在Avalanche、Arbitrum、Ethereum和Fantom Opera上都可以使用mSPELL staking!

#### 怎样使用mSPELL

如下图所示，用户可以选择 "stake "或 "unstake"。要在这两个frame之间切换，只需点击中间的两个箭头。

![[Pasted image 20221019144030.png]]

请注意，所有的mSPELL代币都有一个24小时的锁定期。换句话说，用户每次存入SPELL代币，在接下来的24小时内，他将不能解除锁定。

同样，每次用户存入SPELL，他将不能在24小时内claim。

##### Claiming MIM

要领取待领的MIM奖励，只需点击屏幕右下角的CLAIM按钮，在你的钱包中确认交易，并等待网络的确认！在这之后，你将正确地收获你待领的MIM奖励。在这之后，你将会正确地收获你的待领MIM奖励!

##### 分析

在屏幕的右侧，用户将能够检查他们的钱包里有多少SPELL代币，在mSPELL staking 合约里有多少被赌注，以及这些代币的美元价值。在它的下面，用户将能够找到给出的近似赌注年利率，它是由协议的上个月的年化收入计算出来的。

> 不要直接向合约发送代币，它们会丢失。

### sSPELL

sSPELL表示staked SPELL代币。通过点击屏幕上方的STAKE按钮，用户将能够进入SPELL Staking页面。这个页面分为三个frame。

![[Pasted image 20221019144830.png]]

#### stake/unstake Frame

如上图所示，用户将能够选择 "stake "或 "unstake"。当用户点击这两个按钮时，他们会看到一个弹出的窗口，允许他们在自己的position上增加SPELL代币或从自己的position上withdraw SPELL代币。要在这两个frame之间切换，只需点击中间的两个箭头。

请注意，所有sSPELL代币都有24小时的锁定期。换句话说，每次用户存入SPELL代币，在接下来的24小时内，他将无法解除锁定。

#### Ratio Frame

在屏幕的右上角，用户将能够看到sSPELL-SPELL的比例。随着越来越多的费用被添加，1个sSPELL代币的价值将越来越高。

请记住，这就是押注SPELL的奖励分配方式：以SPELL代币为单位，增加sSPELL代币的价值!

#### Staking参数页面

在这里，用户将能够检查他们的钱包里有多少SPELL或sSPELL代币，以及这些代币的美元价值。在它下面，用户将能够找到给出的近似赌注年利率，它是由协议的上个月年化收入计算出来的。

## 工具

### Markets

网站的市场部分显示了各种可以作为抵押品的ib-token。下面是一些例子。

![[Pasted image 20221019145437.png]]

> 所以SPELL也是ib-token？  所以我可不可以认为SPELL其实就是这个项目的LP代币，这个LP代币是计息代币。SHIB也是这个平台下的ib-token？

#### 借贷(borrow)

在顶部的对面，你会看到。
- 组件 - 这些是你可以用来作为抵押品借入MIM的组件。
- 已借入的MIM总量 - 这显示了使用该市场借入的MIM总量。
- 剩余可借 -这显示了还有多少MIM可以使用该组件进行借贷。一旦这个数字变成0，就不能开出杠杆position或新的贷款。为了借入更多的MIM，用户需要等待该特定市场的补货!
- 利息 - 这是你的债务每年增长的年化百分比。
- 清算费 -这是清算人在清算被标记为清算的position时将获得的折扣。

要借用一些MIM，只需点击你想用作抵押品的组件框架中的任何地方。

> 所以包括SPELL，SHIB在内的代币，都可以抵押借入MIM，借入的都是MIM?

#### Farm

在这里，你会看到在Abracadabra上可以找到的所有耕作机会。Abracadabra提供不同的耕作机会，用户可以用他们的LP代币stake来耕作SPELL代币。这种机制是用来保持特定货币对的深度流动性的。耕作目前在以太坊上可用于ETH-SPELL和MIM-3CRV LP代币。未来的LP对(pairs)的激励计划可以在治理门户上线后通过治理程序进行投票。ETH-SPELL就是一种LP代币

### Farms

#### 使用LP代币来耕作

网站的FARM页面允许其用户添加LP代币和耕作SPELL。
我们可以用ETH-SPELL LP代币作为例子，但这个过程对每一种资产都是一样的。

我们将需要在下拉菜单中选择ETH-SPELL POOL。

![[Pasted image 20221019150520.png]]

首先，我们需要apporve我们的ETH-SPELL LP代币的消费。
完成后，我们只需按下STAKE，将我们的ETH-SPELL LP代币存入农场，开始赚取SPELL。
我们可以确认我们的存款是成功的，方法是进入positions页面并向最后滚动。我们也可以在EARNED中看到我们赚取的SPELL代币，随着时间的推移，我们继续施展这个仪式的法术。
现在我们还可以通过harvest和withdraw按钮，分别收获奖励或撤回押注的代币和奖励。

![[Pasted image 20221019150659.png]]

### 多链桥的功能



### Swap



## 清算(liquidation)

在Abracadabra.money中，每个MIM都有一定的生息代币（ib-token）作为支持。与大多数协议不同的是，所有用户的抵押品都有清算事件的风险，在Abracadabra这里，每个抵押债务position（CDP）都是孤立的，只有在它自己单独清算的风险。（所以用了kashi的借贷技术？）

澄清一下，如果一个用户用不同的ib-token开了两个CDP，他们能够单独借用MIM与这些ib-token，并相应地设置他们的风险容忍度。因此，如果他们认为某个ib-token的价值下降的可能性较大，他们可以选择相对于该ib-token借入较少的MIM。

要阅读更多关于隔离式借贷市场和Kashi的运作方式，以及为什么它们与传统借贷协议不同，请参考这里。[[Kashi借贷]]

也就是说，仍有一些时候，用户的ib-token价值会减少，并被标记为清算。在这种情况下，第三方玩家（通常是机器人）可以选择偿还所有的MIM债务，以换取用于该特定CDP的ib-token抵押品。

这个平台下有SEPLL， SHIB，FTT等等都是ib-token。

### 一个例子

巫师梅林有一些yvWETH（一种ib-token）。这种抵押品目前每块代币的价格是2000.00美元。梅林是个疯狂的巫师，他决定借入最大限度的MIM代币，用来为他的魔法书购买墨水。他得到的清算价格是1696.69美元。（记住，每一种ib-token都有MIM支持，每一个MIM，都有一定的ib-token支持，抵押ib-token借入MIM）

黑暗中的佐尔达克，有一个傀儡被设置为监视这个抵押借贷position(CDP)，并准备在任何时候都进行清算。

天有不测风云，yvWETH的价格跌到了1696.69美元，梅林的抵押品不再有足够的价值来支付他的债务。所以，佐尔塔克的巨魔可以在这个CDP上工作并清算它。佐尔塔克付清了所欠的MIM，并将那些神奇的yvWETH代币收归己有。然而，梅林并不感到非常失望，因为他还有那些MIM代币，好在他没有花在魔法墨水上的那些代币，而且他不再需要偿还债务。

### 清算费用

每个市场都有自己的清算费用，但一般来说，有基础稳定币的ibTKNS将有3%的清算费用(基础稳定币抵押，兑换出来的ib-token)，有价格行为的ibTKNS将有12.5%的清算费用。

清算费用分享 - 这个费用是给执行清算的各方的奖励。此外，这些收取的费用的10%被硬编码，留作每周对特定池子的sSPELL奖励。

从用户的角度来看，在为各自的ibTKNs报出清算价格时，这笔费用已经在计算中被考虑了。当用户的清算价格达到时，他们会被清算。

### 清算价格

清算价格是你的ib-token这种抵押品的价格，在这个价格上你将被清算。如果你的抵押品价值下降到清算价格与作为抵押品的代币价格一致，你的position将被标记为清算。该合约不允许清算者进行高于清算价格的清算，这意味着用户的抵押品在规定的总清算价格之前是安全的。