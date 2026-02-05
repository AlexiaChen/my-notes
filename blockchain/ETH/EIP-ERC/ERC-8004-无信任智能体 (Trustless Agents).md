# ERC-8004: 无信任智能体 (Trustless Agents)


> **原文链接**: https://eips.ethereum.org/EIPS/eip-8004
> **状态**: 草稿 (Draft)
> **类型**: 标准 (Standards Track)
> **创建日期**: 2025-08-13
> **作者**: Marco De Rossi (@MarcoMetaMask), Davide Crapis (@dcrapis) <davide@ethereum.org>, Jordan Ellis <jordanellis@google.com>, Erik Reppel <erik.reppel@coinbase.com>

---

## 目录

- [摘要](#摘要)
- [动机](#动机)
- [规范](#规范)
  - [身份注册表](#身份注册表)
  - [声誉注册表](#声誉注册表)
  - [验证注册表](#验证注册表)
- [基本原理](#基本原理)
- [安全考虑](#安全考虑)
- [核心总结](#核心总结)
- [版权](#版权)

---

  

## 摘要

本协议提议使用区块链来**跨越组织边界发现、选择和与智能体交互**，无需预先存在的信任关系，从而**实现开放式的智能体经济**。

信任模型是可插拔和分层的，安全性与风险价值成正比，从低风险任务（如订购披萨）到高风险任务（如医疗诊断）。开发者可以选择不同的信任模型：使用客户端反馈的声誉系统、通过质押担保的重新执行进行验证、零知识机器学习（zkML）证明，或可信执行环境（TEE）预言机。

---
  
## 动机

  
模型上下文协议（MCP）允许服务器列出和提供其能力（提示词、资源、工具和补全），而 Agent2Agent（A2A）处理智能体认证、通过 AgentCards 进行技能广告、直接消息传递以及完整的任务生命周期编排。然而，这些智能体通信协议本身不涵盖智能体发现和信任。

为了促进开放、跨组织的智能体经济，我们需要在不受信任的设置中发现和信任智能体的机制。本 ERC 通过三个轻量级注册表解决了这一需求，这些注册表可以作为每链单例部署在任何 L2 或主网上：

  

**身份注册表** - 基于带有 URIStorage 扩展的 ERC-721 的最小化链上句柄，解析为智能体的注册文件，为每个智能体提供可移植的、抗审查的标识符。
**声誉注册表** - 用于发布和获取反馈信号的标准接口。评分和聚合既在链上（用于可组合性）也在链下（用于复杂算法）进行，从而实现智能体评分服务、审计网络和保险池的生态系统。
**验证注册表** - 用于请求和记录独立验证者检查的通用钩子（例如，质押者重新运行任务、zkML 验证器、TEE 预言机或可信法官）。

支付与本协议正交，此处不涉及。然而，提供的示例展示了 **x402 支付** 如何丰富反馈信号。

---

## 规范

  本文中的关键词 "MUST"、"MUST NOT"、"REQUIRED"、"SHALL"、"SHALL NOT"、"SHOULD"、"SHOULD NOT"、"RECOMMENDED"、"NOT RECOMMENDED"、"MAY" 和 "OPTIONAL" 应按照 RFC 2119 和 RFC 8174 的描述进行解释。

---

## 身份注册表

  
身份注册表使用带有 URIStorage 扩展的 ERC-721 进行智能体注册，使**所有智能体可以立即通过 NFT 兼容的应用程序进行浏览和转移**。每个智能体通过以下方式全局唯一标识：

- **agentRegistry**：一个冒号分隔的字符串 `{namespace}:{chainId}:{identityRegistry}`（例如，`eip155:1:0x742...`），其中：
- **namespace**：链族标识符（EVM 链为 `eip155`）
- **chainId**：区块链网络标识符
- **identityRegistry**：ERC-721 注册表合约部署的地址
- **agentId**：注册表分配的 ERC-721 tokenId（递增分配）

在本文档中，ERC-721 中的 **tokenId** 被称为 **agentId**，ERC-721 中的 **tokenURI** 被称为 **agentURI**。ERC-721 代币的所有者是智能体的所有者，可以转移所有权或将管理权（例如，更新注册文件）委托给操作者，如 `ERC721URIStorage` 所支持。

---

### 智能体 URI 和智能体注册文件

  
**agentURI** 必须解析为智能体注册文件。它可以使用任何 URI 方案，例如：

- `ipfs://`（例如，`ipfs://cid`）
- `https://`（例如，`https://example.com/agent3.json`）
- base64 编码的 `data:` URI（例如，`data:application/json;base64,eyJ0eXBlIjoi...`）用于完全链上的元数据

当注册 uri 更改时，可以通过 **setAgentURI()** 更新。

注册文件**必须**具有以下结构：  

```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "myAgentName",
  "description": "智能体的自然语言描述，可能包括它的作用、工作原理、定价和交互方式",
  "image": "https://example.com/agentimage.png",
  "services": [
    {
      "name": "web",
      "endpoint": "https://web.agentxyz.com/"
    },
    {
      "name": "A2A",
      "endpoint": "https://agent.example/.well-known/agent-card.json",
      "version": "0.3.0"
    },
    {
      "name": "MCP",
      "endpoint": "https://mcp.agent.eth/",
      "version": "2025-06-18"
    },
    {
      "name": "OASF",
      "endpoint": "ipfs://{cid}",
      "version": "0.8",
      "skills": [],
      "domains": []
    },
    {
      "name": "ENS",
      "endpoint": "vitalik.eth",
      "version": "v1"
    },
    {
      "name": "DID",
      "endpoint": "did:method:foobar",
      "version": "v1"
    },
    {
      "name": "email",
      "endpoint": "mail@myagent.com"
    }
  ],

  "x402Support": false,
  "active": true,
  "registrations": [
    {
      "agentId": 22,
      "agentRegistry": "{namespace}:{chainId}:{identityRegistry}"
    }
  ],

  "supportedTrust": [
    "reputation",
    "crypto-economic",
    "tee-attestation"
  ]
}

```

  
顶部的 **type**、**name**、**description** 和 **image** 字段**应该**确保与 ERC-721 应用程序兼容。端点的数量和类型完全可自定义，允许开发者根据需要添加任意数量的端点。端点中的 **version** 字段是 SHOULD（推荐），而不是 MUST（必须）。


智能体**可以**宣传其端点，这些端点指向 A2A 智能体卡、MCP 端点、ENS 智能体名称、DID，或智能体在任何链上的钱包（即使智能体未在该链上注册）。

---

### 端点域验证（可选）

由于端点可以指向不受智能体所有者控制的域，智能体**可以**选择通过发布 `https://{endpoint-domain}/.well-known/agent-registration.json` 来证明对 HTTPS 端点域的控制，该文件至少包含一个 `registrations` 列表（或完整的智能体注册文件）。如果该文件可通过 HTTPS 访问并且包含一个 `registrations` 条目，其 `agentRegistry` 和 `agentId` 与链上智能体匹配，用户**可以**将端点域视为已验证；如果端点域与提供智能体主注册文件的域相同（由 `agentURI` 引用），则不需要此额外检查，因为域控制已经在此得到证明。

智能体**应该**至少有一个注册（可以有多个），并且注册中的所有字段都是强制性的。

**supportedTrust** 字段是**可选的**。如果缺失或为空，此 ERC 仅用于发现，而不用于信任。


---

### 链上元数据

注册表通过添加以下函数扩展了 ERC-721，用于可选的额外链上智能体元数据：  

```solidity
function getMetadata(uint256 agentId, string memory metadataKey) external view returns (bytes memory)

function setMetadata(uint256 agentId, string memory metadataKey, bytes memory metadataValue) external
```

当设置元数据时，会发出以下事件：

```solidity
event MetadataSet(uint256 indexed agentId, string indexed indexedMetadataKey, string metadataKey, bytes metadataValue)
```

键 `agentWallet` 被保留，不能通过 `setMetadata()` 或在 `register()` 期间设置（包括元数据数组重载）。它代表智能体接收付款的地址，最初设置为所有者的地址。要更改它，智能体所有者必须通过调用以下函数提供对新钱包控制的有效证明（EOA 使用 EIP-712 签名，智能合约钱包使用 ERC-1271）：  

```solidity
function setAgentWallet(uint256 agentId, address newWallet, uint256 deadline, bytes calldata signature) external
```
  
要读取和清除当前设置的钱包，公开以下函数：

```solidity
function getAgentWallet(uint256 agentId) external view returns (address)
function unsetAgentWallet(uint256 agentId) external
```

当智能体被转移时，`agentWallet` 会自动清除（实际上将其重置为零地址），并且必须由新所有者重新验证。

---

### 注册

可以通过调用以下函数之一来铸造新智能体：  

```solidity
struct MetadataEntry {
    string metadataKey;
    bytes metadataValue;
}

function register(string agentURI, MetadataEntry[] calldata metadata) external returns (uint256 agentId)

function register(string agentURI) external returns (uint256 agentId)

// agentURI 稍后使用 setAgentURI() 添加
function register() external returns (uint256 agentId)
```

  
这会发出一个 Transfer 事件、一个保留的 `agentWallet` 键的 MetadataSet 事件、每个额外元数据条目的一个 MetadataSet 事件（如果有），以及：
  

```solidity
event Registered(uint256 indexed agentId, string agentURI, address indexed owner)
```


---

### 更新 agentURI
  
可以通过调用以下函数来更新 agentURI，该函数会发出 URIUpdated 事件：


```solidity
function setAgentURI(uint256 agentId, string calldata newURI) external
event URIUpdated(uint256 indexed agentId, string newURI, address indexed updatedBy)
```


如果所有者想要将整个注册文件存储在链上，**agentURI 应该**使用 base64 编码的数据 URI，而不是序列化的 JSON 字符串：

```
data:application/json;base64,eyJ0eXBlIjoi...
```

  

---

## 声誉注册表
  
当部署声誉注册表时，_identityRegistry_ 地址通过 `initialize(address identityRegistry_)` 设置，并通过调用以下方式公开可见：


```solidity
function getIdentityRegistry() external view returns (address identityRegistry)
```

_clientAddress_ 给智能体的反馈由一个签名的固定点 **value**（`int128`）及其 **valueDecimals**（`uint8`，0-18），加上可选的 **tag1** 和 **tag2**（留给开发者自行决定，以提供最大的链上可组合性和过滤性）、一个 **endpoint** URI、一个指向链下 JSON 的文件 URI（包含额外信息）及其 KECCAK-256 文件哈希（以保证完整性）组成。我们建议使用 IPFS 或类似服务，使反馈易于被子图或类似技术索引。对于 IPFS URI，不需要哈希。

除 **value** 和 **valueDecimals** 外，所有字段都是**可选的**，因此不需要链下文件，可以省略。

---

### 给予反馈

任何 _clientAddress_ 都可以通过调用以下方式添加新反馈：  

```solidity
function giveFeedback(uint256 agentId, int128 value, uint8 valueDecimals, string calldata tag1, string calldata tag2, string calldata endpoint, string calldata feedbackURI, bytes32 feedbackHash) external
```

  
_agentId_ 必须是有效注册的智能体。_valueDecimals_ **必须**在 0 到 18 之间。反馈提交者**绝不能**是 _agentId_ 的智能体所有者或批准的操作者。_tag1_、_tag2_、_endpoint_、_feedbackURI_ 和 _feedbackHash_ 是**可选的**。

如果提供，_feedbackHash_ 是 _feedbackURI_ 引用内容的 KECCAK-256 哈希（`keccak256`），为非内容寻址 URI 启用可验证的完整性。对于 IPFS（或其他内容寻址 URI），_feedbackHash_ 是**可选的**，可以省略（例如，设置为 `bytes32(0)`）。

如果过程成功，会发出一个事件：  

```solidity
event NewFeedback(uint256 indexed agentId, address indexed clientAddress, uint64 feedbackIndex, int128 value, uint8 valueDecimals, string indexed indexedTag1, string tag1, string tag2, string endpoint, string feedbackURI, bytes32 feedbackHash)
```

反馈字段 _value_、_valueDecimals_、_tag1_、_tag2_ 和 _isRevoked_ 与 feedbackIndex 一起存储在合约存储中（feedbackIndex 是 _clientAddress_ 给 _agentId_ 的反馈提交的从 1 开始计数的计数器）。字段 _endpoint_、_feedbackURI_ 和 _feedbackHash_ 被发出但不存储。这向任何智能合约暴露声誉信号，实现链上可组合性。

当反馈由智能体给出（即客户端是智能体）时，智能体**应该**使用链上可选的 `agentWallet` 元数据中设置的地址作为 clientAddress，以便于声誉聚合。
  

---

### value / valueDecimals 示例

| tag1                                               | 测量内容            | 人类可读值示例  | `value` | `valueDecimals` |
| -------------------------------------------------- | --------------- | -------- | ------- | --------------- |
| `starred`                                          | 质量评分 (0-100)    | `87/100` | `87`    | `0`             |
| `reachable`                                        | 端点可达性（二元）       | `true`   | `1`     | `0`             |
| `ownerVerified`                                    | 端点由智能体所有者拥有（二元） | `true`   | `1`     | `0`             |
| `uptime`                                           | 端点正常运行时间 (%)    | `99.77%` | `9977`  | `2`             |
| `successRate`                                      | 端点成功率 (%)       | `89%`    | `89`    | `0`             |
| `responseTime`                                     | 响应时间 (ms)       | `560ms`  | `560`   | `0`             |
| `blocktimeFreshness`                               | 平均区块延迟（区块数）     | `4 个区块`  | `4`     | `0`             |
| `revenues`                                         | 累计收入（例如，USD）    | `$560`   | `560`   | `0`             |
| `tradingYield` (`tag2` = `day, week, month, year`) | 收益率             | `-3.2%`  | `-32`   | `1`             |
  

---

### 链下反馈文件结构

URI 处的**可选**文件可能如下所示：  

```json
{
  // 必填字段
  "agentRegistry": "eip155:1:{identityRegistry}",
  "agentId": 22,
  "clientAddress": "eip155:1:{clientAddress}",
  "createdAt": "2025-09-23T12:00:00Z",
  "value": 100,
  "valueDecimals": 0,

  // 所有可选字段
  "tag1": "foo",
  "tag2": "bar",
  "endpoint": "https://agent.example.com/GetPrice",

  "mcp": { "tool": "ToolName" }, // 或: { "prompt": "PromptName" } / { "resource": "ResourceName" }

  // A2A: 参见 A2A 规范中的"上下文标识符语义"和任务模型。
  "a2a": {
    "skills": ["as-defined-by-A2A"],
    "contextId": "as-defined-by-A2A",
    "taskId": "as-defined-by-A2A"
  },

  "oasf": {
    "skills": ["as-defined-by-OASF"],
    "domains": ["as-defined-by-OASF"]
  },

  "proofOfPayment": { // 这可用于 x402 付款证明
    "fromAddress": "0x00...",
    "toAddress": "0x00...",
    "chainId": "1",
    "txHash": "0x00..."
  },

  // 其他字段
  "...": { "..." } // 可以（MAY）
}
```


---

### 撤销反馈

_clientAddress_ 可以通过调用以下方式撤销反馈：

  
```solidity
function revokeFeedback(uint256 agentId, uint64 feedbackIndex) external
```

这会发出：

```solidity
event FeedbackRevoked(uint256 indexed agentId, address indexed clientAddress, uint64 indexed feedbackIndex)
```

---

### 附加响应

任何人（例如，_agentId_ 显示退款，任何将反馈标记为垃圾邮件的链下数据智能聚合器）都可以调用：

```solidity
function appendResponse(uint256 agentId, address clientAddress, uint64 feedbackIndex, string calldata responseURI, bytes32 responseHash) external
```

其中 _responseHash_ 是 _responseURI_ 文件内容的 KECCAK-256 文件哈希，以保证完整性。对于 IPFS URI，不需要此字段。

这会发出：

```solidity
event ResponseAppended(uint256 indexed agentId, address indexed clientAddress, uint64 feedbackIndex, address indexed responder, string responseURI, bytes32 responseHash)
```

  

---

### 读取函数

```solidity
function getSummary(uint256 agentId, address[] calldata clientAddresses, string tag1, string tag2) external view returns (uint64 count, int128 summaryValue, uint8 summaryValueDecimals)

// agentId 和 clientAddresses 是强制性的；tag1 和 tag2 是可选过滤器。
// 必须提供 clientAddresses（非空）；未按 clientAddresses 过滤的结果容易受到女巫/垃圾邮件攻击。详细信息请参阅安全考虑
function readFeedback(uint256 agentId, address clientAddress, uint64 feedbackIndex) external view returns (int128 value, uint8 valueDecimals, string tag1, string tag2, bool isRevoked)

function readAllFeedback(uint256 agentId, address[] calldata clientAddresses, string tag1, string tag2, bool includeRevoked) external view returns (address[] memory clients, uint64[] memory feedbackIndexes, int128[] memory values, uint8[] memory valueDecimals, string[] memory tag1s, string[] memory tag2s, bool[] memory revokedStatuses)

// agentId 是唯一的强制性参数；其他是可选过滤器。默认情况下省略已撤销的反馈。
function getResponseCount(uint256 agentId, address clientAddress, uint64 feedbackIndex, address[] responders) external view returns (uint64 count)

// agentId 是唯一的强制性参数；其他是可选过滤器。
function getClients(uint256 agentId) external view returns (address[] memory)
function getLastIndex(uint256 agentId, address clientAddress) external view returns (uint64)
```

  
我们预计围绕审查者/clientAddresses 的声誉系统将会出现。__虽然链上已启用简单的按审查者过滤（对缓解垃圾邮件有用）和按标签过滤，但更复杂的声誉聚合将在链下发生__。

---

## 验证注册表

__此注册表使智能体能够请求对其工作的验证，并允许验证者智能合约提供可以在链上跟踪的响应__。验证者智能合约可以使用，例如，质押担保的推理重新执行、zkML 验证器或 TEE 预言机来验证或拒绝请求。

当部署验证注册表时，_identityRegistry_ 地址通过 `initialize(address identityRegistry_)` 设置，并通过调用 `getIdentityRegistry()` 可见，如上所述。

---

### 验证请求

智能体通过调用以下方式请求验证：

```solidity
function validationRequest(address validatorAddress, uint256 agentId, string requestURI, bytes32 requestHash) external
```

此函数**必须**由 _agentId_ 的所有者或操作者调用。_requestURI_ 指向包含验证者验证所需所有信息的链下数据，包括验证所需的输入和输出。_requestHash_ 是对此数据的承诺（请求负载的 `keccak256`），并标识请求。所有其他字段都是强制性的。

发出一个 ValidationRequest 事件：

```solidity
event ValidationRequest(address indexed validatorAddress, uint256 indexed agentId, string requestURI, bytes32 indexed requestHash)
```

---

### 验证响应

验证者通过调用以下方式响应：  

```solidity
function validationResponse(bytes32 requestHash, uint8 response, string responseURI, bytes32 responseHash, string tag) external
```

只有 _requestHash_ 和 _response_ 是强制性的；_responseURI_、_responseHash_ 和 _tag_ 是可选的。此函数**必须**由原始请求中指定的 _validatorAddress_ 调用。_response_ 是一个介于 0 和 100 之间的值，可以用作二元值（0 表示失败，100 表示通过）或具有一系列结果的验证的中间值。可选的 _responseURI_ 指向验证的链下证据或审计，_responseHash_ 是其承诺（如果资源不在 IPFS 上），而 _tag_ 允许自定义分类或额外数据。

可以对同一个 _requestHash_ 多次调用 validationResponse()，从而支持用例，如渐进式验证状态（例如，使用 _tag_ 的"软最终性"和"硬最终性"）或验证状态的更新。

成功执行后，会发出一个包含所有函数参数的 _ValidationResponse_ 事件：

```solidity
event ValidationResponse(address indexed validatorAddress, uint256 indexed agentId, bytes32 indexed requestHash, uint8 response, string responseURI, bytes32 responseHash, string tag)
```

合约存储 _requestHash_、_validatorAddress_、_agentId_、_response_、_responseHash_、_lastUpdate_ 和 _tag_，用于链上查询和可组合性。

---

### 读取函数  

```solidity
function getValidationStatus(bytes32 requestHash) external view returns (address validatorAddress, uint256 agentId, uint8 response, bytes32 responseHash, string tag, uint256 lastUpdate)

//返回智能体的聚合验证统计信息。agentId 是唯一的强制性参数；validatorAddresses 和 tag 是可选过滤器
function getSummary(uint256 agentId, address[] calldata validatorAddresses, string tag) external view returns (uint64 count, uint8 averageResponse)

function getAgentValidations(uint256 agentId) external view returns (bytes32[] memory requestHashes)

function getValidatorRequests(address validatorAddress) external view returns (bytes32[] memory requestHashes)
```

与验证相关的激励和 slashing 由特定验证协议管理，不在本注册表范围内。

---

## 基本原理

  
- **智能体通信协议**：MCP 和 A2A 很流行，其他协议可能会出现。因此，本协议从区块链链接到一个灵活的注册文件，其中包括一个可以随意添加端点的列表，结合了 AI 原语（MCP、A2A）和 Web3 原语，如钱包地址、DID 和 ENS 名称。
- **反馈**：该协议结合了 A2A 已经建立的术语（如任务和技能）和 MCP（如工具和提示）的杠杆作用，以及反馈信号结构的完全灵活性。
- **Gas 赞助**：由于客户端不再需要注册，任何应用程序都可以利用 EIP-7702 实现无摩擦反馈。
- **索引**：由于反馈数据保存在链上，我们建议使用 IPFS 存储完整数据，因此很容易利用子图创建索引器并改善用户体验。
- **部署**：我们期望注册表每链作为单例部署。请注意，在链 A 上注册和接收反馈的智能体仍然可以在其他链上运行和交易。如果需要，智能体也可以在多个链上注册。

---

## 安全考虑

- 可能存在女巫攻击，夸大虚假智能体的声誉。本协议的贡献是使信号公开并使用相同的模式。我们预计许多参与者会构建声誉系统，例如，信任或给予审查者声誉（因此按审查者过滤，正如协议已经支持的那样）。
- 链上指针和哈希无法删除，确保审计追踪的完整性
- 验证者激励和 slashing 由特定验证协议管理
- 虽然此 ERC 在加密方面确保注册文件对应于链上智能体，但它无法在加密上保证广告的功能是有效且非恶意的。三种信任模型（声誉、验证和 TEE 证明）旨在支持此验证需求  

---

## 核心总结

### 一句话概括

ERC-8004 是一个用于 **AI 智能体发现和建立信任** 的链上注册标准，让不同组织之间的 AI 智能体能够在没有预先信任的情况下相互发现、选择和交互，实现开放式的智能体经济。  

---
### 核心问题是什么？

目前有两个主流的 AI 智能体通信协议：

| 协议                               | 功能                             |
| -------------------------------- | ------------------------------ |
| **MCP (Model Context Protocol)** | 服务器列出和提供能力（提示词、资源、工具、补全）       |
| **A2A (Agent2Agent)**            | 处理智能体认证、技能广告、直接消息传递、完整任务生命周期编排 |

**但这些协议不覆盖两个关键问题：**

1. 如何发现智能体？
2. 如何信任陌生智能体？

ERC-8004 就是为解决这个问题而设计的。

---

### 三个核心注册表

  

```
┌─────────────────────────────────────────────────────────────┐
│                    ERC-8004 信任层                          │
├─────────────────┬─────────────────┬─────────────────────────┤
│  身份注册表      │   声誉注册表     │    验证注册表           │
│  (ERC-721)      │  (反馈信号)      │   (第三方验证)          │
└────────┬────────┴────────┬────────┴──────────┬──────────────┘、
         │                 │                   │
         ▼                 ▼                   ▼
    智能体发现         信任评分           工作验证
         │                 │                   │
         └─────────────────┴───────────────────┘
                           │
                           ▼
              ┌─────────────────────┐
              │   开放式智能体经济   │
              └─────────────────────┘
```

  
### 1. 身份注册表 (ERC-721)

| 功能                 | 说明                              |
| ------------------ | ------------------------------- |
| **每个智能体 = 一个 NFT** | 可移植、抗审查的标识符                     |
| **agentURI**       | 指向智能体注册文件 (IPFS/HTTPS/Data URI) |
| **agentWallet**    | 收款地址（需签名证明）                     |
  

**核心函数：**

```solidity
register(agentURI) → agentId
setAgentURI(agentId, newURI)
setAgentWallet(agentId, newWallet, signature)
```

---

### 2. 声誉注册表

| 功能        | 说明                          |
| --------- | --------------------------- |
| **反馈信号**  | value + valueDecimals（固定点数） |
| **标签系统**  | tag1、tag2 自定义（用于筛选）         |
| **链上+链下** | 核心数据上链，详细数据 IPFS            |
  

**反馈类型示例：**

| tag1           | 含义     | 示例值    |
| -------------- | ------ | ------ |
| `starred`      | 质量评分   | 87/100 |
| `uptime`       | 正常运行时间 | 99.77% |
| `responseTime` | 响应时间   | 560ms  |
| `successRate`  | 成功率    | 89%    |
  

**核心函数：**

```solidity
giveFeedback(agentId, value, tag1, tag2)
revokeFeedback(agentId, feedbackIndex)
getSummary(agentId, clients, tag) → (count, avgValue)
```


---
  
### 3. 验证注册表
  
| 功能       | 说明                     |
| -------- | ---------------------- |
| **验证请求** | 智能体请求第三方验证工作结果         |
| **验证方式** | 质押重执行 / zkML / TEE 预言机 |
| **响应值**  | 0-100（0=失败，100=通过）     |
  
**核心函数：**

```solidity
validationRequest(validator, agentId, requestURI, requestHash)
validationResponse(requestHash, response, responseURI)
getValidationStatus(requestHash) → (validator, agentId, response, ...)
```

  
---

### 三种信任模型

| 模型             | 适用场景      | 安全级别  | 实现方式           |
| -------------- | --------- | ----- | -------------- |
| **声誉系统**       | 点披萨、简单任务  | ⭐     | 客户端反馈评分        |
| **加密经济验证**     | 中等风险任务    | ⭐⭐⭐   | 质押担保重新执行       |
| **zkML / TEE** | 医疗诊断、金融交易 | ⭐⭐⭐⭐⭐ | 零知识证明 / 可信执行环境 |
  

---

### 智能体全局标识符

  
```
agentRegistry = {namespace}:{chainId}:{identityRegistry}
例：eip155:1:0x742d35Cc6634C0532925a3b844Bc9e7595f0bEb
agentId = ERC-721 tokenId (递增)
```

---

### 智能体注册文件结构

  
```json
{
  "type": "https://eips.ethereum.org/EIPS/eip-8004#registration-v1",
  "name": "myAgentName",
  "description": "智能体描述",
  "image": "https://example.com/agent.png",
  "services": [
    { "name": "web", "endpoint": "https://..." },
    { "name": "A2A", "endpoint": "https://.../.well-known/agent-card.json" },
    { "name": "MCP", "endpoint": "https://mcp.agent.eth/" },
    { "name": "ENS", "endpoint": "vitalik.eth" }
  ],
  "active": true,
  "supportedTrust": ["reputation", "crypto-economic", "tee-attestation"]
}
```

---

### 安全考虑

| 风险         | 缓解措施           |
| ---------- | -------------- |
| **女巫攻击**   | 按审查者过滤、审查者声誉系统 |
| **虚假功能宣传** | 三种信任模型验证       |
| **数据篡改**   | 链上指针和哈希不可删除    |
  
---

### 状态信息

| 项目       | 值                                                  |
| -------- | -------------------------------------------------- |
| **状态**   | Draft (草稿)                                         |
| **创建日期** | 2025-08-13                                         |
| **依赖**   | EIP-155, EIP-712, EIP-721, EIP-1271                |
| **作者**   | MetaMask / Ethereum Foundation / Google / Coinbase |

---

### 关键设计理念

  
1. **可插拔信任**：根据任务风险选择合适的信任模型
2. **链上+链下**：核心信号上链可组合，复杂数据链下存储
3. **协议无关**：支持 MCP、A2A 等多种智能体协议
4. **跨链兼容**：每链单例部署，智能体可跨链工作


---

## 引用

请将本文档引用为：
  Marco De Rossi (@MarcoMetaMask), Davide Crapis (@dcrapis) <davide@ethereum.org>, Jordan Ellis <jordanellis@google.com>, Erik Reppel <erik.reppel@coinbase.com>, "ERC-8004: Trustless Agents [DRAFT]," _Ethereum Improvement Proposals_, no. 8004, August 2025. [Online serial]. Available: https://eips.ethereum.org/EIPS/eip-8004.

---

## 依赖项
  
本 ERC 依赖于以下以太坊改进提案：

| EIP      | 描述              |
| -------- | --------------- |
| EIP-155  | 链ID（用于跨链标识符）    |
| EIP-712  | 结构化数据签名和验证      |
| EIP-721  | 非同质化代币标准        |
| EIP-1271 | 智能合约钱包的标准签名验证方法 |
  
---

## 讨论链接

https://ethereum-magicians.org/t/erc-8004-trustless-agents/25098