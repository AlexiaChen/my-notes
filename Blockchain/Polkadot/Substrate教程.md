## substrate 启动参数

### 单节点

```bash
./target/release/node-template --dev
```

```txt
--dev  
Specify the development chain.  
  
This flag sets `--chain=dev`, `--force-authoring`, `--rpc-cors=all`, `--alice`, and  
`--tmp` flags, unless explicitly overridden.

--tmp  
Run a temporary node.  
  
A temporary directory will be created to store the configuration and will be deleted at  
the end of the process.  
  
Note: the directory is random per process execution. This directory is used as base path  
which includes: database, node key and keystore.  
  
When `--dev` is given and no explicit `--base-path`, this option is implied.

--force-authoring  
Enable authoring even when offline

--alice  
Shortcut for `--name Alice --validator` with session keys for `Alice` added to keystore

--chain <CHAIN_SPEC>  
Specify the chain specification.  
  
It can be one of the predefined ones (dev, local, or staging) or it can be a path to a  
file with the chainspec (such as one exported by the `build-spec` subcommand).
```

### 双节点网络

```bash
./target/release/node-template --base-path /tmp/alice --chain local --alice --p  
ort 30333 --ws-port 9945 --rpc-port 9933 --node-key 0000000000000000000000000000000000000000000000000000000000000001 --t  
elemetry-url "wss://telemetry.polkadot.io/submit/ 0" --validator
```

```bash
./target/release/node-template --base-path /tmp/bob --chain local --bob --port  
30334 --ws-port 9946 --rpc-port 9934 --telemetry-url "wss://telemetry.polkadot.io/submit/ 0" --validator --bootnodes /ip  
4/127.0.0.1/tcp/30333/p2p/12D3KooWEyoppNCUx8Yx66oV9fJnriXwCcXwDDUA2kj6vnc6iDEp
```

```bash
--bob  
Shortcut for `--name Bob --validator` with session keys for `Bob` added to keystore
```

### 删除数据

```bash
./target/release/node-template purge-chain --base-path /tmp/alice --chain local
./target/release/node-template purge-chain --base-path /tmp/bob --chain local
```


## 创建自己的私有网络

### 创建自己的Keys

不用预定义好的`--chain <local | dev | staging>`

也不用预定义好的账号和Keys `--alice --bob`等这些。需要用自己创建的密钥给Validator node，这样这和key就可以给块签名了，记住，每一个区块链网络的参与者，都需要一个自己的唯一的密钥。

至于生成密钥对的工具用node template subkey命令或者其他第三方工具

```bash
./target/release/node-template key generate --scheme Sr25519 --password-interactive
```

```txt
Secret phrase: across fuel flush example tornado cheese strategy vibrant notice legend foot frost  
Network ID: substrate  
Secret seed: 0x5386174c42477764a333dc0c262d2507fc3eea14b97383902bf76f382109a85e  
Public key (hex): 0x5e4f5309eb2b1e23ab5e68c48c848a3bbbeab6803b55178dc554970737ace758  
Account ID: 0x5e4f5309eb2b1e23ab5e68c48c848a3bbbeab6803b55178dc554970737ace758  
Public key (SS58): 5ECMvTpbBAb3BbZYMoQXAVXJM579jPReWvMy2S7gd87Kqssi  
SS58 Address: 5ECMvTpbBAb3BbZYMoQXAVXJM579jPReWvMy2S7gd87Kqssi
```

这样你就有一个key来生产块了(用aura协议)。然后你用上面`Secret phrase` 这个助记词来Derived其他的Key, 比如Ed25519:

```bash
./target/release/node-template key inspect --password-interactive --scheme Ed25519 "across fuel flush example tornado cheese strategy vibrant notice legend foot frost"
```

```txt
Secret phrase: across fuel flush example tornado cheese strategy vibrant notice legend foot frost  
Network ID: substrate  
Secret seed: 0x5386174c42477764a333dc0c262d2507fc3eea14b97383902bf76f382109a85e  
Public key (hex): 0x0693cc1f380610301ad0eead21b2c2f53696d0d33bc8083a5b4d15adb32beadd  
Account ID: 0x0693cc1f380610301ad0eead21b2c2f53696d0d33bc8083a5b4d15adb32beadd  
Public key (SS58): 5CDL3pcUnEEoQJMhJF2hbmVtBRZ8pNQPGwSurhGo8Wc4HXwc  
SS58 Address: 5CDL3pcUnEEoQJMhJF2hbmVtBRZ8pNQPGwSurhGo8Wc4HXwc
```

这样就有有一个ed25519的Key对块进行确认了(用grandpa协议)。

因为上面的sr25519和ed25519的key只是给一个节点的，我们要模拟两个节点的网络，所以需要，给另一个节点生成Keys，重复以上步骤。或者，因为着是一个教程，第二个节点可以暂时用预定义好的Key，比如Alice或者Bob。

当然，这个教程的第二个节点是用:

```txt
Sr25519
Secret phrase: metal gown mail case fiber bread provide issue much cannon rabbit clap  
Network ID: substrate  
Secret seed: 0x7529887b6cbaf33d8db85abc1266d11159b7d6452c759d3f285f82e292723000  
Public key (hex): 0x5895e5fdb56afa73f8d8cab8497125bd7961bc3a20d7b24c6d616846520ecd15  
Account ID: 0x5895e5fdb56afa73f8d8cab8497125bd7961bc3a20d7b24c6d616846520ecd15  
Public key (SS58): 5E4rbm8PEDuYjxRu2AWyMJr3kwCMkv6c1pyJ3Y4WAKnjZoeY  
SS58 Address: 5E4rbm8PEDuYjxRu2AWyMJr3kwCMkv6c1pyJ3Y4WAKnjZoeY

###################################################

Ed25519

Secret phrase: metal gown mail case fiber bread provide issue much cannon rabbit clap  
Network ID: substrate  
Secret seed: 0x7529887b6cbaf33d8db85abc1266d11159b7d6452c759d3f285f82e292723000  
Public key (hex): 0x9161c93da5a53eb326cf034f5ec8e62cffc72492d5170743b77000ee07c76ea0  
Account ID: 0x9161c93da5a53eb326cf034f5ec8e62cffc72492d5170743b77000ee07c76ea0  
Public key (SS58): 5FMKrCE2Hv8Qiy9t2gJ9h6MNrC1eiWFHPJeDSfzhwj3zP5Ut  
SS58 Address: 5FMKrCE2Hv8Qiy9t2gJ9h6MNrC1eiWFHPJeDSfzhwj3zP5Ut
```

-   Sr25519: 5E4rbm8PEDuYjxRu2AWyMJr3kwCMkv6c1pyJ3Y4WAKnjZoeY for `aura`.
-   Ed25519: 5FMKrCE2Hv8Qiy9t2gJ9h6MNrC1eiWFHPJeDSfzhwj3zP5Ut for `grandpa`.

### 创建自己的Chain Spec

在你生成了与你的区块链一起使用的密钥后，你就可以使用这些密钥对创建一个自定义的Chain Spec，然后与被称为Validator的受信任的网络参与者分享你的自定义Chain Spec。

为了使其他人能够参与你的区块链网络，确保他们生成自己的密钥。在你收集了网络参与者的密钥后，你可以创建一个自定义Chain Spec来取代本地链规范。

为了简单起见，你在本教程中创建的自定义Chain Spec是`--chain local`的修改版，说明了如何创建一个双节点网络。如果你有所需的密钥，你可以按照同样的步骤在网络中添加更多的节点。

#### 修改local chain spec

你可以修改预定义的local chain spec，而不是编写一个全新的chain spec。

```bash
./target/release/node-template build-spec --disable-default-bootnode --chain lo  
cal > customSpec.json
```

![[1661741417914.png]]

spec大概长这样，有一段很长的code字段，实际上runtime的WASM字节码。

然后把之前生成好的sr25519和ed25519的SS58格式的public Key分别添加到aura和grandpa的`authorities`字段下，不必删除以前的，追加就好。

![[10999e34674cea6780527fee551cb08.png]]

请注意，在`gandpa` 字段的 `authorities`字段有两个数据值。第一个值是地址Key(SS58)。第二个值是用来支持加权投票的。在这个例子中，每个Validator的权重是1票。

#### 把修改后的chain spec转换成raw格式

```bash
./target/release/node-template build-spec --chain=customSpec.json --raw --disable-default-bootnode > customSpecRaw.json
```


#### 添加其他Validators

注意，如果你有其他Validator，那么着两个字段，就要添加其他Validator的sr25519和ed25519的SS58 key。每一个Validator都有一个独立唯一的。如果两个validator的key配置重复了，那么就生成冲突的块。

