
## 正文

你一定听说过 Uniswap，甚至可能对它有更深层次的了解。

  
LP、恒定乘积、去中心化、区块链、自动做市商——没有订单簿、不依赖服务器、没有价格预言，这一切仅仅依靠一个公式 x∗y=k 。**X * Y = K 是 Unicorn 的魔法！**

  
当你看完这篇文章后，**会有一只 Unicorn 跳出屏幕来给你一拳**！

好吧，让我们快来通过代码来看看它到底是怎么工作的！ 这篇文章给出的代码可以在[这里]([monokh/looneyswap: A rudimentary implementation of uniswap for educational purposes. You'd be crazy to actually use this. (github.com)](https://github.com/monokh/looneyswap))找到。

### 交易池

Uniswap的原理要从交易池开始说起。这个交易池中存在着两个智能合约地址（即两个erc-20代币）token0 与 token1。

```d
contract LooneySwapPool is ERC20 { 
  address public token0; 
  address public token1; 
 
 // Reserve of token 0 
  uint public reserve0; 
 
 // Reserve of token 1 
  uint public reserve1; 
  ... 
} 
```

> _译者注：以太坊上普遍使用的智能合约语言为 solidity，它在语法上类似 javascript。在 solidity 中，有一个其他编程语言没有的类“address”，它既可以表示外部用户地址、也可以表示智能合约地址。在本段代码中它表示后者。_

  

构造这样一个交易池很简单，只要在构造函数中声明并保存这两个地址：

```d
constructor(address _token0, address _token1) ERC20("LiquidityProvider", "LP") {
  token0 = _token0;
  token1 = _token1;
}
```

合约创建后，交易池会将 token0 和 token1 两个代币的数量存储在 reserve0 和 reserve1 中。

注意，这个合约本身也会产生一个 erc-20 代币，它依据用户提供流动性的多少来分配，是流动性提供者为交易池做贡献的证明。我们称之为 LP（Liquidity Provider）代币。

这里的初始化构造函数对应的uniswap v2的代码是在UniswapV2Pair中，这个Pair其实就是一个交易对的交易池子， https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L66-L70 其对应的LP代币就是 UniswapV2ERC20这个LP代币 https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L3-L11 

### 添加流动性

作为用户，为了给我们的池子增加流动性，需要调用 add 指定我们存入代币的数量：

```d
function add(uint amount0, uint amount1)
```

首先，用户将代币转移至池中。

```d
assert(IERC20(token0).transferFrom(msg.sender, address(this), amount0));
assert(IERC20(token1).transferFrom(msg.sender, address(this), amount1));
```

由于用户不再控制这些代币，我们需要一种方法来为他们提供一个“代表他们为池子流动性做出的贡献”的对象。这就是该合约原生代币的用武之地！

我们根据用户在池中提供的流动性份额（即充值的 token1, token2 的数量）按比例铸造该合约的代币（即刚才提到的 LP 代币）。

如果这是第一个提供流动性的用户，我们会铸造初始金额的 LP 代币：

```d
_mint(msg.sender, INITIAL_SUPPLY);
```

否则，我们将按比例计算份额并铸造等比例的 LP 代币：

```d
uint reserve0After = reserve0 + amount0;
uint reserve1After = reserve1 + amount1;

...

uint currentSupply = totalSupply(); // Current supply of LP tokens
uint newSupplyGivenReserve0Ratio = reserve0After * currentSupply / reserve0;
uint newSupplyGivenReserve1Ratio = reserve1After * currentSupply / reserve1;
uint newSupply = Math.min(newSupplyGivenReserve0Ratio, newSupplyGivenReserve1Ratio);
_mint(msg.sender, newSupply - currentSupply);
```

太好了！我们的池子现在公平地给每一个流动性提供者都分配了代币。在池子具有手续费收益时，我们可以知道这些收益应该按什么比例分配给流动性提供者。

以上添加流动性的代码对应UniswapV2的Pair中的mint方法 https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L110-L131 ，当然，与这个教程已经很不一样了。

是时候开始 Swap 了！

### 魔法公式 X\*Y = K

在我们执行交易之前，需要得出市场价格。在传统交易所中，买入或卖出现货需要明确设定一个价格和规模，例如：_在 \$2210/ETH 卖出 2.4 ETH_，然后等待有意交易的用户吃单。但在区块链的背景下，它带来了许多问题，订单簿更新过于频繁会使得整个系统效率低下、无法负载。

Uniswap 使用公式$x∗y=k$ 计算价格，而不是明确定价。它看起来很抽象，但我们可以通过例子看到它的方便之处——

-   式中 x 为 reserve0 （池中 token0 的数量）
-   式中 y 为 reserve1
-   k = reserve0 * reserve1，在定价时 k 必须保持不变。这意味着，如果你减少了池中的 reserve0，你就必须增加池中的 reserve1。

用代码编写如下：

```d
function getAmountOut (uint amountIn, address fromToken) {
  ...
  uint k = reserve0 * reserve1;
  ...
  newReserve0 = amountIn + reserve0;
  newReserve1 = k / newReserve0;
  amountOut = reserve1 - newReserve1;
}
```

让我们再来看一个具体的例子。假设我们已经向池中添加流动性，使得池子里有 10 ETH (reserve0) + 20000 DAI (reserve1)。我们想出售 1 ETH 来获得 DAI。

首先让我们计算 k 的值：

```d
k = x(reserve0) * y(reserve1)
k = 10 * 20000
k = 200000
```

要向池子中出售 1 ETH，我们会使 reserve0 加1，然后计算 reserve1。

```d
newReserve0 = 10 + 1 = 11
newReserve0 * newReserve1 = knewReserve1 = k / newReserve0
newReserve1 = 200000 / 11
newReserve1 = 18182
```

收到的 DAI 数量是 reserve1 的变化量。

```d
amountOut = reserve1 - newReserve1
amountOut = 20000 - 18182
amountOut = 1818
```

就这样，价格就被简洁地确认下来啦！

![[Pasted image 20221101091647.png]]

> Uniswap中，Token A 和 Token B 的一次交易

### 交易

得知了如何计算交换出代币的数量，下面让我们执行它吧！

```d
function swap(uint amountIn, uint minAmountOut, address fromToken, address toToken, address to)
```

首先，用上一节的函数计算应该得到的代币数量，以及池子中两种代币的更新后的数量：

```d
(uint amountOut, uint newReserve0, uint newReserve1) = getAmountOut(amountIn, fromToken);
```

但是，amountOut 还面临另一个问题：由于你的交易可能需要一些时间，并且其他用户也在与这个程序进行交互，因此你最终可能会得到与预期价格大不相同的价格。因此，我们还设置了一个限制，使得用户认为“不合理的交易”不会被执行：

```d
require(amountOut >= minAmountOut, 'Slipped... on a banana');
```

“实际交易价格与目标交易价格之差”称为滑点。更多关于滑点的资料可以在[这里]([Swaps | Uniswap](https://docs.uniswap.org/protocol/concepts/V3-overview/swaps#slippage))阅读。

现在，我们需要对两个代币进行分别转账，以完成交易。

```d
assert(IERC20(fromToken).transferFrom(msg.sender, address(this), amountIn));
assert(IERC20(toToken).transfer(to, amountOut));

reserve0 = newReserve0;
reserve1 = newReserve1;
```

对应UniswapV2的Pair中的swap方法  https://github.com/Uniswap/v2-core/blob/ee547b17853e71ed4e0101ccfd52e70d5acded58/contracts/UniswapV2Pair.sol#L159-L187 

### 去除流动性

如果某位用户不再希望为池子提供流动性，他可以收回他的代币。

```d
function remove(uint liquidity)
```

首先，将代表他流动性份额的 LP 代币收回：

```d
assert(transfer(address(this), liquidity));
```

计算这部分 LP 代币在池子中代表的 token0 和 token1 的数量：

```d
uint currentSupply = totalSupply(); 
uint amount0 = liquidity * reserve0 / currentSupply; 
uint amount1 = liquidity * reserve1 / currentSupply;
```

然后销毁该用户的LP代币：

```d
_burn(address(this), liquidity); 
```

并将 token0 和 token1 归还给该用户：

```d
assert(IERC20(token0).transfer(msg.sender, amount0)); 
assert(IERC20(token1).transfer(msg.sender, amount1)); 
reserve0 = reserve0 - amount0; 
reserve1 = reserve1 - amount1; 
```

### 结束语

我们用100行代码，成功演示了这个简易版本的 Uniswap！如果想更深入地学习，可以在阅读[Uniswap-v2的源码](https://github.com/Uniswap/uniswap-v2-core/tree/master/contracts)。

像我们开头所提到的——读到这里，一只 Unicorn 跳出屏幕给了你一拳！！


### 进阶阅读

因本教程是学习性质的，只学习uniswap的最核心的本质，对于实际的uniswapV2还是有很多地方(架构，算法)不一样了。所以直接推荐看uniswapV2的源码分析 这个系列文章，[UniSwap 学习笔记1: 概览 以及 交易对地址计算 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/article/3915) 

[UniswapV2源码分析 - chuwt's blog](https://chuwt.github.io/post/uniswapv2%E6%BA%90%E7%A0%81%E5%88%86%E6%9E%90/) 

当然，为了学习UniswapV2，可以试着部署uniswapV2到测试网，其前端代码也可以部署到github.io上也有一些原理分析，也有一些文章参考

- [使用Uniswap V2部署自己的去中心化交易所 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/article/4925)
- [Uniswap V2部署 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/article/3285)
- [Uniswap V2 可交互演示 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/article/3017)
- [Uniswap V2 设计迷思 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/article/3004)
- [10分钟快速部署 Uniswap-v2 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/article/3515)
- [手把手教你部署自己的uniswap交易所 | 登链社区 | 区块链技术社区 (learnblockchain.cn)](https://learnblockchain.cn/article/1383)
- [去中心化交易所 - YouTube](https://www.youtube.com/playlist?list=PLV16oVzL15MRR_Fnxe7EFYc3MAykL-ccv) 
- [区块链 - 在以太坊测试网络部署 uniswap v2 去中心化交易所_个人文章 - SegmentFault 思否](https://segmentfault.com/a/1190000040401731) 

## 参考

- [从零开始，带你看懂 Uniswap 代码 - 知乎 (zhihu.com)](https://zhuanlan.zhihu.com/p/401811923)
- [monokh/looneyswap: A rudimentary implementation of uniswap for educational purposes. You'd be crazy to actually use this. (github.com)](https://github.com/monokh/looneyswap)
- [Uniswap from Scratch (monokh.com)](https://monokh.com/posts/uniswap-from-scratch)
