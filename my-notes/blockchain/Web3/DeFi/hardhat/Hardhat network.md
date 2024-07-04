## 概览

Hardhat内置了Hardhat Network，这是一个为开发而设计的本地Ethereum网络节点。它允许你部署你的合约，运行你的测试和调试你的代码，所有这些都在你的本地机器的范围内。

### 它是怎样工作的？

它可以作为进程内或独立的守护程序运行，为JSON-RPC和WebSocket请求提供服务。

默认情况下，它在收到每笔交易时都会按顺序开采一个区块，而且没有延迟。

它由@ethereumjs/vm EVM实现支持，与ganache、Remix和Ethereum Studio使用的相同。

### 我怎样使用它？

默认情况下，如果你正在使用Hardhat，那么你已经在使用Hardhat Network。

当Hardhat执行你的tests、scripts或tasks时，一个进程中(in-process)的Hardhat Network节点会自动启动，所有Hardhat的插件（ethers.js、web3.js、Waffle、Truffle等）将直接连接到该节点的provider上

不需要对你的tests或scripts做任何修改。

Hardhat Network只是另一个网络。如果你想明确一点（显式运行），你可以运行，比如说，npx hardhat run --network hardhat scripts/my-script.js。

#### 独立运行，以支持钱包和其他软件

另外，Hardhat Network可以以独立的方式运行，以便外部客户可以连接到它。这可以是MetaMask，你的Dapp前端，或一个脚本。要以这种方式运行Hardhat Network，请运行。

```bash
npx hardhat node
```

这将启动Hardhat Network，并将其作为一个JSON-RPC和WebSocket服务器公开。

然后，只要将你的钱包或应用程序连接到http://127.0.0.1:8545。

如果你想把Hardhat连接到这个节点，你只需要使用 --network localhost运行。

### 为什么我需要使用它

#### Solidity的stack traces

Hardhat Network拥有一流的Solidity支持。它总是知道哪些智能合约正在运行，它们具体做什么，以及它们为什么失败。

如果一个交易或调用失败，Hardhat Network将抛出一个异常。这个异常将有一个联合的JavaScript和Solidity堆栈跟踪：堆栈跟踪从JavaScript/TypeScript开始，直到你对合约的调用，并继续完整的Solidity调用堆栈。

这是一个使用 TruffleContract 的 Hardhat Network 异常的例子。

```txt
Error: Transaction reverted: function selector was not recognized and there's no fallback function at ERC721Mock.<unrecognized-selector> (contracts/mocks/ERC721Mock.sol:9) at ERC721Mock._checkOnERC721Received (contracts/token/ERC721/ERC721.sol:334) at ERC721Mock._safeTransferFrom (contracts/token/ERC721/ERC721.sol:196) at ERC721Mock.safeTransferFrom (contracts/token/ERC721/ERC721.sol:179) at ERC721Mock.safeTransferFrom (contracts/token/ERC721/ERC721.sol:162) at TruffleContract.safeTransferFrom (node_modules/@nomiclabs/truffle-contract/lib/execute.js:157:24) at Context.<anonymous> (test/token/ERC721/ERC721.behavior.js:321:26)
```

最后两行对应于执行失败交易的 JavaScript 测试代码。其余的是 Solidity 堆栈跟踪。这样你就能清楚地知道为什么你的测试没有通过。

#### 自动化错误消息

Hardhat Network 总是知道你的交易或调用失败的原因，它使用这些信息来使你的合约调试更容易。

当交易无故失败时，Hardhat Network会在以下情况下创建一个清晰的错误信息。

- 用ETH调用一个non-payable的函数
- 向一个没有可payable fallback或receive功能的合约发送ETH
- 在没有fallback函数的情况下调用一个不存在的函数
- 调用一个参数不正确的函数
- 调用一个没有返回正确数据量的external函数
- 在一个非合约账户上调用一个external函数
- 由于参数的原因，无法执行external调用（例如，试图发送过多的ETH）
- 调用一个没有DELEGATECALL的库
- 不正确地调用一个预编译的合约
- 试图部署一个超过EIP-170规定的字节码大小限制的合约

#### console.log

这个可以在合约里面打印相关的数据。

#### 主网forking

Hardhat Network有能力将主网区块链的状态复制到你的本地环境中，包括所有余额和部署的合约。这被称为 "分叉主网"。

