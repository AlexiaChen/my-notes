ETH是用merkle patricia trie来存储状态的，当然，其实修改版的，全称modified merkle patricia trie(MPT)。



因为Archival node(归档节点)要存储全量状态数据，所以ETH官方提出了对状态树进行裁剪，减少对硬盘存储的占用，有以下相关的几篇文章

- [State Tree Pruning | Ethereum Foundation Blog](https://blog.ethereum.org/2015/06/26/state-tree-pruning)
- [Ethereum’s state trie pruning. Ethereum stores the current state in a… | by Seung Woo Kim | CodeChain | Medium](https://medium.com/codechain/ethereums-state-trie-pruning-45ea73ed2c78)

- [Understanding the ethereum trie | Easy There Entropy (wordpress.com)](https://easythereentropy.wordpress.com/2014/06/04/understanding-the-ethereum-trie/)




当然还有以下社区的提问

- [blockchain - What is state-trie pruning and how does it work? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/174/what-is-state-trie-pruning-and-how-does-it-work)
- [synchronization - Difference between a pruned and unpruned blockchain - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/1229/difference-between-a-pruned-and-unpruned-blockchain)
- [blockchain - ELI5 How does a Merkle-Patricia-trie tree work? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/6415/eli5-how-does-a-merkle-patricia-trie-tree-work)
- [go ethereum - What is Geth's "light" sync, and why is it so fast? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/11297/what-is-geths-light-sync-and-why-is-it-so-fast)
- [synchronization - What is Parity's “warp” sync, and why is it faster than Geth "fast"? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/9991/what-is-paritys-warp-sync-and-why-is-it-faster-than-geth-fast)
- [go ethereum - What is Geth's "fast" sync, and why is it faster? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/1161/what-is-geths-fast-sync-and-why-is-it-faster)

- [eth/63 fast synchronization algorithm by karalabe · Pull Request #1889 · ethereum/go-ethereum (github.com)](https://github.com/ethereum/go-ethereum/pull/1889)
- 