[Cryptography | Substrate_ Docs](https://docs.substrate.io/reference/cryptography/#public-key-cryptography)


### 把chain spec分享给别人

如果你正在创建一个私有区块链网络与其他参与者分享，确保只有一个人创建chain spec，并与网络中的所有其他Validator分享该规范的结果raw格式--例如，customSpecRaw.json文件。必须要是raw格式。

因为Rust编译器产生的优化的WASM二进制字节文件是不能确定地重现的，每个生成Wasm runyime的人都会产生一个稍微不同的Wasm blob。为了确保确定性，区块链网络的所有参与者必须使用完全相同的chain spec的raw格式文件。关于这个问题的更多信息，请参阅 [Hunting down a non-determinism-bug in our Rust Wasm build - DEV Community 👩‍💻👨‍💻](https://dev.to/gnunicorn/hunting-down-a-non-determinism-bug-in-our-rust-wasm-build-4fk1)

### 准备启动私有网络

启动网络之前，请确定以下步骤是否完成:

- 生成区块的Authority账户，就是前面提到的sr25519和ed25519的public keys
- 把这两个key加入chain spec的相关字段种
- 然后生成chain spec的raw格式

### 启动第一个节点

```bash
./target/release/node-template \
  --base-path /tmp/node01 \
  --chain ./customSpecRaw.json \
  --port 30333 \
  --ws-port 9945 \
  --rpc-port 9933 \
  --telemetry-url "wss://telemetry.polkadot.io/submit/ 0" \
  --validator \
  --rpc-methods Unsafe \
  --name MyNode01 \
  --password-interactive
```

然后输入KeyStore的密码，你就可以见到第一个节点运行起来了，检查下控制台的输出是否用了你自定义的chain spec，没问题就行。你会看到这个链没有生成任何块，是因为还你需要把之前生成的密钥导入到KeyStore里。

### 导入Key到第一个节点的KeyStore

- 先导入Sr25519的Key来为aura协议服务

```bash
./target/release/node-template key insert --base-path /tmp/node01 --chain customSpecRaw.json --scheme Sr25519 --suri 0x5386174c42477764a333dc0c262d2507fc3eea14b97383902bf76f382109a85e --password-interactive --key-type aura
```

或者

```bash
./target/release/node-template key insert --base-path /tmp/node01 --chain customSpecRaw.json --scheme Sr25519 --suri "across fuel flush example tornado cheese strategy vibrant notice legend foot frost" --password-interactive --key-type aura
```

- 然后导入Ed25519的Key来为grandpa服务

```bash
./target/release/node-template key insert --base-path /tmp/node01 --chain customSpecRaw.json --scheme Ed25519 --suri "across fuel flush example tornado cheese strategy vibrant notice legend foot frost" --password-interactive --key-type gran
```

或者

```bash
./target/release/node-template key insert --base-path /tmp/node01 --chain customSpecRaw.json --scheme Ed25519 --suri 0x5386174c42477764a333dc0c262d2507fc3eea14b97383902bf76f382109a85e --password-interactive --key-type gran
```

- 检查Key是否导入

```bash
ls /tmp/node01/chains/local_testnet/keystore  


617572615e4f5309eb2b1e23ab5e68c48c848a3bbbeab6803b55178dc554970737ace758  
6772616e0693cc1f380610301ad0eead21b2c2f53696d0d33bc8083a5b4d15adb32beadd
```


### 让第二个节点加入网络

```bash
./target/release/node-template \
  --base-path /tmp/node02 \
  --chain ./customSpecRaw.json \
  --port 30334 \
  --ws-port 9946 \
  --rpc-port 9934 \
  --telemetry-url "wss://telemetry.polkadot.io/submit/ 0" \
  --validator \
  --rpc-methods Unsafe \
  --name MyNode02 \
  --bootnodes /ip4/127.0.0.1/tcp/30333/p2p/12D3KooWLmrYDLoNTyTYtRdDyZLWDe1paxzxTw5RgjmHLfzW96SX \
  --password-interactive
```

把为第二个节点生成的sr25519和ed25519的Key导入到第二个节点的KeyStore中。

```bash
./target/release/node-template key insert --base-path /tmp/node02 --chain cus  
tomSpecRaw.json --scheme Sr25519 --suri "metal gown mail case fiber bread provide issue much cannon rabbit clap" -  
-password-interactive --key-type aura

./target/release/node-template key insert --base-path /tmp/node02 --chain c  
ustomSpecRaw.json --scheme Ed25519 --suri "metal gown mail case fiber bread provide issue much cannon rabbit clap"  
--password-interactive --key-type gran
```


重启节点，这样你就可以看到这两个Validator正在共识并且出块了。并且通过前端的ws链接到两个节点上，看到的balances没有这些Keys的Address，就说明这些Keys默认不会当作普通的Address显示余额。只做纯粹的Aura和grandpa共识用。

### RPC查询链的状态

以上我们只是在Web前端界面上通过Polkadot-JS这样的JS接口来查询节点的状态，并且它还是用WebSocket协议连接的节点，我们想用http来调用JSON-RPC。看上面的节点启动命令已经有`--rpc-port 9933` 这些开关了可以用Curl来试着调用了:

```bash
curl -H "Content-Type: application/json" -d '{"id":1, "jsonrpc":"2.0", "  
method": "rpc_methods"}' http://localhost:9933/
```

```json
{"jsonrpc":"2.0","result":{"methods":["account_nextIndex","author_hasKey","author_hasSessionKeys","author_insertKey","au  
thor_pendingExtrinsics","author_removeExtrinsic","author_rotateKeys","author_submitAndWatchExtrinsic","author_submitExtr  
insic","author_unwatchExtrinsic","chain_getBlock","chain_getBlockHash","chain_getFinalisedHead","chain_getFinalizedHead"  
,"chain_getHead","chain_getHeader","chain_getRuntimeVersion","chain_subscribeAllHeads","chain_subscribeFinalisedHeads","  
chain_subscribeFinalizedHeads","chain_subscribeNewHead","chain_subscribeNewHeads","chain_subscribeRuntimeVersion","chain  
_unsubscribeAllHeads","chain_unsubscribeFinalisedHeads","chain_unsubscribeFinalizedHeads","chain_unsubscribeNewHead","ch  
ain_unsubscribeNewHeads","chain_unsubscribeRuntimeVersion","childstate_getKeys","childstate_getKeysPaged","childstate_ge  
tKeysPagedAt","childstate_getStorage","childstate_getStorageEntries","childstate_getStorageHash","childstate_getStorageS  
ize","offchain_localStorageGet","offchain_localStorageSet","payment_queryFeeDetails","payment_queryInfo","state_call","s  
tate_callAt","state_getChildReadProof","state_getKeys","state_getKeysPaged","state_getKeysPagedAt","state_getMetadata","  
state_getPairs","state_getReadProof","state_getRuntimeVersion","state_getStorage","state_getStorageAt","state_getStorage  
Hash","state_getStorageHashAt","state_getStorageSize","state_getStorageSizeAt","state_queryStorage","state_queryStorageA  
t","state_subscribeRuntimeVersion","state_subscribeStorage","state_traceBlock","state_unsubscribeRuntimeVersion","state_  
unsubscribeStorage","subscribe_newHead","system_accountNextIndex","system_addLogFilter","system_addReservedPeer","system  
_chain","system_chainType","system_dryRun","system_dryRunAt","system_health","system_localListenAddresses","system_local  
PeerId","system_name","system_nodeRoles","system_peers","system_properties","system_removeReservedPeer","system_reserved  
Peers","system_resetLogFilter","system_syncState","system_unstable_networkState","system_version","unsubscribe_newHead"]  
,"version":1},"id":1}
```

```bash
curl -H "Content-Type: application/json" -d '{"id":1,  
"jsonrpc":"2.0", "method": "chain_getBlock"}' http://localhost:9933/
```

```json
{"jsonrpc":"2.0","result":{"block":{"header":{"parentHash":"0xbab3c3ee72118f41e3ac8b073eff26dbe2b7a61860eba9f16eebc4f389  
ea9d0d","number":"0x6e","stateRoot":"0xe029714536af72c65999b972973bec2ecd68b19f037b1cc908d5badcd101b162","extrinsicsRoot  
":"0xd6b3bfdb9906d61d1eba1e9e9431c427799d3e6bd15585ede673353b72604442","digest":{"logs":["0x0661757261202b11821000000000  
","0x056175726101014e8e818600ae7e7ef179dc861f8f3c0aac34990946da7587434e8c68a9f8166f880bb5d827f661f65288f9b9efe592628a7bb  
312099cde44645d9ba5e2cae08f"]}},"extrinsics":["0x280402000bd05f72e88201"]},"justifications":null},"id":1}
```

```bash
curl -H "Content-Type: application/json" -d '{"id":1, "jsonrpc":"2.0", "me  
thod": "chain_getBlock", "params": {"BlockHash": "0xbab3c3ee72118f41e3ac8b073eff26dbe2b7a61860eba9f16eebc4f389"}}' http:  
//localhost:9933/
```

```json
{"jsonrpc":"2.0","result":{"block":{"header":{"parentHash":"0x6230dbf6df6cb2fb2832f0983fdf183f58c11a10f477d71b7e1e7ce028  
844aba","number":"0x7c","stateRoot":"0x74a50401d11b4aa467c0336632491beb38cfe4624a6e1d0f7c383ddb1df626c9","extrinsicsRoot  
":"0xd0e4d875d56cc2a65e91c177aec295a57eb64a2182765fece34cfbe1ea76dcf7","digest":{"logs":["0x0661757261204711821000000000  
","0x05617572610101c4d068f538ea537f29c6cd83867505f402210b4329ab062e1ede950acb848366bdf173c78287d989911e7663958bb8a61684d  
2c2c5be145403cde57b87959885"]}},"extrinsics":["0x280402000b11f074e88201"]},"justifications":null},"id":1}
```

- [JSON-RPC | polkadot{.js}](https://polkadot.js.org/docs/substrate/rpc/#chain)
- [Querying Substrate Storage via RPC | Shawn Tabrizi](https://www.shawntabrizi.com/substrate/querying-substrate-storage-via-rpc/)

## 创建自己的Permission网络

这里是带许可的链，也就是不是permissionless了。

## 监控节点指标

TODO

## 升级运行中的网络

TODO


## 准备一个本地的平行链测试网

在本教程中，你将配置一个本地中继链并连接一个parachain模板，以便在本地测试环境中使用。

一定要注意匹配版本号

- Polkadot v0.9.26
- Substrate Parachain template `polkadot-v0.9.26`
- Polkadot-JS App `v0.116.2-34`

### 构建中继链节点

```bash
git clone --depth 1 --branch release-v0.9.26 https://github.com/paritytech/polkadot.git

cd polkadot
cargo build --release

# Check if the help page prints to ensure the node is built correctly
./target/release/polkadot --help
```

### 中继链的Chain Spec

[Chain specification | Substrate_ Docs](https://docs.substrate.io/build/chain-spec/)

你将需要为你的中继链网络制定一个chain spec。由于我们想使用本地测试网，可能需要一个自定义的配置来设置Validator的开发或自定义密钥，启动节点地址等。

一个中继链运行的Validator节点必须比连接的parachain collators(收集人节点)总数多一个。为了测试，这些通常必须硬编码到你的chain spec中。例如，如果你想用一个collator连接2个parachain，那么需要运行3个或更多的中继链Validator节点，并确保它们都在你的chain spec中被指定。

无论你选择使用哪种chain spec文件，我们都会在下面的说明中把该文件简称为chain-spec.json。你需要提供你所使用的chain spec的正确路径。

#### 预定义好的chain spec文件

本教程包括一个样本chain spec文件，其中有两个Validator中继链节点--Alice和Bob--作为出块节点。你可以在不修改的情况下将这个样本chain spec用于本地测试网络和单一的parachain。这对于注册一个单一的parachain是很有用的。

- plain spec https://raw.githubusercontent.com/substrate-developer-hub/substrate-docs/main/static/assets/tutorials/cumulus/chain-specs/rococo-custom-2-plain.json
- raw spec https://raw.githubusercontent.com/substrate-developer-hub/substrate-docs/main/static/assets/tutorials/cumulus/chain-specs/rococo-custom-2-raw.json

你可以修改上面的plain spec然后再转换成raw来使用，也可以直接使用 SCALE-encoded raw的格式。

详情请看怎样定制chain spec的文档

[Customize a chain specification | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/basics/customize-a-chain-specification/)

这个样本chain spec只对有两个Validator节点的单一parachain有效。如果你添加了其他Validator，或者在你的中继链上添加了额外的parachain，或者想使用自定义的非开发密钥，你需要创建一个符合你需求的自定义chain spec。

### 启动你的中继链

#### 启动alice validator


Alice的Validator节点:

```bash
# Start Relay `Alice` node
./target/release/polkadot --alice --validator --base-path /tmp/relay/alice --chain ./rococo-custom-2-raw.json --port 2033 --ws-port 9944 --rpc-port 9934
```



#### 启动bob validator

Bob的Validator节点:

```bash
./target/release/polkadot --bob --validator --base-path /tmp/replay/bob --chain ./rococo-custom-2-raw.json --bootnodes /ip4/127.0.0.1/tcp/2033/p2p/12D3KooWPftctF23pkU858Uj5LV584xJ2KTXTC256YvpKGt8pEx4 --port 2034 --ws-port 9945 --rpc-port 9935
```

可以发现，波卡的命令行参数与Substrate一样的，没做任何自定义修改，

注意： 在本教程中，你的最终chain spec文件名必须以rococo开头，否则节点将不知道要包括哪个runtime逻辑。

## 连接一个本地平行链

这教程达到的目的就是，注册一个平行连到relay chain上，然后平行连的区块产生，也在中继链上。

### 编译substrate的平行连模板

```bash
# Clone the parachain template with the correct Polkadot version
git clone --depth 1 --branch polkadot-v0.9.26 https://github.com/substrate-developer-hub/substrate-parachain-template

# Switch into the parachain template directory
cd substrate-parachain-template

# Build the parachain template collator
cargo b -r
```

### 预留一个ParaID

因为平行链要连接中继链，所以中继链上要保留一个平行链ID。

要连接到任何中继链，你必须首先为你的parachain保留一个ParaID。它被称为ParaID，因为同一标识符可以作为parachain ID或parathread ID使用，这取决于你当前的中继链插槽状态。你必须有足够的资金来保留一个ParaID。请参考你的目标中继链来确定所需的代币数量。

中继链负责分配所有的ParaID，对于所有不是common good Parachain [Common Good Parachains · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-common-goods)，只需从2000开始递增。这些链使用不同的方法来分配ParaIDs。

这个练习说明了如何使用Polkadot-JS Apps [Polkadot/Substrate Portal](https://polkadot.js.org/apps/?rpc=ws%3A%2F%2F127.0.0.1%3A9944#/parachains/parathreads) 连接到你的本地中继链来保留一个ParaID。在network > Parachains子页面下，点击Parathreads标签，使用+ParaId按钮。

![[Pasted image 20220830093204.png]]

请注意，用于保留ParaID的账户也必须是使用这个`ParaID`的起源(Origin)。在你提交这个交易并看到一个成功的`Registrar.Reserved`事件与你的`ParaID`后，你可以使用保留的ID启动你的parachain或parathread。你现在已经准备好使用保留的`ParaID（ParaID 2000）`生成你的parachain所需的文件。

这里我选择保留ParaID的账户是Alice。Polkadot-JS App连接到的也是Alice的validator节点（ws协议的9944端口）。现在预留的ParaID是2000.

### 自定义parachain的chain spec

你的chain spec必须为你的parachain设置正确的保留ParaID。

请参阅配置自定义链规范的方法指南，[Customize a chain specification | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/basics/customize-a-chain-specification/) 以了解更多关于生成plain spec、修改规范和生成raw spec的深入说明。

我们首先生成一个plain spec:

```bash
# Assumes that `rococo-local` is in `node/chan_spec.rs` as the relay you registered with
./target/release/parachain-template-node build-spec --disable-default-boot  
node > rococo-local-parachain-plain.json
```

编辑这个json文件，修改paraID为之前保留的2000:

![[b3c4e38b115ad7e8d99870df12ee5ba.png]]

保存并生成raw spec:

```bash
./target/release/parachain-template-node build-spec --chain rococo-local-p  
arachain-plain.json --raw --disable-default-bootnode > rococo-local-parachain-id-2000-raw.json
```


#### 获取WASM runtime校验功能

中继链也需要parachain特有的runtime验证逻辑来验证parachain块。parachain collator节点也有一个命令来产生(导出)这个Wasm blob。

```bash
./target/release/parachain-template-node export-genesis-wasm --chain rococo-local-parachain-id-2000-raw.json > para-id-2000-wasm
```

#### 生成一个平行链的创世状态(genesis state)

为了注册一个平行链，中继链需要知道平行链的创世状态。collator节点可以将该状态导出到一个文件。进入你的Parachain Template文件夹，下面的命令将创建一个包含parachain的整个创世状态的文件，并以十六进制编码:

```bash
./target/release/parachain-template-node export-genesis-state --chain rococo-local-parachain-id-2000-raw.json > para-id-2000-genesis
```

##### 不允许有创世块之前的区块

这个导出的runtime和状态只适用于parachain的创世块。你不能把一个有任何先前状态的parachain连接到中继链上。所有的在中继链上的parachain，它只能从自己的区块0开始。最终，迁移建立在Substrate上的单独链(solo chain)的区块历史预计是可能的，但这个功能并不打算很快实现。

关于如何创建parachain模板以及如何将你的链的逻辑(而不是它的历史或状态迁移)转换为parachain的细节，请参见关于将单独链(solo chain)转换为parachain的指南。[Convert a solo chain | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/parachains/convert-a-solo-chain/)


### 启动collator节点

我们现在可以启动一个单一的collator节点。注意，我们需要在命令的后半部分提供我们用于启动中继链节点的相同中继链的chain spec。用下面的命令启动一个collator:

```bash
./target/release/parachain-template-node --alice --collator --force-authoring --chain rococo-local-parachain-id-2000-raw.json --base-path /tmp/parachain/alice --port 40333 --ws-port 8844 -- --execution wasm --chain ../polkadot/rococo-custom-2-raw.json --port 30343 --ws-port 9977
```

```bash
./target/release/parachain-template-node \
--alice \
--collator \
--force-authoring \
--chain rococo-local-parachain-id-2000-raw.json \
--base-path /tmp/parachain/alice \
--port 40333 \
--ws-port 8844 \
-- \
--execution wasm \
--chain ../polkadot/rococo-custom-2-raw.json \
--port 30343 \
--ws-port 9977
```

```txt
2022-08-30 14:54:59 💤 Idle (0 peers), best: #0 (0xc42e…7d95), finalized #0 (0xc42e…7d95), ⬇ 0 ⬆ 
02022-08-30 14:54:59 💤 Idle (2 peers), best: #1543 (0x3f0a…5613), finalized #1541 (0x4736…2f31), ⬇ 0.4kiB/s ⬆ 0.4kiB/s
2022-08-30 14:55:04 💤 Idle (0 peers), best: #0 (0xc42e…7d95), finalized #0 (0xc42e…7d95), ⬇ 0 ⬆ 
02022-08-30 14:55:04 💤 Idle (2 peers), best: #1543 (0x3f0a…5613), finalized #1541 (0x4736…2f31), ⬇ 0.2kiB/s ⬆ 0.2kiB/s
```


关于这个命令，首先要注意的是，在`--`之前传递了几个参数，而在它之后又传递了几个参数。一个parachain collator包含实际的collator节点和一个嵌入式的中继链节点。--之前的参数是针对collator的，而--之后的参数是针对嵌入式中继链节点的。

我们给collator一个基本路径和端口，就像我们之前为中继链节点做的那样。

如果你第二次为第二个parachain执行这些指令，记住要改变collator的特定值。你必须使用相同的中继链chain spec，但为第二个parachain collator绑定不同的ParaID、基本路径和端口号。

目前还没有办法在没有嵌入中继链节点的情况下运行parachain节点，但预计这最终将成为非collator节点的可能。

运行以后，你应该看到你的collator节点作为一个独立的节点运行，它的内嵌中继节点作为一个对等体与之前已经运行的中继链Validator节点连接。

注意，如果你没有看到嵌入式中继链与本地中继链节点在P2P网络上，请尝试禁用你的防火墙或用中继节点的地址添加bootnodes参数。

你还没有看到它还没有开始编写parachain的区块。当collator节点在中继链上实际注册时，将开始创作。

### 平行链注册

有几种可能的方法来注册一个parachain到一个中继链。对于测试来说，使用sudo是最常见的。

#### 使用`sudo`注册

我们的中继链已经启动，我们的parachain的collator节点也准备就绪。现在我们必须在中继链上注册parachain。在一个生产网络(polkadot网络)中，这通常是通过parachain拍卖来完成的 [Parachain Slot Auctions · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-auction)。但在本教程中，我们将用sudo调用来完成。

#### 注册交易

交易可以通过Polkadot-JS应用程序用户界面在中继链节点上提交。这方面有多种选择，我们可以选择以下任何一种。请注意，这里的所有选项都取决于之前那个预留的paraID哪一步骤--所以一定要先做这个预留paraID。

如果你正在运行一个有两个以上Validator nodes的中继链网络，你可以通过相同的界面添加更多的parachains，并相应调整参数。

##### Option 1: `paraSudoWrapper.sudoScheduleParaInitialize`

这个选项完全绕过了slot租赁机制，从下一个中继链会话开始，为预留的parachain或parathread搭载一个paraID。这是最简单和最快的测试方法。这些注册parachain所需的文件包括在你的chain spec中设置的细节，必须明确针对正确的中继链并使用正确的ParaID--在这种情况下，`rococo`就是用这样的方法（而不是本教程中使用的`rococo-local`）。

-  访问 [Polkadot Apps UI](https://polkadot.js.org/apps/), 连接你的中继链
-  访问Apps的Developer下的sudo页面，来执行一个`sudo extrinsics` 交易
-  选择`paraSudoWrapper` -> `sudoScheduleParaInitialize(id, genesis)` 作为此交易的extrinsics 类型，如下
![[Pasted image 20220830151000.png]]
![[c40945f602d1fcbf5a25da867374b2c.png]]
- 在此交易的参数(extrinsics parameters)中，指定:
    - 设置`id: paraId` 为之前设置的2000
    - `genesisHead`: 上传之前导出的文件`para-id-2000-genesis`
    - `validationCode`: 上传之前导出的文件 `para-id-2000-wasm`
    - 设置`parachain: Bool` 选项为Yes

如果这步成功了。就会在你的中继链浏览器页面上看到`sudo.Sudid`事件被发送出来了。

##### Option 2: `slots.forceLease`

这是在生产中使用的更正式的注册流程（除了使用sudo来强制租赁插槽）：你用保留它的同一个账户来注册你的保留paraID，这个教程使用Alice账户来发送预留paraId的交易的，或者如果你愿意的话，使用sudo的egistrar.forceRegister 交易。

在这里，我们将使用sudo来授予自己一个租赁。你应该有一个在看板上的parathread。

![[Pasted image 20220830151823.png]]

- 访问 [Polkadot Apps UI](https://polkadot.js.org/apps/), 连接你的中继链
- 访问Apps的Developer下的sudo页面，来执行一个`sudo extrinsics` 交易
- 选择`slots` -> `forceLease(para, leaser, amount, period_begin, period_end)` 作为一个交易类型，如下

![[Pasted image 20220830152433.png]]

请确保将begin period设置为你希望开始的插槽。在测试中，如果你开始一个新的中继链，这很可能是已经激活的槽0。除非你想测试入职(onboarding)和离职(offboading)周期，否则将其延长到你希望测试这个parachain的时间之外可能是最好的。在这种情况下，那么选择有空隙的paraID的插槽租赁将是有序的。一旦完全onboard，在区块生产开始后，你应该看到。

![[Pasted image 20220830152720.png]]

#### 区块生成

一旦注册交易成功上中继链成，下一个中继链epoch开始，collator就应该开始生产parachain链上的区块（又称整理）。你会看到collator既同步中继链的区块，也生成parachain的区块。生成parachain区块是从高度0开始，与中继链的高度并不保持一致。

> 这可能需要一些时间! 请耐心等待下一个epoch的开始。这是[Prepare a local parachain testnet](https://docs.substrate.io/tutorials/connect-other-chains/local-relay/)教程中的例子[`rococo-custom-2raw.json`](https://github.com/substrate-developer-hub/substrate-docs/blob/main/static/assets/tutorials/cumulus/chain-specs/rococo-custom-2-raw.json)的10个块。

> 注意中继链的epoch的变化! 只有这样，我们才开始整理parachain!

```txt
2022-08-31 09:32:08 💤 Idle (2 peers), best: #123 (0x0611…ed9a), finalized #120 (0x6bf4…7367), ⬇ 1.3ki  
B/s ⬆ 0.7kiB/s
2022-08-31 09:32:08 💤 Idle (0 peers), best: #3 (0x7ebc…79cc), finalized #3 (0x7ebc…79cc), ⬇ 0 ⬆ 0
2022-08-31 09:32:12 ✨ Imported #124 (0x40e5…6ceb)2022-08-31 09:32:12 Starting collation. relay_parent=0x40e523d96789989ea634aff65d3cd497e493a72a7ebeadb  
273dd8d1b31c16ceb at=0x12e1c08be3d808a2dae9a959a97fff4d0ff0a3100f8bcfca36496dd2136d8155
2022-08-31 09:32:12 👴 Applying authority set change scheduled at block #121
2022-08-31 09:32:12 👴 Applying GRANDPA set change to new set [(Public(88dc3417d5058ec4b4503e0c12ea1a0  
a89be200fe98922423d4334014fa6b0ee (5FA9nQDV...)), 1), (Public(d17c2d7823ebf260fd138f2d7e27d114c0145d96  
8b5ff5006125f2414fadae69 (5GoNkf6W...)), 1)]
...
...
2022-08-31 09:33:36 ✨ Imported #8 (0x0740…b0ce)2022-08-31 09:33:36 PoV size { header: 0.1787109375kb, extrinsics: 2.7470703125kb, storage_proof: 2.70  
8984375kb }
2022-08-31 09:33:36 Compressed PoV size: 4.9033203125kb
2022-08-31 09:33:36 Produced proof-of-validity candidate. block_hash=0x07408ab05ff0cb13d1f3fef314bd4f9  
4b18db618e93bfe6b18c73e74becfb0ce
```

#### 区块确认

中继链跟踪每个parachain的最新区块（头部）。当一个中继链的区块被确定后，每一个已经完成验证过程[The Path of a Parachain Block (polkadot.network)](https://polkadot.network/blog/the-path-of-a-parachain-block/)  的parachain区块也被确定。这就是Polkadot如何为它的parachain实现池化、共享的安全!

你可以通过点击Apps UI中的Network > Parachains来查看注册的parachains和它们的最新数据。

![[1661911054261.png]]

### 与你的平行链交互

启动和注册parachains的全部意义在于，我们可以向parachains提交交易并与之互动。我们终于准备好在我们的parachains上提交extrinsics了!

#### 用Polkadot-JS Apps UI连接你的平行链

我们已经将Apps UI连接到中继链节点。现在我们也可以连接到parachain collator。在一个新的浏览器窗口中打开另一个Apps UI的实例，并将其连接到相应的端点。如果你遵循本教程到目前为止，你可以连接到parachain节点上。

[Polkadot/Substrate Portal](https://polkadot.js.org/apps/?rpc=ws%3A%2F%2Flocalhost%3A8844#/explorer)

#### 提交交易(extrinsics)

你可以做一些简单的代币转移，以确保parachain的正常运行。你也可以通过进入Extrinsics页面，选择System pallet和 remark extrinsic来做一些链上备注。

如果交易如期进行，你就有了一个工作的parachain。

对于XCMP跨链交易

将准绳连接到一个共同的中继链的决定性特征是在所有连接的链之间进行通信的能力。这方面的功能处于发展的前沿，不包括在本教程中。

要了解更多关于XCMP的信息，请参考Polkadot关于XCMP的wiki。[Cross-Consensus Message Format (XCM) · Polkadot Wiki](https://wiki.polkadot.network/docs/learn-crosschain)

### 链裁剪(purging)

你唯一的collator是parachain区块链数据的唯一归宿，因为你整个parachain网络上只有一个节点（所以生产环境下是多个）。中继链只存储parachain的头信息。如果parachain数据丢失，你将无法恢复该链。不过在测试中，你可能需要从头开始。

为了从中继链中清除你的parachain链数据，你需要取消注册并重新注册parachain collator。在测试中可能更容易，而只是清除中继链和parachain，然后从创世块重新开始。你可以通过运行以下命令清除所有链的数据:

```bash
# Purge the collator(s)
./target/release/parachain-template-node purge-chain \
  --base-path <your collator DB path set above>

# Purge the validator(s)
polkadot purge-chain \
  --base-path <your relay chain DB path set above>
```












## 添加一个pallet到runtime中

正如你在构建本地区块链中所看到的，Substrate node template提供了一个工作的runtime，包括一些默认的FRAME开发模块--pallets，让你开始构建一个自定义的区块链。  
  
本教程介绍了为node template的runtime添加一个新pallet的基本步骤。任何时候你想在runtime添加一个新的FRAME pallet，步骤都是类似的。然而，每个pallet需要特定的配置设置--例如，执行pallet实现的函数所需的特定参数和类型。在本教程中，你将把Nicks pallet添加到node template的runtime中，因此你将看到如何配置Nicks pallet [pallet_nicks - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_nicks/index.html) 的特定设置。Nicks pallet允许区块链用户支付押金(deposit)，为他们控制的账户保留一个昵称。它实现了以下功能。  
  
- `set_name`函数用于收取押金，并在账户名称尚未被占用的情况下设置该名称。  
- `clear_name`函数用于删除与账户相关的名称并返回押金。  
- `kill_name`函数用于强行删除账户名称而不返回存款。  

请注意，本教程是通向更高级教程的垫脚石，这些教程说明了如何用更复杂的配置设置添加pallet，如何创建自定义pallet，以及如何发布pallet。  

### 添加nicks pallet到依赖中

在你使用一个新的pallet之前，你必须在编译器用来构建runtime二进制文件的配置文件中(cargo.toml)添加一些关于它的信息。

对于Rust程序，你使用Cargo.toml文件来定义配置设置和依赖关系，以决定在生成的二进制文件中被编译的内容。因为Substrate runtime既可以编译成包括标准Rust库函数的Native平台二进制文件，也可以编译成不包括标准Rust库的WebAssembly（Wasm）[WebAssembly](https://webassembly.org/) 二进制文件，所以Cargo.toml文件控制着两个重要的信息。

- 要作为runtime的依赖项导入的pallet，包括要导入的pallet的位置和版本。
- 在编译本地Rust二进制文件时应启用的每个pallet的特性。通过启用每个pallet的标准（std）特性集(feature set)，你可以在编译runtime时包含函数、类型和primitives，否则在构建WebAssembly二进制时就会丢失。

关于在Cargo.toml文件中添加依赖项的信息，请参见Cargo文档中的依赖项 [Dependencies - The Cargo Book (rust-lang.org)](https://doc.rust-lang.org/cargo/guide/dependencies.html)。关于启用和管理依赖包的特性的信息，请参见Cargo文档中的特性(features)。[Features - The Cargo Book (rust-lang.org)](https://doc.rust-lang.org/cargo/reference/features.html)

要将Nicks pallet的依赖项添加到runtime中。

- 打开一个终端shell，改变到node template的根目录。
- 在文本编辑器中打开 `runtime/Cargo.toml` 配置文件。
- 找到`[dependencies]`部分，注意其他pallet的导入方式。
- 复制一个现有的pallet依赖描述，用pallet-nicks替换pallet名称，使该pallet对node template的runtime可用。

例如，添加类似以下的一行。

```toml
pallet-nicks = { version = "4.0.0-dev", default-features = false, git = "https://github.com/paritytech/substrate.git", branch = "polkadot-v0.9.28" }
```

这一行将 `pallet-nicks` crate 作为依赖关系导入，并指定了以下内容。

- 版本，以确定你要导入的是哪个版本的crate。
- 在用标准Rust库编译runtime时，包含pallet特性的默认行为。
- 用于检索 `pallet-nicks` crate 的存储库位置。
- 检索crate时使用的分支。

这些细节对于任何给定版本的node template中的每个pallet都应该是相同的。

- 将`pallet-nicks/std` feature添加到编译runtime要启用的特性列表中。

```toml
[features]
default = ["std"]
std = [
  ...
  "pallet-aura/std",
  "pallet-balances/std",
  "pallet-nicks/std",
  ...
]
```

本节指定该runtime的默认特性集为std特性集进行编译。当runtime使用`std`特性集编译时，所有被列为依赖项的pallet中的`std`特性都被启用。关于runtime如何被编译为带有标准Rust库的平台native二进制文件和使用`no_std`属性编译为WebAssembly二进制文件的更多详细信息，请参见构建过程。[Build process | Substrate_ Docs](https://docs.substrate.io/build/build-process/)

如果你忘记更新`Cargo.toml`文件中的`features`部分，你可能会在编译runtime二进制文件时看到找不到函数的错误。

- 通过运行以下命令检查新的依赖关系是否正确解决。

```bash
cargo check -p node-template-runtime
```



### Review Balances的配置

每个pallet都有一个名为`Config`的Rust trait。`Config trait`用于识别pallet执行其功能(函数)所需的参数和类型。

添加一个pallet所需的大部分pallet代码都是通过Config trait实现的。你可以通过参考Rust文档或pallet的源代码来查看任何pallet需要实现的功能。例如，要想知道nicks pallet需要实现什么，可以参考`pallet_nicks::Config`的Rust文档 [Config in pallet_nicks::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_nicks/pallet/trait.Config.html) 或者Nicks pallet源代码中的trait定义。[substrate/lib.rs at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/blob/master/frame/nicks/src/lib.rs)

在本教程中，你可以看到nicks pallet中的Config trait声明了以下类型。

```rust
pub trait Config: Config {
    type Event: From<Event<Self>> + IsType<<Self as Config>::Event>;
    type Currency: ReservableCurrency<Self::AccountId>;
    type ReservationFee: Get<<<Self as Config>::Currency as Currency<<Self as Config>::AccountId>>::Balance>;
    type Slashed: OnUnbalanced<<<Self as Config>::Currency as Currency<<Self as Config>::AccountId>>::NegativeImbalance>;
    type ForceOrigin: EnsureOrigin<Self::Origin>;
    type MinLength: Get<u32>;
    type MaxLength: Get<u32>;
}
```

在确定你的pallet需要的类型后，你需要在runtime中添加代码来实现Config trait。为了了解如何实现pallet的Config trait，让我们用Balances pallet作为一个例子。

要查看Balances pallet的Config trait。

- 用文本编辑器打开`runtime/src/lib.rs`文件。
- 找到Balances pallet，注意它由以下实现（impl）代码块组成。

```rust
impl pallet_balances::Config for Runtime {
  type MaxLocks = ConstU32<50>;
  type MaxReserves = ();
  type ReserveIdentifier = [u8; 8];
  /// The type for recording an account's balance.
  type Balance = Balance;
  /// The ubiquitous event type.
  type Event = Event;
  /// The empty value, (), is used to specify a no-op callback function.
  type DustRemoval = ();
  /// Set the minimum balanced required for an account to exist on-chain
  type ExistentialDeposit = ConstU128<500>;
  /// The FRAME runtime system is used to track the accounts that hold balances.
  type AccountStore = System;
  /// Weight information is supplied to the Balances pallet by the node template runtime.
  type WeightInfo = pallet_balances::weights::SubstrateWeight<Runtime>;
}
```

正如你在这个例子中看到的，` impl pallet_balances::Config`块允许你配置由`Balances pallet Config trait`指定的类型和参数。例如，这个impl块将Balances pallet配置为使用`u128`类型来跟踪余额。

### 实现Nicks的配置

现在你已经看到了如何为Balances pallet实现`Config trait`的例子，你已经准备好为Nicks pallet实现`Config trait`了。

要在你的runtime中实现Nicks pallet需要:

- 用文本编辑器打开`runtime/src/lib.rs`文件。
- 找到Balances代码块的最后一行。
- 为Nicks pallet添加以下代码块。

```rust
impl pallet_nicks::Config for Runtime {
// The Balances pallet implements the ReservableCurrency trait.
// `Balances` is defined in `construct_runtime!` macro.
type Currency = Balances;

// Set ReservationFee to a value.
type ReservationFee = ConstU128<100>;

// No action is taken when deposits are forfeited.
type Slashed = ();

// Configure the FRAME System Root origin as the Nick pallet admin.
// https://paritytech.github.io/substrate/master/frame_system/enum.RawOrigin.html#variant.Root
type ForceOrigin = frame_system::EnsureRoot<AccountId>;

// Set MinLength of nick name to a desired value.
type MinLength = ConstU32<8>;

// Set MaxLength of nick name to a desired value.
type MaxLength = ConstU32<32>;

// The ubiquitous event type.
type Event = Event;
}
```

- 把Nicks pallet添加进`construct_runtime!` macro

```rust
construct_runtime!(
pub enum Runtime where
   Block = Block,
   NodeBlock = opaque::Block,
   UncheckedExtrinsic = UncheckedExtrinsic
 {
   /* --snip-- */
   Balances: pallet_balances,

   /*** Add This Line ***/
   Nicks: pallet_nicks,
 }
);
```

- `cargo build --release`


### 启动区块链节点

```rust
./target/release/node-template --dev
```

看到节点不停地出块确认就可以了

### 启动前端模板

进入substrate-front-end-template

```bash
yarn start
```

一般会自动打开http://127.0.0.1:8000端口，连接上ws协议的端口，默认是9944

### 使用Nicks pallet设置一个昵称

在你启动前端模板后，你可以用它来与你刚刚添加到runtime的Nicks pallet进行交互。

要为一个账户设置昵称。

- 检查账户选择列表，确认当前选择的是Alice账户。
- 在Pallet Interactor组件中，确认`Extrinsic`被选中。
- 从可调用的托盘列表中选择`nicks`。
- 选择`setName` [Call in pallet_nicks::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_nicks/pallet/enum.Call.html#variant.set_name) 作为从nicks pallet中调用的函数。
- 输入一个长于MinNickLength（8个字符）但不长于MaxNickLength（32个字符）的名称。
- 点击`Signed` 按钮执行这个setName的函数调用
- 最后就可以看到call change从Ready 到InBlock再到Finalized了，然后看到事件被Nicks pallet发出

![[9a7d0651fce9e89b900bb583143e383.png]]




### 使用Nicks pallet查询一个账户的信息

输入Alice账户的地址查询

![[5b27cecd43d147b6e869dfe9ae01fbb.png]]

返回的第一个参数`0x4d6174687848204368656e`是Alice账户昵称的Hex编码，`100`是在Alice账户中预留的金额以保证Alice昵称的使用。如果是输入Bob的账户地址查询，会返回None。然后调用clear_name，就可以把昵称删除了。再查就没有了。

### 探索其他函数

本教程说明了如何向runtime添加一个简单的pallet，并演示了如何使用前端模板与新的pallet进行交互。在本例中，你向runtime添加了nicks pallet，并使用前端模板调用set_name和nameOf函数。nicks pallet还提供了两个额外的函数--clear_name函数和kill_name函数--使账户所有者能够删除保留的名称，或者使root级用户能够强行删除账户名称。

## 配置contracts pallet

如果你已经学习完了添加Nicks pallet到runtime中，那么就可以学习本章了。这里也是添加一个pallet到runtime中，只是这个pallet是contracts pallet，有更复杂的配置，你添加这个pallet已给你的区块链提供智能合约开发的功能。

### 添加pallet依赖

- 在`runtime/Cargo.toml`的`[dependencies]` 下面添加`pallet-contracts`：

```toml
pallet-contracts = { version = "4.0.0-dev", default-features = false, git = "https://github.com/paritytech/substrate.git", branch = "polkadot-v0.9.28" }
```

- 导入 `pallet-contracts-primitives` crate，通过将其添加到依赖项列表中，使其在node template runtime可用。

在大多数情况下，你为任何给定版本的node template中的每个pallet指定相同的信息。然而，如果编译器表明发现了与你所指定的版本不同的版本，你可能需要修改依赖关系以匹配编译器所识别的版本。例如，如果编译器发现`pallet-contracts-primitives` crate的版本为6.0.0:

```toml
pallet-contracts-primitives = { version = "6.0.0", default-features = false, git = "https://github.com/paritytech/substrate.git", branch = "polkadot-v0.9.28" }
```

- 然后把pallet加入std feature列表:

```toml
[features]
default = ["std"]
std = [
  "codec/std",
  "scale-info/std",
  "frame-executive/std",
  "frame-support/std",
  "frame-system-rpc-runtime-api/std",
  "frame-system/std",
  "pallet-aura/std",
  "pallet-balances/std",
  "pallet-contracts/std",
  "pallet-contracts-primitives/std",
]
```


### 实现contracts pallet的Config trait

现在你已经成功导入了智能合约的pallet，你已经准备好在runtime中实现这些参数和类型。如果你探索过其他教程，你可能已经知道每个pallet都有一个`Config trait`--称为Config，runtime必须实现它。

如果你查看了`pallet_contracts::Config`的Rust API文档 [Config in pallet_contracts::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_contracts/pallet/trait.Config.html) ，你会注意到--与Nicks pallet不同--这个pallet有很多关联类型(associal type)，所以本教程的代码会比以前的教程更复杂。

要在runtime中为Contracts pallet实现`Config trait`。

- 用文本编辑器打开`runtime/src/lib.rs`文件。
- 找到`pub use frame_support`块，将Nothing添加到traits列表中。

```rust
pub use frame_support::{
construct_runtime, parameter_types,
traits::{
  ConstU128, ConstU32, ConstU64, ConstU8, KeyOwnerProofSystem, Randomness, StorageInfo,Nothing // Nothing keyword added on this Line
},
weights::{
 constants::{BlockExecutionWeight, ExtrinsicBaseWeight, RocksDbWeight, WEIGHT_PER_SECOND},
 IdentityFee, Weight,
},
StorageValue,
};
```

- 增加一行，从合同pallet中导入默认的合同权重。例如。

```rust
pub use frame_system::Call as SystemCall;
pub use pallet_balances::Call as BalancesCall;
pub use pallet_timestamp::Call as TimestampCall;
use pallet_transaction_payment::CurrencyAdapter;
use pallet_contracts::DefaultContractAccessWeight; // Add this line
```

- 将Contracts pallet要求的常量添加到runtime中。

```rust
/* After this block */
// Time is measured by number of blocks.
pub const MINUTES: BlockNumber = 60_000 / (MILLISECS_PER_BLOCK as BlockNumber);
pub const HOURS: BlockNumber = MINUTES * 60;
pub const DAYS: BlockNumber = HOURS * 24;

/* Add this block */
// Contracts price units.
pub const MILLICENTS: Balance = 1_000_000_000;
pub const CENTS: Balance = 1_000 * MILLICENTS;
pub const DOLLARS: Balance = 100 * CENTS;

const fn deposit(items: u32, bytes: u32) -> Balance {
items as Balance * 15 * CENTS + (bytes as Balance) * 6 * CENTS
}
const AVERAGE_ON_INITIALIZE_RATIO: Perbill = Perbill::from_percent(10);
/*** End Added Block ***/
```

- 添加参数类型，并在runtime为`pallet_contracts`实现`Config trait`。

```rust
/*** Add a block similar to the following ***/
parameter_types! {
  pub const DepositPerItem: Balance = deposit(1, 0);
  pub const DepositPerByte: Balance = deposit(0, 1);
  pub const DeletionQueueDepth: u32 = 128;
  pub DeletionWeightLimit: Weight = AVERAGE_ON_INITIALIZE_RATIO * BlockWeights::get().max_block;
  pub Schedule: pallet_contracts::Schedule<Runtime> = Default::default();
}

impl pallet_contracts::Config for Runtime {
  type Time = Timestamp;
  type Randomness = RandomnessCollectiveFlip;
  type Currency = Balances;
  type Event = Event;
  type Call = Call;
  type CallFilter = frame_support::traits::Nothing;
  type WeightPrice = pallet_transaction_payment::Pallet<Self>;
  type WeightInfo = pallet_contracts::weights::SubstrateWeight<Self>;
  type ChainExtension = ();
  type Schedule = Schedule;
  type CallStack = [pallet_contracts::Frame<Self>; 31];
  type DeletionQueueDepth = DeletionQueueDepth;
  type DeletionWeightLimit = DeletionWeightLimit;
  type DepositPerByte = DepositPerByte;
  type DepositPerItem = DepositPerItem;
  type AddressGenerator = pallet_contracts::DefaultAddressGenerator;
  type ContractAccessWeight = DefaultContractAccessWeight<BlockWeights>;
  type MaxCodeLen = ConstU32<{ 256 * 1024 }>;
  type RelaxedMaxCodeLen = ConstU32<{ 512 * 1024 }>;
  type MaxStorageKeyLen = ConstU32<{ 512 * 1024 }>;
}
/*** End added block ***/
```

关于合约pallet的配置以及如何使用类型和参数的更多信息，请参阅合约pallet的源代码。 [substrate/lib.rs at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/blob/master/frame/contracts/src/lib.rs)

- 把`pallet_contracts` 添加到`construct_runtime!` 宏中

```rust
// Create the runtime by composing the FRAME pallets that were previously configured
construct_runtime!(
  pub enum Runtime where
    Block = Block,
    NodeBlock = opaque::Block,
    UncheckedExtrinsic = UncheckedExtrinsic
  {
     System: frame_system,
     RandomnessCollectiveFlip: pallet_randomness_collective_flip,
     Timestamp: pallet_timestamp,
     Aura: pallet_aura,
     Grandpa: pallet_grandpa,
     Balances: pallet_balances,
     TransactionPayment: pallet_transaction_payment,
     Sudo: pallet_sudo,
     Contracts: pallet_contracts,
  }
);
```

### 暴露contracts的API

一些pallet，包括合约pallet，暴露了自定义的runtime API和RPC端点。你不需要启用合约pallet上的RPC调用来在链上使用它。然而，为Contracts pallet暴露API和端点是非常有用的，因为这样做可以让你执行以下任务。

- 从链外读取合同状态。
- 在不进行交易的情况下，对节点存储进行调用。

要暴露合约的RPC API。

- 在文本编辑器中打开`runtime/Cargo.toml`文件。
- 在`[dependencies]`部分添加`pallet-contracts-rpc-runtime-api` pallet的描述，使用与其他pallet相同的版本和分支信息。

```toml
pallet-contracts-rpc-runtime-api = { version = "4.0.0-dev", default-features = false, git = "https://github.com/paritytech/substrate.git", branch = "polkadot-v0.9.28" }
```

- 把`pallet-contracts-rpc-runtime-api`添加进std features列表

```toml
[features]
default = ["std"]
std = [
  "codec/std",
  ...
  "pallet-contracts/std",
  "pallet-contracts-primitives/std",
  "pallet-contracts-rpc-runtime-api/std",
  ...
]
```

- 在文本编辑器中打开`runtime/src/lib.rs`文件，并通过添加以下常量来启用合约的调试。

```rust
const CONTRACTS_DEBUG_OUTPUT: bool = true;
```

- 打开`runtime/src/lib.rs`文件，在lib.rs文件末尾附近的` impl_runtime_apis!` 宏中实现合约的runtime API。

例如，在 `impl_runtime_apis! { }`的部分。

```rust
/*** Add this block ***/
impl pallet_contracts_rpc_runtime_api::ContractsApi<Block, AccountId, Balance, BlockNumber, Hash>
 for Runtime
 {
  fn call(
     origin: AccountId,
     dest: AccountId,
     value: Balance,
     gas_limit: u64,
     storage_deposit_limit: Option<Balance>,
     input_data: Vec<u8>,
  ) -> pallet_contracts_primitives::ContractExecResult<Balance> {
     Contracts::bare_call(origin, dest, value, gas_limit, storage_deposit_limit, input_data, CONTRACTS_DEBUG_OUTPUT)
  }
  
  fn instantiate(
     origin: AccountId,
     value: Balance,
     gas_limit: u64,
     storage_deposit_limit: Option<Balance>,
     code: pallet_contracts_primitives::Code<Hash>,
     data: Vec<u8>,
     salt: Vec<u8>,
  ) -> pallet_contracts_primitives::ContractInstantiateResult<AccountId, Balance> {
     Contracts::bare_instantiate(origin, value, gas_limit, storage_deposit_limit, code, data, salt, CONTRACTS_DEBUG_OUTPUT)
     }
     
  fn upload_code(
     origin: AccountId,
     code: Vec<u8>,
     storage_deposit_limit: Option<Balance>,
  ) -> pallet_contracts_primitives::CodeUploadResult<Hash, Balance> {
     Contracts::bare_upload_code(origin, code, storage_deposit_limit)
  }
  
  fn get_storage(
     address: AccountId,
     key: Vec<u8>,
     ) -> pallet_contracts_primitives::GetStorageResult {
     Contracts::get_storage(address, key)
     }
  }
```

- 这时候可以`cargo build --release`检查下有没有编译报错

### 更新外层节点

至此，你已经完成了向runtime添加Contracts pallet的工作。现在，你需要考虑外层节点是否需要任何相应的更新。Substrate提供了一个RPC服务器来与节点交互。然而，默认的RPC服务器并不提供对Contracts pallet的访问。为了与Contracts pallet交互，你必须扩展现有的RPC服务器以包括Contracts pallet和Contracts RPC API。为了让Contracts pallet利用RPC端点API，你需要在外层节点配置中添加自定义RPC端点。

要添加RPC API扩展到外层节点。

- 在文本编辑器中打开`node/Cargo.toml`文件，将Contracts和Contracts RPC crates添加到`[dependencies]`部分。

```toml
pallet-contracts = { version = "4.0.0-dev", git = "https://github.com/paritytech/substrate.git", branch = "polkadot-v0.9.28" }
pallet-contracts-rpc = { version = "4.0.0-dev", git = "https://github.com/paritytech/substrate.git", branch = "polkadot-v0.9.28" }

```

因为你已经暴露了runtime的API，并且现在正在为外层节点编写代码，你不需要使用`no_std`配置，所以你不需要维护一个专门的`std`功能列表。

- 用文本编辑器打开`node/src/rpc.rs`文件，找到以下一行。

```rust
use node_template_runtime::{opaque::Block, AccountId, Balance, Index};
```

- 在现有的`use node_template_runtime`声明中增加`BlockNumber`和`Hash`。

```rust
use node_template_runtime::{opaque::Block, AccountId, Balance, Index, BlockNumber, Hash}; 
```

- 在文件中添加`use pallet_contracts_rpc`。

```rust
// 这段在create_full fn中
use pallet_transaction_payment_rpc::{TransactionPayment, TransactionPaymentApiServer};
use substrate_frame_rpc_system::{System, SystemApiServer};
use pallet_contracts_rpc::{Contracts, ContractsApiServer}; // Add this line
```

- 把Contracs RPC pallet添加进`create_full`函数中，为RPC作扩展，主要是其中的Where子句:

```rust
/// Instantiate all full RPC extensions.
pub fn create_full<C, P>(
  deps: FullDeps<C, P>,
) -> Result<RpcModule<()>, Box<dyn std::error::Error + Send + Sync>>
where
  C: ProvideRuntimeApi<Block>,
  C: HeaderBackend<Block> + HeaderMetadata<Block, Error = BlockChainError> + 'static,
  C: Send + Sync + 'static,
  C::Api: substrate_frame_rpc_system::AccountNonceApi<Block, AccountId, Index>,
  C::Api: pallet_transaction_payment_rpc::TransactionPaymentRuntimeApi<Block, Balance>,
  C::Api: pallet_contracts_rpc::ContractsRuntimeApi<Block, AccountId, Balance, BlockNumber, Hash>, // Add this line
  C::Api: BlockBuilder<Block>,
  P: TransactionPool + 'static,
```

- 把扩展加入到`create_full`中

```rust
module.merge(System::new(client.clone(), pool.clone(), deny_unsafe).into_rpc())?;
module.merge(TransactionPayment::new(client.clone()).into_rpc())?;
module.merge(Contracts::new(client.clone()).into_rpc())?; // Add this line
```

- `cargo build --release` 编译

注意，以上部分其实涉及到了自定义RPC的部分，RPC的自定义部分会涉及到外层节点，详情请参考 RPC的文档 [Remote procedure calls | Substrate_ Docs](https://docs.substrate.io/build/custom-rpc/)

### 启动本地的substrate节点

```rust
./target/release/node-template --dev
```

看到出块并不断地确认就可以了。

然后启动frontend-template：

```bash
yarn start
```

然后就可以在页面上看到合约暴露出来的API uploadCode了:

![[Pasted image 20220904165733.png]]

## 在自定义的pallet中使用宏

本教程的目的就是带着你开发一个自定义的pallet，一个简单的proof of existence 应用。PoE是一种通过在区块链上存储有关对象的信息来验证数字对象的真实性和所有权的方法。由于区块链将时间戳和账户与对象相关联，区块链记录可以用来 "证明 "某个特定对象在特定日期和时间存在。它还可以验证在那个日期和时间，记录的主人是谁。

## 针对一个call指定其origin

这篇教程是在nicks pallet基础上做的。

在之前的教程 “添加pallet到runtime中”，你将nicks pallet的函数添加到substrate node template的runtime中。

Nicks pallet允许区块链用户支付押金(deposit)，为他们控制的账户保留一个昵称。它实现了以下函数。

- `set_name`函数，使账户所有者(owner)能够设置他或她自己的账户的名称，如果该名称尚未被保留(占用)。
- `clear_name`函数，使账户所有者能够删除与账户相关的名称，并返回存款(deposit)。
- `kill_name`函数，为另一方强行删除账户名称，而不退还押金(deposit)。
- `force_name`函数为另一方设置一个账户名称，而不要求押金。

本教程说明了如何使用不同的原始账户(originating accounts)调用这些函数，以及为什么使用不同的原始账户调用这些函数很重要。

### 识别管理账户

正如你在《向runtime添加pallet》教程中所看到的，nicks pallet的`Config trait`声明了几种类型。在本教程中，重点是`ForceOrigin`类型。`ForceOrigin`类型是用来指定可以执行某些选项的账户。对于这个pallet，`ForceOrigin`类型指定了可以为另一个账户设置或删除名称的账户。通常情况下，只有具有管理权限的账户--如root超级用户账户--才能代表另一个账户行事。在Nicks pallet的情况下，只有账户的所有者或Root账户可以设置或删除一个保留的昵称。当你确定FRAME System的`Root origin` [RawOrigin in frame_system - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_system/enum.RawOrigin.html#variant.Root) 为nicks pallet管理员时，你在实现（implication）块中配置了这个Root账户。比如说

```rust
type ForceOrigin = frame_system::EnsureRoot<AccountId>;
```

在node template的chain spec中，`Sudo pallet` [pallet_sudo - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_sudo/index.html) 被配置为使用Alice账户作为FRAME system的`Root origin`。由于这种配置，默认情况下，只有Alice账户可以调用需要`ForceOrigin`类型的函数。

如果你试图用Alice账户以外的账户调用`kill_name`或`force_name`，调用将不能被执行。


### 为一个账户设置一个名称

为了演示对origin的调用影响操作，让我们设置并尝试强行删除另一个账户的账户名。对于这个演示，要确保你有。  
  
- 以开发模式运行的节点模板： `./target/release/node-template --dev`  
- 前端模板正在运行并连接到本地节点：`yarn start`  
- 你的浏览器连接到本地网络服务器：`http://localhost:8000/``  
- 将前端模板中的活动账户从Alice改为Bob。  
- 在Pallet Interactor中选择Extrinsic。  
    - 选择nicks pallet。  
    - 选择setName函数。  
    - 给一个账户输入一个昵称。  
    - 点击Signed，提交这个由Bob签名的交易。  

因为Bob是这个账户的所有者，所以交易成功了。作为账户的所有者，Bob也可以在签名的交易中执行`clearName`函数来删除账户的昵称。  
  
- 在选择了Extrinsic的情况下。  
    - 选择nicks pallet。  
    - 选择clearName函数。  
    - 点击Signed，提交这个由Bob签名的交易。  
  
因为Bob是这个账户的所有者，所以交易是成功的。如果Bob要为另一个账户设置或删除昵称，他必须使用为该pallet配置的ForceOrigin调用forceName或killName函数。  
  
- 在选择了Extrinsic的情况下。  
    - 选择nicks pallet。  
    - 选择ForceName函数。  
     - 复制并粘贴Charlie的账户地址作为目标。  
     - 键入一个账户昵称。  
     - 点击 "签名"，提交这个由Bob签名的交易。  
  
因为你用Bob的账户签署了这个交易，所以这个函数是用`Signed origin`而不是`Root origin`来dispatch的。在这种情况下，函数的调用本身是成功的。但是，名字的保留无法完成，因此发出了一个`BadOrigin`错误。

![[Pasted image 20220904212016.png]]

正如你在事件中所看到的，该交易导致从Bob的账户中提取了一笔钱作为提交交易的费用，但由于`Root origin`没有提交该交易，所以没有发生状态变化。状态改变的失败也说明了数据库读写的验证是--先校验--最后写的原则，以确保只有成功的操作被提交到磁盘上。

如果你好奇，你可以看nicks pallet的源码，里面的force_name或kill_name的验证是这样的:

```rust
T::ForceOrigin::ensure_origin(origin)?;
```

必须确保是root origin，不然会报错。


### 使用root origin来分发一个调用

Sudo pallet使你可以使用`Root origin`来dispatch调用。在Nick pallet中，`forceName`和`killName`函数必须使用`ForceOrigin`配置所指定的`Root origin`来调用。在前端模板中，你可以访问Sudo pallet，通过点击SUDO来分配一个使用Root 的call。  
  
对于这个演示，请确保你有。  
  
- 以开发模式运行的节点模板：`./target/release/node-template --dev`  
- 前端模板运行并连接到本地节点：`yarn start`  
- 你的浏览器连接到本地网络服务器： `http://localhost:8000/``  
- 将活动账户改为Alice。  
  
    正如在识别管理账户中提到的，当以开发模式运行链时，Alice是与`Root origin`相关的账户。  
  
- 在Pallet Interactor中选择Extrinsic。  
     - 选择nicks pallet。  
     - 选择forceName函数。  
     - 复制并粘贴Charlie的账户地址作为目标。  
     - 输入账户的昵称。  
     - 点击SUDO，使用root origin提交此交易。  

![[Pasted image 20220904215739.png]]

- 在选择了Extrinsic的情况下。
    - 选择nicks pallet。
    - 选择killName函数。
    - 复制并粘贴Bob的账户地址作为目标。
    - 点击SUDO，使用Root origin提交此交易。

在这种情况下，Sudo pallet会发出一个`Sudid事件` [Event in pallet_sudo::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_sudo/pallet/enum.Event.html) ，通知网络参与者，`Root origin`发出了一个调用，但发生了错误。

![[Pasted image 20220904220302.png]]

这个dispatch错误包括两块元数据。

- 一个index，表示该错误来自哪个pallet。
- 一个error number，表示从该pallet的 error enum中发出的错误。

index与pallet在`construct_runtime！` 宏中的位置相对应，`construct_runtime！`宏中的第一个托盘的索引号为0（0）。

在这个例子中，index是6（第七个pallet），error是2(0x02000000)（第三个错误）。

```rust
construct_runtime!(
pub enum Runtime where
  Block = Block,
  NodeBlock = opaque::Block,
  UncheckedExtrinsic = UncheckedExtrinsic
{
  System: frame_system,                                        // index 0
  RandomnessCollectiveFlip: pallet_randomness_collective_flip, // index 1
  Timestamp: pallet_timestamp,                                 // index 2
  Aura: pallet_aura,                                           // index 3
  Grandpa: pallet_grandpa,                                     // index 4
  Balances: pallet_balances,                                   // index 5
  Nicks: pallet_nicks,                                         // index 6
}
```


```rust
/// Error for the nicks pallet.
#[pallet::error]
pub enum Error<T> {
	/// A name is too short.
	TooShort,
	/// A name is too long.
	TooLong,
	/// An account isn't named.
	Unnamed,   // 0x02000000
}
```

以上的操作就是，对一个没有昵称的账户做kill_name操作，那么就会报这个错误了。[Error in pallet_nicks::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_nicks/pallet/enum.Error.html)

不管index的值是多少，错误值2对应于Nicks palley中的Unnamed错误。如果Bob没有保留昵称或者之前清除了名字的保留，这就是你期望的错误。

你可以确认Alice可以使用SUDO调用`killName`函数来删除任何当前有名字保留的账户的昵称。

- 在选择了Extrinsic的情况下。
    - 选择nicks pallet。
    - 选择killName函数。
    - 复制并粘贴Charlie的账户地址作为目标。
    - 点击SUDO，使用`Root origin`提交此交易。

![[Pasted image 20220904221138.png]]

## 发布自定义的pallet

TODO