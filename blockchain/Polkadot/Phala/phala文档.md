## Readme overview

![[Pasted image 20220922142356.png]]

-   `node/`: substrate构建的区块主链
-   `standalone/pherry/`: 消息中继器，连接区块链节点和pRuntime的
-   `standalone/pruntime/`: 合约执行内核，运行在TEE enclave中

区块链是系统的核心组成部分。它记录命令（保密合约调用），作为pRuntime注册处，运行本地代币和链上治理模块。

pherry是消息中继器。它连接着区块链和pRuntime（区块链和pRuntime通信，都需要经过它）。它将区块数据从链上传给pRuntime，并将pRuntime的副作用传回给链上。这里正在开发一个多客户端版本的运行时桥接器 [Phala-Network/runtime-bridge (github.com)](https://github.com/Phala-Network/runtime-bridge) ，现在是alpha版本。

pRuntime（Phala Network Secure Enclave Runtime）是一个执行保密智能合约的runtime，基于保密计算。

## phat合约

phat合约兼容substrate的contracts pallet。完全支持未修改的ink!合约。还有其他可以发http的功能。(这个安全吗？) 其实phat合约就是parity的ink!语言和phala ink! 扩展 (pink) [phala-blockchain/crates/pink at master · Phala-Network/phala-blockchain (github.com)](https://github.com/Phala-Network/phala-blockchain/tree/master/crates/pink) 

合约教程 [Phat Contract Tutorial - Phala Network Wiki](https://wiki.phala.network/en-us/build/developer/fat-contract-tutorial/)

合约的一些exmaple [phala-blockchain/crates/pink at master · Phala-Network/phala-blockchain (github.com)](https://github.com/Phala-Network/phala-blockchain/tree/master/crates/pink)

[Phala Native Fat Contract Demo - Phala Ghost Auction - YouTube](https://www.youtube.com/watch?v=zaogHCuySD0&t=147s)

[A BTC price bot with ink! smart contracts | Substrate Seminar - YouTube](https://www.youtube.com/watch?v=aZGj4FhkY6A)

## phala区块链的细节

### Overview

Phala是下一代互联网的无信任云计算解决方案，即Web3。 我们建立在Parity的Substrate上，并在Kusama parachain上的Polkadot生态系统中运行。

Phala通过将计算从链上解耦给链下的安全工人（矿工，worker node），解决了在云环境中运行的传统区块链解决方案的隐私和性能问题：区块链只被用作典型的（加密的）消息队列，而经过认证的安全工人(worker node)从链上获取请求（即交易），验证并执行它们，然后将计算结果写回去。

### 架构

![[Pasted image 20220922152115.png]]

跟Readme的overview的内容解释一致。pherry是连接pRuntime和susbtrate区块链的消息中继器。

### 交易安全

我们系统设计的核心观点是，区块链可以作为安全enclave的典型输入源(Event Sourcing)，即使workder node的运营者是恶意的，安全enclave硬件也会执行链上指示的保密和忠实的执行。  
  
虽然攻击者不能偷看安全enclave，但他们可以通过伪造交易或重放/重排有效交易来欺骗其中的合约。确保保密合约只接受有效的交易，并按照预期的顺序处理交易是很重要的。这就是为什么我们引入了Phala区块链，并通过pherry将其连接到pRuntime。  
  
如图所示，Phala区块链作为有效交易的典范来源。只有提交的交易才能被pRuntime接受，而且它们将按照区块链上的顺序被处理。我们在pRuntime中实现了一个轻型验证客户端，以确定有效交易是否按照预期的顺序被接受。另外，还将引入一个密钥轮换机制，以防止历史交易的重放。最棒的是，pRuntime隐藏了所有这些复杂的实现细节，让你像开发普通程序一样实现保密合约。  
  
pherry作为Phala区块链和pRuntime之间的桥梁。它确保区块链上的所有交易都忠实地转发到pRuntime，并且所有的enclave实例都运行未经修改的pRuntime版本。虽然值得注意的是，pRuntime并不信任pherry，但它仍然会验证它从pherry收到的每一个区块和交易。  
  
## phala区块链的组件实体

### Overview

上一章介绍了Phala的架构，而这一页将涉及到Phala的实体，即构成Phala网络的节点类型。

在Phala网络中，有三种类型的实体。

- 客户端，在普通设备上运行，没有任何特殊的硬件要求。
- Worker，在安全enclave上运行，作为保密智能合约的计算节点。
- 守门人（Gatekeeper），在安全enclave上运行，作为授权和密钥管理者。

### 实体秘钥初始化

在Phala中，任何实体之间的通信都应该是加密的，所以每个实体在初始化时都会用伪随机数生成器生成以下实体密钥对。

1. 身份密钥
    -  一个sr25519密钥对，用于唯一地识别一个实体。
2. EcdhKey
    - 一个用于安全通信的sr25519密钥对。

### pRuntime初始化

在初始化过程中，pRuntime会用随机数发生器自动生成上述实体密钥对。生成的密钥对在pRuntime的安全enclave中进行管理，这意味着workder node和守门人(Gatekeeper)只能通过pRuntime导出的有限API来使用它，而永远无法获得明文密钥对来读取安全enclave中的加密数据。

生成的密钥对可以通过Sealing [Sealing - SGX 101 (gitbook.io)](https://sgx101.gitbook.io/sgx101/sgx-bootstrap/sealing) 在本地加密并缓存在磁盘上，在重启时解密并加载。这同时适用于GateKeeper和worker。

### 安全通信信道

[Blockchain Entities - Phala Network Wiki](https://wiki.phala.network/en-us/build/infrastructure/blockchain-entities/#secure-communication-channels)

### Client与worker通信的payload例子

客户端只为合约调用而与worker进行通信。一个调用至少由以下有效载荷组成。

```json
{ 
	from: Client_IdentityKey, 
    payload: { 
	    to: Contract_IdentityKey, 
	    input: "0xdeadbeef", 
    }, 
    nonce: 12345, 
    sig: UserSignature, 
}
```

- `nonce`对于防御双重消费和重放攻击是必要的。
- `from`字段显示调用者的身份，可以用`sig`验证。 from将进一步传递给合约。
- 由于一个worker可以运行多个合约（甚至是同一合约的不同实例），所以需要指定调用目标。
- `input`对被调用的函数和参数进行编码，它应该根据合约的ABI进行序列化。

## 运行本地开发环境网络

可能有点过时 [Archived: Run a Local Development Network - Phala Network Wiki](https://wiki.phala.network/en-us/build/developer/run-a-local-development-network/)


