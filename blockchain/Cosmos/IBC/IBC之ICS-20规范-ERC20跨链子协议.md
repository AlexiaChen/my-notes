[ibc/spec/app/ics-020-fungible-token-transfer at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/app/ics-020-fungible-token-transfer)

实现代码 https://github.com/cosmos/ibc-go/tree/main/modules/apps/transfer 

## 概要

该标准文档指定了数据包数据结构、状态机处理逻辑和编码细节，用于通过 IBC 通道在不同链上的两个模块之间传输可替代令牌。 所呈现的状态机逻辑允许安全的多链面额(denoms)处理和无需许可的通道开通。 该逻辑构成了“同质化代币传输桥接模块”，连接了 IBC 路由模块和主机状态机上现有的资产跟踪模块。

### 动机

通过 IBC 协议连接的一组链的用户可能希望在另一条链上使用一条链上发布的资产，也许是为了利用附加功能，例如交换或隐私保护，同时保留与发布链上原始资产的可互换性 . 该应用层标准描述了一种用于在与 IBC 连接的链之间传输同质化代币的协议，该协议保留资产同质化、保留资产所有权、限制拜占庭故障的影响，并且不需要额外的许可。

### 定义

IBC handler接口和 IBC 路由模块接口分别在 ICS 25 [ibc/spec/core/ics-025-handler-interface at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-025-handler-interface)  和 ICS 26 [ibc/spec/core/ics-026-routing-module at main · cosmos/ibc (github.com)](https://github.com/cosmos/ibc/tree/main/spec/core/ics-026-routing-module) 中定义。

### 期望的特性

- 保持同质性（双向挂钩，two way peg）。 
- 保持总供应量（单个源链和模块上的恒定或通货膨胀）。 
- 无需许可的代币传输，无需将连接、模块或面额(denoms)列入白名单。 
- 对称的（所有链都实现相同的逻辑，没有hubs和zones的协议内差异）。 
- 故障遏制：防止由于链 B 的拜占庭行为（尽管向链 B 发送代币的任何用户都可能面临风险）导致源自链 A 的代币发生拜占庭式膨胀。

## 技术规范

### 数据结构

只需要一种数据包数据类型：`FungibleTokenPacketData`，指定面额(denom)、金额、发送账户和接收账户。

```go
interface FungibleTokenPacketData {
  denom: string
  amount: uint256
  sender: string
  receiver: string
  memo: string
}
```

注意：由于本规范的早期版本不包含备注字段（memo），因此实现必须确保新的数据包数据仍然与期望旧数据包数据的链兼容。遗留实现必须能够将带有空字符串 memo 的新数据包数据解组到遗留 `FungibleTokenPacketData` 结构中。类似地，支持memo的实现必须能够将遗留数据包数据解组到当前结构中，并将memo字段设置为空字符串。

memo 字段不在转账中使用，但它可以用于外部链下用户（即交易所）或中间件包装转账，可以根据传入的 memo 解析和执行自定义逻辑。如果memo旨在由更高级别的中间件进行解析和解释，则建议这些中间件将它们添加到memo字符串中的命名空间命名，以便它们不会相互覆盖。链应确保对整个数据包数据有一定的长度限制，以确保数据包不会成为 DOS vector造成被攻击。但是，这些不需要是协议定义的限制。如果接收方由于长度限制而无法接受数据包，这将导致发送方超时。

当代币使用 ICS 20 协议跨链发送时，它们开始累积代币转移通道的记录。此信息被编码到 `denom `字段中。

ics20 代币面额以 `{ics20Port}/{ics20Channel}/{denom}` 的形式表示，其中 `ics20Port` 和 `ics20Channel` 是资金所在的当前链上的 `ics20` 端口和通道。带前缀的端口和通道对指示资金先前通过哪个通道发送。如果 `{denom}` 包含 `/`，则它也必须是 ics20 形式，表示此代币具有多跳记录。请注意，这要求在非 IBC 代币面额名称中禁止使用 `/`（斜杠字符）。

发送链可以充当source zone或sink zone。 当链通过不等于最后一个前缀端口和通道对的端口和通道发送代币时，它充当source zone。 当从source zone发送代币时，目标端口和通道将作为面额的前缀（一旦收到令牌），将另一个跳点添加到代币记录。 当一条链通过等于最后一个前缀端口和通道对的端口和通道发送代币时，它充当sink zone。 当代币从接收器区域发送时，面额上的最后一个前缀端口和通道对将被删除（一旦收到代币），撤消代币记录中的最后一跳。 ibc-go 实现中有更完整的解释。[ibc-go/relay.go at 457095517b7832c42ecf13571fee1e550fec02d0 · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/457095517b7832c42ecf13571fee1e550fec02d0/modules/apps/transfer/keeper/relay.go#L18-L49) 

确认数据类型描述了传输是成功还是失败，以及失败的原因（如果有的话）。

```go
type FungibleTokenPacketAcknowledgement = FungibleTokenPacketSuccess | FungibleTokenPacketError;

interface FungibleTokenPacketSuccess {
  // This is binary 0x01 base64 encoded
  result: "AQ=="
}

interface FungibleTokenPacketError {
  error: string
}
```

请注意，当 `FungibleTokenPacketData` 和 `FungibleTokenPacketAcknowledgement` 序列化为数据包数据时，它们都必须是 JSON 编码的（而不是 Protobuf 编码的）。 另请注意，`uint256` 在转换为 JSON 时是字符串编码的，但必须是 `[0-9]+` 形式的有效十进制数。

同质化代币传输桥模块跟踪与状态中特定通道相关联的托管地址。 假定 `ModuleState` 的字段在范围内。

```go
interface ModuleState {
  channelEscrowAddresses: Map<Identifier, string>
}
```

### 子协议

此处描述的子协议应该在“同质化代币传输桥”模块中实现，该模块可以访问`bank`模块和 IBC 路由模块。

#### 端口和通道的建立

`setup` 函数必须在创建模块时调用一次（可能是在区块链本身初始化时）以绑定到适当的端口并创建托管地址(escrow address)（由模块拥有）。

```go
function setup() {
  capability = routingModule.bindPort("bank", ModuleCallbacks{
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

一旦调用了`setup`函数，就可以通过 IBC 路由模块在不同链上的同质化代币传输模块实例之间创建通道。

管理员（有权在主机状态机上创建连接和通道）负责建立与其他状态机的连接，并在其他链上为该模块（或支持该接口的其他模块）的其他实例创建通道。 本规范仅定义数据包处理语义，并以模块本身不需要担心在任何时间点可能存在或不存在哪些连接或通道的方式定义它们。

#### 路由模块回调

##### 通道的生命周期管理

机器A和机器B都接受来自对端机器的任何模块的通道， 当且仅当：

- 正在创建的通道是无序的。
- 版本字符串是 `ics20-1`。

```go
function onChanOpenInit(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  version: string) => (version: string, err: Error) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // assert that version is "ics20-1" or empty
  // if empty, we return the default transfer version to core IBC
  // as the version for this channel
  abortTransactionUnless(version === "ics20-1" || version === "")
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
  return "ics20-1", nil
```

```go
function onChanOpenTry(
  order: ChannelOrder,
  connectionHops: [Identifier],
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyPortIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  counterpartyVersion: string) => (version: string, err: Error) {
  // only unordered channels allowed
  abortTransactionUnless(order === UNORDERED)
  // assert that version is "ics20-1"
  abortTransactionUnless(counterpartyVersion === "ics20-1")
  // allocate an escrow address
  channelEscrowAddresses[channelIdentifier] = newAddress()
  // return version that this chain will use given the
  // counterparty version
  return "ics20-1", nil
}
```

```go
function onChanOpenAck(
  portIdentifier: Identifier,
  channelIdentifier: Identifier,
  counterpartyChannelIdentifier: Identifier,
  counterpartyVersion: string) {
  // port has already been validated
  // assert that counterparty selected version is "ics20-1"
  abortTransactionUnless(counterpartyVersion === "ics20-1")
}
```

```go
function onChanOpenConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // accept channel confirmations, port has already been validated, version has already been validated
}
```

```go
function onChanCloseInit(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
    // always abort transaction
    abortTransactionUnless(FALSE)
}
```

```go
function onChanCloseConfirm(
  portIdentifier: Identifier,
  channelIdentifier: Identifier) {
  // no action necessary
}
```

##### 数据包的回复

用简单的英语来说，在链 A 和 B 之间：

- 当作为source zone时，bridge模块在发送链上托管现有的本地资产面额，并在接收链上铸造凭证（vouchers）。
- 作为sink zone时，bridge模块在发送链上销毁本地凭证，并在接收链上取消托管本地资产面额。
- 当数据包超时时，本地资产将被取消托管并返回给发送者或凭证适当地返回给发送者。
- 确认数据用于处理失败，例如无效的面额或无效的目的地账户。 返回失败的确认比中止交易更可取，因为它更容易使发送链根据失败的性质采取适当的行动。

`sendFungibleTokens` 必须由模块中的交易处理程序调用，该模块执行适当的签名检查，特定于主机状态机上的帐户所有者。

```go
function sendFungibleTokens(
  denomination: string,
  amount: uint256,
  sender: string,
  receiver: string,
  sourcePort: string,
  sourceChannel: string,
  timeoutHeight: Height,
  timeoutTimestamp: uint64) {
    prefix = "{sourcePort}/{sourceChannel}/"
    // we are the source if the denomination is not prefixed
    source = denomination.slice(0, len(prefix)) !== prefix
    if source {
      // determine escrow account
      escrowAccount = channelEscrowAddresses[sourceChannel]
      // escrow source tokens (assumed to fail if balance insufficient)
      bank.TransferCoins(sender, escrowAccount, denomination, amount)
    } else {
      // receiver is source chain, burn vouchers
      bank.BurnCoins(sender, denomination, amount)
    }

    // create FungibleTokenPacket data
    data = FungibleTokenPacketData{denomination, amount, sender, receiver}

    // send packet using the interface defined in ICS4
    handler.sendPacket(
      getCapability("port"),
      sourcePort,
      sourceChannel,
      timeoutHeight,
      timeoutTimestamp,
      data
    )
}
```

`onRecvPacket` 在接收到发往该模块的数据包时由路由模块调用。

```go
function onRecvPacket(packet: Packet) {
  FungibleTokenPacketData data = packet.data
  // construct default acknowledgement of success
  FungibleTokenPacketAcknowledgement ack = FungibleTokenPacketAcknowledgement{true, null}
  prefix = "{packet.sourcePort}/{packet.sourceChannel}/"
  // we are the source if the packets were prefixed by the sending chain
  source = data.denom.slice(0, len(prefix)) === prefix
  if source {
    // receiver is source chain: unescrow tokens
    // determine escrow account
    escrowAccount = channelEscrowAddresses[packet.destChannel]
    // unescrow tokens to receiver (assumed to fail if balance insufficient)
    err = bank.TransferCoins(escrowAccount, data.receiver, data.denom.slice(len(prefix)), data.amount)
    if (err !== nil)
      ack = FungibleTokenPacketAcknowledgement{false, "transfer coins failed"}
  } else {
    prefix = "{packet.destPort}/{packet.destChannel}/"
    prefixedDenomination = prefix + data.denom
    // sender was source, mint vouchers to receiver (assumed to fail if balance insufficient)
    err = bank.MintCoins(data.receiver, prefixedDenomination, data.amount)
    if (err !== nil)
      ack = FungibleTokenPacketAcknowledgement{false, "mint coins failed"}
  }
  return ack
}
```

`onAcknowledgePacket` 在路由模块发送的数据包被确认时被路由模块调用。

```go
function onAcknowledgePacket(
  packet: Packet,
  acknowledgement: bytes) {
  // if the transfer failed, refund the tokens
  if (!ack.success)
    refundTokens(packet)
}
```

`onTimeoutPacket` 由路由模块调用，当该模块发送的数据包超时（这样它就不会在目标链上接收到）。

```go
function onTimeoutPacket(packet: Packet) {
  // the packet timed-out, so refund the tokens
  refundTokens(packet)
}
```

`refundTokens` 由 `onAcknowledgePacket`（失败时）和 `onTimeoutPacket`（将托管代币退还给原始发送人）调用。

```go
function refundTokens(packet: Packet) {
  FungibleTokenPacketData data = packet.data
  prefix = "{packet.sourcePort}/{packet.sourceChannel}/"
  // we are the source if the denomination is not prefixed
  source = data.denom.slice(0, len(prefix)) !== prefix
  if source {
    // sender was source chain, unescrow tokens back to sender
    escrowAccount = channelEscrowAddresses[packet.srcChannel]
    bank.TransferCoins(escrowAccount, data.sender, data.denom, data.amount)
  } else {
    // receiver was source chain, mint vouchers back to sender
    bank.MintCoins(data.sender, data.denom, data.amount)
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

此实现保留了同质性和供应。

- 同质性：如果代币已被发送到交易对手链，它们可以在源链上以相同的面额和数量赎回。
- 供应：将供应重新定义为未锁定的代币。 所有发送-接收对总和为零。 源链可以改变供应。

##### 多链的注意事项

本规范不直接处理“钻石问题”(很极端的问题)，用户将源自链 A 的代币发送到链 B，然后再发送到链 D，并希望通过 D -> C -> A 返回它——因为供应被跟踪由于 B 链拥有（面额为“`{portOnD}/{channelOnD}/{portOnB}/{channelOnB}/denom`”），C 链不能充当中介。目前尚不清楚这种情况是否应该在协议内处理——只需要原始的赎回路径可能就可以了（如果流动性频繁且两条路径上都有一些盈余，钻石路径将在大多数情况下起作用时间）。长赎回路径引起的复杂性可能导致网络拓扑中出现中央链。

为了跟踪以各种路径在链网络中移动的所有面额，对于特定链实施将跟踪每个面额的“全球”源链的注册表可能是有帮助的。最终用户服务提供商（例如钱包作者）可能希望集成这样的注册表或保留他们自己的规范源链和人类可读名称的映射，以改善用户体验。

> 也就是说，你发送的路径是怎样走的，那么在赎回时，必须按照园路返回。

#### 可选附录

- 每个链，在本地，可以选择保留一个查找表，以在状态下使用简短的、用户友好的本地面额，在发送和接收数据包时将其转换为较长的面额或从较长的面额转换而来。
- 可能会对哪些其他机器可以连接到哪些通道可以建立额外的限制。

## 向后兼容性

## 向前兼容性

主要是通道握手时候，会使用不同的版本号

