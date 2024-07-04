# Moonbeam的预编译的pallets学习

## X-tokens pallet

可以使用该pallet发送XC-20的资产。

### 介绍

为可替换的资产(FT) transfer建立一个XCM消息并不是一件容易的事。因此，有一些包装函数/pallet，开发者可以利用它们来使用Polkadot/Kusama的XCM功能。

其中一个例子是x-tokens pallet [open-runtime-module-library/xtokens at master · open-web3-stack/open-runtime-module-library (github.com)](https://github.com/open-web3-stack/open-runtime-module-library/tree/master/xtokens)，它提供了不同的方法来通过XCM转移可替换资产(FT)。

本指南将告诉你如何利用x-tokens pallet将XC-20从基于Moonbeam的网络发送到生态系统中的其他链上（中继链/parachains）。此外，你还将学习如何使用x-tokens预编译，通过Ethereum API执行同样的操作。

开发人员必须明白，发送不正确的XCM信息会导致资金损失。因此，在转移到生产环境之前，在测试网络上测试XCM功能是非常重要的。

### 相关XCM定义

- XCM - 代表跨共识信息。它是共识系统之间相互通信的一种通用方式。  
- VMP - 代表垂直消息传递，是XCM的传输方式之一。它允许parachain与中继链交换消息。UMP（向上的消息传递）使parachain能够向其中继链发送消息，而DMP（向下的消息传递）使中继链能够将消息向下传递给其parachains之一  
- XCMP - 代表跨共识消息传递，是XCM的传输方法之一。它允许parachain与同一中继链上的其他parachain交换消息。  
- HRMP - 代表水平中继路由消息传递，是在启动完整的XCMP实现时的一个临时协议。它与XCMP有相同的接口，但消息被存储在中继链上。  
- 主权账户 (sovereign account)- 生态系统中每个链都有一个账户，一个用于中继链，另一个用于其他parachains。它被计算为特定单词和parachain ID串联的blake2哈希值。`blake2(para+ParachainID)`用于中继链的主权账户，`blake2(sibl+ParachainID)`用于其他parachains的主权账户，将哈希值截断到正确的长度。该账户由root拥有，只能通过SUDO（如果有的话）或民主（技术委员会或公投）使用。主权账户通常在生态系统的其他链中签署XCM消息  
- multilocation - 一种指定整个中继链/parachain生态系统中相对于特定origin的点(位置)的方法。例如，它可以用来指定一个特定的parachain、资产、账户，甚至是parachain内的一个pallet。一般来说，一个multilocation被定义为有一个父母(`parents`)和一个内部(`interior`)。Parents指的是你需要从一个给定的origin "跳 "到父区块链中的多少次。内部指的是你需要多少个字段来定义目标点（也就是说用多个字段描述目标点）。例如，为了从另一个parachain瞄准一个ID为1000的parachain，multilocation将是：

```json
{ "parents": 1, "interior": { "X1": [{ "Parachain": 1000 }]}}
```
。  

### X-tokens pallet的接口

[Send XC-20s to Other Chains | Moonbeam Docs](https://docs.moonbeam.network/builders/xcm/xc20/xtokens/#x-tokens-pallet-interface)

### 用X-tokens pallet构建XCM消息

本指南涵盖了使用x-tokens pallet建立XCM消息的过程，更具体地说，是使用transfer和transferMultiasset函数。不过，这两种情况可以推及到其他函数，尤其是在熟悉了multilocation之后。

> 每个parachain可以允许/禁止来自pallet的特定方法。因此，开发者必须确保使用被允许的方法。反之，交易将失败，出现类似`system.CallFiltered`的错误。

你将转移xcUNIT代币，这是Alphanet中继链代币UNIT的XC-20表示。xcUNIT是一个external XC-20。你可以为另一个external XC-20或一个mintable XC-20修改本指南（换句话说是不用本指南的中继链等测试环境）

#### 检查预备的环境要求

- 启动本地的平行链开发环境

```bash
npm run launch -- --parachain moonbase-0.26.0 --parachain moonbase-0.26.0
```

注意ws端口:

```txt
中继链4个节点的ws port分别是: 34002, 34012,34022,34032
parachain 1000的ws port分别是 34102 34112 
parachain 1001的ws port分别是 34202 34212
```


- 用polkadot js App连接到本地的ws-port为34002的中继链的ws端口。就可以看到一下账户了，也有相应的token余额。[Polkadot/Substrate Portal](https://polkadot.js.org/apps/?rpc=wss%3A%2F%2Fwss.api.moonbase.moonbeam.network#/accounts) 后面这个连接也是一个Moonbase Alpha 的样例，是一个parachain 的collator节点。下面的截图是中继链和parachain的截图。

![[43c04e2a0c8f3cff47c5ba49d0218cd.png]]

![[5beeeeb1ca85b6fde410dc1d3436245.png]]

中继链和parachain都有Account，就是那几个预定义的，还有余额。

- 先给两个平行链parachain 1000和1001创建一个revert code，好像已经有了，不用自己去设置了。

[XCM Initialization on Moonbeam (polkassembly.network)](https://moonbeam.polkassembly.network/referendum/33)
[Register DOT from Polkadot as xcDOT on Moonbeam (polkassembly.network)](https://moonbeam.polkassembly.network/referendum/34)

setStorage的Hex-encoded call是:

```txt
0x00050411011da53b775b270400e7e61ed5cbc5a146ea70f53d5a3306ce02aaf97049cf181a5df9223506da77de558984990247c7240000000000000000000000000000000000000806181460006000fd
```

以上的编码，在中继链上和parachain上都可以解码出来

![[ff091a90e7726fa5813cfe6e35228cf.png]]


- 确认的要转移的资产ID， 对于 external XC-20s请连接Network下面的Asset页面，但是本地运行的开发环境Asset的下面没有资产，只有parachain才有Asset，这个对于Moonbase Alphanet也是一样的。这里的assets，主要是external的XC20s。[Polkadot/Substrate Portal](https://polkadot.js.org/apps/?rpc=wss%3A%2F%2Fwss.api.moonbase.moonbeam.network#/assets)

![[5e749cf760c738433742f1635c0405d.png]]

现在是没有，就需要创建，用SODU调用`assetManager.registerForeignAsset`

![[207084bf10b279104026bfb9aed01ce.png]]

encoded call data `0x1f00000100147863444f54147863444f540c000100000000000000000000000000000001`

注意这个decimal参数必须是external XC-20绑定的对应的token强相关，在这里decimal必须是`12` 不然是bug，到后面平行链转到中继链转账转不回来。

点击提交，然后事件显示无问题:

```txt
system.ExtrinsicSuccess
balances.Withdraw
assets.ForceCreated
assets.MetadataSet
assetManager.ForeignAssetRegistered
sudo.Sudid
balances.Deposit
transactionPayment.TransactionFeePaid
extrinsic event
```

然后你就可以在parachain的Assets页面看到资产了:

![[d6c0a20d2efc4f585e728d6ec146683.png]]

这里的资产，你在其中一条parachain上设置，那么两条parachain 1000和1001上都看得到。看来元数据是存在中继链上的？

然后还需要设置XCM的手续费。调用`assetManager.setAssetUnitsPerSecond`

参数unitsPerSecond可以设置为0，numAssetsWeightHint也为0，但是Hint参数每调用一次该接口，就要加1.

![[5108d5dbf8482e565ad9767236f4a08.png]]

encoded call data `0x1f010001000000000000000000000000000000000000000000`

```txt
system.ExtrinsicSuccess
balances.Withdraw
assetManager.UnitsPerSecondChanged
sudo.Sudid
balances.Deposit
transactionPayment.TransactionFeePaid
extrinsic event
```

还可以用Chain state查看external ex-20的资产，通过assets调用metadata:

![[9052a4790a68e5367c9e508f66bced4.png]]


asset id `42259045809535163221576417993425387648`

- 确认要转移的资产ID ，这里主要是mintable XC-20s [Mintable XC-20s | Moonbeam Docs](https://docs.moonbeam.network/builders/xcm/xc20/mintable-xc20/#retrieve-list-of-mintable-xc-20s)

同样，对于Mintable XC-20s也需要创建，因为没有。mintable XC-20资产在Asset页面是看不到的要在Chain state的页面的localAssets下调用asset或者metadata才可以看到，以下就是没有

![[b4a4ffd666ed1a5275f4126e3aa39d5.png]]

所以最终还是要创建mintable xc-20。用SODU调用`assetManager.registerLocalAsset`

isSufficient: 若为 True ，则代表可以将该代币发送给一个没有原生代币（如DOT）的账户

![[c2a3525001418ca0bde951a6ae51160.png]]

这样等待交易上链就可以了。然后可以再次从Chain state页面查看，就可以看到了

![[29243fc5fcbaae295eb37dfafebad5a.png]]

上面的`270,195,117,769,614,861,929,703,564,202,131,636,628` 就是Asset Id。需要手动去除逗号 `270195117769614861929703564202131636628`

好了，之后就是用Extrinsic调用`localAssets.setMetadata`了。

![[a1e6099b3a96bff5d45d9d0796cb85f.png]]

好了，到此为止，external 和 mintable的XC-20资产都有了。

```json
# assets.metadata: PalletAssetsAssetMetadata
[
  [
    [
      42,259,045,809,535,163,221,576,417,993,425,387,648
    ]
    {
      deposit: 0
      name: xcDOT
      symbol: xcDOT
      decimals: 12
      isFrozen: false
    }
  ]
]

# localAssets.asset: Option<PalletAssetsAssetDetails>
[
  [
    [
      270,195,117,769,614,861,929,703,564,202,131,636,628
    ]
    {
      owner: 0xf24FF3a9CF04c71Dbc94D0b566f7A27B94566cac
      issuer: 0xf24FF3a9CF04c71Dbc94D0b566f7A27B94566cac
      admin: 0xf24FF3a9CF04c71Dbc94D0b566f7A27B94566cac
      freezer: 0xf24FF3a9CF04c71Dbc94D0b566f7A27B94566cac
      supply: 0
      deposit: 0
      minBalance: 1
      isSufficient: true
      accounts: 0
      sufficients: 0
      approvals: 0
      isFrozen: false
    }
  ]
]
```

#### 中继链到平行链的跨链转账

用以下命令启动:

```bash
npm run launch -- --parachain moonbase-0.26.0
```


用Poladot js app切换到中继链的ws端口上。然后通过extrinsic调用`xcmPallet`的`limitedReserveTransferAsset`这个接口。

![[Pasted image 20220914163001.png]]

encoded call data `0x630801000100a10f0100010300c0f0f4ab324c46e55d02d0033343b4be8a55532d010400000000070010a5d4e80000000000`

注意，dest肯定是选择目标平行链parachainid是1000. 第二个参数bebeficary选择目标链的账号地址，目标链是Moonbeam的链，所以账号格式是以太坊的格式，选择AccountKey20这个类型，把
目标平行链的任意一个地址复制进去(这里选择的是名叫`FAITH`这个账户)，assets参数就是选择转多少金额了，因为当时在平行链parachain 1000上创建的external XC-20的资产decimal是12，所以就是`1000000000000` 才相当于1个xcDOT的xc-20资产。最后提交交易，等到上链，就可以在平行链的polkadot js app上看到 Asset上有相应的余额了。

![[c4fa2c7dee3964cb7b8fab11b7581d4.png]]

![[768df609e4311d99c7a6e9bb13968e2.png]]

#### 平行链到中继链的转账

从上面那一节中可以看到xcDOT的资产上有余额了（上面是转到FAITH账户下，但是统一显示到了xcDOT的资产下面，以此类推），这一小节中，我们把xcDOT又转回到中继链上。在平行链上打开Extrinsic的页面，选择xToken pallet的`transfer`接口。

currencyId肯定就填入ForeignAsset了， dest填写目标中继链的账户地址。 destWeight填写手续费权重 

![[ad543b3c9c4a138c556aaa86917c8e4.png]]

注意以上的transfer的XCM转账会报错，会有一个XcmExecutionFail的事件弹出来，是因为FAITH这个账户下面没有平行链的原生代币，来支付手续费。要往FAITH账户里面充点token。或者换一个方式，从中继链转到平行链的xcDOT，转到ALITH这个有token的账户下面，这样这个ALITH账户就可以成功转回xcDOT到中继链的任意地址了

![[07b05d8c4f6dbcf1f96125627fd0c56.png]]

encoded call data
`0x1e00018080778c30c20fa2ebc0ed18d2cbca1f0010a5d4e800000000000000000000000101010100d43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d0000000000000000`


根据上图，我们转了1200 amount过去，也就是1.2 xcDOT过去。是转到中继链的FERDIE账户下。你可以在平行链的xcDOT资产下面看到ALITH的账户少了余额

![[71c4e4fe03369b009b5316408d2e791.png]]

并且你还会在中继链的区块链浏览器中看到一个ump pallet的完成事件:

![[09cc19509cba889229945e24d419668.png]]


![[06e279add1c3ed246cb2342e98da106.png]]

这里转到中继链的钱，<del>好像在相关账户下找不到金额</del>  这找不到金额是因为中继链是rococo的中继链，里面的原生token是1e12 普朗克。所以把中继链的原生token注册为平行链的external xc-20资产时，`decimal参数必须是12`，这样才正确，不然随便设置，会导致兑换问题，external xc-20必须1:1的兑换。如果可以随便乱设置，那么可以进行跨链套利了。


#### 平行链到平行链的转账



#### X-tokens transfer函数

#### X-tokens transfer multiasset函数

### X-tokens的预编译

#### X-tokens的solidity接口

#### 构建预编译的multiAsset

#### 使用库与X-tokens交互

## XCM-transactor pallet

可以用该pallet来做远程执行。

### 介绍

XCM消息由一系列指令组成[Cross-Consensus Messaging (XCM) | Moonbeam Docs](https://docs.moonbeam.network/builders/xcm/overview/#xcm-instructions) ，由跨共识虚拟机（XCVM）执行。这些指令的组合导致预定的行动，如跨链token转移，更有趣的是，远程跨链执行。有点像RPC。这样给跨链合约调用带来了可能。

尽管如此，从头开始构建一个XCM消息还是有些麻烦的。此外，XCM消息是从root账户（即SUDO或通过民主投票）发送给生态系统中的其他参与者的，这对于想要通过简单的交易来利用远程跨链调用的项目来说并不理想。

为了克服这些问题，开发者可以利用包装函数/pallet来使用Polkadot/Kusama上的XCM功能，例如XCM-transactor pallet [moonbeam/pallets/xcm-transactor at master · PureStake/moonbeam (github.com)](https://github.com/PureStake/moonbeam/tree/master/pallets/xcm-transactor) 。在这方面，XCM-transactor pallet允许用户通过不同的方法进行远程跨链调用，这些方法从不同的派生账户dispatch调用，这样就可以通过一个简单的交易轻松执行。

pallet的两个主要extrinsics是通过一个主权衍生账户或一个由特定multilocation计算的衍生账户进行交易。每一个extrinsic都被相应命名。


# Moonbeam的预编译合约学习

# Moonbeam本地XCM跨链环境搭建

## 搭建本地Rococo中继链网络和平行链

因为我们目前需要测试并学习XCM跨链，所以需要2个parachains和3个validator nodes，因为之前我们学习到，中继链validator node的数量要比平行链的数量多一个。才可以正常工作。

很多开源项目有例子了，这里我们参考:

- [Thales/generate-relay-specs.sh at main · Thales-network/Thales (github.com)](https://github.com/Thales-network/Thales/blob/main/scripts/generate-relay-specs.sh)
- [moonbeam/generate-relay-specs.sh at master · PureStake/moonbeam (github.com)](https://github.com/PureStake/moonbeam/blob/master/scripts/generate-relay-specs.sh)

- [oysternetwork/generate-relay-specs.sh at d8a0481471178ac14021d7ca7dd52c41b300c307 · nfttoken/oysternetwork (github.com)](https://github.com/nfttoken/oysternetwork/blob/d8a0481471178ac14021d7ca7dd52c41b300c307/scripts/generate-relay-specs.sh)

最好用第二个，因为我们的parachain打算基于用Moonbeam来构建，所以最好是参考MoonBeam的scripts。看这些scripts大量使用了Docker。我大概看了这些scripts, 本质上跟substrate官网的搭建中继链和平行链本地测试网络并流程和一些基本步骤参数没有什么区别，只是更复杂了。

好了，先开启docker:

```bash
sudo service docker start
docker images
```

进入moonbeam项目的根目录cd到tools目录下:

```bash
npm --registry https://registry.npm.taobao.org install
# rococo-9230
npm run launch -- --parachain moonbase-0.26.0
```

```txt
@polkadot/util has multiple versions, ensure that there is only one installed.  
Either remove and explicitly install matching versions or dedupe using your package manager.  
The following conflicting packages were found:  
cjs 9.7.1 node_modules/polkadot-launch/node_modules/@polkadot/util/cjs  
7.9.2 node_modules/@polkadot/util  
cjs 9.7.1 node_modules/@polkadot/api-augment/node_modules/@polkadot/util/cjs  
cjs 9.7.1 node_modules/@polkadot/rpc-augment/node_modules/@polkadot/util/cjs  
cjs 9.7.1 node_modules/@polkadot/types-augment/node_modules/@polkadot/util/cjs  
cjs 9.7.1 node_modules/@polkadot/types-create/node_modules/@polkadot/util/cjs  
cjs 9.7.1 node_modules/@polkadot/types-codec/node_modules/@polkadot/util/cjs  
🚀 Relay: rococo-9230 - purestake/moonbase-relay-testnet:sha-2fd38f09 (rococo-local)  
Missing build/moonbase-0.26.0/moonbeam locally, downloading it...
```

如果pull moonbase的image过慢，可以用`docker events`查看一下:

```bash
mathxh@MathxH:~/metablock-projects$ docker events 
2022-09-01T15:02:47.173422400+08:00 image pull purestake/moonbeam:v0.26.0 (description=Binary for Moon  
beam Collator, maintainer=alan@purestake.com, name=purestake/moonbeam, org.opencontainers.image.create  
d=2022-08-26T07:36:41Z, org.opencontainers.image.description=An Ethereum-compatible smart contract par  
achain on Polkadot, org.opencontainers.image.licenses=GPL-3.0, org.opencontainers.image.revision=c6dc6  
b86bc9da0e7492cb4bcb1e69130579bb8e0, org.opencontainers.image.source=https://github.com/PureStake/moon  
beam.git, org.opencontainers.image.title=moonbeam, org.opencontainers.image.url=https://github.com/Pur  
eStake/moonbeam)

2022-09-01T15:03:33.685254700+08:00 container create 3ade2dea387e88d1bd42487a5541f918e43fe09c339f25708  
fe738366380a0aa (description=Polkadot for Moonbeam Relay Chains, image=purestake/moonbase-relay-testne  
t:sha-2fd38f09, maintainer=alan@purestake.com, name=polkadot-tmp)  
2022-09-01T15:03:33.776034500+08:00 volume mount 2585d70a21e4c786baa0cdbf1aed9d7b212af2a9df4bd6479239b  
bffdaa733da (container=3ade2dea387e88d1bd42487a5541f918e43fe09c339f25708fe738366380a0aa, destination=/  
data, driver=local, propagation=, read/write=true)  
2022-09-01T15:03:33.776579100+08:00 container archive-path 3ade2dea387e88d1bd42487a5541f918e43fe09c339  
f25708fe738366380a0aa (description=Polkadot for Moonbeam Relay Chains, image=purestake/moonbase-relay-  
testnet:sha-2fd38f09, maintainer=alan@purestake.com, name=polkadot-tmp)  
2022-09-01T15:03:34.313554900+08:00 volume unmount 2585d70a21e4c786baa0cdbf1aed9d7b212af2a9df4bd647923  
9bbffdaa733da (container=3ade2dea387e88d1bd42487a5541f918e43fe09c339f25708fe738366380a0aa, driver=loca  
l)  
2022-09-01T15:03:36.905126600+08:00 container destroy 3ade2dea387e88d1bd42487a5541f918e43fe09c339f2570  
8fe738366380a0aa (description=Polkadot for Moonbeam Relay Chains, image=purestake/moonbase-relay-testn  
et:sha-2fd38f09, maintainer=alan@purestake.com, name=polkadot-tmp)
```

`npm run lauch` 以后，你看到有以下日志输出，就说明测试网络启动完成了:

```txt
2022-09-01 15:05:50 API/INIT: Not decorating unknown runtime apis: 0xe5bdc752b8ec2ba1/1, 0x91a3  
c6922fd0a608/1, 0xb69782c4499462e5/1  
Current status is {"finalized":"0x636e85bfcfcd238244fae7df7a1a83834759a31d3dc2458ae80f796b4567253a"}  
Transaction finalized at blockHash 0x636e85bfcfcd238244fae7df7a1a83834759a31d3dc2458ae80f796b4567253a  
🚀 POLKADOT LAUNCH COMPLETE 🚀
```

启动了两个relay node，就是两个validator nodes，和两个parachain collator nodes，通过`htop`命令可以看到本质上的命令行，当然，用`ps auxf > top.txt` 也可以导入文件中看到

```bash
# relay validator node 1
moonbeam/tools/build/rococo-9230/polkadot --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --ws-port=34002 --port=34000 --node-key=501ac10069c62a64b4fbb77ba4e8ecd823c12bb87228f3298da90b1034096dc7 --alice --rpc-port=34001 --tmp --log=info,parachain::pvf=trace
# relay validator node 2
moonbeam/tools/build/rococo-9230/polkadot --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --ws-port=34012 --port=34010 --node-key=5a18b9c939c082e0b1271adfb9792dc635bb84a6dcb5c03fc90182f1b1b8fbfb --bob --rpc-port=34011 --tmp --log=info,parachain::pvf=trace
```

从上面看到，实际上就是用rococo-local的raw chain spec启动了两个中继链的validator nodes。接下来我们再看parachain的collator节点的命令行:

```bash
# parachain collator node 1 Alice
moonbeam/tools/build/moonbase-0.26.0/moonbeam --ws-port=34102 --port=34100 --rpc-port=34101 --collator --tmp --chain=moonbase-local --alice --unsafe-rpc-external --unsafe-ws-external --rpc-methods=Unsafe --rpc-cors=all -- --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --execution=wasm
# parachain collator node 2 Bob
moonbeam/tools/build/moonbase-0.26.0/moonbeam --ws-port=34112 --port=34110 --rpc-port=34111 --collator --tmp --chain=moonbase-local --bob --unsafe-rpc-external --unsafe-ws-external --rpc-methods=Unsafe --rpc-cors=all -- --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --execution=wasm
```

从这里`--chain`参数可以看出，实际上只有一个平行链的chain spec，只是这个平行链运行了两个collator nodes。并且通过 Polkadot-JS Apps [Polkadot/Substrate Portal](https://polkadot.js.org/apps/?rpc=ws://localhost:34002#/explorer) 可以看到本地中继链Alice node的状态，上面只有一个parachain，这个parachain的paraId是1000。通过 [Polkadot/Substrate Portal](https://polkadot.js.org/apps/?rpc=ws://localhost:34102#/explorer) 连接平行链的Alice节点

停止本地网络 也就是`ctrl + C`了，然后docker的容器会被杀死，重新运行又开始从创世块启动了。

## 跨链

### 跨链测试环境启动

要启动两个parachains连接到中继链:

```bash
npm run launch -- --parachain moonbase-0.26.0 --parachain local
```

```txt
🚀 Relay: rococo-9230 - purestake/moonbase-relay-testnet:sha-2fd38f09 (rococo-local)  
🚀 Parachain: moonbase-0.26.0 - purestake/moonbeam:v0.26.0 (moonbase-local)  
🚀 Parachain: local - ../target/release/moonbeam (moonbase-local)

...

⛓ Adding Genesis Parachains  
✓ Added Genesis Parachain 1000  
✓ Added Genesis Parachain 1001  
⛓ Adding Genesis HRMP Channels  
✓ Added HRMP channel 1000 -> 1001  
✓ Added HRMP channel 1001 -> 1000

...

Waiting for finalization...  
2022-09-01 16:43:27 API/INIT: Not decorating unknown runtime apis: 0xe5bdc752b8ec2ba1/1, 0x91a3c6922fd0a608/1, 0xb69782c4499462e5/1  
Current status is {"finalized":"0x34f383c53780f9b178252029d27fe719212fab3ddfa7e12a95298965a3b823c1"}  
Transaction finalized at blockHash 0x34f383c53780f9b178252029d27fe719212fab3ddfa7e12a95298965a3b823c1  
🚀 POLKADOT LAUNCH COMPLETE 🚀

```


可以从上述日志看到，HRMP给两个parachain开通了双向通道。一个paraId是1000，另一个是1001。

用`ps auxf > file.txt` 查看进程:

```bash
# relay chain validator node 1 Alice
moonbeam/tools/build/rococo-9230/polkadot --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --ws-port=34002 --port=34000 --node-key=2a689fc05425c9faa3e3a0ab435307cbf2cd7634e359e81a3fad97602aeb6b38 --alice --rpc-port=34001 --tmp --log=info,parachain::pvf=trace

# relay chain validator node 2 bob
moonbeam/tools/build/rococo-9230/polkadot --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --ws-port=34012 --port=34010 --node-key=555d3dff2e977ddcaa95edc62ddc6e87713a84367abc5e16405cd8c72d891d92 --bob --rpc-port=34011 --tmp --log=info,parachain::pvf=trace

# relay chain validator node 3 charlie
moonbeam/tools/build/rococo-9230/polkadot --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --ws-port=34022 --port=34020 --node-key=be19216b9410b3fa6ca7100aa58fb15e16a911f6cb9f6314c754f2ef08965b8d --charlie --rpc-port=34021 --tmp --log=info,parachain::pvf=trace

# relay chain validator node 4 dave
moonbeam/tools/build/rococo-9230/polkadot --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --ws-port=34032 --port=34030 --node-key=7810551e24a173c0a2595002f15181194c71b8356d63a4714de9a014bd8c8a4c --dave --rpc-port=34031 --tmp --log=info,parachain::pvf=trace

# parachain 1000 collator node 1 alice 
moonbeam/tools/build/moonbase-0.26.0/moonbeam --ws-port=34102 --port=34100 --rpc-port=34101 --collator --tmp --chain=moonbase-local --alice --unsafe-rpc-external --unsafe-ws-external --rpc-methods=Unsafe --rpc-cors=all -- --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --execution=wasm

# parachain 1000 collator node 2 bob 
moonbeam/tools/build/moonbase-0.26.0/moonbeam --ws-port=34112 --port=34110 --rpc-port=34111 --collator --tmp --chain=moonbase-local --bob --unsafe-rpc-external --unsafe-ws-external --rpc-methods=Unsafe --rpc-cors=all -- --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --execution=wasm

# parachain 1001 collator node 1 alice 
moonbeam/target/release/moonbeam --ws-port=34202 --port=34200 --rpc-port=34201 --collator --tmp --chain=moonbase-local --alice --unsafe-rpc-external --unsafe-ws-external --rpc-methods=Unsafe --rpc-cors=all -- --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --execution=wasm

# parachain 1001 collator node 2 bob
moonbeam/target/release/moonbeam --ws-port=34212 --port=34210 --rpc-port=34211 --collator --tmp --chain=moonbase-local --bob --unsafe-rpc-external --unsafe-ws-external --rpc-methods=Unsafe --rpc-cors=all -- --chain=/home/mathxh/metablock-projects/moonbeam/tools/rococo-local-raw.json --execution=wasm
```

以上可以看到中继链的Validator node启动了4个。Moonbeam(0.26.0)平行链启动了2个collator nodes分别是Alice和Bob，同样，Moonbeam local（已经编译好的）平行链也启动了2个collator nodes也分别是Alice和Bob，这下子知道有两个平行链了。跨链环境准备完成。

# XCM的交互性

## 概览

## XC-20s

### XC-20s的概览

#### 介绍

跨共识消息（XCM）格式定义了如何在可互操作的区块链之间发送消息。这种格式为Moonbeam/Moonriver和中继链或Polkadot/Kusama生态系统中的其他parachains之间传输消息和资产（substrate资产）打开了大门。

substrate资产是原生就支持可互操作的。然而，开发人员需要利用Substrate API与它们进行互动，这使得开发人员的体验不理想，特别是对于那些来自以太坊世界的人。因此，为了帮助开发者利用Polkadot/Kusama提供的原生互操作性，Moonbeam引入了XC-20s的概念。

XC-20是Moonbeam上一个独特的资产类别。它结合了Substrate资产的力量（原生互操作性），但允许用户和开发者通过熟悉的ERC-20接口 [moonbeam/ERC20.sol at master · PureStake/moonbeam (github.com)](https://github.com/PureStake/moonbeam/blob/master/precompiles/assets-erc20/ERC20.sol) ，通过预编译合约（Ethereum API）与他们互动。此外，开发人员可以将XC-20与常规的以太坊开发框架或dApps整合。

![[Pasted image 20220913100241.png]]


#### XC-20s的类型

external XC-20是原生的跨链资产，从另一个parachain或中继链转移到Moonbeam。因此，实际的代币存在于Moonbeam在这些链(parachain或中继链)中的主权账户中。external XC-20都会在其名称前加上xc，以区分其为本地跨链资产。  
  
可铸造(mintable)的XC-20也是跨链资产，但是，它们是直接在Moonbeam上铸造和燃烧的，可以转移到其他parachain。由于可铸造的XC-20是在Moonbeam上创建的，不是其他parachain或中继链的原生资产，资产的名称、符号和小数点都是完全可配置的。因此，它们不一定会在资产名称或符号前加上xc。  
  
这两种类型的XC-20的核心都是Substrate资产，在低层次上通过Substrate API进行交互。然而，Moonbeam提供了一个ERC-20接口来与这些资产互动，因此不需要Substrate的知识。从用户的角度来看，两种类型的XC-20都是以同样的方式进行交互。这里唯一的区别是，可铸币的XC-20包含了ERC-20接口的扩展，有一些管理资产的额外功能，如铸币和燃烧。  
  
XC-20的跨链转移是使用X-Tokens pallet进行的。转移可铸币XC-20和externa XC-20的指令略有不同，取决于给定资产的multilocation。  
  
#### XC-20s vs. ERC-20

虽然XC-20s和ERC-20s非常相似，但也要注意一些明显的区别。

首先，XC-20s是基于substrate的资产，因此，它们也会受到substrate特性的直接影响，如治理。此外，通过Substrate API完成的XC-20s交易不会从基于EVM的区块链浏览器（如Moonscan）[Moonbeam (GLMR) Blockchain Explorer (moonscan.io)](https://moonscan.io/) 中看到。只有通过Ethereum API完成的交易才能通过这些探索器看到。

然而，XC-20s可以通过ERC-20接口进行交互，因此它们有额外的好处，即可以从Substrate和Ethereum APIs中访问。这最终为开发者在处理这些类型的资产时提供了更大的灵活性，并允许与基于EVM的智能合约无缝集成，如DEX、借贷平台等。

#### ERC-20的solidity接口

- [moonbeam/ERC20.sol at master · PureStake/moonbeam (github.com)](https://github.com/PureStake/moonbeam/blob/master/precompiles/assets-erc20/ERC20.sol)
- [EIP-20: Token Standard (ethereum.org)](https://eips.ethereum.org/EIPS/eip-20)

#### ERC-20的许可的solidity接口

Moonbeam上的Permit.sol接口遵循EIP-2612标准，它用许可功能扩展了ERC-20接口。许可证是签名的信息，可以用来改变一个账户的ERC-20 allowance。这个allowance也就是对授权账户的限额。跟ERC-20的approve接口有关。

- [moonbeam/Permit.sol at master · PureStake/moonbeam (github.com)](https://github.com/PureStake/moonbeam/blob/master/precompiles/assets-erc20/Permit.sol)
- [EIP-2612: permit – 712-signed approvals (ethereum.org)](https://eips.ethereum.org/EIPS/eip-2612)





### External XC-20s

#### 介绍

正如XC-20概述中所述，有两种类型的XC-20：extarnal和mintable。external XC-20和可铸币XC-20的关键区别是，external XC-20代表锁定在Moonbeam的主权账户中的资产，无论是中继链还是其他parachains。相比之下，可铸币XC-20代表直接在Moonbeam中铸币/燃烧的资产，但具有原生的XCM互操作性特征。本指南将介绍外部XC-20。  
  
虽然XC-20是Substrate资产，但Moonbeam通过一个预编译合约的简单易用的ERC-20接口，抽象出与Substrate API的低级互动。这允许开发者与XC-20s互动，就像他们与任何其他ERC-20一样。请注意，XC-20预编译不支持跨链转移，这是有意为之，以尽可能地接近标准的ERC-20接口。  
  
外部XC-20资产都会在其名称前加上xc。例如，Kusama在Moonriver上的KSM代表将被称为xcKSM。在使用之前，它们需要注册并链接到生态系统中的另一个资产。这是通过民主提案的白名单程序来完成的。  

### Mintable XC-20s

#### 介绍

正如XC-20概述中所述，有两种类型的XC-20：外部和可铸币。外部XC-20和可铸币XC-20的关键区别是，可铸币XC-20代表的是直接在Moonbeam中铸币/烧制的资产，但具有本地XCM互操作性的特点。此外，可铸型XC-20可以转移到任何其他parachain，只要它在该链上注册为XCM资产，正如XCM概述页中所涉及的那样。相比之下，外部XC-20代表锁定在Moonbeam在中继链或其他parachains的主权账户中的资产，并在Moonbeam上注册为资产。本指南将介绍可铸币的XC-20。  
  
所有XC-20的核心都是Substrate资产。通常，对于Substrate资产，开发者需要直接与Substrate API交互。然而，Moonbeam消除了对Substrate知识的需求，允许用户和开发者通过预编译合约的ERC-20接口与这些资产进行互动。因此，开发者可以使用标准的以太坊开发者工具来与这些资产进行互动。可铸币XC-20包括ERC-20接口的扩展，有一些额外的功能，用于管理资产和设置元数据，如资产的名称、符号和小数。还有一些额外的角色用于资产注册和管理。  
  
目前，可铸的XC-20资产需要通过民主提案创建，并通过链上治理进行投票。一旦提案获得多数票并被批准，该资产就可以在Moonbeam上注册和铸造了。此外，创建一个可铸造的XC-20代币需要交纳押金（保证金）。  
  


### Send XC-20s

这里可以参考X-token pallet那节。

## XC的channel注册

## 远程执行

## XCM Fees
