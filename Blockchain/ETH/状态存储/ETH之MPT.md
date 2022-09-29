## 修改版的Merkle Patricia Tree(MPT)是怎样存储以太坊的状态的？

这里其实主要是MPT的概览(overview)，还不会涉及太多的细节。

抛开网络部分，我们可以说以太坊是一个状态机，交易在以太坊网络上修改状态。一个状态可以被表达为一个键值对。虽然有几种表示键值对的方法，但以太坊规范定义了修改的Merkle Patricia Trie（又称MPT）作为保存状态的方法。

基本上，MPT是Patricia trie和Merkle树的组合，有一些额外的优化，符合Ethereum的特点。因此，在了解MPT之前，应该先了解Patricia trie和Merkle tree。

### Patricia trie

其实就是Radix tree的一个变种，我有在 [[理解Radix树]] 中提到。目前你就可以按照Radix tree来理解Patricia trie。不需要按照最原始论文中的patricia trie理解。当然，了解Radix Tree之前，建议先了解下[[理解Trie树]]。

### Merkle Tree

这里我有在 [[Merkle Tree和其证明]] 中写到原理。

### Merkle Patricia Trie(MPT)

![[Pasted image 20220926100726.png]]

在MPT中，以及在Merkle tree中，每个节点都有一个哈希值。每个节点的哈希值是由其内容的`SHA3`(keccak hash)哈希值决定的。这个哈希值也被用作指向该节点的key。Go-ethereum使用levelDB，而parity使用rocksDB来存储状态。它们都是KV存储引擎。`存储引擎中保存的key和value不是Ethereum状态的键值`。**存储引擎中的value是MPT节点的内容，而key是这个节点的`SHA3` hash值**。

> 注意，这里的SHA3之前就叫keccak，但是ETH使用的叫做keccak-256，也就是Hash值是一个256bit的值，这个keccak-256算法与NIST标准的SHA3-256不一样，[cryptography - Which cryptographic hash function does Ethereum use? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/550/which-cryptographic-hash-function-does-ethereum-use)

以太坊状态的key-values被用作MPT上的路径。Nibble是用来区分MPT中key-values的单位，所以每个节点最多可以有16个分支。此外，由于一个节点有自己的值，一个分支节点（branch node）是由1个节点值和16个分支组成的17项数组。

```cpp
// 分支节点就是下面还有子节点(叶子节点和分支节点)
struct BranchNode{
	std::array array[17] // array[16] is node self value。 rest of array items are branches (array[0]~array[15])
}
```

一个没有子节点的节点被称为叶子节点（leaf node）。一个leaf node由两个item组成：它的路径和值。例如，我们假设键 "0xBEA "包含1000，键 "0xBEE "包含2000。那么，应该有一个路径为 "0xBE "的分支节点（因为有一个公共前缀`0xBE`），在这个节点下，将连接两个路径为 "0xA "和 "0xE "的叶子节点。所以这些路径，你可以就理解为Radix Tree中的edge了，就是trie中的字符串，路径就是trie中的key。这些大量的路径肯定是有公共前缀的。

```cpp
// 叶子节点就是下面没有任何节点了
struct LeafNode{
  std::string path;
  char* value;
}
```

![[Pasted image 20220926103153.png]]

> 从上图你就可以理解了，为什么其分支是一个16个元素的数组，因为是采用16进制，存储`0~9 A-F`这十六个字符。用16进制表示的路径，自然可以大概猜想出来一些，这些路径肯定就是一些编码，一般我们都用十六进制来表示编码后的二进制数据。我们再猜想一下，ETH肯定是把ETH上的状态编码成“十六进制”的路径，把这个路径作为MPT的key，插入进MPT中。**注意，MPT中的key和value跟KV存储引擎中的key和value不一样**。

在MPT中，除了分支节点(branch node)和叶子节点(leaf node)之外，还有一种类型的节点。它们是扩展节点(extend node)。扩展节点是分支节点(branch node)的一个优化节点。在以太坊状态下，经常有一些只有一个子节点的分支节点。这就是为什么MPT将只包含一个子节点的分支节点压缩成具有路径和子节点哈希值的扩展节点的原因。（这里跟Radix Tree中的优化手段一样，只有一个父节点的子节点可以压缩为一个edge）

```cpp
// 扩展节点就是只有一个子节点的分支节点
struct ExtendNode{
	std::string path;
	std::vector<uint8> child_hash256;
}
```

由于叶子节点和扩展节点都是由两个items组成的数组，应该有一种方法来区分这两个不同的节点。为了进行这样的区分，MPT在路径上添加了一个前缀。

如果节点是一个叶子节点(Leaf Node):
   - 路径由奇数个nibbles组成，则添加`0x20`作为前缀。
   - 路径由偶数个nibbles组成，应该添加`0x3`作为前缀。

如果该节点是一个扩展节点(Extension Node):
   - 路径由偶数个nibbles组成，你要添加`0x00`作为前缀。
   - 路径由奇数个nibbles组成，你应该添加`0x1`作为前缀。

因为由奇数个nibbles组成的路径有一个nibble作为前缀，由偶数个nibbles组成的路径有两个nibbles作为前缀，所以路径总是以字节的形式表示。

> 上面提到的nibble就是十六进制中`0~F`的单个字符，一个nibble就是其中16个字符中的一个


**上面偶数个奇数个nibbles，未必准确，需要看ETH官网文档，以下面这篇文章为主**

## Patricia Merkle Trees

