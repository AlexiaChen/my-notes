[ibc/spec/app/ics-721-nft-transfer at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/app/ics-721-nft-transfer)

实现代码 https://github.com/bianjieai/ibc-go/tree/ics-721-nft-transfer 

该标准文档遵循与  [[IBC之ICS-20规范-ERC20跨链子协议]] 相同的设计原则，并继承了其大部分内容，同时将基于`bank`模块的资产跟踪逻辑替换为 `nft` 模块的资产跟踪逻辑。

## 概要

该标准文档指定了数据包数据结构、状态机处理逻辑和编码细节，用于在不同链上的两个模块之间通过 IBC 通道传输NFT代币。 所呈现的状态机逻辑允许安全的多链 `classId` 处理，无需许可的通道打开。 该逻辑构成了一个NFT代币传输桥接模块，在 IBC 路由模块和主机状态机上的现有资产跟踪模块之间进行接口，该状态机可以是 Cosmos 风格的本机模块或在虚拟机中运行的智能合约。

### 动机

通过 IBC 协议连接的一组链的用户可能希望在最初发行代币的链以外的链上使用NFT代币——也许是为了利用额外的功能，例如交换、版税支付或隐私 保护。 该应用层标准描述了一种用于在与 IBC 连接的链之间传输NFT代币的协议，该协议保留资产非同质性、保留资产所有权、限制拜占庭故障的影响，并且不需要额外的许可。

### 定义

IBC handler 接口和 IBC 路由模块接口分别在 ICS 25 和 ICS 26 中定义。

### 期望的特性

- 非同质性的保存（即，任何代币只有一个实例在所有 IBC 连接的区块链中存在）。
- 无需许可的代币传输，无需将连接、模块或`classID` 列入白名单。
- 对称的（所有链都实现相同的逻辑，没有hubs和zones的协议内差异）。
- 故障遏制：防止链 B 的拜占庭行为导致链 A 上产生代币的拜占庭式创建。

## 技术规范

### 数据结构

只需要一种数据包数据类型：`NonFungibleTokenPacketData`，它指定classId、class uri、token id、token uri、发送方地址和接收方地址。

```go
interface NonFungibleTokenPacketData {
  classId: string
  classUri: string
  tokenIds: []string
  tokenUris: []string
  sender: string
  receiver: string
}
```

`classId` 唯一标识正在传输的代币在发送链中所属的类/集合。例如，对于符合 ERC-1155 标准的智能合约，这可能是token ID 前 128 位的字符串表示形式。

`classUri` 是可选的，但对于与 OpenSea 等 NFT 市场的跨链互操作性非常有益，可以在其中添加类/集合元数据以获得更好的用户体验。

`tokenIds` 唯一标识正在传输的给定类的一些代币。例如，在符合 ERC-1155 标准的智能合约中，`tokenId` 可以是令牌 ID 低 128 位的字符串表示形式。

每个` tokenId` 在 `tokenUris` 中都有对应的条目，它指的是链下资源，通常是包含令牌元数据的不可变 JSON 文件。

当代币使用 ICS-721 协议跨链发送时，它们开始累积代币传输通道的记录。此记录信息被编码到 `classId` 字段中。

ICS-721 代币类以 `{ics721Port}/{ics721Channel}/{classId}` 的形式表示，其中 `ics721Port` 和 `ics721Channel` 标识当前链上代币到达的通道。如果 `{classId}` 包含 `/`，那么它也必须是 ICS-721 形式，表示令牌具有多跳记录。请注意，这要求在非 IBC 代币的class ID 中禁止使用 `/`（斜杠字符）。

发送链可以充当source zone或sink zone。当链通过不等于最后一个前缀端口和通道对的端口和通道发送代币时，它充当source zone。当从source zone发送代币时，目标端口和通道将作为前缀添加到 `classId` 上（一旦收到代币），向代币记录添加另一个跃点。当一条链通过等于最后一个前缀端口和通道对的端口和通道发送代币时，它充当sink zone。当从接收器区域发送代币时，将删除 `classId` 上的最后一个前缀端口和通道对（一旦收到代币），撤消代币记录中的最后一跳。

