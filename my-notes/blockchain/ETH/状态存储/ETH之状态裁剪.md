## State Tree裁剪

这个主要是来自ETH基金会的博客，而且是2015年的早期研究了。而且是V神亲自写的。[State Tree Pruning | Ethereum Foundation Blog](https://blog.ethereum.org/2015/06/26/state-tree-pruning)

<del>这里主要是对StateRoot所属的这个MPT进行裁剪，因为随着区块的不停增高，账户余额都在改变，这个StateRoot是全局唯一的，[[ETH之MPT#State trie--有且仅有一个]]  而TransactionTrie是每个区块中的交易的 [[ETH之MPT#Transaction trie---在每一个区块中都有一个]]，而且是固定不会改变，这样的裁剪没有意义，所以就是对不停变化的StateRoot所属的state trie进行裁剪。

所以这个state trie也叫做世界状态 World state trie。[[ETH之MPT#修改版的Merkle Patricia Tree MPT 是怎样存储以太坊的状态的？]] </del>

> 以上为什么把其删除掉，是因为随着我阅读文章，我发现这篇文章其实把Transaction trie和receipts trie也给裁剪掉了。不仅仅是state trie。

### 正文

在奥林匹克stress-net发布的过程中，一个重要的问题是客户端需要存储大量的数据；在三个多月的运行中，特别是在上个月，每个以太坊客户端的区块链文件夹中的数据量已经膨胀到惊人的10-40G，这取决于你使用的是哪个客户端以及是否启用了压缩。尽管需要注意的是，这确实是一个压力测试场景，用户被激励在区块链上dump交易，只支付免费的测试ETH代币作为交易费用，交易吞吐量水平因此比比特币高几倍，但对于用户来说，这仍然是一个合理的担忧，在许多情况下，他们没有数百GB的空间来储存其他人的交易历史。

首先，让我们开始探讨为什么目前的以太坊客户端数据库这么大。以太坊与比特币不同，它的特性是每个区块都包含一个叫做 "State Root "的东西：一种专门的Merkle树的root hash，它存储了系统的全部状态：所有账户余额、合约存储、合约代码和账户nonce都在里面。[[ETH之MPT#State trie]] 合约的存储在Storage root中，但是Storage root是挂在State trie中的。

![[Pasted image 20220930091606.png]]

这样做的目的很简单：它允许一个仅有最后一个区块的节点，加上最后一个区块实际上是最近的区块的一些保证，为了不需要处理任何历史交易，非常快速地“同步”区块链，通过简单地从网络中的节点下载树的其余部分（拟议的HashLookup wire protocol message [Ethereum Wire Protocol · ethereum/wiki Wiki (github.com)](https://github.com/ethereum/wiki/wiki/Ethereum-Wire-Protocol) 将促进这一点），通过检查所有的哈希值是否匹配来验证树是正确的，然后从那里（高度）继续进行。在一个完全去中心化的背景下，这可能是通过比特币的 "优先校验Block Header "策略的高级版本来完成的，它看起来大致如下。

- 客户端可以下载尽可能多的区块头。
- 确定在最长链的末端的头。从该块头开始，为了安全起见，往前追溯100个块，并将该位置的块称为$P^{100}(H)$（"块头的第100代祖先"）。
- 使用HashLookup操作码，从$P^{100}(H)$的state root下载state tree（注意，在最初的一两轮之后，可以根据需要在许多P2P Peers之间并行化下载）。验证树的所有部分是否匹配。
- 从这里开始正常进行。

> 这里写一下我的个人猜想，以太坊的快速同步功能可能就是这样，为了实现轻客户端的基础就在这里，必须还是要下载state trie。从最近N个区块下载state trie，因为state trie只是每个块之间才有变化，所以数据量不会很大。

对于轻客户来说，state root甚至更有优势：他们可以立即确定任何账户的确切余额和状态，只需向网络查询树的特定分支，而不需要遵循比特币的多步骤1-of-N "要求所有交易输出，然后要求所有交易花费这些输出，并采取剩余的 "轻客户模式。

然而，这种state tree机制如果被单纯地实现，有一个重要的缺点：树上的中间节点大大增加了存储所有数据所需的磁盘空间。要想知道为什么，请考虑这里的这个图。

![[Pasted image 20220930092921.png]]

在每个单独的区块中，树的变化是相当小的，而且树作为数据结构的神奇之处在于，大多数数据可以简单地被引用两次而不被复制。然而，即使如此，对于每一次对状态的改变，都需要将对数数量的节点（即1000个节点时~5次改变，1000000个节点时~10次改变，1000000000个节点时~15次改变）被存储两次，一个版本用于旧树，一个版本用于新的trie。最终，由于一个节点处理每一个区块，因此我们可以预期总的磁盘空间利用率，用计算机科学术语来说，大约是$O(n*log(n))$，其中$n$是交易负载数量。从实际情况来看，以太坊区块链只有1.3GB，但包括所有这些额外节点的数据库大小为10-40GB(整个网络的节点)

那么，我们能做什么呢？一个向后看的解决方法是简单地去实现区块头信息优先同步，基本上将新用户的硬盘消耗重置为零，并允许用户通过每一两个月重新同步来保持低的硬盘消耗，但这是一个有点丑陋的解决方案。

第一种方式相当于，除了高度最近几个块，其他块都是区块头的形式存储在本地硬盘，最近的块都是整个区块，但是随着用户使用了一两个月后，又积累了很多区块了，占用空间，用户又需要删除本地数据，重新同步区块链，把之前同步的区块变成区块头的形式存储。这样太笨了。

^a68459

另一种方法是实现state tree的修剪：基本上，使用引用计数来跟踪state tree中的节点（这里使用计算机科学术语中的 "节点"，意思是 "图或树结构中某处的数据块"，而不是 "网络上的计算机"）判断节点何时从树中删除，并在此时将其置于 "死囚区"：除非该节点在接下来的`X`个块（例如，`X=5000`）内以某种方式再次使用，在该`X`块数过去之后，该节点应从KV数据库中永久删除。本质上，我们存储属于当前状态的树节点，我们甚至存储最近的历史，但我们不存储超过5000块的历史。 ^f4566a

> 这也就是，Parity实现的substrate框架中也有轻客户端模式 [[Substrate学习笔记#轻客户端节点]]，就是采用类似的原理。其实也就是对state trie的节点实现了一个基于引用计数的GC原理。

为了节省空间，`X`应该被设置得越低越好，但将`X`设置得太低会损害鲁棒性：一旦实现了这种技术，一个节点就不能在不完全重启同步的情况下回退超过`X`个块。因为`X`个块之前，都没有历史了，现在，让我们看看如何完全实现这种方法，并考虑到所有的边角情况。

- 当处理一个编号为`N`的区块时，跟踪所有引用计数下降到零的trie节点（在state trie和receipts trie中）。将这些节点的哈希值放入某种数据结构的 "死亡行 "数据库中，这样以后就可以通过区块编号（具体来说，区块编号`N+X`）来调用该列表，并将节点数据库条目(entry)本身标记为在区块`N+X`处可被删除。
- 如果一个被关在死囚牢房的trie节点被重新恢复（一个实际的例子是账户A获得了一些特定的余额/nonce/合约代码/存储 组合成的项，简称`f`，然后切换到一个不同的值`g`，然后账户B在`f`的trie节点被关在死囚牢房时获得了状态`f`，然后将其引用计数增加到`1`。如果该节点在未来的某个区块编号`M（M>N）`再次被删除，那么就把它放回未来区块的死囚区，在区块编号`M+X`被删除。
- 当你在处理区块`N+X`时，检查与每个哈希值相关的trie节点；如果该节点在该特定区块中仍被标记为删除（即没有恢复，重要的是没有恢复，然后又被重新标记为删除），则删除它。将死囚数据库中的哈希值列表也删除。
- 有时，一个链的新头(top of blockchain)不会在前一个头的上面，你将需要恢复一个块（这里是对于硬分叉的情况考虑得多，主要是PoW算法，如果是PoS，这样的概率很低）。对于这些情况，你将需要在数据库中保留所有引用计数变化的日志（这就是日志文件系统中的 "日志"；本质上是所做变化的有序列表，有点类似redo log）；当恢复一个块时，删除产生该块时的死行列表，并undo根据日志所做的变化（完成后删除日志）。
- 当处理一个区块时，在区块`N-X`处删除日志；无论如何，你没有能力恢复超过`X`个区块，所以日志是多余的（如果保留，实际上会破坏修剪的整个意义）。

> 由上可以看到，可能对于这种修剪模式的轻型客户端，会遇到分叉情况，处理不了X个块之前的分叉状态恢复，所以一般这种轻型客户端也不参与共识 [[Substrate学习笔记#轻客户端节点]]

一旦这样做了，数据库应该只存储与最后`X`个区块相关的state trie节点，所以你仍然会有你需要的这些区块的所有信息，但没有其他的了。在此基础上，还有进一步的优化。特别是，在`X`个区块之后，transaction trie和receipts trie应该被完全删除，甚至区块也可以说是被删除了（因为区块头占用太小了）--尽管有一个重要的论点是保留一些 "归档节点(archive nodes) "的子集，这些节点存储了绝对的一切，以帮助网络的其他部分获得它需要的数据。  

现在，这能给我们带来多大的节省？事实证明，相当多！特别是，如果我们要把所有的数据都存储在一个地方，那么我们就必须要把所有的数据都存储起来。特别是，如果我们采取最终的大胆路线，并采取`X = 0`（即完全失去处理甚至单块分叉的能力，不存储任何历史），那么数据库的大小将基本上是状态的大小：`一个值`，即使是现在（这个数据是在670000块抓取的）大约是40MB - 其中大部分是由像这样的账户组成的 [explorer.etherapps.info](https://explorer.etherapps.info/address/0x798d86e782c8c34da97f7389a464c8af76ea3442) ，存储槽(stroge slots)被填满，故意垃圾邮件的网络。在`X=100000`时，我们将得到基本上是目前的10-40GB的大小，因为大部分的增长发生在最后十万个区块，而存储日志和死囚名单所需的额外空间将弥补其余的差异。在这两者之间的每一个值，我们可以预期磁盘空间的增长是线性的（即`X=10000`将使我们有大约百分之九十的方式接近零）。  

请注意，我们可能想采取一种混合策略：保留每个区块，但不保留每个state tree的节点；在这种情况下，我们将需要增加大约1.4GB的空间来存储区块数据。值得注意的是，区块链大小的原因不是快速的区块生成时间；目前，过去三个月的区块头大约占300MB，其余的是过去一个月的交易，所以在高水平的使用下，我们可以期望继续看到交易占主导地位。也就是说，轻型客户端如果要在低内存情况下生存，也需要修剪区块头，相当于也要把区块头给干掉。

上面描述的策略已经在pyeth [Page not found · GitHub](https://github.com/ethereum/pyethereum/tree/pruning) 中以非常早期的alpha形式实现了；在Frontier推出后，它将在适当的时候在所有客户端中正确实施，因为这样的存储膨胀只是一个中期而不是短期的可扩展性问题。  

> 注意，这里的Frontier不是substrate做的那个以太坊兼容层frontier，是以太坊的一个里程碑叫frontier。因为文章写的比较早，所以pyeth已经改成 py-evm了 [ethereum/py-evm: A Python implementation of the Ethereum Virtual Machine (github.com)](https://github.com/ethereum/py-evm)

## 以太坊的state trie裁剪

这个主要是2018年的Medium文章 [Ethereum’s state trie pruning. Ethereum stores the current state in a… | by Seung Woo Kim | CodeChain | Medium](https://medium.com/codechain/ethereums-state-trie-pruning-45ea73ed2c78) ，会对ETH更加近期的state trie的裁剪实现进行描述

### 正文

以太坊将当前状态存储在一个修改过的Merkle patricia trie（又称MPT）中，这是一种前缀树的类型。为了计算state root的哈希值，MPT不需要查看整个state trie，而只需要重新计算修改过的分支的哈希值，这导致更快地计算出root hash。
![[Pasted image 20220930110744.png]]

在使用MPT时，可以尽量减少新增加的节点数。例如，上图中N块和N+1块之间的区别是，A的右子的值从10变成了20。在这种情况下，除了从10变为20的节点的父节点外，所有的节点都可以被重复使用。因此，唯一需要添加的节点是用蓝色画的三个节点。

那么，那些无法到达的节点会怎样呢？在上面的例子问题中，3个红色节点在区块N+1中是不需要的。那么一旦添加了3个蓝色节点，这3个红色节点是否可以被删除？如果是这样的话，这将是一个很容易解决的问题。然而，它不是那么简单。以太坊并不保证其区块的最终性(finality)，也就是说会分叉，这时候还是PoW算法。换句话说，区块`N + 1`可以在任何时候撤回到区块N。此外，过去的状态可以通过Web3 API获取 [JavaScript API · ethereum/wiki Wiki (github.com)](https://github.com/ethereum/wiki/wiki/JavaScript-API) ，因此，仅仅因为节点在当前状态下没有被使用，就不能简单地删除。(要给它留一个时间窗口区。类似于GC技术中的新生代，老年代什么的。)

然而，这些节点不能只是无限期地留在那里。以太坊目前的状态约为25GB，但如果要存储所有过去的状态，那么它将超过300GB [The Ethereum-blockchain size will not exceed 1TB anytime soon. - DEV Community 👩‍💻👨‍💻](https://dev.to/5chdn/the-ethereum-blockchain-size-will-not-exceed-1tb-anytime-soon-58a) 。此外，这个规模只会越来越大，因此，存储所有东西是不现实的。作为一个解决方案，以太坊将可访问的过去状态的数量限制为127个块的状态 https://github.com/ethereum/go-ethereum/blob/63687f04e441c97cbb39d6b0ebea346b154d2e73/core/blockchain.go#L649 ，只有那些包含在较早状态中的节点可以被删除。然而，只是被允许删除和能够删除之间是有区别的。在127个最近的节点中找到一个不能被访问的trie节点并删除该节点并不是一件容易的事。

这个问题类似于一个长期存在的计算机科学问题，自动内存管理 [Garbage collection (computer science) - Wikipedia](https://en.wikipedia.org/wiki/Garbage_collection_%28computer_science%29) 。Vitalik Buterin[Vitalik Buterin - Wikipedia](https://en.wikipedia.org/wiki/Vitalik_Buterin) 写的state trie修剪提到了引用计数 [[#^f4566a]]。然而，Ethereum的state trie修剪有一些不同于我们所习惯的正常内存管理。

在正常的自动内存管理中，我们处理的是不稳定的资源。因此，不需要考虑程序被非正常关闭的情况。这是因为一旦程序被关闭，就没有更多的资源需要管理。然而，state trie 的树节点是一种存储在数据库中的持久性内存。如果程序非正常关闭，state trie 也可能变得非正常，使整个事情变得不可恢复。这也是为什么Vitalik建议的state trie修剪不能进入这里的名单。

state trie 修剪不一定要用引用计数来实现。例如，跟踪垃圾收集 [Tracing garbage collection - Wikipedia](https://en.wikipedia.org/wiki/Tracing_garbage_collection) ，一种使用跟踪的方法，是自动内存管理中常用的东西。但是，追踪需要额外的内存，而且还有其他的性能问题，比如stop-the-world，这些问题必须首先解决。

由于这样的问题，go-ethereum以非常有限的方式执行state trie 修剪。在state trie 修剪中，它使用一个基于state trie 的cache。存储在这个缓存中的trie节点会被修剪，而存储在KV数据库中的trie节点则不会被修剪。

**被缓存的trie节点只有在服务器正常关闭 https://github.com/ethereum/go-ethereum/blob/63687f04e441c97cbb39d6b0ebea346b154d2e73/core/blockchain.go#L658 、超过128区块标记 、超过缓存大小 ，或者最后被缓存的trie节点在数据库中存储超过5分钟的情况下才会被存储到数据库中。
https://github.com/ethereum/go-ethereum/blob/63687f04e441c97cbb39d6b0ebea346b154d2e73/core/blockchain.go#L927 **


然而，trie节点在创建后超过5分钟之前被删除的情况并不常见。因此，大多数应该被删除的trie节点仍然在数据库内。以太坊的开发者们正在不断尝试实施状态修剪。然而，在实施状态修剪之前，我建议使用fast sync尝试下面的方法。

> 我理解的上面这句话，好像跟2015年V神写的设想有点不一样啊，感觉到了2018年还在没有完全实现，状态修剪还在慢慢的进行中？

- 用一个新的以太坊客户端（没有存储数据，还没有开始同步区块）
- 用新客户端采用fast sync同步模式来向已存在的客户端同步区块链
- 删除已存在的客户端

> 这种好像是手动进行state trie的裁剪，V神在2015年的文章中提到 [[#^a68459]]

通过上述过程，已经被fast sync模式同步的状态可以在不涉及垃圾trie节点的情况下被管理。虽然这种方法看起来很简陋，但比起实现垃圾收集器，这是一个最现实的解决方案，因为垃圾收集器本身就会带来新的风险。在以太坊正式实现安全的垃圾收集器之前，开发者可能应该坚持使用上面建议的方法。

> 原来如此，原来到了2018年，以太坊还是没有完全实现V神2015年的设想，是用类似第一种最笨最保险的方法。而且这种裁剪模式，也不能用在Archive node上，Archive node本身是归档节点，需要存储所有状态历史。

> 那么当时我在harmony干的时候，先不论2021年的时候，以太坊到底有没有实现2015年的V神的state trie裁剪设想，我BOSS布置一个对Archive node的ValidatorWrapper状态修剪的任务给我，现在回头来看，当时是可以的，[Do not save validator wrapper state data for every block in archival node · Issue #3843 · harmony-one/harmony (github.com)](https://github.com/harmony-one/harmony/issues/3843)  [[core] Do not save ValidatorWrapper state for every block by AlexiaChen · Pull Request #3844 · harmony-one/harmony (github.com)](https://github.com/harmony-one/harmony/pull/3844) [feat: Prune validator rewards in archival node... by MaxMustermann2 · Pull Request #4097 · harmony-one/harmony (github.com)](https://github.com/harmony-one/harmony/pull/4097) ，最后也被实现了，虽然不是我实现的。

> 当时harmony的archive node硬盘占用增长太快了，我想着主要原因也不是ValidatorWrapper的问题，还是费用太低，大量交易导致的。[S0 archival node is growing too fast · Issue #3805 · harmony-one/harmony (github.com)](https://github.com/harmony-one/harmony/issues/3805), 这个issue还被一个前harmony的员工点了一个赞 [fxfactorial (@edgararout) (github.com)](https://github.com/fxfactorial) 


## 一些社区的回答更新

社区提问 状态裁剪是怎么回事

- [blockchain - What is state-trie pruning and how does it work? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/174/what-is-state-trie-pruning-and-how-does-it-work)

> 2016年回答的

> 这是一个类似于编程语言和tree-based的版本控制系统（如git）中的垃圾收集的概念。当以太坊合约运行时，或者交易被执行，它们会修改其状态。由于state trie是一个不可改变的只附加的结构，这意味着每次状态被修改时，它都会得到一个新的state root。一些从旧的根部可以到达的元素可能无法从新的根部到达（由于删除或修改条目的操作）。理论上，它们可以被修剪掉（垃圾收集）。然而，由于目前的工作证明（Proof-Of-Work consensus）并没有定义什么时候状态转换是最终的(PoW可能会导致分叉)，所以理论上总是有一种可能性，即状态会被恢复到旧的根上，被修剪的trie node会被再次需要。所以，修剪目前是一种权衡。我们说，例如，在5000个区块之后，我们假设状态不会被恢复，并修剪所有不可达的节点。有人可能仍然想禁用这个功能，为特殊目的保留区块链的整个历史。

- [synchronization - Difference between a pruned and unpruned blockchain - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/1229/difference-between-a-pruned-and-unpruned-blockchain)

> 这个回答是ETH核心开发者2016年回答的。

> 让我们一步步来吧。  
> 
> 区块链的工作方式一般是有一个起源（创世）状态，有几个账户有资金，然后你放在链上的每一个区块都会把这些起源资金转移到周围，也会给矿工一点额外的好处。因此，每当你把一个新的区块导入你现有的链中，看看你对世界的看法（状态）是什么，并根据区块中包含的交易来改变这个状态，得出你认为世界是什么样子的新看法。你不会抛弃你过去对世界的看法，因为如果区块链中出现分叉（例如，一个矿工出现了一个更好的区块，或者可能是两个更好的区块），那么你需要将你的看法从过去的状态转变为更好的版本。这导致你过渡到的所有过去的状态被永恒地积累起来。这是一个未修剪的状态/区块链，目前以太坊为7GB。  
> 
> 需要注意的是，大多数时候你并不关心一个账户在3年前有多少资金，你只关心目前的状态是什么（也许几天前也是）。那么，为什么要把那些极老的过去的过渡状态保留下来呢？状态修剪本质上是把所有的中间状态，冲进厕所。重要的是要意识到，你只是扔掉了中间世界观，而不是区块本身或任何其他可能对网络不健康的数据（即一个加入的节点需要这些数据来同步）。因此，通过修剪你的state trie，你失去了查询过去账户余额的能力(在某个区块高度，你的账户余额是多少，这样的需求就没法实现了)，但好处是将你的存储数据量减少到原来的1/5-1/6。  
> 
> 好吧，那么fast sync是怎么回事？好吧，按照之前的思维模式，如果你不关心三年前的随机账户余额，你为什么要重放区块链的整个交易历史，只是为了达到当前的状态。因此，快速同步所做的是，它下载所有的区块链（包括区块头和区块体），但它不执行交易，而是一次一个区块地生成当前同步到的世界观(state)。相反，它只验证PoW，当整个链被下载后，它查看state root（定义当前世界观的哈希值）并直接从网络中下载statedb（**也就是还是要拿到KV数据库中的最新的state trie，对于最新的root来说，一些老的，不可达的trie节点就不会被下载了，相当于被修剪了**），从一开始就重建最终状态，而不需要中间状态。这意味着，除了下载区块，它还需要下载额外的数据，即state trie 本身，所以它是在用带宽交换处理能力（**即我下载state，而不是生成计算它**）。fast sync的最终结果是**一个经过修剪的statedb，从所有的意图和目的来看，只是通过一个不同的手段（类似2015年V神提出的第一种设想 [[ETH之状态裁剪#^a68459]], 其实本质上还是第二个设想没实现）。目前这样一个数据库的大小是1.2-1.3GB。  

- [go ethereum - What is Geth's "light" sync, and why is it so fast? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/11297/what-is-geths-light-sync-and-why-is-it-so-fast)

> 这个是2017年的回答

> 我将采取我的镜头。专家们，请纠正我。
> 
> full sync, 获取块头、块体，并验证从创世块开始，每一个区块中的每一个元素。
> 
> fast sync, 获取区块头和区块体，在当前区块编号减去64之前的区块不处理(执行)任何交易。然后它得到一个快照状态，并像full sync一样进行。
> 
> light sync, 只获取当前状态。为了验证元素，它需要向全节点或Archive node获取相应的trie。

^6cda4c

- [synchronization - What is Parity's “warp” sync, and why is it faster than Geth "fast"? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/9991/what-is-paritys-warp-sync-and-why-is-it-faster-than-geth-fast)

> 2016年的回答

> 如果不重述Parity  wiki上的解释，就很难给出答案......
> 
> 相关的部分如下。
> 
>**这些快照可以用来快速获得一个特定区块的完整状态副本。每隔30000个区块，节点会对该区块的状态进行一次关键性的共识快照。任何节点都可以通过网络获取这些快照，实现fast sync。**
>
>快照本身由三部分组成。
>
>1. 一个清单，基本上是关于快照的元数据。
   2. 区块Chunks，它包含关于区块的原始数据和它们的交易收据(receipts)。
> 3.状态Chunks，包含关于某个区块的状态的数据。
> 
> 目前，块的大小被设定为4MB。
> 
> 那么，这实际上是如何加快同步速度的呢？那个维基页面没有说的是，我们最初只同步快照。因此，对于每个区块，我们以30,000的时间间隔获得一组4MB的chunks。然后在后台我们继续同步剩余的区块数据。
> 
> 这相当于 Geth 的 `--fast sync`，它首先同步区块头，然后在后台同步其余的数据。只是 `--warp` 在第一遍时同步的数据更少，并在以后填补更大的空白。

- [go ethereum - What is Geth's "fast" sync, and why is it faster? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/1161/what-is-geths-fast-sync-and-why-is-it-faster)

> 2016年的回答

>fast sync不是一次处理整个区块链，重放历史上曾经发生过的所有交易，而是沿着区块下载交易收据(transaction receipts)，[[ETH之MPT#receipts trie]] 并拉出整个最近的状态数据库。[eth/63 fast synchronization algorithm by karalabe · Pull Request #1889 · ethereum/go-ethereum (github.com)](https://github.com/ethereum/go-ethereum/pull/1889)

^062e17


其实经过我的理解，fast sync跟Substrate那样的轻客户端的实现模式还是不一样 [[Substrate学习笔记#^0b135b]]，还是会同步整个区块链，只是不执行交易（不处理区块）而已，只是拉取state trie对区块头的trie root hash进行验证，包括验证PoW。只是执行到最近的区块（最新高度的区块减去1024个区块）区间，对这个区间内的区块进行完全处理，回归到full sync那样的方式。**fast sync本质上有历史数据，还是可以查询所有的历史记录的。** 