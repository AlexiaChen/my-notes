#### Solana的基本原理

Solana扩容主要立足于：高效利用网络带宽、减少节点间通讯次数、提高交易执行速度这3方面。为了透彻理解Solana，本Thread将先从投票机制说起，先理解POS共识下怎么敲定新区块，再去看Solana的创新。

1.共识投票：在采用BFT（拜占庭容错）的POS共识中，节点们要对新区块进行投票，区块收集了占2/3权重的节点给出的投票才能上链，这一举措主要是为了防止链条分叉。

由于POS共识中出块节点不需要多少算力就可以产生一个区块，它可能会同时生产好几个版本的区块。如果其他节点同时认可不同版本的区块给出投票，就会导致网络分叉，这种现象叫“无利害攻击”。

![[Pasted image 20220812140817.png]]

2.为了解决无利害攻击问题，采用POS的共识协议会引入惩罚机制，如果一个节点在某个区块高度上向多个不同版本的区块给出投票，就会受到惩罚（这种左右逢源的行为被诚实节点观察到，就会在记录全网Stake质押状态的合约里扣掉作恶节点的一部分资产）。

如果网络内有N个节点，每个节点要与其他N-1个节点交换投票，共识通讯次数会与N^2（N的平方）成正比，显然开销巨大。所以大多数PBFT算法无法让太多节点参与到共识过程，一次只能维持100多个节点，这不利于去中心化。

![[Pasted image 20220812140846.png]]

4.公开的出块者：Solana和Avalanche对节点投票问题进行了大幅修正。Solana在每个时间段的出块者Leader都是提前选好且公开的，这与Cosmos的Tendermint共识算法类似。Leader具有很高的权力，负责汇总并广播其他节点的共识投票、提前接收网络内全部的交易等。

![[Pasted image 20220812140905.png]]

5.验证节点只需要把自己的投票结果单独发给Leader，Leader汇总了所有节点的投票后，会捆绑成一个数据包，一次性发送到网络当中，其他节点很快就能收到这个投票汇总数据包。这就好像一个村落里有了大喇叭后，村民很快就能获知消息，效率会得到大幅提升。

Solana中该过程的通讯复杂度往往与节点数量N或其对数Log N成正比，远小于普通PBFT算法的N^2。

![[Pasted image 20220812140936.png]]

6.Solana所有节点都能知道未来几天内的出块者Leader分别是谁，每个Leader任期为4个Slot（区块时长），一个Slot约为400ms~ 800ms。除了简化投票流程，Solana通过提前确定出块者，目的在于配合其“无内存池的交易传输协议”—Gulf Stream。

![[Pasted image 20220812140958.png]]

7.Gulf Stream：用户发起交易后，直接被Solana的RPC节点提交给当前的出块者Leader，随即被Leader处理。这种做法可以大幅提升交易传播效率，往往只需要1~3秒，一笔交易就会被出块节点获取。

![[Pasted image 20220812141018.png]]

换作以太坊等无法预知出块者的网络，一笔交易要通过“流言协议”，以口耳相传的形式被节点传播给其他节点，最终被所有节点获取。这种传播方式速度很慢，由于受到光纤等物理层因素限制，一笔交易传遍以太坊网络需要6s~12s，这意味着出块的节点可能要很久才收到待处理的交易。

8.但Solana的这种机制有利有弊，好处在于提高了交易提交速度，可以让出块者尽快执行交易，坏处则是让Leader的压力太大，许多未经其他节点过滤和处理的交易都汇集到了Leader处，在极端情况下会诱使Leader宕机。比如，采取了相同交易传播机制的Arbitrum曾因出块者收到太多未经过滤的交易事件而崩溃。

同时，Solana用UDP协议传输交易，UDP虽然快但是不安全，很容易发生丢包等问题，许多用户发起交易后却屡次失败，可能因为交易在传播过程中就弄丢了。

![[Pasted image 20220812141117.png]]

Solana过去并没有像以太坊那样设置gas机制，大多数交易的手续费都是一样的，低于0.01美分，这可能是因为gas竞价机制会抬高交易手续费，而无gas可遏制MEV爱好者拉高整体的手续费水准。虽然这的确让solana手续费维持在近0的水平，但却衍生出了另一种形式的MEV攻击：

