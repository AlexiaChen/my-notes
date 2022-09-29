## state Tree裁剪

这个主要是来自ETH基金会的博客，而且是2015年的早期研究了。

这里主要是对StateRoot所属的这个MPT进行裁剪，因为随着区块的不停增高，账户余额都在改变，这个StateRoot是全局唯一的，[[ETH之MPT#State trie--有且仅有一个]]  而TransactionTrie是每个区块中的交易的 [[ETH之MPT#Transaction trie---在每一个区块中都有一个]]，而且是固定不会改变，这样的裁剪没有意义，所以就是对不停变化的StateRoot所属的state trie进行裁剪。

所以这个state trie也叫做世界状态 World state trie。[[ETH之MPT#修改版的Merkle Patricia Tree MPT 是怎样存储以太坊的状态的？]]




- [State Tree Pruning | Ethereum Foundation Blog](https://blog.ethereum.org/2015/06/26/state-tree-pruning)

## 以太坊的state trie裁剪

这个主要是2018年的Medium文章


- [Ethereum’s state trie pruning. Ethereum stores the current state in a… | by Seung Woo Kim | CodeChain | Medium](https://medium.com/codechain/ethereums-state-trie-pruning-45ea73ed2c78)

## 资料

TODO

因为Archival node(归档节点)要存储全量状态数据，所以ETH官方提出了对状态树进行裁剪，减少对硬盘存储的占用，有以下相关的几篇文章



社区提问 状态裁剪是怎么回事

- [blockchain - What is state-trie pruning and how does it work? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/174/what-is-state-trie-pruning-and-how-does-it-work)
- [synchronization - Difference between a pruned and unpruned blockchain - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/1229/difference-between-a-pruned-and-unpruned-blockchain)

- [go ethereum - What is Geth's "light" sync, and why is it so fast? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/11297/what-is-geths-light-sync-and-why-is-it-so-fast)
- [synchronization - What is Parity's “warp” sync, and why is it faster than Geth "fast"? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/9991/what-is-paritys-warp-sync-and-why-is-it-faster-than-geth-fast)
- [go ethereum - What is Geth's "fast" sync, and why is it faster? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/1161/what-is-geths-fast-sync-and-why-is-it-faster)

- [eth/63 fast synchronization algorithm by karalabe · Pull Request #1889 · ethereum/go-ethereum (github.com)](https://github.com/ethereum/go-ethereum/pull/1889)