这篇主要是ETH官方的MPT翻译 [Patricia Merkle Trees | ethereum.org](https://ethereum.org/en/developers/docs/data-structures-and-encoding/patricia-merkle-trie/)

Patricia Merkle Trie提供了一个加密认证的数据结构，可用于存储所有（key，value）的KV绑定映射关系。

Patricia Merkle Tries是完全确定的，这意味着在一个trie中具有相同（key，value）的绑定被保证是相同的，直到最后一个字节。这意味着它们有相同的根哈希(root hash)，为插入、查找和删除提供了O(log(n))效率的圣杯。此外，它们比更复杂的基于比较的替代方案（如红黑树）更容易理解和编码。

在读此文章之前，你最好对Merkle Tree和序列化有一定的理解才可以继续。


### 基本的Radix Trie

在一个基本的Radix trie中，每个节点像以下这样:

$$
[i_0, i_1 ... i_n, value]
$$

其中$i_0 ... i_n$代表字母表的符号（通常是二进制或十六进制），value是节点的终端值，$i_0 ... i_n$槽中的值要么是NULL，要么是指向其他节点的指针（在我们的例子中是哈希值）。这就形成了一个基本的（key，value）存储。例如，如果你对trie中当前映射到`dog`的值感兴趣，把`dog`作为key，在MPT中查找，你会首先将`dog`转换成字母的十六进制表示（给出0x64 0x6f 0x67），然后沿着这个路径下降trie，直到找到该值。也就是说，你会先在一个平坦的KV数据库中查找根哈希（root hash），以找到trie的根节点（这是一个指向其他节点的键（hash）的数组），使用索引6的值作为key（并在平坦的KV数据库中查找它），以获得一级的节点，然后挑选其中的索引4来查找下一个值，然后挑选其中的索引6，以此类推，直到，一旦你遵循路径。根->6->4->6->15(f)->6->7，你就可以查到你所拥有的节点的值并返回结果。

在 "trie "和底层平坦的KV引擎中查找东西是有区别的。它们都定义了KV的键值对组织，但是底层DB可以对一个key进行传统的一步查询。在 trie 中查询一个key需要多次底层 DB 查询以获得上述的最终值。让我们把后者的key称为路径，以消除歧义。

Radix Trie的更新和删除操作很简单，可以大致定义如下。

```python
def update(node,path,value):
	if path == '':
		curnode = db.get(node) if node else [ NULL ] * 17
		newnode = curnode.copy()
		newnode[-1] = value
	else:
		curnode = db.get(node) if node else [ NULL ] * 17
		newnode = curnode.copy()
		newindex = update(curnode[path[0]],path[1:],value)
		newnode[path[0]] = newindex
	db.put(hash(newnode),newnode)
	return hash(newnode)

def delete(node,path):
	if node is NULL:
		return NULL
	else:
		curnode = db.get(node)
		newnode = curnode.copy()
		if path == '':
			newnode[-1] = NULL
		else:
			newindex = delete(curnode[path[0]],path[1:])
			newnode[path[0]] = newindex

		if len(filter(x -> x is not NULL, newnode)) == 0:
			return NULL
		else:
			db.put(hash(newnode),newnode)
			return hash(newnode)

```

Radix trie的 "Merkle "部分产生于这样一个事实：一个节点的确定性密码学哈希值被用作节点的指针（对于key/value DB（levelDB）中的每一次查找，key == keccak256(rlp(value))），而不是像在C语言实现的更传统的trie中可能发生的一些32位或64位内存位置。这为数据结构提供了一种加密认证形式；如果一个给定的trie的root hash是公开的，那么任何人都可以通过提供连接特定值到trie root的每个节点的哈希值来证明该trie在特定路径上有一个特定的值。攻击者不可能提供一个不存在的（key(path)，value）对的证明，因为root hash最终是基于它下面的所有哈希值，所以任何修改都会改变root hash。  
  
如上所述，在一次一个nibble地遍历路径时，大多数节点（因为是分支节点）包含一个17元素的数组。一个索引表示路径中下一个十六进制字符（nibble）所持有的每个可能的值，还有一个索引表示如果路径已经被完全遍历，则持有最终的目标值。这些17元素的数组节点被称为分支节点  
  
### MPT

然而，radix tries有一个主要的限制：它们的效率很低。如果你只想存储一个(key/path,value)绑定，其中路径（在以太坊state trie的情况下），64个字符长（32个字节中的nibbles数），你将需要超过一千字节的额外空间来存储每个字符的级别，而且每次查找或删除都要花费整整64步。这里介绍的Patricia trie解决了这个问题。

#### 优化

Merkle Patricia试图通过给数据结构增加一些额外的复杂性来解决低效率问题。Merkle Patricia trie中的一个节点是以下之一。  
  
1. NULL (表示为空字符串)  
2. 分支节点 一个17个items的节点 $[ v_0 ... v_{15}, v_t ]$ 。  
3. 叶节点 一个2个items的节点 $[ encodedPath, value ]$ 。  
4. 扩展节点(extension node) 一个2个items的节点 $[ encodedPath, key ]$ 。  

在64个字符的路径中，在遍历trie的前几层后，你将不可避免地到达一个节点，在这个节点上至少有一部分是不存在分支路径的。如果要求这样的节点除了目标索引（路径中的下一个nibble）外，每个索引（16个十六进制字符中的每一个）都有空值，那就太天真了(空间利用率低)。相反，我们通过设置一个形式为$[ encodedPath, key ]$的扩展节点来简化下降过程，其中$encodedPath$包含了向前跳过的 "部分路径"（使用下面描述的紧凑编码），而$key$是用于下一个数据库的查询。  

> 一个索引也就是对应一个nibble，十六进制字符中的每一个。

在叶子节点的情况下，可以通过$encodedPath$的第一个nibble中的标志来确定，上述情况会发生，也可以通过 "部分路径 "来提前跳过完成一个路径的全部剩余部分。在这种情况下，值就是目标值本身。

然而，上面的优化引入了一些不明确的地方。

当以nibbles为单位遍历路径时，我们最终可能会有奇数的nibbles需要遍历，但是因为所有的数据都是以字节格式存储的，所以不可能区分，比如说nibble 1，和nibble 01（两者都必须存储为<01>）。为了指定奇数长度，部分路径的前缀是一个标志。

#### 规范：可选终止符的十六进制序列的紧凑型编码

上述奇数与偶数的剩余部分路径长度和叶子与扩展节点的标记都存在于任何2-items节点的部分路径的第一个nibble中。它们的结果如下。2-items节点只有扩展节点和叶子节点。

![[60e7d16503db2d271f68458ceaede0e.png]]

> 注意，奇数是odd，偶数是even

对于偶数的剩余路径长度（0或2），另一个0的 "padding "nibble将始终跟随。

```python
def compact_encode(hexarray):
	term = 1 if hexarray[-1] == 16 else 0
	if term: hexarray = hexarray[:-1]
	oddlen = len(hexarray) % 2
	flags = 2 * term + oddlen
	if oddlen:
		hexarray = [flags] + hexarray
	else:
		hexarray = [flags] + [0] + hexarray
	// hexarray now has an even length whose first nibble is the flags.
	o = ''
	for i in range(0,len(hexarray),2):
		o += chr(16 * hexarray[i] + hexarray[i+1])
	return o
```

编码例子:

```txt
> [ 1, 2, 3, 4, 5, ...]      # 扩展节点，奇数个nibbles，加入0x1前缀
'11 23 45'
> [ 0, 1, 2, 3, 4, 5, ...]    # 扩展节点，偶数个nibbles，加入0x00前缀
'00 01 23 45'
> [ 0, f, 1, c, b, 8, 10]    # 叶子节点，奇数个nibbles，加入前缀0x20
'20 0f 1c b8'
> [ f, 1, c, b, 8, 10]   # 叶子节点，偶数个niblles，加入前缀0x3
'3f 1c b8'
```

下面是获取Merkle Patricia trie中的一个节点的扩展代码:

```python
def get_helper(node,path):
	if path == []: return node
	if node = '': return ''
	curnode = rlp.decode(node if len(node) < 32 else db.get(node))
	if len(curnode) == 2:
		(k2, v2) = curnode
		k2 = compact_decode(k2)
		if k2 == path[:len(k2)]:
			return get(v2, path[len(k2):])
		else:
			return ''
	elif len(curnode) == 17:
		return get_helper(curnode[path[0]],path[1:])

def get(node,path):
	path2 = []
	for i in range(len(path)):
		path2.push(int(ord(path[i]) / 16)) # 获取一个字节中的高位nibble 比如0x3f中的3
		path2.push(ord(path[i]) % 16) # 获取一个字节中的低位nibble，比如0x3f中的f
	path2.push(16)
	return get_helper(node,path2)
```

#### 一个实际例子

假设我们想要一个包含了四个 (路径/值) 键值对（'do', 'verb'）、（'dog', 'puppy'）、（'doge', 'coin'）、（'horse', 'stallion'）的 trie。

首先，我们将路径和值都转换为字节。下面，路径的实际字节表示法是用<>表示的，尽管值仍然显示为字符串，用' ' 表示，以便于理解（它们实际上也是字节）。

```txt
<64 6f> : 'verb'
<64 6f 67> : 'puppy'
<64 6f 67 65> : 'coin'
<68 6f 72 73 65> : 'stallion'
```

现在，我们用底层的KV存储引擎中的以下键/值对建立这样一个 trie。

```txt
rootHash: [ <16>, hashA ]
hashA:    [ <>, <>, <>, <>, hashB, <>, <>, <>, [ <20 6f 72 73 65>, 'stallion' ],<>, <>, <>, <>, <>, <>, <>, <> ]
hashB:    [ <00 6f>, hashD ]
hashD:    [ <>, <>, <>, <>, <>, <>, hashE, <>, <>, <>, <>, <>, <>, <>, <>, <>,'verb' ]
hashE:    [ <17>, [ <>, <>, <>, <>, <>, <>, [ <35>, 'coin' ], <>, <>, <>, <>, <>, <>, <>, <>, <>, 'puppy' ] ]
```

当一个节点被另一个节点引用时，包括的是H(rlp.encode(x))，其中`H(x)= keccak256(x) if len(x) >= 32 else x`和`rlp.encode`是RLP编码函数。

请注意，当更新一个 trie 时，如果新创建的节点长度 >= 32字节，则需要将键/值对（keccak256(x), x）存储在一个持久的查找表中。然而，如果节点的长度短于此，则不需要存储任何东西，因为函数f(x) = x是可逆的。

### 在ETH中的tries

以太坊执行层中的所有merkle tries都使用Merkle Patricia Trie。

从一个区块头来看，有3个root来自这些trie的3个root。

- stateRoot
- transactionsRoot
- receiptsRoot

#### State trie

有一个全局状态 trie，它随着时间的推移而更新。在其中，一个路径总是：$keccak256(ethereumAddress)$ 也就是以太坊地址的Hash，一个值总是：rlp(ethereumAccount)，也就是以太坊账户的RLP编码。更具体地说，一个以太坊账户是一个$[nonce,balance,storageRoot,codeHash]$的4项数组。在这一点上，值得注意的是，这个$storageRoot$是另一个Patricia trie的根

#### Storage trie

Storage trie是所有合约数据的所在地。每个账户都有一个单独的storage trie。要检索给定地址的特定存储位置的值，需要存储地址、存储数据的整数位置和块ID。这些可以作为参数输入到JSON-RPC API中定义的`eth_getStorageAt`，例如，检索地址为`0x295a70b2de5e3953354a6a8344e616ed314d7251`的存储槽0的数据。

```txt
curl -X POST --data '{"jsonrpc":"2.0", "method": "eth_getStorageAt", "params": ["0x295a70b2de5e3953354a6a8344e616ed314d7251", "0x0", "latest"], "id": 1}' localhost:8545

{"jsonrpc":"2.0","id":1,"result":"0x00000000000000000000000000000000000000000000000000000000000004d2"}
```

检索存储中的其他元素要稍微复杂一些，因为必须首先计算存储trie中的位置。该位置被计算为地址和存储位置的keccak256哈希值，两者都用零填充，长度为32字节。例如，地址为`0x391694e7e0b0cce554cb130d723a9d27458f9298`的存储槽1中的数据位置是。

```txt
curl -X POST --data '{"jsonrpc":"2.0", "method": "eth_getStorageAt", "params": ["0x391694e7e0b0cce554cb130d723a9d27458f9298", "0x1", "latest"], "id": 1}' localhost:8545
```

在Geth控制台中，这可以被计算如下：

```txt
> var key = "000000000000000000000000391694e7e0b0cce554cb130d723a9d27458f9298" + "0000000000000000000000000000000000000000000000000000000000000001"
undefined
> web3.sha3(key, {"encoding": "hex"})
"0x6661e9d6d8b923d5bbaab1b96e1dd51ff6ea2a93520fdc9eb75d059238b8c5e9"
```

因此路径是keccak256(<6661e9d6d8b923d5bbaab1b96e1dd51ff6ea2a93520fdc9eb75d059238b8c5e9>)。现在可以像以前一样从存储trie中检索数据了。

```txt
curl -X POST --data '{"jsonrpc":"2.0", "method": "eth_getStorageAt", "params": ["0x295a70b2de5e3953354a6a8344e616ed314d7251", "0x6661e9d6d8b923d5bbaab1b96e1dd51ff6ea2a93520fdc9eb75d059238b8c5e9", "latest"], "id": 1}' localhost:8545

{"jsonrpc":"2.0","id":1,"result":"0x000000000000000000000000000000000000000000000000000000000000162e"}
```

#### transaction trie

每个区块都有一个单独的交易trie，同样存储着（键，值）对。这里的一个路径是：`rlp(transactionIndex)`，它代表对应于一个由以下因素决定的值的键。

```python
if legacyTx:
  value = rlp(tx)
else:
  value = TxType | encode(tx)
```

更多请参考 [EIP-2718: Typed Transaction Envelope (ethereum.org)](https://eips.ethereum.org/EIPS/eip-2718)


#### receipts trie

每个区块都有自己的Receipts trie。这里的路径是：`rlp(transactionIndex)`。transactionIndex是它在所开采的区块中的索引。Receipts trie从不更新。与Transactions trie类似，也有当前和遗留的收据。要在Receipts trie中查询一个特定的收据，需要其区块中的交易索引、收据有效载荷和交易类型。返回的收据可以是**Receipt类型**，定义为交易类型和交易payload的集合，也可以是**LegacyReceipt类型**，定义为$rlp（[status, cumulativeGasUsed, logsBloom, logs]）$。

更多请参考 [EIP-2718: Typed Transaction Envelope (ethereum.org)](https://eips.ethereum.org/EIPS/eip-2718)

## 以太坊MPT详解

### 介绍

Merkle Patricia Trie(MPT)是以太坊存储层的关键数据结构之一。我想了解它到底是如何工作的。所以我对我能找到的所有材料进行了深入研究，并自己实现了这个算法。

在这个区块链帖子中，我将分享我所学到的东西。解释Merkle Patricia Trie到底是如何工作的，并向你展示一个Merkle证明是如何生成和验证的演示。

该算法的源代码和本博文中使用的例子都是开源的。

[zhangchiqing/merkle-patricia-trie: A simplified golang implementation of Ethereum's Modified Patricia Trie. (github.com)](https://github.com/zhangchiqing/merkle-patricia-trie)

让我们开始吧。

### 一个基本的KV映射

以太坊的Merkle Patricia Trie本质上是一种键值映射，提供以下标准方法:

```go
type Trie interface {
  // methods as a basic key-value mapping
  Get(key []byte) ([]byte, bool) {
  Put(key []byte, value []byte)
  Del(key []byte, value []byte) bool
}
```

上述Trie接口的实现应该通过以下测试案例：

```go
func TestGetPut(t *testing.T) {
	t.Run("should get nothing if key does not exist", func(t *testing.T) {
		trie := NewTrie()
		_, found := trie.Get([]byte("notexist"))
		require.Equal(t, false, found)
	})

	t.Run("should get value if key exist", func(t *testing.T) {
		trie := NewTrie()
		trie.Put([]byte{1, 2, 3, 4}, []byte("hello"))
		val, found := trie.Get([]byte{1, 2, 3, 4})
		require.Equal(t, true, found)
		require.Equal(t, val, []byte("hello"))
	})

	t.Run("should get updated value", func(t *testing.T) {
		trie := NewTrie()
		trie.Put([]byte{1, 2, 3, 4}, []byte("hello"))
		trie.Put([]byte{1, 2, 3, 4}, []byte("world"))
		val, found := trie.Get([]byte{1, 2, 3, 4})
		require.Equal(t, true, found)
		require.Equal(t, val, []byte("world"))
	})
}
```

https://github.com/zhangchiqing/merkle-patricia-trie/blob/6e11cbee5b1d839c0c60cc39a2badb912355d990/trie_test.go#L15-L38

从接口实现的功能上看，其实就是最简单的方式就是实现了一个内存map，内存map多了去了，有hash table，trie等等。目前我们把它看成一个最简单的内存trie的实现。

### 校验数据完整性

merkle patricia trie与标准映射有什么不同？

嗯，merkle patricia trie允许我们验证数据的完整性。(在本教程的其余部分，为了简单起见，我们将称它为trie。）

我们可以用Hash函数计算trie的Merkle root hash，这样，如果任何键值对被更新，trie的Merkle根哈希值就会不同；如果两个Tries有相同的所有键值对，它们应该有相同的Merkle根哈希。

```go
type Trie interface {
  // compute the merkle root hash for verifying data integrity
  Hash() []byte
}
```

让我们用一些测试案例来解释这种行为

```go
// verify data integrity
func TestDataIntegrity(t *testing.T) {
	t.Run("should get a different hash if a new key-value pair was added or updated", func(t *testing.T) {
		trie := NewTrie()
		hash0 := trie.Hash()

		trie.Put([]byte{1, 2, 3, 4}, []byte("hello"))
		hash1 := trie.Hash()

		trie.Put([]byte{1, 2}, []byte("world"))
		hash2 := trie.Hash()

		trie.Put([]byte{1, 2}, []byte("trie"))
		hash3 := trie.Hash()

		require.NotEqual(t, hash0, hash1)
		require.NotEqual(t, hash1, hash2)
		require.NotEqual(t, hash2, hash3)
	})

	t.Run("should get the same hash if two tries have the identicial key-value pairs", func(t *testing.T) {
		trie1 := NewTrie()
		trie1.Put([]byte{1, 2, 3, 4}, []byte("hello"))
		trie1.Put([]byte{1, 2}, []byte("world"))

		trie2 := NewTrie()
		trie2.Put([]byte{1, 2, 3, 4}, []byte("hello"))
		trie2.Put([]byte{1, 2}, []byte("world"))

		require.Equal(t, trie1.Hash(), trie2.Hash())
	})
}
```

https://github.com/zhangchiqing/merkle-patricia-trie/blob/6e11cbee5b1d839c0c60cc39a2badb912355d990/trie_test.go#L41-L71

### 校验是否包含一个键值对

是的，trie可以验证数据的完整性，但为什么不简单地通过对整个键值对列表进行哈希比较，为什么要费力地创建一个trie数据结构呢？$hash(kv_0 | kv_1 | ... | kv_n)$ vs. MPT

**这是因为trie还允许我们在不访问整个键值对的情况下验证键值对的包含性。** 就是某个键值对是否包含在这个trie的root hash下面。

这意味着trie可以提供一个证明，证明某个键值对包含在产生某个merkle root hash的键值映射中。

```go
type Proof interface {}

type Trie interface {
  // generate a merkle proof for a key-value pair for verifying the inclusion of the key-value pair
  Prove(key []byte) (Proof, bool)
}

// verify the proof for the given key with the given merkle root hash
func VerifyProof(rootHash []byte, key []byte, proof Proof) (value []byte, err error)
```

这在以太坊中是很有用的。例如，设想以太坊世界状态是一个键值映射，**键是每个账户地址，值是每个账户的余额。** （这个举例跟实际的以太坊有点不一致，所以是举例）

作为一个轻客户端，它不像全节点那样可以访问完整的区块链状态，而只是访问某些区块的merkle root hash，那么它怎么能相信全节点返回的账户余额结果呢？

答案是，全节点可以提供一个merkle证明，其中包含merkle根哈希值、账户的键和其余额值，以及其他数据。这个merkle证明允许轻量级客户通过自己的方式来验证正确性，而无需访问完整的区块链状态。

让我们用测试案例来解释这种行为。

```go
func TestProveAndVerifyProof(t *testing.T) {
	t.Run("should not generate proof for non-exist key", func(t *testing.T) {
		tr := NewTrie()
		tr.Put([]byte{1, 2, 3}, []byte("hello"))
		tr.Put([]byte{1, 2, 3, 4, 5}, []byte("world"))
		notExistKey := []byte{1, 2, 3, 4}
		_, ok := tr.Prove(notExistKey)
		require.False(t, ok)
	})

	t.Run("should generate a proof for an existing key, the proof can be verified with the merkle root hash", func(t *testing.T) {
		tr := NewTrie()
		tr.Put([]byte{1, 2, 3}, []byte("hello"))
		tr.Put([]byte{1, 2, 3, 4, 5}, []byte("world"))

		key := []byte{1, 2, 3}
		proof, ok := tr.Prove(key)
		require.True(t, ok)

		rootHash := tr.Hash()

		// verify the proof with the root hash, the key in question and its proof
		val, err := VerifyProof(rootHash, key, proof)
		require.NoError(t, err)

		// when the verification has passed, it should return the correct value for the key
		require.Equal(t, []byte("hello"), val)
	})

	t.Run("should fail the verification if the trie was updated", func(t *testing.T) {
		tr := NewTrie()
		tr.Put([]byte{1, 2, 3}, []byte("hello"))
		tr.Put([]byte{1, 2, 3, 4, 5}, []byte("world"))

		// the hash was taken before the trie was updated
		rootHash := tr.Hash()

		// the proof was generated after the trie was updated
		tr.Put([]byte{5, 6, 7}, []byte("trie"))
		key := []byte{1, 2, 3}
		proof, ok := tr.Prove(key)
		require.True(t, ok)

		// should fail the verification since the merkle root hash doesn't match
		_, err := VerifyProof(rootHash, key, proof)
		require.Error(t, err)
	})
}
```

https://github.com/zhangchiqing/merkle-patricia-trie/blob/6e11cbee5b1d839c0c60cc39a2badb912355d990/proof_test.go#L37-L84

一个轻量级的客户可以要求得到一个 state trie的 merkle root hash，并使用它来验证其账户的余额。如果trie被更新了，即使更新的是其他键值，那么验证也会失败。

而现在，轻型客户只需要相信Merkle root hash，这是一个小的数据，以说服他们自己全节点是否为其账户返回了正确的余额。

好吧，但为什么轻客户端要相信merkle root hash呢？

因为以太坊的共识机制是工作证明，而世界状态的Merkle根哈希值包含在每个区块头中，计算工作就是验证/信任Merkle根哈希值的证明。

这样小的Merkle根哈希值可以用来验证一个巨大的键值映射的状态，这是非常酷的。

### 验证实现

我已经解释了merkle patricia trie的工作原理。这个 repo 提供了一个简单的实现。但是，我们怎样才能验证我们的实现呢？

一个简单的方法是用Ethereum主网数据和官方Trie golang实现来验证。

以太坊有3个Merkle Patricia Tries。`Transaction Trie`、`Receipt Trie`和`State Trie`。在每个区块头中，它包括3个Merkle根哈希值：`TransactionRoot`、`ReceiptRoot`和`StateRoot`。

其实还有`StorageRoot`，这个root hash相对不是那么重要，因为它没被包含在区块头中。

我们重点讲解前三个MPT的root hash。

由于`transactionRoot`是区块中所有交易的Merkle root hash，我们可以通过获取所有交易来验证我们的实现，然后将它们存储在我们的trie中，计算其Merkle root hash，最后与区块头中的transactionRoot进行比较。

例如，我在mainnet上挑选了10467135区块 [Ethereum Blocks #10467135 | Etherscan](https://etherscan.io/block/10467135) ，并将所有193个交易保存到`transactions.json`文件中。

由于块10467135的`TransactionRoot`是`0xbb345e208bda953c908027a45aa443d6cab6b8d2fd64e83ec52f1008ddeafa58`。我可以创建一个测试案例，将10467135区块的193个交易添加到我们的Trie中，并检查Merkle根哈希值是否为`0xbb345e208bda953c908027a45aa443d6cab6b8d2fd64e83ec52f1008ddeafa58`。

从我们的trie实现中产生的某个交易的merkle证明是否可以被官方实现所验证。
但是，交易列表的键和值是什么？键是无符号整数的RLP编码，从索引0开始；值是相应交易的RLP编码。

好吧，让我们看看测试案例。

```go
import (
	"github.com/ethereum/go-ethereum/common"
	"github.com/ethereum/go-ethereum/core/types"
	"github.com/ethereum/go-ethereum/trie"
)
// use the official golang implementation to check if a valid proof from our implementation can be accepted
func VerifyProof(rootHash []byte, key []byte, proof Proof) (value []byte, err error) {
	return trie.VerifyProof(common.BytesToHash(rootHash), key, proof)
}
// load transaction from json
func TransactionJSON(t *testing.T) *types.Transaction {
	jsonFile, err := os.Open("transaction.json")
	defer jsonFile.Close()
	require.NoError(t, err)
	byteValue, err := ioutil.ReadAll(jsonFile)
	require.NoError(t, err)
	var tx types.Transaction
	json.Unmarshal(byteValue, &tx)
	return &tx
}
func TestTransactionRootAndProof(t *testing.T) {
	trie := NewTrie()
	txs := TransactionsJSON(t)
	for i, tx := range txs {
		// key is the encoding of the index as the unsigned integer type
		key, err := rlp.EncodeToBytes(uint(i))
		require.NoError(t, err)
		transaction := FromEthTransaction(tx)
		// value is the RLP encoding of a transaction
		rlp, err := transaction.GetRLP()
		require.NoError(t, err)
		trie.Put(key, rlp)
	}
	// the transaction root for block 10467135
	// https://api.etherscan.io/api?module=proxy&action=eth_getBlockByNumber&tag=0x9fb73f&boolean=true&apikey=YourApiKeyToken
	transactionRoot, err := hex.DecodeString("bb345e208bda953c908027a45aa443d6cab6b8d2fd64e83ec52f1008ddeafa58")
	require.NoError(t, err)
	t.Run("merkle root hash should match with 10467135's transactionRoot", func(t *testing.T) {
		// transaction root should match with block 10467135's transactionRoot
		require.Equal(t, transactionRoot, trie.Hash())
	})
	t.Run("a merkle proof for a certain transaction can be verified by the offical trie implementation", func(t *testing.T) {
		key, err := rlp.EncodeToBytes(uint(30))
		require.NoError(t, err)
		proof, found := trie.Prove(key)
		require.Equal(t, true, found)
		txRLP, err := VerifyProof(transactionRoot, key, proof)
		require.NoError(t, err)
		// verify that if the verification passes, it returns the RLP encoded transaction
		rlp, err := FromEthTransaction(txs[30]).GetRLP()
		require.NoError(t, err)
		require.Equal(t, rlp, txRLP)
	})
}
```

https://github.com/zhangchiqing/merkle-patricia-trie/blob/6e11cbee5b1d839c0c60cc39a2badb912355d990/transaction_test.go#L20-L65

上述测试案例通过了，并且表明如果我们将10467135区块的所有193个交易加入到我们的trie中，那么trie的哈希值与该区块中公布的TransactionRoot相同。而由我们的trie生成的索引为30的交易的merkle证明，被golang官方的trie实现认为是有效的。

### MPT内部实现之trie的节点

现在，让我们来看看MPT的内部。

在内部，MPT有4种类型的节点: 空节点、叶节点、分支节点和扩展节点。每个节点将被编码并作为键值对存储在KV存储引擎中。

作为一个例子，让我们看看mainnet上的10593417区块[Ethereum Blocks #10593417 | Etherscan](https://etherscan.io/block/10593417)，以显示一个transaction trie是如何建立的，以及它是如何存储的。

10593417区块只有4个交易的transactionRoot hash。`0xab41f886be23cd786d8a69a72b0f988ea72e0b2e03970d0798f5e03763a442cc`. 因此，为了将4个交易存储到一个 trie，我们实际上是以十六进制字符串的形式存储以下键值对。

```txt
(80, f8ab81a5852e90edd00083012bc294a3bed4e1c75d00fa6f4e5e6922db7261b5e9acd280b844a9059cbb0000000000000000000000008bda8b9823b8490e8cf220dc7b91d97da1c54e250000000000000000000000000000000000000000000000056bc75e2d6310000026a06c89b57113cf7da8aed7911310e03d49be5e40de0bd73af4c9c54726c478691ba056223f039fab98d47c71f84190cf285ce8fc7d9181d6769387e5efd0a970e2e9)

(01, f8ab81a6852e90edd00083012bc294a3bed4e1c75d00fa6f4e5e6922db7261b5e9acd280b844a9059cbb0000000000000000000000008bda8b9823b8490e8cf220dc7b91d97da1c54e250000000000000000000000000000000000000000000000056bc75e2d6310000026a0d77c66153a661ecc986611dffda129e14528435ed3fd244c3afb0d434e9fd1c1a05ab202908bf6cbc9f57c595e6ef3229bce80a15cdf67487873e57cc7f5ad7c8a)

(02, f86d8229f185199c82cc008252089488e9a2d38e66057e18545ce03b3ae9ce4fc360538702ce7de1537c008025a096e7a1d9683b205f697b4073a3e2f0d0ad42e708f03e899c61ed6a894a7f916aa05da238fbb96d41a4b5ec0338c86cfcb627d0aa8e556f21528e62f31c32f7e672)

(03, f86f826b2585199c82cc0083015f9094e955ede0a3dbf651e2891356ecd0509c1edb8d9c8801051fdc4efdc0008025a02190f26e70a82d7f66354a13cda79b6af1aa808db768a787aeb348d425d7d0b3a06a82bd0518bc9b69dc551e20d772a1b06222edfc5d39b6973e4f4dc46ed8b196)
```

`80`是无符号整数`0`的RLP编码结果的字节的十六进制形式：`RLP(uint(0))`。`01`是RLP(uint(1))的结果，以此类推。

键`80`的值是第一笔交易的RLP编码的结果。键`01`的值是第二个交易的值，以此类推。

因此，我们将把上述4个键值对添加到MPT中，让我们看看添加每一个键值对时，MPT的内部结构如何变化。

为了更直观，我将用一些图来解释它的工作原理。你也可以通过在测试案例中添加日志来检查每一步的状态。

#### 空trie

trie结构只包含一个根字段，指向一个根节点。而节点类型是一个接口，它可以是4种类型的节点之一。

```go
type Trie struct {
	root Node
}
```

当一个 trie 被创建时，根节点指向一个 EmptyNode。

![[Pasted image 20220927105034.png]]

相当于空的MPT是有两个节点，一个根节点，指向一个空的节点，root指针是hash，在KV引擎中root指向一个keccak(RLP(null))的value。

#### 加入第一个交易

当添加第一个交易的键值对时，一个LeafNode被创建，交易数据存储在其中。根节点被更新以指向该LeafNode。

![[Pasted image 20220927142407.png]]

从上图可以看到，root hash 在KV中指向hash1，hash1指向该第一个（索引为0）交易的LeafNode的RLP编码。拿到root hash读取到hash1，再由hash1去KV引擎中去查找，然后解码。注意这里的在KV引擎中的RLP编码，除了交易本身，已经把交易索引也编码成RLP了。你可以看到`80`这个数字。

可以看到LeafNode就两个字段，一个路径，一个value本身。路径就是现在不是`80`这两个nibbles。

#### 加入第二个交易

当添加第2个交易时，根部的LeafNode将变成一个BranchNode，有两个分支指向2个LeafNode。右边的LeafNode持有剩余的nibbles（nibbles是单个十六进制字符）-1，以及第2个交易的值。

而现在，根节点指向了新的BranchNode。

![[Pasted image 20220927144059.png]]
可以看到，第一个交易的nibbles组成的路径是`80`， 第二个交易的nibbles组成的路径是`01`，当然，这些nibbles是索引0和1经过RLP编码之后的。

同时也可以看到，BranchNode，只有一个成员，这个成员是一个16个元素的数组。这个数组就是存储`0~f`这些16进制的16个nibbles的数组了。

第一个交易的path，也就是trie中的key是`80`，所以先找到8这个索引，这个索引下存储着第一个交易的LeafNode的hash1，拿到hash1，就可以去KV引擎里面拿到第一个交易的LeafNode的编码RLP(0,f8ab81a5....), RLP解码出来，发现LeafNode上的第一个成员是不是`0`这个剩余的path nibble，如果是，就返回正确，不是，就说明没找到。

第二个交易的path，也就是trie中的key是`01`，所以先找到0这个索引，0这个索引存放的是hash2，拿到hash2，就可以去KV引擎里面拿到第二个交易的LeafNode的编码RLP(1,f8ab81a6...)，然后RLP解码出来，发现LeafNode上的第一个成员是不是`1`这个剩余的path nibble，如果是就返回正确，不是，就说明没找到。

![[Pasted image 20220927160359.png]]

#### 加入第三个交易

添加第3个交易将使左侧的LeafNode变成一个BranchNode，与添加第2个交易的过程类似。虽然根节点没有改变，但它的根的哈希值已经改变了，因为它的索引0分支指向不同的节点（从LeafNode变到），有不同的哈希值。

![[Pasted image 20220927161646.png]]
![[Pasted image 20220927161655.png]]
这第三个交易大部分就同第二个交易一样了，因为第二个和第三个交易的，RLP编码的索引path `01`和`02`有公共的前缀`0`，所以主要是左边的BranchNode有变化。依次类推。

#### 加入第四个交易

添加最后一个交易与添加第三个交易类似。现在我们可以验证根哈希值是否与区块中包含的transactionRoot相同。

![[Pasted image 20220927162009.png]]

![[Pasted image 20220927162016.png]]


#### 获取第三个交易的Merkle proof

第3笔交易的Merkle proof只是通往存储第3笔交易值的LeafNode的路径，这个路径就是`02`。在VerifyProof时，根据函数的参数 $VerifyProof(rootHash, needtoverified key/path, proof)$ 可以从根哈希值开始，解码Node，匹配nibbles，然后重复，直到找到匹配所有剩余nibbles的Node。如果找到了，那么这个值就是与密钥配对的那个值；如果没有找到，那么Merkle Proof就无效了。

这里的前提就是，验证方也要有整个Merkle trie，不然无法验证。

### 更新trie的规则

在上面的例子中，我们已经建立了一个有3种类型节点的MPT。空节点、叶节点和分支节点。然而，我们没有机会使用扩展节点(ExtensionNode)。请找到其他使用ExtensionNode的测试案例。

一般来说，规则是:

- 当停在一个EmptyNode时，用一个新的LeafNode替换它的剩余路径。
- 当停在一个LeafNode时，将其转换为ExtensionNode并添加一个新的分支和一个新的LeafNode。
- 当停在一个扩展节点时，将其转换为另一个路径较短的扩展节点，并创建一个新的分支节点指向该扩展节点。

详情可以看以下代码 https://github.com/zhangchiqing/merkle-patricia-trie/blob/6e11cbee5b1d839c0c60cc39a2badb912355d990/trie.go#L9-L202

### 总结

Merkle Patricia Trie是一个存储键值对的数据结构，就像一个map。除此之外，它还允许我们验证数据的完整性和键值对的包含性。


## 深入以太坊的世界状态

### 什么是区块链的状态

### trie树

[[理解Trie树]]

### Patricia trie

[[理解Radix树]]

### 近距离查看以太坊的trie结构

对于没有采用账户模型的状态的BTC来说，其实就是UTXO，这些未花费和已花费的output了。但是对于以太坊采用账户余额模型的，就不一样了。

以太坊的世界状态能够管理账户余额，以及更多。以太坊的状态不是一个抽象的概念。它是Ethereum基础层协议的一部分。正如黄皮书所提到的，以太坊是一个基于交易的 "状态 "机；所有基于交易的状态机概念都可能建立在这一技术之上。

让我们从头说起。与所有其他区块链一样，以太坊区块链的生命始于它自己的创世区块。从这一点（0区块的创世状态）开始，交易、合约和挖矿等活动将不断地改变以太坊区块链的状态。在以太坊中，一个例子是账户余额（存储在state trie中），每次与该账户有关的交易发生了，都会发生变化。

重要的是，像账户余额这样的数据并不直接存储在以太坊区块链的区块头中。只有交易trie、state trie和receipts trie的root hash直接存储在区块头中。这在下图中得到了说明。

![[Pasted image 20220929094919.png]]

你还会注意到，从上图中，storage trie的root hash（所有的智能合约数据都保存在这里）实际上指向了state trie，而state trie又被Hash后并存储在块头中。我们很快详细介绍这一切。

在以太坊有两种截然不同的数据类型；永久数据和短暂数据。永久数据的一个例子是交易。一旦一个交易被完全确认，它就被记录在transaction trie中；它永远不会被改变。短暂数据的一个例子是一个特定的以太坊账户地址的余额。帐户地址的余额存储在state trie 中，每当针对该特定帐户的交易发生时就会被改变。永久性数据，如挖出的交易，和短暂性数据，如账户余额，应该分开存储，这是合理的。以太坊使用 trie 数据结构（如上所述），来管理数据。这里有一个关于tries的快速概述。

下面我们直接来看看state trie storage trie  transaction trie到底是什么

#### State trie--有且仅有一个

在以太坊中，有且只有一个，全局的state trie。

这个全局的state trie是不断更新的。

state trie包含了以太坊网络上存在的每个账户的键值对。

键是一个单一的160bit的标识符（一个以太坊账户的地址）。

全局state trie 中的 "值 "是通过对一个以太坊账户的以下账户详情进行编码而创建的（使用递归长度前缀编码（RLP）方法）。 ^86aa4e

- nonce
- balance
- storageRoot
- codeHash

可以看到，有账户余额，有账户关联的storage root，相当于每个账户都有一个属于自己的storage root等等。 ^13e6a8

state trie 的root hash（整个state trie 在特定时间点的哈希值）被用作state trie 的安全和唯一标识符；state trie 的root hash在加密上依赖于所有内部state trie 数据。

![[Pasted image 20220929095836.png]]

![[Pasted image 20220929095903.png]]

如果好奇，MPT的内部节点怎样与KV存储引擎结合，那么请看 [[#MPT内部实现之trie的节点]] 

#### Storage trie----合约的数据在哪

^9b422e

之前提到了每一个账户都有自己的Storage trie [[#^13e6a8]]  。

一个Storage trie是所有合约数据的所在。每个以太坊账户都有自己的Storage trie。Storage trie的root hash是256位的哈希值被存储为全局state trie（我们刚刚讨论过 [[#^86aa4e]]）的storageRoot值。

![[Pasted image 20220929101132.png]]

#### Transaction trie---在每一个区块中都有一个

每个以太坊区块都有自己独立的交易trie。一个区块包含许多交易。当然，一个区块中的交易顺序是由组装区块的矿工决定的。通往交易trie中的特定交易的路径，是通过交易在区块中的索引（RLP编码）来实现的。已被矿工开采的区块从不更新；交易在区块中的位置从不改变。这意味着，一旦你在区块的交易trie中找到一个交易，你就可以一次又一次地输入相同的路径来检索相同的结果。

![[Pasted image 20220929101411.png]]


### 以太坊中trie的具体例子

主要的Ethereum客户端使用两种不同的数据库软件解决方案来存储他们的尝试。Ethereum的Rust客户端Parity（这个Parity就是做波卡和substrate那个公司的名称，substrate使用的是RocksDB）使用rocksdb。而Ethereum的Go、C++和Python客户端都使用leveldb。

#### 以太坊和Rocksdb

这个不过多讨论，RocksDB也是基于levelDB改造的高性能数据库。

#### 以太坊和levelDB

levelDB也不过多讲了，也是KV存储引擎的一种，Google Jeff Dean编写。

### 分析以太坊的数据库

可以用web3.js

#### 安装npm,node,level和ethereum.js

```bash
sudo apt-get update
sudo apt-get upgrade
curl -sL https://deb.nodesource.com/setup_9.x | sudo -E bash - sudo apt-get install -y nodejs
sudo apt-get install nodejs
npm -v
nodejs -v
npm install levelup leveldown rlp merkle-patricia-tree --save
git clone https://github.com/ethereumjs/ethereumjs-vm.git
cd ethereumjs-vm
npm install ethereumjs-account ethereumjs-util --save
```

```js
//Just importing the requirements
var Trie = require('merkle-patricia-tree/secure');
var levelup = require('levelup');
var leveldown = require('leveldown');
var RLP = require('rlp');
var assert = require('assert');

//Connecting to the leveldb database
var db = levelup(leveldown('/home/timothymccallum/gethDataDir/geth/chaindata'));

//Adding the "stateRoot" value from the block so that we can inspect the state root at that block height.
var root = '0x8c77785e3e9171715dd34117b047dffe44575c32ede59bde39fbf5dc074f2976';

//Creating a trie object of the merkle-patricia-tree library
var trie = new Trie(db, root);

//Creating a nodejs stream object so that we can access the data
var stream = trie.createReadStream()

//Turning on the stream (because the node js stream is set to pause by default)
stream.on('data', function (data){
  //printing out the keys of the "state trie"
  console.log(data.key);
});
```

#### 解码数据

```js
//Mozilla Public License 2.0 
//As per https://github.com/ethereumjs/ethereumjs-vm/blob/master/LICENSE
//Requires the following packages to run as nodejs file https://gist.github.com/tpmccallum/0e58fc4ba9061a2e634b7a877e60143a

//Getting the requirements
var Trie = require('merkle-patricia-tree/secure');
var levelup = require('levelup');
var leveldown = require('leveldown');
var utils = require('ethereumjs-util');
var BN = utils.BN;
var Account = require('ethereumjs-account');

//Connecting to the leveldb database
var db = levelup(leveldown('/home/timothymccallum/gethDataDir/geth/chaindata'));

//Adding the "stateRoot" value from the block so that we can inspect the state root at that block height.
var root = '0x9369577baeb7c4e971ebe76f5d5daddba44c2aa42193248245cf686d20a73028';

//Creating a trie object of the merkle-patricia-tree library
var trie = new Trie(db, root);

var address = '0xccc6b46fa5606826ce8c18fece6f519064e6130b';
trie.get(address, function (err, raw) {
    if (err) return cb(err)
    //Using ethereumjs-account to create an instance of an account
    var account = new Account(raw)
    console.log('Account Address: ' + address);
    //Using ethereumjs-util to decode and present the account balance 
    console.log('Balance: ' + (new BN(account.balance)).toString());
})
```