# Substrate学习笔记

一些资料:

- [A guide to building Blockchain in Rust & Substrate - BLOCKGENI](https://blockgeni.com/a-guide-to-building-blockchain-in-rust-substrate/#:~:text=Substrate%20is%20Rust%20based%20and%20compiles%20to%20a,and%20various%20consensus%20mechanisms%20we%20can%20choose%20from.)
- [First impressions of Rust smart contracts with Substrate and Ink, part 1 (brson.github.io)](https://brson.github.io/2020/12/03/substrate-and-ink-part-1)
- [Building A Blockchain in Rust & Substrate: [A Step-by-Step Guide for Developers] | HackerNoon](https://hackernoon.com/building-a-blockchain-in-rust-and-substrate-a-step-by-step-guide-for-developers-kc223ybp)


### 快速开始

所有的Substrate教程和指南都要求你在你的开发环境中构建和运行一个Substrate节点。为了帮助你快速建立一个工作环境，[Substrate开发者中心]([Substrate Developer Hub (github.com)](https://github.com/substrate-developer-hub/))维护了几个模板供你使用。

[Substrate-node-template](https://github.com/substrate-developer-hub/substrate-node-template)的开发者中心快照包括了你开始使用一组核心功能所需的一切。

本快速入门假设你是第一次建立一个开发环境，并想尝试在本地计算机上运行一个单一的区块链节点。

为了保持简单，你将使用网络浏览器连接到本地节点，并查询预定义样本账户的余额。

#### 构建node template


```bash
sudo apt install build-essential
sudo apt install --assume-yes git clang curl libssl-dev llvm libudev-dev make protobuf-compiler
rustup default stable
rustup update
rustup update nightly
rustup target add wasm32-unknown-unknown --toolchain nightly
rustup show
rustup +nightly show
```

```bash
git clone --branch polkadot-v0.9.25 https://github.com/substrate-developer-hub/substrate-node-template
cd substrate-node-template
cargo build  --release
```

#### 启动节点

```bash
./target/release/node-template --help
```

以上帮助给你了一个启动节点，与account工作，修改节点操作的一个帮助提示，

```bash
./target/release/node-template key inspect //Alice
```

以上是查看预定义好的Alice开发账号的account信息，显示如下:

```txt
Secret Key URI `//Alice` is account:
Network ID:        substrate 
Secret seed:       0xe5be9a5092b81bca64be81d212e7f2f9eba183bb7a90954f7b76361f6edb5c0a
Public key (hex):  0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d
Account ID:        0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d
Public key (SS58): 5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
SS58 Address:      5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY
```

可以以开发模式启动节点:

```bash
./target/release/node-template --dev
```

在开发模式下，链不需要任何P2P对端的计算机来最终完成区块。当节点启动时，终端会显示关于所执行操作的输出。如果你看到正在提议和敲定区块的信息，你就有一个正在运行的节点。



