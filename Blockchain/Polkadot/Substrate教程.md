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








