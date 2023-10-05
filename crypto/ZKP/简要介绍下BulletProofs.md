之前我们提到的非交互式零知识证明都是从Sigma Protocol来的，这个是ZKP里面比较简单的构造零知识证明的手段，首先设计一个交互式协议，再用Fiat-Shamir变换(Fiat-Shamir heuristic)把交互式变成非交互式(NIZK)，commit challange prove就是它的核心三步骤。现在要讲的防弹证明(BulletProofs)是比较新的一种零知识证明算法。它有两大应用就是：

- 做范围证明(Range Proof)，也就是证明一个值在某个给定范围内，但是不泄露这个具体的值(witness)，这里就是关于这个值是零知识的。这里就可以应用在Confidential Transaction上面，防止泄露转账金额。
- 作为Sigma Protocol的一般替代算法

对比当今的zk-SNARKs和zk-STARK系的ZKP有什么优缺点:

优点：

- 对比zk-SNARKs，它不需要可信设置(trusted setup)
- 对比zk-STARK，它的生成证明的数据大小比较小，大概可以小到1kb以下，Proof size与witess成对数关系 O(log(witness))
- 支持证明数据的聚合(aggregation)， 也就是把多个证明合并在一个单个证明字段中，这样就减小了存储的占用，对于BTC这样UTXO很多的链来说，能省不少存储。

缺点：

- 对比zk-SNARKs，它的verify证明的过程，需要消耗更多的时间

## References:

- https://crypto.stanford.edu/bulletproofs/