Radix树，中文叫做基数树，有时候也叫Radix trie，压缩前缀树，压缩trie树。之前我们学过trie树了，[[理解Trie树]] 了解到了其内存占用大的特点，因为一个字符占一个节点，空间利用率太低了，会造成很多浪费。这个数据结构相当于尽可能把相同的前缀字符串压缩了。

Radix Tree是这样的，树中的一个节点是父节点的唯一子节点(the only child)的话，那么该子节点将会与父节点进行合并，这样就使得radix tree中的每一个内部节点最多拥有`r`个孩子， `r`为正整数且等于`2^n`(n>=1)。不像是一般的trie树，radix tree的边沿(edges)可以是一个或者多个元素。这里的`r`就是基数的缩写。参看如下

![[Pasted image 20220926091847.png]]

看上图就可以了解到，其实Raxdix Tree就是把单一路径的节点压缩成一条边。

不像是平常的树结构（在进行key的比较时，是整个key从头到尾进行比较），radix key在每个节点进行key的比较时是以`bit块`(chunk-of-bits)为单位来进行的，每一个chunk中的bit数目等于`radix tree`的基数`r`。

### 搜索

查找操作用于判定一个字符串是否存在于`trie`树中。本操作与普通的`trie树`查找相似，除了某些`edges`可能会包含多个元素之外。

Edge的结构：

![[Pasted image 20220926093300.png]]
Node结构:

```cpp
struct Node{
	struct Edge* edges[];
	fn is_leaf();
};
```

![[Pasted image 20220926092551.png]]

### 插入操作

要插入一个字符串，首先我们需要执行查找，直到查找到某个节点处停止。此时，我们或者可以直接将`input string`的剩余部分添加到一个新的`Edge`中； 或者有一个edge与我们的`input string`的剩余部分有相同的前缀，此种情况下我们需要将该`edge`分裂成两个`edge`（其中第一个edge存储common prefix）然后再进行处理。这里分裂edge确保了一个节点的孩子(children)数目不会超过总的字符串的个数。

如下我们展示了插入时的多种情况（仍有部分未列出）。下图中`r`代表的是根节点（root)。这里我们假设`edge`以空字符串(empty string)作为结尾。

![[Pasted image 20220926093329.png]]
### 删除

要从radix tree中删除一个字符串`x`，我们必须首先定位到代表`x`的叶子节点。假设该叶子节点存在，我们就移除该叶子节点。此时，我们需要检查该被删节点的父节点，假如父节点只剩下一个孩子节点的话，则将该孩子节点的`incomming label`追加到父节点的`incomming label`上并将该孩子节点移除。

### 分析

radix tree允许在O(k)的时间复杂度内进行查找、插入、删除等操作，而平衡树在进行这些操作时时间复杂度一般为`O(logn)`。这看起来并不像是一个优势，因为通常情况下`k>=log(n)`，但是在平衡树中key的比较通常是字符串的完整比较，时间复杂度是`O(k)`，并且在实际使用过程中很多key可能还拥有相同的前缀。假如使用`trie`数据结构来进行存储，所有key的比较都是`constant time`，但是如果我们查询的字符串长度为`m`，那么也必须执行`m`次的比较。使用`radix tree`数据结构时，则需要更少的比较操作，并且使用的节点个数也更少。

当然，radix tree也与`trie`树一样有相同的劣势，然而：因为它们只能被应用于字符串元素或者能够高效映射为字符串的元素，它们缺少了平衡查找树那样的通用性，平衡查找树可以应用于任何类型以及任何比较顺序。

### Radix Tree的变种

PATRICIA trie是radix 2 trie(Radix二叉trie)的一个特殊变体，其中节点不是明确地存储每个key的每一个bit，而是只存储区分两个子树的第一个bit的位置。在遍历过程中，该算法检查search key的索引bit，并根据情况选择左边或右边的子树。PATRICIA trie的显著特点包括：该tree只需要为每一个存储的唯一key插入一个节点，这使得PATRICIA比标准的二叉trie更加紧凑。另外，由于实际的key不再被明确地存储，因此有必要对索引的记录进行一次全key比较，以确认匹配。在这方面，PATRICIA与使用哈希表的索引有一定的相似之处。这里的二进制bit比较方式就是原论文提到的方式。

当然，你就可以认为Patricia Trie是Radix tree的一个特例。

参考论文:

- PATRICIA—Practical Algorithm To Retrieve Information Coded in Alphanumeric
- Patricia Tries: A Better Index For Prefix Searches

