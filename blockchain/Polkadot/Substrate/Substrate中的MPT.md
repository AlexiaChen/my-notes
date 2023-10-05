之前写过以太坊中的状态存储  [[ETH之MPT]],  其中trie的格式差不多，都是用MPT数据结构。只是一个hash和编码选取的算法不一样

在Substrate的MPT中，在Trie的基础上，给每个节点计算了一个哈希值，在Substrate中，该值通过对节点内容进行Blake2哈希运算取得，用来索引数据库和计算merkle root。也就是说，MPT用到了两种key的类型(这里的Blake2运算就充当了KV数据库中的key。类似以太坊的keecak256)。
其实，这两种key的类型，跟以太坊是一样的，trie的key和KV引擎中的Key是两个不同的概念，这里要分清楚。

#### 一种是Trie路径所对应的key，由runtime模块的存储单元决定

使用Substrate开发的应用链，它所拥有的各个模块的存储单元会通过交易进行修改，成为链上状态（简称为state）。每个存储单元的状态都是通过键值对以trie节点的形式进行索引或者保存的，这里键值对的value是原始数据（如数值、布尔）的SCALE编码结果（在以太坊中是RLP编码），并作为MPT节点内容的一部分进行保存；key是模块、存储单元等的哈希组合，且和存储数据类型紧密相关，如：

-   单值类型（即Storage Value），它的key是`Twox128(module_prefix) ++ Twox128(storage_prefix)`；
-   简单映射类型（即map），可以表示一系列的键值数据，它的存储位置和map中的键相关，即`Twox128(module_prefix) + Twox128(storage_prefix) + hasher(encode(map_key))`；
-   链接映射类型（即linked_map），和map类似，key是`Twox128(module_prefix) + Twox128(storage_prefix) + hasher(encode(map_key))`；它的head存储在`Twox128(module) + Twox128("HeadOf" + storage_prefix)`；
-   双键映射类型（即double_map），key是`twox128(module_prefix) + twox128(storage_prefix) + hasher1(encode(map_key1)) + hasher2(encode(map_key2))`。

上面中的`encode`函数，你可以认为就是SCALE编码函数。

