> 都知道hardhat的部署和测试都可以通过networks这个配置项来配置。

networks config字段是一个可选的对象，网络名称映射到其配置。

Hardhat 中有两种网络。基于JSON-RPC的network，以及内置的Hardhat network。

你可以通过设置config的defaultNetwork字段来定制运行Hardhat时默认使用的网络。如果你省略了这个配置，其默认值是 "hardhat"。

## Hardhat network

Hardhat内置了一个名为hardhat的特殊网络。当使用这个网络时，当你运行一个task、script或test你的智能合约时，Hardhat网络的一个实例将被自动创建。

Hardhat network对Solidity有一流的支持。它总是知道哪些智能合约正在运行，以及它们到底做了什么，为什么会失败。在此了解更多信息。

请参阅 Hardhat Network 配置参考 以了解可配置的细节。[[Hardhat network]]

## JSON-RPC based network

这些是连接到外部节点的网络。节点可以在你的电脑中运行，如Ganache，也可以远程运行，如Alchemy或Infura。  
  
这种网络是用带有以下字段的对象来配置的。  
  
- url, 节点的网址。这个参数对于自定义网络是必需的。  
- chainId, 一个可选的数字，用于验证Hardhat所连接的网络。如果不存在，这个验证将被省略。
- from: 用来作为默认发件人的地址。如果不存在，则使用该节点的第一个账户。  
- gas。它的值应该是 "auto "或一个数字。如果使用一个数字，它将是每笔交易中默认使用的气体限额。如果使用的是 "auto"，则会自动估算出气体限额。默认值。"auto"。  
- gasPrice: 它的值应该是 "auto "或一个数字。这个参数的作用与气体一样。默认值。"自动"。  
- gasMultiplier。一个数字，用于乘以gas估计的结果，由于估计过程的不确定性，给它一些松懈。默认值：1。  
- accounts. 这个字段控制Hardhat使用哪些账户。它可以使用节点的账户（通过设置为 "remote"），本地账户列表（通过设置为十六进制编码的私钥keys），或使用HD钱包。默认值。"remote"。  
- httpHeaders。你可以使用这个字段来设置额外的HTTP Headers，以便在进行JSON-RPC请求时使用。它接受一个JavaScript对象，将头的名称映射到它们的值。默认值：undefined。  
- timeout: 发送到JSON-RPC服务器的请求的超时，单位是ms。如果请求的时间超过这个时间，它将被取消。默认值。40000用于localhost网络，20000用于其他网络。  
  
## HD钱包配置

要在Hardhat中使用HD钱包，你应该将你的网络的accounts字段设置为具有以下字段的对象。

- mnemonic。一个必要的字符串，包含钱包的助记词组。
- path。所有派生钥匙(derived keys)的HD父级。默认值。"m/44'/60'/0'/0".
- initialIndex。派生的初始索引。默认值：0。
- count: 要推导的账户数量。默认值：20。
- passphrase: 钱包的口令。默认值：空字符串。

例如。

![[Pasted image 20221018145437.png]]

## 默认的networks对象

![[Pasted image 20221018145533.png]]