每逢有热门项目开启公售，就有科学家发起大量的重复交易，他们通过这种方式提升自己的交易被Leader处理的概率，挤兑正常用户。Solana对这种重复交易的惩罚机制并不明确，也未曾有效发挥作用，单一的Leader无力过滤海量重复交易，最终导致崩溃。

10.Turbine传输协议：为了提升新区块的传播速度，Leader会把区块（条目）切成多个不同的碎片，结合纠删码发送给多个不同的节点，再由它们把碎片进一步扩散。之后节点们可以彼此交换碎片，拼凑出完整的区块。Turbine照搬了BT种子，通过让更多节点参与到碎片交换流程，提升各个节点获取完整区块的速度。

![[Pasted image 20220812141207.png]]

同时，Solana按照节点质押资产的数量划分层级，层级高的先收到Leader发出的碎片，层级低的再从层级高的节点那里接收数据，这就使得占有2/3质押权重的节点们可以率先收到新区块，只要2/3资产质押权重的大户节点都给出了投票，一个区块就完成敲定Finality，理论上不可回滚。

![[Pasted image 20220812141226.png]]

根据数据，Solana有132个节点占据67%的质押份额，其中的25个节点占据33%的质押份额，基本构成了“元老院”。

11.Sealevel（并行执行模式）：Solana支持多个CPU核心并行处理交易，加快处理速度。为了防止不同的CPU核心同时访问一个合约，产生读写上的冲突，Solana采用了Actor模型，为每个合约都标记一个ID号，系统可以提前知道未来将访问的多个合约的ID，并判断出哪些情况可能冲突。

![[Pasted image 20220812141302.png]]

Actor模型曾被EOS采用，现在也被Gear、Thinkium等公链使用，被认为是解决并发并行执行中冲突情况的最佳方式。

12.超高硬件要求：综上所述，Solana通过典型的纵向扩容方式，通过软硬件结合提高了TPS，其真实的TPS应该在500左右（官方浏览器的TPS数据包含节点投票，75%左右都和用户交易无关）。但纵向扩容本身会对节点硬件水平提出极高要求，以下为Solana的Validator节点硬件要求：

Cpu 12或24核，内存至少128 GB，硬盘2T SSD，网络带宽至少达到300 MB/s，一般为1GB/s。 再对比当前以太坊节点的硬件要求（转型POS前）： CPU 4核以上，内存至少16 GB，硬盘0.5 T SSD，网络带宽至少25 MB/s。

考虑到以太坊转型POS后节点硬件配置要求会调低，Solana对节点硬件的要求远远高于前者。根据部分说法，一个Solana节点的硬件成本，相当于几百个转型POS后的以太坊节点。

#### solana的协议

PoH(Proof of history) 这个共识基本就是不可靠，有问题的。

Proof of history: what is it good for?  [poh.pdf (shoup.net)](https://www.shoup.net/papers/poh.pdf)  作者Victor Shoup

```txt
Wow, really great. I never understood what PoH brings to the table. I even thought it's a rebranded PoW. But a bit worse: all verifiers need to redo the work...

n theory, VDFs could enforce delays ala Fantomette https://arxiv.org/abs/1805.06786 except.. VDFs admit an adversarial advantage, not as bad as proof-of-work, but still 10x or worse. In really, you discover the time by treating message arrival times as votes on the time.

great work! i don’t think that PoH provides any utility in making consensus faster or reducing consensus complexity. IMO, the strongest argument for it is as an embedded “objective” history metric, to address so-called “weak subjectivity” of typical PoS systems
```

[buffalu 🤖💦 在 Twitter: "didn't look at any source code yet cited a lack of information? nice. quick points: PoH is a recursive hash combined with merkle root of tx signatures: https://t.co/j7ojcD2SLY it keeps track of time and at what point txs were executed bc blocks are "streamed" out" / Twitter](https://twitter.com/buffalu__/status/1523501818260725760)

[Victor Shoup 在 Twitter: ""Proof of history: what is it good for?" Some thoughts on Solana's proof of history mechanism. https://t.co/AiB4yOaZz4 #solana #proofofhistory" / Twitter](https://twitter.com/VictorShoup/status/1523015525814894597)

[David Wong 在 Twitter: "So looking into Solana. It’s a BFT, and it uses an old-school VDF to do leader election, having said that I still don’t see where the perfs come from" / Twitter](https://twitter.com/cryptodavidw/status/1562774435018330112)

