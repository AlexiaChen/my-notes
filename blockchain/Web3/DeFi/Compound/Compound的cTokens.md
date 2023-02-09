主要是翻译官方文档了，没有翻译完成  [Compound v2 Docs | cTokens](https://docs.compound.finance/v2/ctokens/#ctokens)，， 也不打算翻译完成，因为随时会变

## 介绍

Compound协议所支持的每种资产都是通过cToken合约整合的，cToken是提供给协议的余额的EIP-20兼容表示（ERC-20就是EIP-20建议的）。通过铸造(mint)cToken，用户（1）通过cToken的汇率赚取利息，相对于基础资产而言，cToken的价值会增加；（2）获得使用cToken作为抵押的能力。

cToken是与Compound协议互动的主要手段；当用户铸造、赎回、借贷、偿还借款、清算借款或转移cToken时，她将使用cToken合约进行操作。

目前有两种类型的cTokens。CErc20和CEther。虽然这两种类型都暴露了EIP-20接口，但CErc20包装了一个底层的ERC-20资产，而CEther只是包装了以太币本身。因此，涉及到将资产转移到协议中的核心功能，根据不同的类型有稍微不同的接口，每一种都显示在下面。

#### cTokens是怎样赚取利息的

每个市场都有自己的供应利率（APR）。利息是不分配的；相反，仅仅通过持有cToken，你就可以获得利息。

cTokens通过其汇率来积累利息--随着时间的推移，每个cTokens可以转换为越来越多的相关资产，即使你钱包里的cTokens数量保持不变。

#### 我需要计算cToken的汇率吗

当一个市场启动时，cToken的汇率（一个cETH值多少ETH）从0.020000开始 ( 这里与ib-token[[计息代币(ib-token, interest-bearing token)#^43f258]]  提到的不一样)--并以相当于复利市场利率的速度增加。例如，一年后，汇率可能等于0.021591。

每个用户都有相同的cToken汇率；你的钱包没有任何独特之处，你必须担心。

#### 一个例子

假设你向Compound协议提供了1000个DAI，当时的汇率是0.020070；你会收到49,825.61个cDAI（1000/0.020070）。

几个月后，你决定是时候从协议中撤回(Withdraw)你的DAI；现在的汇率是0.021591。

- 你的49,825.61cDAI现在等于1,075.78DAI (49,825.61 * 0.021591)
- 你可以提取1,075.78DAI，这将赎回所有49,825.61cDAI
- 或者，你可以提取一部分，比如你原来的1,000DAI，这将赎回46,315.59cDAI（在你的钱包里保留3,510.01cDAI）。

#### 我怎么查看我的cTokens

每个cToken在Etherscan [Just a moment... (etherscan.io)](https://etherscan.io/tokens/label/compound) 上都是可见的，您应该能够在与您的地址相关的代币列表中查看它们。

cToken余额已被整合到Coinbase钱包和MetaMask；其他钱包可能会增加对cToken的支持

#### 我可以转移我的cTokens吗

是的，但要谨慎! 通过转移cToken，你正在转移你在Compound协议内的相关资产的余额。如果你发送一个cToken给你的朋友，你的余额（可在Compound接口中查看 [Compound | Dashboard](https://app.compound.finance/) ，其实就是一些钱包）将减少，而你的朋友将看到他们的余额增加。

如果账户已经进入该cToken市场 [Compound v2 Docs | Comptroller](https://docs.compound.finance/v2/comptroller/#enter-markets) ，并且转移会使账户进入负流动性状态 [Compound v2 Docs | Comptroller](https://docs.compound.finance/v2/comptroller/#account-liquidity) ，那么cToken转移将失败。

## 铸造Mint

铸币功能将资产转移到Compound协议中，开始根据该资产的当前供应率积累利息。用户收到的cTokens数量等于所提供的基础代币，除以当前汇率。
![[0bbff85efa1bcaa6ef2cef4e5f68a7b.png]]

## 赎回(Redeem)

赎回函数将指定数量的cTokens转换为基础资产，并将其返回给用户。收到的基础代币的数量等于赎回的cTokens数量，乘以当前汇率 [Compound v2 Docs | cTokens](https://docs.compound.finance/v2/ctokens/#exchange-rate) 。赎回的数量必须小于用户的账户流动资金 [Compound v2 Docs | Comptroller](https://docs.compound.finance/v2/comptroller/#account-liquidity) 和市场的可用流动资金。

![[3cc3c03c6848f6e2272b424c17709a1.png]]

## 赎回的底层(underlying)

赎回underlying函数将cTokens转换为指定数量的基础资产，并将其返回给用户。赎回的cTokens数量等于收到的基础代币数量，除以当前汇率。赎回的数量必须小于用户的账户流动资金和市场的可用流动资金。

![[2dfbd4b79efba2a223361bd2cff54c6.png]]

## 借款(borrow)

借款功能将一项资产从协议中转移给用户，并创建一个借款余额，开始根据该资产的借款利率累积利息。借款金额必须小于用户的账户流动资金和市场的可用流动资金。要借入以太币，借款人必须是 "可支付的"（solidity）。

![[ff8fd587c2de782a5de23c5bc25cf84.png]]

## 偿还借款(repay borrow)

偿还功能将一项资产转移到协议中，减少用户的借款余额。

![[4b08dc00a5b0dbb95647d5ede2da4c3.png]]

更多请参考 [Compound v2 Docs | cTokens](https://docs.compound.finance/v2/ctokens/) 