假设以下时NFT代币的转移路径

A -> B -> C -> A -> C -> B -> A

- A(p1,c1) -> (p2,c2)B : A 是source zone。 B 中的 `classId`：'p2/c2/nftClass'
- B(p3,c3) -> (p4,c4)C : B 是source zone。 C 中的 `classId`：'p4/c4/p2/c2/nftClass'
- C(p5,c5) -> (p6,c6)A : C 是source zone。 A 中的 `classId`：'p6/c6/p4/c4/p2/c2/nftClass'
- A(p6,c6) -> (p5,c5)C : A 是source zone。 C 中的 `classId`：'p4/c4/p2/c2/nftClass'
- C(p4,c4) -> (p3,c3)B : C 是sink zone。 B 中的 `classId`：'p2/c2/nftClass'
- B(p2,c2) -> (p1,c1)A : B 是sink zone。 A 中的 `classId`：'nftClass'

确认数据类型描述了传输是成功还是失败，以及失败的原因（如果有的话）。

```go
type NonFungibleTokenPacketAcknowledgement =
  | NonFungibleTokenPacketSuccess
  | NonFungibleTokenPacketError;

interface NonFungibleTokenPacketSuccess {
  // This is binary 0x01 base64 encoded
  success: "AQ==";
}

interface NonFungibleTokenPacketError {
  error: string;
}
```

请注意，在序列化为数据包数据时，`NonFungibleTokenPacketData` 和 `NonFungibleTokenPacketAcknowledgement` 都必须采用 JSON 编码（而非 Protobuf 编码）。

NFT传输桥模块为每个 NFT 通道维护一个单独的托管地址。

```go
interface ModuleState {
  channelEscrowAddresses: Map<Identifier, string>;
}
```

### 子协议

此处描述的子协议应该在“NFT牌传输桥”模块中实现，该模块可以访问 NFT 资产跟踪模块和 IBC 路由模块。

NFT资产追踪模块应实现以下功能：

```go
function SaveClass(classId: string, classUri: string) {
  // creates a new NFT Class identified by classId
}
```

```go
function Mint(
  classId: string,
  tokenId: string,
  tokenUri: string,
  receiver: string
) {
  // creates a new NFT identified by <classId,tokenId>
  // receiver becomes owner of the newly minted NFT
}
```

```go
function Transfer(classId: string, tokenId: string, receiver: string) {
  // transfers the NFT identified by <classId,tokenId> to receiver
  // receiver becomes new owner of the NFT
}
```

```go
function Burn(classId: string, tokenId: string) {
  // destroys the NFT identified by <classId,tokenId>
}
```

```go
function GetOwner(classId: string, tokenId: string) {
  // returns current owner of the NFT identified by <classId,tokenId>
}
```

```go
function GetNFT(classId: string, tokenId: string) {
  // returns NFT identified by <classId,tokenId>
}
```

```go
function HasClass(classId: string) {
  // returns true if NFT Class identified by classId already exists;
  // returns false otherwise
}
```

```go
function GetClass(classId: string) {
  // returns NFT Class identified by classId
}
```

#### 端口和通道的建立

`setup` 函数必须在创建模块时调用一次（可能是在初始化区块链本身时）以绑定到适当的端口（由模块拥有）。

```go
function setup() {
  capability = routingModule.bindPort("nft", ModuleCallbacks{
    onChanOpenInit,
    onChanOpenTry,
    onChanOpenAck,
    onChanOpenConfirm,
    onChanCloseInit,
    onChanCloseConfirm,
    onRecvPacket,
    onTimeoutPacket,
    onAcknowledgePacket,
    onTimeoutPacketClose
  })
  claimCapability("port", capability)
}
```

一旦调用了`setup`函数，就可以通过 IBC 路由模块在不同链上的NFT传输模块实例之间创建通道。

本规范仅定义数据包处理语义，并以模块本身不需要担心在任何时间点可能存在或不存在哪些连接或通道的方式定义它们。

