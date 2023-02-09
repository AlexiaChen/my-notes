![[Pasted image 20221007110033.png]]

##### 什么是Arbitrum

由Ed Felten（美国前Deputy CTO和FTC首席技术专家）、Steven Goldfeder和Harry Kalodner在2021年8月由Offchain Labs创建，Arbitrum是一个第二层解决方案，通过实施Rollups，提高以太坊的交易吞吐量，从而改善以太坊智能合约的性能。Arbitrum将多个智能合约或交易Roll up成一个，然后提交给以太坊在第一层，即共识层进行结算。Arbitrum的另一个独特之处在于，它与以太坊虚拟机（EVM）兼容，允许dApps开发者以零修改的方式进行整合。这意味着什么呢？它意味着Arbitrum并没有实现EVM本身。相反，它实现了一个足够小的兼容子集，能够在提交欺诈证明时，识别Rollup内交易的不一致状态变化。

##### Arbitrum One

与我们第2层指南第一部分的主题 "Optimism "类似  [[L2指南之Optimism]]  ，Arbitrum One是一种被称为Optimistic Rollups的技术。它允许以太坊智能合约通过在被称为共识层的以太坊主链上的智能合约和Arbitrrum第二层链上的智能合约之间传递消息来进行扩展。大部分交易处理是在第二层完成的，其结果被记录在主链上，这就减少了以太坊第一层的负载，极大地提高了速度和效率。  
  
它的乐观之处在于，任何validator都能够发布一个rollup区块，并确认其他区块的有效性，而rollup这个词是用来描述公共信息如何被用来从优化的事件日志中重建一个完整的链的历史。Arbitrum协议确保只要任何验证者是诚实的，代码就会正确运行（即按照预期），帮助网络抵制串通和其他形式的攻击。交易被发送到sequencer，然后发布。由sequencer发布的交易在主网上直接被认为是有效的，并从那里被命名为乐观的。因此，Optimistic Rollups是乐观的，他们可以在Ethereum上发布交易，而不需要发布证明，除非有假交易的争议。广播到主网的交易有一定时间的等待期，以防止sequencer的错误交易。在这期间，提供者（provider）可以通过提交欺诈证明来反对已发布的交易，迫使系统使相关交易无效，并惩罚发布错误交易的sequencer。该平台也有自己的定制虚拟机，恰当地命名为Arbitrum虚拟机（AVM）。这是Arbitrum智能合约的执行环境，存在于与Arbitrum链对接的一组智能合约之上，被称为ETHBridge。与Ethereum兼容的智能合约会被自动翻译成在AVM上运行。  
  
Arbitrum Nitro是Arbitrum One L2的升级版，它用Web Assembly（WASM）目标取代了定制设计的AVM，将负责欺诈证明。这意味着L2 Arbitrum引擎可以使用标准语言和工具进行编写和编译，取代了以前Arbitrum版本中使用的定制设计的语言和编译器。这也将使整个系统与EVM更加兼容。另一个变化是，EVM-emulator被Geth取代，Geth是目前运行最多的以太坊客户端。这一升级将被无缝推出，估计将提高20-50倍的执行速度，并大大降低交易成本。

##### Arbitrum Nova and AnyTrust

与Arbitrum One（Arbitrum的Optimistic Rollups第二层）分开，Offchain Labs还宣布推出Arbitrum Nova，一个使用AnyTrust技术建立的独立链。  
  
> Nova是我们建立在AnyTrust技术之上的新链 [Inside AnyTrust | Arbitrum Documentation Center (offchainlabs.com)](https://developer.offchainlabs.com/inside-anytrust) ，而Arbitrum One是我们建立在Optimistic Rollups技术之上的现有链。关键的技术区别在于，Arbitrum One将所有交易数据一直放在以太坊上，而Nova则利用了数据可用性委员会。在这样做的时候，Nova通过首先将数据发送到委员会，并在委员会未能完成其工作的情况下才回落到将数据放在链上，从而实现了显著的成本节约。Nova是一个全新的链，有一套不同的假设，是为游戏、社交应用和对成本更敏感的用例设计的。- Off-chains Labs  
  
Nova与EVM完全兼容，一旦Arbitrrum One如上节所述迁移到Nitro，在这两条链上开发对开发者来说感觉完全一样。至于这两个链如何运作，有两个关键的区别。  
  

首先，在Arbitrum Nova中，现在在处理交易数据的过程中加入了一个最小的信任假设--特别是假设至少有两个委员会成员是诚实的。sequencer不是将所有的`calldata`批处理并发布到L1，而是与委员会共享数据。然后，委员会为成批的交易签署数据可用性证书（又称DACerts），只有这些证书被发布到L1，这导致基础层的存储需求小得多。  
  
AnyTrust由一个节点委员会运作，只需要对有多少委员会成员是诚实的进行最小的假设，而不是拜占庭容错（BFT）。委员会负责数据的持续可用性和使用。假设有20个委员会成员，其中至少有两个人是诚实的。如果20个委员会成员中的19个签署了数据是可用的，那么可以理解为至少有一个诚实的成员签署了数据是可用的，并且数据被认为是正确的。但是，如果委员会的全部或部分成员没有签署数据是可用的呢？在这种情况下，AnyTrust使用 "Fallback to Rollup "功能，像Rollup一样工作，直到委员会再次诚实起来。在标准的BFT链中，对20个成员中的14个成员的诚实性的需求，在虚构的AnyTrust链中已经减少到20个成员中的2个。  
  
其次，最吸引开发者的是，由于使用DAC，Nova上的交易将比Arbitrum One的交易费用低很多。虽然Arbitrum One在许多交易类型上已经比Ethereum便宜了97%，但Nova将明显低于Arbitrum One的成本。这使得Nova非常适合对成本敏感的高交易量预期的项目，如经常铸造新物品或货币的玩赚游戏或有高水平的微交易或有许多不同杠杆的链上互动的社会项目。随着这些类型的项目规模的扩大，将交易成本降到更低的需求成为首要任务，而拥有一个支持这种增长的链就是我们希望通过Nova实现的目标。当然，随着Arbitrum Nova的成熟，我们将继续增加改进，进一步降低成本。

![[Pasted image 20221007112132.png]]

尽管Arbitrum是相当新的，并且还没有激励人们使用该系统的原生代币，但Arbitrum是以太坊生态系统内锁定总价值最大的第二层，并且也是以太坊安全费用的最大贡献者。截至发稿时，Arbitrum One每天向以太坊贡献约23000美元的费用，用于在其上面运行。

![[Nitro-whitepaper.pdf]]