在从mainnet分叉出来的本地环境中，你可以执行交易来调用mainnet部署的合约，或以任何其他方式与网络互动，就像你与mainnet一样。此外，你可以做任何非分叉Hardhat Network支持的事情：查看控制台日志，获得堆栈跟踪，或使用默认账户来部署新合约。

更广泛地说，Hardhat Network 可以用来分叉任何网络，而不仅仅是 mainnet。甚至更进一步，Hardhat Network可以用来分叉任何与EVM兼容的区块链，而不仅仅是Ethereum。

你还可以用分叉的Hardhat Network做其他事情。查看我们的指南以了解更多。[[#Forking其他的networks]]

#### Mining模式

Hardhat Network可以被配置为自动（automine）开采区块，在收到每笔交易后立即开采，也可以被配置为间隔开采，即定期开采新区块，尽可能多地纳入待处理交易。

你可以使用这些模式中的一种，也可以两种都不使用。默认情况下，只有自动模式（automine）被启用。

如果这两种挖矿模式都没有启用，就不会有新的区块被挖出，但是你可以使用evm_mine RPC方法手动挖出新区块。这将产生一个新的区块，其中将包括尽可能多的待处理交易。

关于采矿模式和mempool行为的更多细节，请看采矿模式。[[#Mining模式]]

#### 日志

Hardhat Network利用其追踪基础设施提供丰富的日志记录，这将有助于你开发和调试智能合约。

例如，一个成功的交易和一个失败的调用将看起来像这样。

```txt
eth_sendTransaction
  Contract deployment: Greeter
  Contract address: 0x8858eeb3dfffa017d4bce9801d340d36cf895ccf
  Transaction: 0x7ea2754e53f09508d42bd3074046f90595bedd61fcdf75a4764453454733add0
  From: 0xc783df8a850f42e7f7e57013759c285caa701eb6
  Value: 0 ETH
  Gas used: 568851 of 2844255
  Block: #2 - Hash: 0x4847b316b12170c576999183da927c2f2056aa7d8f49f6e87430e6654a56dab0

  console.log:
    Deploying a Greeter with greeting: Hello, world!

eth_call
  Contract call: Greeter#greet
  From: 0xc783df8a850f42e7f7e57013759c285caa701eb6

  Error: VM Exception while processing transaction: revert Not feeling like it
      at Greeter.greet (contracts/Greeter.sol:14)
      at process._tickCallback (internal/process/next_tick.js:68:7)
```



#### debug_traceTransaction方法

在使用Hardhat Network的节点（即npx hardhat node）时，这种日志记录是默认启用的，但在使用进程中(in-process)的Hardhat Network provider时，则禁用。请参阅Hardhat Network的配置，以了解更多关于如何控制其日志记录。[[#Reference]]

## Forking其他的networks

你可以启动一个分叉主网的Hardhat Network实例。这意味着它将模拟拥有与mainnet相同的状态，但它将作为一个本地开发网络工作。这样你就可以与部署的协议（部署在其他远程网络的合约）进行交互，并在本地测试复杂的交互。

要使用这个功能，你需要连接到一个存档节点(archive node)。我们推荐使用Alchemy。[Alchemy - the web3 development platform](https://www.alchemy.com/) 

### 从Mainnet分叉

尝试这一功能的最简单方法是在命令行中启动一个节点。

```bash
npx hardhat node --fork https://eth-mainnet.alchemyapi.io/v2/<key>
```

你也可以这样配置hardhat network:

```js
networks: {
  hardhat: {
    forking: {
      url: "https://eth-mainnet.alchemyapi.io/v2/<key>",
    }
  }
}
```
(注意，你需要用你的个人Alchemy API key替换URL中的\<key\>部分）。

通过访问mainnet上存在的任何状态，Hardhat Network将提取数据并透明地公开，就像它在本地可用一样。


### 钉住一个区块(Pinning a block)

Hardhat Network 默认会从最新的主网块分叉。虽然这可能是实用的，但要建立一个依赖于分叉的测试套件，我们建议从一个特定的区块编号分叉。

这有两个原因。

- 你的测试运行所针对的状态可能会在运行之间发生变化。这可能会导致你的测试或脚本的行为发生变化。
- 钉住可以实现缓存。每次从主网获取数据时，Hardhat Network 都会将其缓存在磁盘上，以加快未来的访问速度。如果你不钉住区块，每一个新区块都会有新的数据，缓存就没有用了。我们测得，用区块钉住后，速度最多可提高20倍。

你需要访问一个有归档数据的节点（archive node），这样才能发挥作用。这就是为什么我们推荐Alchemy，因为他们的免费计划包括存档数据(archival data)。

要钉住区块编号。

```js
networks: {
  hardhat: {
    forking: {
      url: "https://eth-mainnet.alchemyapi.io/v2/<key>",
      blockNumber: 14390000
    }
  }
}
```
如果你使用的是节点任务，你也可以用--fork-block-number标志指定一个块的编号。

```bash
npx hardhat node --fork https://eth-mainnet.alchemyapi.io/v2/<key> --fork-block-number 14390000
```

### 自定义http headers

这里忽略，请参考最新的文档

###  冒充账户

Hardhat Network允许你冒充任何地址。这让你可以从该账户发送交易，即使你无法访问其私钥。

最简单的方法是使用 ethers.getImpersonatedSigner 方法，该方法是由 ethers 对象中的 hardhat-ethers插件 [hardhat-ethers | Ethereum development environment for professionals by Nomic Foundation](https://hardhat.org/hardhat-runner/plugins/nomiclabs-hardhat-ethers) 添加到ethers对象中。

```js
const impersonatedSigner = await ethers.getImpersonatedSigner("0x1234567890123456789012345678901234567890");
await impersonatedSigner.sendTransaction(...);
```

或者，你可以使用 impersonateAccount helper，然后获得该地址的Signer。

```js
const helpers = require("@nomicfoundation/hardhat-network-helpers");

const address = "0x1234567890123456789012345678901234567890";
await helpers.impersonateAccount(address);
const impersonatedSigner = await ethers.getSigner(address);
```

### 自定义Hardhat network的行为

一旦你得到了一个mainnet链状态的本地实例，将该状态设置为你的测试的具体需要可能是下一步。为此，你可以使用我们的Hardhat network helper库 [Hardhat Network Helpers | Ethereum development environment for professionals by Nomic Foundation](https://hardhat.org/hardhat-network-helpers/docs/overview) ，它允许你做一些事情，如操纵网络的时间或修改账户的余额。

### 重置分叉

你可以在运行期间操作分叉，重置到一个新的分叉状态，从另一个区块编号分叉，或者通过调用hardhat_reset来禁用分叉。

```js
await network.provider.request({
  method: "hardhat_reset",
  params: [
    {
      forking: {
        jsonRpcUrl: "https://eth-mainnet.alchemyapi.io/v2/<key>",
        blockNumber: 14390000,
      },
    },
  ],
});
```
你可以通过传递空参数来禁用分叉。

```js
await network.provider.request({
  method: "hardhat_reset",
  params: [],
});
```
这将重置Hardhat Network，在这里描述的状态下启动一个新的实例。[[#Reference]]

### 使用一个自定义的hardfork历史

这个暂时忽略，请参考最新文档 

### troubleshooting

##### ["Project ID does not have access to archive state"](https://hardhat.org/hardhat-network/docs/guides/forking-other-networks#%22project-id-does-not-have-access-to-archive-state%22)

当使用Infura而不使用归档插件(archival add-on)时，你只能访问最近区块链的状态。为了避免这个问题，你可以使用本地存档节点或提供存档数据的服务，如Alchemy。

## Mining模式

这个暂时参考hardhat文档 [Mining Modes | Ethereum development environment for professionals by Nomic Foundation (hardhat.org)](https://hardhat.org/hardhat-network/docs/explanation/mining-modes)

Hardhat Network可以被配置为自动开采区块，在收到每笔交易后立即开采，也可以被配置为间隔开采，即定期开采新区块，尽可能多地纳入待处理交易。  
  
你可以使用这些模式中的一种，也可以两种都不使用。默认情况下，只有automine模式被启用。  
  
当automine被禁用时，每个发送的交易都会被添加到mempool中，其中包含了所有未来可能被开采的交易。默认情况下，Hardhat Network的mempool遵循与Geth相同的规则。这意味着，除其他事项外，交易的优先级是根据支付给矿工的费用（然后是到达时间），无效的交易会被放弃。除了默认的mempool行为之外，还有一个替代的FIFO行为。  
  
当automine被禁用时，可以通过eth_getBlockByNumber RPC方法（用 "pending "作为区块编号的参数）查询待处理交易，可以使用hardhat_dropTransaction RPC方法删除它们，并且可以通过提交一个新的交易来取代它们，该交易具有相同的nonce，但向矿工支付的费用增加了10％以上。  
  
如果这两种挖矿模式都没有启用，就不会有新的区块被挖出来，但你可以使用evm_mine RPC方法手动挖出新的区块。这将产生一个新的区块，其中将包括尽可能多的待处理交易。  
  
### mempool的行为

当automine被禁用时，每个发送的交易都会被添加到mempool中，其中包含了所有未来可能被开采的交易。默认情况下，Hardhat Network的mempool遵循与Geth相同的规则。这意味着，除其他事项外，。

- gas较高的交易会被首先纳入
- 如果有两笔交易可以被纳入，并且都为矿工提供了相同的总费用，那么先收到的那笔交易会先被纳入
- 如果一个交易是无效的（例如，它的nonce值低于发送它的地址的nonce值），该交易将被放弃。

你可以通过使用 "pending"区块标签来获得将包括在下一个区块中的未决交易列表。

```js
const pendingBlock = await network.provider.send("eth_getBlockByNumber", [ "pending", false, ]);
```

#### 用FIFO顺序来开采交易

Hardhat Network的mempool排序交易的方式是可定制的。默认情况下，它们按照 Geth 的规则进行优先排序，但你可以启用先进先出的行为，这可以确保事务按照它们被发送的相同顺序被添加到区块中，这对于从其他网络重新创建区块是很有用的。

你可以在你的配置中用以下方式启用先进先出行为。

```js
networks: {
  hardhat: {
    mining: {
      mempool: {
        order: "fifo"
      }
    }
  }
}
```

### 删除和替换交易

mempool中的交易可以使用 `DropTransaction` network helper删除。

```js
const txHash = "0xabc...";
await helpers.dropTransaction(txHash);
```

你也可以通过发送一个新的交易来取代一个交易，这个交易的nonce和它已经在mempool中的交易相同，但gas价格更高。请记住，就像在Geth中一样，要使这一方法奏效，新的gas/费用价格必须比当前交易的气体价格至少高10%。

### 配置Mining模式

可以使用hardhat的配置文件来配置。  也可以使用RPC方法来配置。

```js
networks: {
  hardhat: {
    mining: {
      auto: false,
      interval: 5000
    }
  }
}
```

在这个例子中，自动挖矿被禁用，间隔挖矿被设置为每5秒生成一个新区块。你也可以配置间隔采矿，在随机延迟后生成一个新的区块。

```js
networks: {
  hardhat: {
    mining: {
      auto: false,
      interval: [3000, 6000]
    }
  }
}
```

在这种情况下，一个新的区块将在3到6秒之间的随机延迟后被开采出来。例如，第一个区块可能在4秒后被开采，第二个区块在5.5秒后被开采，以此类推。

还可以手动把自动挖矿和间隔采矿关闭

```js
networks: {
  hardhat: {
    mining: {
      auto: false,
      interval: 0
    }
  }
}
```

这意味着Hardhat network不会开采新的区块，但你可以使用evm_mine RPC方法手动开采新区块。这将产生一个新的区块，其中将包括尽可能多的待处理交易。

交易排序

Hardhat Network可以用两种不同的方式对mempool交易进行排序。它们的排序方式将改变mempool中的哪些交易被包含在下一个区块中，以及哪个顺序。

第一种排序方式，称为 "prioriy"，模仿Geth的行为。这意味着，它根据支付给矿工的费用对交易进行优先排序。这是默认的。

第二种排序模式，称为 "fifo"，保持mempool交易按其到达顺序排序。

你可以通过以下方式改变排序模式。

```js
networks: {
  hardhat: {
    mining: {
      mempool: {
        order: "fifo"
      }
    }
  }
}
```

你可以使用两个RPC方法在运行时改变采矿行为：`evm_setAutomine`和`evm_setIntervalMining`。例如，要禁用自动采矿。

```js
await network.provider.send("evm_setAutomine", [false]);
```

启用间隔采矿

```js
await network.provider.send("evm_setIntervalMining", [5000]);
```

## Reference

以下的子标题暂时全部参考官方文档 [Reference | Ethereum development environment for professionals by Nomic Foundation (hardhat.org)](https://hardhat.org/hardhat-network/docs/reference)

### 支持的hardforks

### 配置

#### 支持的字段Fields

#### Mining模式

### console.log

### Initial State

### JSON-RPC方法支持