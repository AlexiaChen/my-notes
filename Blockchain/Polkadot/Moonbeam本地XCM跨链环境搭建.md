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