计算key所用到 [Twox128](https://link.zhihu.com/?target=https%3A//cyan4973.github.io/xxHash/) 是一种非加密的哈希算法，计算速度非常快，但去除了一些严格的要求，如不会碰撞、很小的输入改变导致极大的输出改变等，从而无法保证安全性，适用于输入固定且数量有限的场景中。`module_prefix`通常是模块的实例名称；`storage_prefix`通常是存储单元的名称；原始的key通过SCALE编码器进行编码，再进行哈希运算，这里的哈希算法是可配置的，如果输入来源不可信，如用户输入，则使用Blake2（也是默认的哈希算法），否则可以使用Twox。

#### 一种是KV引擎存储和计算Merkle root hash使用的key

可以通过对节点内容进行哈希运算得到，在键值数据库（即RocksDB，和LevelDB相比，[RocksDB有更多的性能优化和特性](https://link.zhihu.com/?target=https%3A//github.com/facebook/rocksdb/wiki/Features-Not-in-LevelDB)）中索引相应的trie节点。

Trie节点主要有三类，即叶子节点（Leaf）、有值分支节点（BranchWithValue）和无值分支节点（BranchNoValue）；有一个特例，当trie本身为空的时候存在唯一的空节点（Empty）。根据类型不同，trie节点存储内容有稍许不同，通常会包含header、trie路径的一部分key（一般是剩余部分）、children节点以及value。下面举一个具体例子。

trie的路径的KV键值对:

![[Pasted image 20220929104557.png]]

将上表展开成trie的树形结构就是:


![[Pasted image 20220929104636.png]]

图上所示内容的说明如下：

-   这里使用不同的header值来表示不同的节点类型，即01表示叶子节点，10表示无值的分支节点，11表示有值的分支节点。
-   为了提高存储和查找的效率，Substrate使用的MPT和基本的trie不同的是，并不是trie path key的每一位都对应一个节点，当key的连续nibbles之间没有分叉时，节点可以直接使用该连续nibbles。对于叶子节点，连续nibbles可以是trie path key的最后几位；对于分支节点，连续位是从当前位开始到出现分叉位结束。
-   之前提到过，Substrate采用了base-16的trie结构，即一个分支节点最多可以有16个子节点。

由这里可以看到，有值分支节点和无值分支节点，与以太坊稍微有点区别，但是大体都一样的。

接着我们要计算Merkle root  hash了，添加节点Hash值:

![[Pasted image 20220929105518.png]]

首先计算出叶子节点的哈希值，它们被上一级的分支节点所引用，用来在数据库中查找对应的节点；然后计算分支节点的哈希值，直至递归抵达根节点，这里用到的哈希算法是Blake2。

_数据库的物理存储大致如下_：

![[Pasted image 20220929105607.png]]
可以看到，对于substrate来说，在KV存储中的节点都是以SCALE编码的，跟ETH用RLP编码不一样。

RocksDB提供了column的概念，用来存储互相隔离的数据。例如，HEADER column存储着所有的区块头；BODY column存储着所有的区块体，也就是所有的存储单元状态。


#### Child trie

Substrate还提供child trie这样的数据结构，它也是一个MPT。Substrate的应用链可以有很多child trie，每个child trie有唯一的标识进行索引，它们可以彼此隔离，提升存储和查询效率。State trie或者main trie的叶子节点可以是child trie的根哈希，来保存对应child trie的状态。Child trie的一个典型应用是Substrate的contracts功能模块，每一个智能合约对应着自己唯一的child trie。

这里的child trie其实很类似以太坊中的Storage trie，因为Storage trie也是挂在state trie下面的。 [[ETH之MPT#Storage trie----合约的数据在哪]]   ，只是substrate中的child trie可以不仅仅是合约。还可以挂其他自定义的数据状态。

#### 区块裁剪

这里的裁剪也可以理解成状态裁剪，区块的无限增长往往给去中心的节点造成较大的存储消耗，通常只有少数的节点需要提供过往历史数据，大部分节点在能够确保状态最终性之后，可以将不需要的区块数据删除，从而提升节点的性能。Substrate也内置了裁剪的功能，示意图如下：

![[Pasted image 20220929110405.png]]


在区块13中，只有node-4的值从04变为40，其哈希也跟着改变，而其它节点的数据并没有变化，所以新的区块根节点可以复用原有节点，仅仅更新发生变化的节点。假设区块13已经具有最终性，那么区块12中的node-4就可以删除掉。Substrate默认的区块生成算法是BABE或者Aura，而最终性是通过GRANDPA来决定的，在网络稳定的情况下，仅保留一定数量的最新区块是可行的。


#### Substrate的状态裁剪和同步模式

类似于在 [[ETH之状态裁剪#一些社区的回答更新]] 提到的，Substrate也有类似以太坊的fast sync。
下面来总结一下跟状态有关的命令行。

- 轻客户端模式

	`--light` 这个是实验性质的参数，让节点以轻客户端模式运行 [[Substrate学习笔记#轻客户端节点]]

- 状态修剪相关的命令
    - `--unsafe-pruning`, 强制节点以不安全的修剪设置启动。当作为Validator运行时，强烈建议禁用默认的状态修剪（即Archive）。除非设置了这个选项，否则如果修剪功能被启用，节点将拒绝作为Validator启动。相当于Validator要保证全部状态都有。
    - `--pruning <pruning-mode>` 指定保留最大的区块状状态数量（保留最近的N个区块状态），或者以archive保留所有区块状状态。如果节点作为Validator运行，默认是保留所有区块的状态。如果节点不作为Validator运行，只保留最后的256个区块的状态。

- sync模式的相关命令
    - `--sync <sync-mode>` 指定区块链同步模式 有效值是`Full`，`Full`用于下载和验证完整的区块链历史；`Fast`，仅下载区块和最新状态；`FastUnsaf`，下载最新状态，但跳过下载state proof。默认值是`Full`。

另外在，一个社区的同步模式的解释 [What kinds of sync mechanisms does Substrate implement? - Substrate and Polkadot Stack Exchange](https://substrate.stackexchange.com/questions/334/what-kinds-of-sync-mechanisms-does-substrate-implement)

> 回答是2022年2月份的

> Light Sync- 仅仅下载区块头，因此有关于涉及交易的状态转换(STF)也不会被执行，这里跟[[ETH之状态裁剪#^6cda4c]] 所提到的light有点不一样，ETH的light是，会获得最新的当前状态。
> 
> Full sync- "常规 "的同步，区块头和区块体按顺序从创世块下载，交易被执行（所以状态存储被保持）。相当于 [[ETH之状态裁剪#^6cda4c]] 提到的full sync一致，交易被执行，意味着MPT也要被重新计算并做验证，更加耗时。
> 
> Light state sync- 只下载区块头，但一旦链被同步，它就会下载最佳区块的整个状态（而不是从创世区块开始一个区块一个区块地执行区块中的状态转移STF）。相当于下载到最新的被确认的块的区块头以后，就去请求这个区块头里面的state root，获取整个state trie。这里才与[[ETH之状态裁剪#^6cda4c]] 提到的light sync描述的一致
> 
> Warp sync- 最初下载最好的被确认的区块和这个区块的整个状态（相当于以开始就下载best chain最新被确认的区块以及区块的整个state trie），以及从创世块到最好的被确认的区块的所有的validator set的转换的finality proof（目前来自GRANDPA）。在最初的warp sync完成后（因此节点已经准备好导入(同步)最新的区块），历史前序区块可以在后台慢慢下载。这个其实相当于是ETH的fast sync的优化版本，实质是一样的 [[ETH之状态裁剪#^062e17]] ,这个warp sync也不会执行交易的状态转移STF，MPT也不会被在同步的时候被重新计算。

#### 其他资料

- Deep into Substrate Storage

![[substrate-storage-deep-dive.pdf]]