#### 路由模块的回调

##### 通道的生命周期管理

机器A和机器B都接受来自对端机器的任何模块的通道， 当且仅当：

- 正在创建的通道是无序的。
- 版本字符串是 `ics721-1`。

```go
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) (version: string, err: Error) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // assert that version is "ics721-1"
  // or relayer passed in empty version
  abortTransactionUnless(version === "ics721-1" || version === "")
  return "ics721-1", nil
}
```

```go
function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  counterpartyVersion: string) (version: string, err: Error) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // assert that version is "ics721-1"
  abortTransactionUnless(counterpartyVersion === "ics721-1")
  return "ics721-1", nil
}
```

```go
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  counterpartyVersion: string
) {
  // port has already been validated
  // assert that version is "ics721-1"
  abortTransactionUnless(counterpartyVersion === "ics721-1");
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress();
}
```

```go
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier
) {
  // accept channel confirmations, port has already been validated, version has already been validated
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress();
}
```

```go
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier
) {
  // accept channel confirmations, port has already been validated, version has already been validated
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress();
}
```

```go
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier
) {
  // abort and return error to prevent channel closing by user
  abortTransactionUnless(FALSE);
}
```

```go
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier
) {
  // no action necessary
}
```

##### 数据包的回复

- 当一个NFT代币从其来源发送出去时，桥接模块在发送链上托管该代币，并在接收链上铸造相应的凭证（voucher）。
- 当一个NFT代币被发送回其源头时，桥接模块会在发送链上销毁该代币，并在接收链上取消托管相应的锁定代币。
- 当数据包超时时，数据包中表示的代币要么取消托管，要么适当地返回给发送者——这取决于代币是从它们的来源移开还是移回它们的来源。
- 确认数据用于处理故障，例如无效的目标帐户。 返回失败的确认比中止交易更可取，因为它更容易使发送链根据失败的性质采取适当的行动。

`createOutgoingPacket` 必须由模块中的交易处理程序调用，该模块执行适当的签名检查，特定于主机状态机上的帐户所有者。

```go
function createOutgoingPacket(
  classId: string,
  tokenIds: []string,
  sender: string,
  receiver: string,
  source: boolean,
  destPort: string,
  destChannel: string,
  sourcePort: string,
  sourceChannel: string,
  timeoutHeight: Height,
  timeoutTimestamp: uint64) {
  prefix = "{sourcePort}/{sourceChannel}/"
  // we are source chain if classId is not prefixed with sourcePort and sourceChannel
  source = classId.slice(0, len(prefix)) !== prefix
  tokenUris = []
  for (let tokenId in tokenIds) {
    // ensure that sender is token owner
    abortTransactionUnless(sender === nft.GetOwner(classId, tokenId))
    if source {
      // escrow token
      nft.Transfer(classId, tokenId, channelEscrowAddresses[sourceChannel])
    } else {
      // we are sink chain, burn voucher
      nft.Burn(classId, tokenId)
    }
    tokenUris.push(nft.GetNFT(classId, tokenId).GetUri())
  }
  NonFungibleTokenPacketData data = NonFungibleTokenPacketData{classId, nft.GetClass(classId).GetUri(), tokenIds, tokenUris, sender, receiver}
  ics4Handler.sendPacket(Packet{timeoutHeight, timeoutTimestamp, destPort, destChannel, sourcePort, sourceChannel, data}, getCapability("port"))
}
```

`onRecvPacket` 在接收到发往该模块的数据包时由路由模块调用。

