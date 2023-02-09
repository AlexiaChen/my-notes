[ibc/README.md at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/blob/main/spec/relayer/ics-018-relayer-algorithms/README.md)

代码实现：
- [cosmos/relayer: An IBC relayer for ibc-go (github.com)](https://github.com/cosmos/relayer)
- [informalsystems/hermes: IBC Relayer in Rust (github.com)](https://github.com/informalsystems/hermes)
- [confio/ts-relayer: IBC Relayer in TypeScript (github.com)](https://github.com/confio/ts-relayer)

## 概要

中继算法是 IBC 的“物理”连接层——链下进程负责在运行 IBC 协议的两条链之间中继数据，方法是扫描每条链的状态，构建适当的数据报，并在允许的对端链上执行它们 协议。

### 动机

在 IBC 协议中，区块链只能记录将特定数据发送到另一条链的意图——它不能直接访问网络传输层。 物理数据报中继必须由可访问传输层（如 TCP/IP）的链下基础设施执行。 该标准定义了中继器算法的概念，该算法可由具有查询链状态能力的链下进程执行，以执行此中继。

### 定义

中继器是一个链下进程，能够使用 IBC 协议读取状态并将交易提交到某些分类账集。

### 期望的特性

- IBC 的 exactly-once 或 deliver-or-timeout 安全属性不应依赖于中继器行为（假设拜占庭中继器）。
- IBC 的数据包中继活性属性应该仅取决于至少一个正确的、实时的中继器的存在。
- 中继应该是无许可的，所有必要的验证都应该在链上执行。
- 应尽量减少 IBC 用户和中继器之间的必要通信。
- 在应用层应该可以提供中继激励。

## 技术规范

### 基本的relayer算法

中继器算法是在一组 链的集合`C`上定义的，这些链实现了 IBC 协议。每个中继器可能不一定有权从链间网络中的所有链读取状态和写入数据报（尤其是在许可链或私有链的情况下）——不同的中继器可能在不同的子集之间中继。

`pendingDatagrams` 根据两条链的状态计算要从一条链中继到另一条链的所有有效数据报的集合。中继者必须事先了解 IBC 协议的哪个子集是由它们所中继的集合中的区块链实现的（例如，通过阅读源代码）。下面定义了一个例子。

`submitDatagram` 是按链定义的过程（提交某种交易）。如果链支持，数据报可以作为单个交易单独提交，也可以作为单个交易原子地提交。

`relay` 每隔一段时间就会被中继者调用——不超过每条链上每个区块一次，也可能不那么频繁，这取决于中继者希望中继的频率。

不同的中继器可以在不同的链之间进行中继——只要每对链都有至少一个正确且活跃的中继器并且链保持活动状态，网络中链之间流动的所有数据包最终都会被中继。

```go
function relay(C: Set<Chain>) {
  for (const chain of C)
    for (const counterparty of C)
      if (counterparty !== chain) {
        const datagrams = chain.pendingDatagrams(counterparty)
        for (const localDatagram of datagrams[0])
          chain.submitDatagram(localDatagram)
        for (const counterpartyDatagram of datagrams[1])
          counterparty.submitDatagram(counterpartyDatagram)
      }
}
```

### 数据包，ack消息，超时

#### 在有序的通道中relay数据包

有序通道中的数据包可以以基于事件的方式或基于查询的方式进行中继。 对于前者，中继器应在发送数据包时观察源链中发出的事件，然后使用事件日志中的数据组成数据包。 对于后者，中继器应该定期查询源链上的发送序列，并保留最后一个被中继的序列号，这样介于两者之间的任何序列都是需要查询然后中继的数据包。 无论哪种情况，随后，中继进程都应通过检查接收序列来检查目标链是否尚未接收到数据包，然后将其中继。

#### 在无序的通道中relay数据包

无序通道中的数据包可以以基于事件的方式进行转发。中继器应该观察源链在发送数据包时发出的事件，然后用事件日志中的数据组成数据包。随后，中继器应通过查询数据包的序列号是否存在确认来检查目的地链是否已经收到数据包，如果还没有确认，中继器应转发该数据包。

#### relay ack消息

确认可以以一种基于事件的方式进行中继。中继器应该观察目的地链在收到数据包和写下确认时发出的事件，然后使用事件日志中的数据组成确认，检查数据包的承诺是否仍然存在于源链上（一旦确认被中继，它将被删除），如果是，则将确认中继给源链。

#### relay超时消息

超时中继稍微复杂一些，因为当数据包超时时，没有特定的事件发出--只是数据包不能再被中继了，因为目标链的超时高度或时间戳已经过了。中继过程必须选择跟踪一组数据包（可以通过扫描事件日志来构建），一旦目标链的高度或时间戳超过被跟踪的数据包的高度或时间戳，就检查该数据包的承诺是否仍然存在于源链上（一旦超时被中继，它将被删除），如果是，就向源链中继一个超时。

> 其实，个人认为，该中继器就是类似于实现了一个极简版的TCP和UDP的有序和无序机制。所以又可以简单认为，中继器是IBC的传输层

### Pending 数据报文

`pendingDatagrams`整理将从一台机器发送到另一台机器的数据报。这个函数的实现将取决于两台机器所支持的IBC协议的子集和源机器的状态布局。特定的中继者也可能希望实现他们自己的过滤功能，以便只中继可能被中继的数据报的子集（例如，他们已经被支付以某种链外方式进行中继的子集）。

下面是一个在两条链之间进行单向中继的实施实例。通过切换`chain`和`counterparty`对手方，它可以被改变为执行双向中继。哪个中继进程负责哪些数据报是一个灵活的选择--在这个例子中，中继进程中继所有在`chain`上开始的握手（向两个链发送数据报），中继所有从`chain`上发送到`counterparty`对手方的数据包，并中继所有从`counterparty`发送到`chain`上的数据包的确认。

```go
function pendingDatagrams(chain: Chain, counterparty: Chain): List<Set<Datagram>> {
  const localDatagrams = []
  const counterpartyDatagrams = []

  // ICS2 : Clients
  // - Determine if light client needs to be updated (local & counterparty)
  height = chain.latestHeight()
  client = counterparty.queryClientConsensusState(chain)
  if client.height < height {
    header = chain.latestHeader()
    counterpartyDatagrams.push(ClientUpdate{chain, header})
  }
  counterpartyHeight = counterparty.latestHeight()
  client = chain.queryClientConsensusState(counterparty)
  if client.height < counterpartyHeight {
    header = counterparty.latestHeader()
    localDatagrams.push(ClientUpdate{counterparty, header})
  }

  // ICS3 : Connections
  // - Determine if any connection handshakes are in progress
  connections = chain.getConnectionsUsingClient(counterparty)
  for (const localEnd of connections) {
    remoteEnd = counterparty.getConnection(localEnd.counterpartyIdentifier)
    if (localEnd.state === INIT &&
          (remoteEnd === null || remoteEnd.state === INIT))
      // Handshake has started locally (1 step done), relay `connOpenTry` to the remote end
      counterpartyDatagrams.push(ConnOpenTry{
        desiredIdentifier: localEnd.counterpartyConnectionIdentifier,
        counterpartyConnectionIdentifier: localEnd.identifier,
        counterpartyClientIdentifier: localEnd.clientIdentifier,
        counterpartyPrefix: localEnd.commitmentPrefix,
        clientIdentifier: localEnd.counterpartyClientIdentifier,
        version: localEnd.version,
        counterpartyVersion: localEnd.version,
        proofInit: localEnd.proof(),
        proofConsensus: localEnd.client.consensusState.proof(),
        proofHeight: height,
        consensusHeight: localEnd.client.height,
      })
    else if (localEnd.state === INIT && remoteEnd.state === TRYOPEN)
      // Handshake has started on the other end (2 steps done), relay `connOpenAck` to the local end
      localDatagrams.push(ConnOpenAck{
        identifier: localEnd.identifier,
        version: remoteEnd.version,
        proofTry: remoteEnd.proof(),
        proofConsensus: remoteEnd.client.consensusState.proof(),
        proofHeight: remoteEnd.client.height,
        consensusHeight: remoteEnd.client.height,
      })
    else if (localEnd.state === OPEN && remoteEnd.state === TRYOPEN)
      // Handshake has confirmed locally (3 steps done), relay `connOpenConfirm` to the remote end
      counterpartyDatagrams.push(ConnOpenConfirm{
        identifier: remoteEnd.identifier,
        proofAck: localEnd.proof(),
        proofHeight: height,
      })
  }

  // ICS4 : Channels & Packets
  // - Determine if any channel handshakes are in progress
  // - Determine if any packets, acknowledgements, or timeouts need to be relayed
  channels = chain.getChannelsUsingConnections(connections)
  for (const localEnd of channels) {
    remoteEnd = counterparty.getConnection(localEnd.counterpartyIdentifier)
    // Deal with handshakes in progress
    if (localEnd.state === INIT &&
          (remoteEnd === null || remoteEnd.state === INIT))
      // Handshake has started locally (1 step done), relay `chanOpenTry` to the remote end
      counterpartyDatagrams.push(ChanOpenTry{
        order: localEnd.order,
        connectionHops: localEnd.connectionHops.reverse(),
        portIdentifier: localEnd.counterpartyPortIdentifier,
        channelIdentifier: localEnd.counterpartyChannelIdentifier,
        counterpartyPortIdentifier: localEnd.portIdentifier,
        counterpartyChannelIdentifier: localEnd.channelIdentifier,
        version: localEnd.version,
        counterpartyVersion: localEnd.version,
        proofInit: localEnd.proof(),
        proofHeight: height,
      })
    else if (localEnd.state === INIT && remoteEnd.state === TRYOPEN)
      // Handshake has started on the other end (2 steps done), relay `chanOpenAck` to the local end
      localDatagrams.push(ChanOpenAck{
        portIdentifier: localEnd.portIdentifier,
        channelIdentifier: localEnd.channelIdentifier,
        version: remoteEnd.version,
        proofTry: remoteEnd.proof(),
        proofHeight: localEnd.client.height,
      })
    else if (localEnd.state === OPEN && remoteEnd.state === TRYOPEN)
      // Handshake has confirmed locally (3 steps done), relay `chanOpenConfirm` to the remote end
      counterpartyDatagrams.push(ChanOpenConfirm{
        portIdentifier: remoteEnd.portIdentifier,
        channelIdentifier: remoteEnd.channelIdentifier,
        proofAck: localEnd.proof(),
        proofHeight: height
      })

    // Deal with packets
    // First, scan logs for sent packets and relay all of them
    sentPacketLogs = queryByTopic(height, "sendPacket")
    for (const logEntry of sentPacketLogs) {
      // relay packet with this sequence number
      packetData = Packet{logEntry.sequence, logEntry.timeoutHeight, logEntry.timeoutTimestamp,
                          localEnd.portIdentifier, localEnd.channelIdentifier,
                          remoteEnd.portIdentifier, remoteEnd.channelIdentifier, logEntry.data}
      counterpartyDatagrams.push(PacketRecv{
        packet: packetData,
        proof: packet.proof(),
        proofHeight: height,
      })
    }

    // Then, scan logs for acknowledgements, relay back to sending chain
    recvPacketLogs = queryByTopic(height, "writeAcknowledgement")
    for (const logEntry of recvPacketLogs) {
      // relay packet acknowledgement with this sequence number
      packetData = Packet{logEntry.sequence, logEntry.timeoutHeight, logEntry.timeoutTimestamp,
                          localEnd.portIdentifier, localEnd.channelIdentifier,
                          remoteEnd.portIdentifier, remoteEnd.channelIdentifier, logEntry.data}
      counterpartyDatagrams.push(PacketAcknowledgement{
        packet: packetData,
        acknowledgement: logEntry.acknowledgement,
        proof: packet.proof(),
        proofHeight: height,
      })
    }
  }

  return [localDatagrams, counterpartyDatagrams]
}
```

中继者可以选择过滤这些数据报，以便中继特定的客户、特定的连接、特定的信道，甚至是特定种类的数据包，也许是按照收费模式（本文件没有具体说明，因为它可能有所不同）。

### 顺序约束

在中继过程中，有一些隐含的顺序限制，决定了哪些数据报必须以何种顺序提交。例如，在数据包可以被中继之前，必须提交一个头，以最终确定存储的共识状态和轻客户端的特定高度的commitment root。中继者进程负责经常查询他们所中继的链的状态，以确定什么时候必须中继。

### 打包(bundling)

如果主机状态机支持它，中继器进程可以将许多数据报捆绑到一个交易中，这将导致它们被依次执行，并摊销任何开销成本（例如费用支付的签名检查）。

### 条件竞争

在同一对模块和链之间中继的多个中继者可能试图同时中继同一个数据包（或提交同一个头）。如果两个中继者这样做，第一个交易将成功，第二个将失败。中继者之间或发送原始数据包的行为者与中继者之间的带外协调是必要的，以减轻这种情况。进一步的讨论超出了本标准的范围。

### 激励

中继过程必须能够访问两个链上的账户，并有足够的余额来支付交易费用。中继者可以采用应用层面的方法来收回这些费用，例如在数据包数据中包括对自己的小额付款--中继者费用支付的协议将在本ICS的未来版本或单独的ICS中描述。

任何数量的中继器进程都可以安全地并行运行（事实上，预计独立的中继器将服务于链间的独立子集）。然而，如果他们多次提交相同的证明，可能会消耗不必要的费用，所以一些最小的协调可能是理想的（例如将特定的中继器分配给特定的数据包或扫描mempools的待定交易）。

## 向后兼容性

没有，因为relayer进程是链下进程，可以单独升级降级

### 向前兼容性

同上，因为进程是链下的

