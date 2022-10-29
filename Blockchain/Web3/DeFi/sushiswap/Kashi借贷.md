
## 什么是kashi

Kashi是一个借贷和保证金交易平台，建立在BentoBox上，允许任何人创建定制的和gas费用高效的市场，用于借贷和抵押各种DeFi代币、稳定币和合成资产。Kashi的代币的广泛多样性是通过使用一个独特的孤立的市场框架来支持的。与传统的DeFi货币市场不同，高风险资产会给整个协议带来风险，在Kashi，每个市场都是完全独立的（类似于SushiSwap DEX），这意味着一个借贷市场内的资产风险对另一个借贷市场的风险没有影响。  
  
传统的借贷项目已经允许用户将流动性添加到基于池的系统中。在这些系统中，如果其中一个资产的价格下降速度超过了清算人的反应速度，每个用户和每个资产都会受到负面影响。从这个意义上说，基于资金池的平台的总风险主要由平台上列出的风险最大的资产决定。这种风险随着每一种额外资产的增加而增加，导致大多数平台上的资产选择非常有限。Kashi的独特设计实现了一种新的借贷方式。将风险隔离到个别借贷市场的能力意味着Kashi可以允许用户添加任何代币。  
  
此外，隔离不同借贷市场的风险使用户能够一键实现杠杆，而无需离开平台。过去，通过直接借贷寻求资产杠杆的用户不得不在一个平台上借款，以便在另一个平台上贷款，如此反复。由于Kashi将市场分成了几对，在同一市场的借贷是可组合的，这意味着Kashi可以在一次点击中自动实现杠杆化。  
  

## kashi与其他的借贷平台有什么不同

主要区别在于，Kashi使用借贷市场(lending markets)，以及孤立的风险市场(risk markets)，而其他借贷平台则是在全球范围内计算风险，这样，任何代币的偿付能力都会影响整个平台的偿付能力。使用孤立的风险市场的一个重要后果是，Kashi能够允许任何代币上市（见什么是孤立的风险市场 [BentoBox/Kashi FAQ | SushiSwap](https://docs.sushi.com/docs/FAQ/Bentobox%20FAQ#what-are-isolated-risk-markets) ）。

另一个重要后果是，使用弹性利率(elastic interest rate)来激励一定范围内的流动性（见什么是弹性利率？[Kashi Lending | SushiSwap](https://docs.sushi.com/docs/Products/Kashi%20Lending) ） 然而，借贷对(lending pairs)和孤立的风险市场(isolated risk market)的另一个后果是，Kashi的oracles需要可定制，以提供无限多代币的喂价(price feeds)。

![[754407dc6cc8002dba7035255eb097f.png]]

