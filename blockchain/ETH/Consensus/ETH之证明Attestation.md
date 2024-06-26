
> 其实现在回过头想了一下，attestation意思有证明的意思，也可以作为委员会的出席人，这两个意思合并起来就是见证的意思，attester这个单词也可以理解为见证人。

以太坊是由1000台计算机组成的，每台计算机脑都通过提供他们的PoS来为其安全作出贡献。但它实际上是如何工作的？1000台计算机如何参与？他们在做什么？

这是一个用ETH代币来进行投票的指南。

这个由EVM组成的分布式网络通过一个共识机制保持同步。

共识机制是一个系统，由leader（区块提议者）更新他们的EVM的状态，而网络的其他部分则以无脑相信leader的方式跟随leader的状态，以保持网络的状态同步。[[ETH之共识系统]]

今天，以太坊使用股权证明（PoS）。

PoS随机选择一个区块提议者，他可以构建下一个区块。网络的其他部分对这些区块进行验证，将它们投票到区块链中。

为了参与，你必须把ETH代币作为股权抵押。[[ETH之共识系统#^ed728f]]

通过将ETH代币作为赌注，网络参与者（validators）将他们的资本置于被slashing的风险之中--这是网络对运作不良（有意或无意）的validators的惩罚。

这就是ETH代币股权投射经济安全的机制。

最明显的是，这种动态适用于区块提议者；如果他们提交了一个无效的区块，网络将认为他们是恶意行为者，并将他们从网络中删除。

但这也适用于下一个步骤--由网络的其他部分对区块进行验证。

区块生产者发出一个区块；validators对其进行评估，如果它是有效的，他们会广播一个证明(attestation)。

如果一个validator证明(见证 attest)了一个无效的区块，网络认为他们和无效区块的提出者一样有敌意。(所谓的连坐)

attestations是ETH代币安全的核心。

好了，我们终于做到了。现在我们明白了attestations是如何融入世界计算机的，我们终于可以深入了解：什么是attestation？

它从一个区块开始，广播到网络上。

![[Pasted image 20221009160024.png]]

将区块发送给几个（甚至1000个）validators是没有问题的，但一旦你开始超过一万个，P2P广播的要求变得如此巨大，以至于没有可信的去中心化网络可以处理它。 ^cd083c

目前有大约440k个以太坊的validators。

![[Pasted image 20221009160152.png]]

一个以太坊的每个validator都验证每个区块是不现实的。相反，我们有一个不同的目标：每个validators将验证每个epoch。

一个epoch=32个区块=6.4分钟。

每隔6.4分钟，每个validator都会对经典区块链(canonical chain)进行投票。[[ETH之时间-Slots和Epochs]]

在每个epoch的开始（每32个区块一次epoch），整个validators集合被随机地洗成32个组，称为委员会，也就是32个委员会(committee)

区块提议者向每个validator发送一份区块的副本，但只有区块提议者所属的委员会成员需要证明（见证）attest。

![[Pasted image 20221009160600.png]]

放大来看，一个attestation实际上是一个BLS签名，这个BLS签名包括了一些可以被识别的元数据

BLS签名是一种数字签名的类型，它保持了普通数字签名的所有属性，但有一个特殊的属性：多个密钥/信息可以被聚合起来。[[BLS数字签名]]

BLS签名的神奇之处在于，在数据层面上，聚合的签名与单个签名是完全相同的东西。

因此，BLS签名可以以一种非常有效的方式存储（和验证）大量的签名。[[BLS数字签名#^458b39]]

如果有44万个validators和32个委员会，那么每个委员会必须有大约14000个validators。

14000个validators仍然是一个问题；因为是P2P网络广播，也有1400个签名，无法一下子聚合。

因此，我们再分一次。1个委员会=64个子网(subnets)。

 validators向被分配给它们的子网广播他们的attestations，子网由大约250名validators组成。

每个委员会中的16个validators被指定为聚合者。他们听取接收证明(attestations)，并开始将它们捆绑到一个BLS签名中。

![[Pasted image 20221009161154.png]]

所有16个聚合者都试图建立包含所有子网validators的同一聚合签名，但网络条件可能导致一些聚合者错过一些验证。你能做的只有这么多。

每个validator都尽力而为，并公布结果。

在这一点上，球又回到了区块生产者的法庭。提出该节点的validator开始逐个子网，并选择最佳的聚合签名。

一旦收集到64个最佳签名，区块提议者就会最后一次聚合它们。

这个最后的聚合签名被添加到区块上，表示该委员会的所有14000个validators都看到了这个区块并同意它--以slashing为威胁作恶节点

顺便说一下，这个过程限制了网络可以支持的validators的数量。

在这一点上，我们有一个有效的区块，由其整个委员会attest见证，并精心准备以允许快速验证。因此，现在是时候把它添加到区块链上了...

事实上，凭借这个过程，它已经被添加到区块链上了。

![[Pasted image 20221009161738.png]]

每当一个区块被提出，1/32的validators网络（32个委员会中的一个）将他们的利益放在线上并投票。

每32个区块，整个validators网络都会参与。

就这样，每个epoch一次。

以太坊就会被所有押注的ETH代币的全部价值所保障。