```go
function onRecvPacket(packet: Packet) {
  NonFungibleTokenPacketData data = packet.data
  // construct default acknowledgement of success
  NonFungibleTokenPacketAcknowledgement ack = NonFungibleTokenPacketAcknowledgement{true, null}
  err = ProcessReceivedPacketData(data)
  if (err !== null) {
    ack = NonFungibleTokenPacketAcknowledgement{false, err.Error()}
  }
  return ack
}

function ProcessReceivedPacketData(data: NonFungibleTokenPacketData) {
  prefix = "{packet.sourcePort}/{packet.sourceChannel}/"
  // we are source chain if classId is prefixed with packet's sourcePort and sourceChannel
  source = data.classId.slice(0, len(prefix)) === prefix
  for (var i in data.tokenIds) {
    if source {
      // unescrow token to receiver
      nft.Transfer(data.classId.slice(len(prefix)), data.tokenIds[i], data.receiver)
    } else { // we are sink chain
      prefixedClassId = "{packet.destPort}/{packet.destChannel}/" + data.classId
      // create NFT class if it doesn't exist already
      if (nft.HasClass(prefixedClassId) === false) {
        nft.SaveClass(data.classId, data.classUri)
      }
      // mint voucher to receiver
      nft.Mint(prefixedClassId, data.tokenIds[i], data.tokenUris[i], data.receiver)
    }
  }
}
```

`onAcknowledgePacket` 在路由模块发送的数据包被确认时被路由模块调用。

```go
function onAcknowledgePacket(packet: Packet, acknowledgement: bytes) {
  // if the transfer failed, refund the tokens
  if (!ack.success) refundToken(packet);
}
```

`onTimeoutPacket` 当该模块发送的数据包超时（这样它就不会在目标链上接收到）时由路由模块调用。

```go
function onTimeoutPacket(packet: Packet) {
  // the packet timed-out, so refund the tokens
  refundToken(packet);
}
```

`refundToken` 在 `onAcknowledgePacket` 失败时 和 `onTimeoutPacket` 被调用，以将托管代币退还给原始发件人。

```go
function refundToken(packet: Packet) {
  NonFungibleTokenPacketData data = packet.data
  prefix = "{packet.sourcePort}/{packet.sourceChannel}/"
  // we are the source if the classId is not prefixed with the packet's sourcePort and sourceChannel
  source = data.classId.slice(0, len(prefix)) !== prefix
  for (var i in data.tokenIds) { {
    if source {
      // unescrow token back to sender
      nft.Transfer(data.classId, data.tokenIds[i], data.sender)
    } else {
      // we are sink chain, mint voucher back to sender
      nft.Mint(data.classId, data.tokenIds[i], data.tokenUris[i], data.sender)
    }
  }
}
```

```go
function onTimeoutPacketClose(packet: Packet) {
  // can't happen, only unordered channels allowed
}
```

#### 分析

##### 正确性

此实现保留了代币的非同质性和可赎回性。

- 非同质性：任何代币只有一个实例存在于所有 IBC 连接的区块链中。
- 可赎回性：如果代币已经发送到交易对手链，它们可以在源链上的相同 `classId` 和 `tokenId` 中赎回。

##### 可选附录

- 每个链，在本地，可以选择保留一个查找表，以在状态下使用简短的、用户友好的本地 `classId`，在发送和接收数据包时，这些 `classId` 与较长的 `classId` 相互转换。
- 可以对可以连接到哪些其他机器以及可以建立哪些通道施加额外的限制。

## 更多的讨论

在此规范之上可以支持扩展和复杂的用例，例如特许权使用费、市场或许可转让。 解决方案可以是模块、hooks、IBC 中间件 [ibc/spec/app/ics-030-middleware at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/app/ics-030-middleware) 等。 为此设计指南超出了范围。

假定主机状态机中的应用程序逻辑将负责根据本规范铸造的 IBC 代币的元数据不变性。 对于任何 IBC 代币，强烈建议 NFT 应用程序检查上游区块链（一直追溯到源头）以确保其元数据未沿途被修改。 如果在未来的某个时候决定通过 IBC 来适应 NFT 元数据的可变性，我们将更新此规范或创建一个全新的规范——也许通过使用高级 DID 功能。

## 向后兼容性

没有

## 向前兼容性

此初始标准在通道握手中使用版本“ics721-1”。

该标准的未来版本可以在通道握手中使用不同的版本，并安全地更改数据包数据格式和数据包处理程序语义。