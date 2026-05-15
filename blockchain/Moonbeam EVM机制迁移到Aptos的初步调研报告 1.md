
### Moonbeam的EVM机制的基本原理

首先来看总体的一个设计，其实Parity已经有了一个substrate EVM兼容层的方案，这个方案就是frontier [paritytech/frontier: Ethereum compatibility layer for Substrate. (github.com)](https://github.com/paritytech/frontier)

目标是在substrate的区块链可以运行不修改的Dappa，不仅仅是合约代码，还有兼容以太坊的RPC，因为EVM使用的地址，非对称加密曲线与substrate的都不一样，所以frontier做了很多兼容层的设计，包括`pallet-evm-precompile-simple` `pallet-evm-precompile-blake2` `pallet-evm-precompile-ed25519` `pallet-evm-precompile-dispatch` 等pallet来适配substrate的Runtime机制。

Moonbeam目标就是在波卡上做一个EVM完全兼容的平行链，为了自己特定的业务需求，对frontier做了自己的定制 ，所以它们fork了一个frontier的仓库，自己修改 [PureStake/frontier: Ethereum compatibility layer for Substrate. (github.com)](https://github.com/PureStake/frontier)

首先我们来看Moonbeam的Runtime目录下面的`cargo.toml`做了怎样的引用。[PureStake/moonbeam: An Ethereum-compatible smart contract parachain on Polkadot (github.com)](https://github.com/PureStake/moonbeam/blob/master/runtime/moonbeam/Cargo.toml)

```toml

...

# Moonbeam precompiles
pallet-evm-precompile-author-mapping = { path = "../../precompiles/author-mapping", default-features = false }
pallet-evm-precompile-balances-erc20 = { path = "../../precompiles/balances-erc20", default-features = false }
pallet-evm-precompile-batch = { path = "../../precompiles/batch", default-features = false }
pallet-evm-precompile-call-permit = { path = "../../precompiles/call-permit", default-features = false }
pallet-evm-precompile-collective = { path = "../../precompiles/collective", default-features = false }
pallet-evm-precompile-crowdloan-rewards = { path = "../../precompiles/crowdloan-rewards", default-features = false }
pallet-evm-precompile-democracy = { path = "../../precompiles/pallet-democracy", default-features = false }
pallet-evm-precompile-parachain-staking = { path = "../../precompiles/parachain-staking", default-features = false }
pallet-evm-precompile-proxy = { path = "../../precompiles/proxy", default-features = false }
pallet-evm-precompile-randomness = { path = "../../precompiles/randomness", default-features = false }
pallet-evm-precompile-relay-encoder = { path = "../../precompiles/relay-encoder", default-features = false }
pallet-evm-precompile-xcm-transactor = { path = "../../precompiles/xcm-transactor", default-features = false }
pallet-evm-precompile-xcm-utils = { path = "../../precompiles/xcm-utils", default-features = false }
pallet-evm-precompile-xtokens = { path = "../../precompiles/xtokens", default-features = false }
pallet-evm-precompileset-assets-erc20 = { path = "../../precompiles/assets-erc20", default-features = false }

...

# Frontier
fp-evm = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
fp-rpc = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
fp-self-contained = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-base-fee = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-ethereum = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-evm = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-evm-precompile-blake2 = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-evm-precompile-bn128 = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-evm-precompile-dispatch = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-evm-precompile-modexp = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-evm-precompile-sha3fips = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }
pallet-evm-precompile-simple = { git = "https://github.com/purestake/frontier", branch = "moonbeam-polkadot-v0.9.26", default-features = false }

...

```

从以上的内容看，对于frontier，Moonbeam引用了都是自己修改的frontier的git地址，包括之前提到的各种预编译的pallet。其次，Moonbeam也引用了本地的EVM相关的其他预编译了的模块。
所以可以看到，为了让EVM生态运行在基于substrate的链上，Parity和Moonbeam做了大量的工作，不仅仅只是移植EVM虚拟机，因为EVM虚拟机上的生态跟substrate的很大不一样，包括底层的概念，所以做了很多EVM生态相关的迁移和适配，目的就是为了完全让EVM生态的Dapp无缝运行在substrate的Moonbeam链上。

对于这些相关的适配的pallet或者说是可复用的组件，是针对substrate的，所以被引用到substrate的runtime中，这些适配pallet中间件完全是按照substrate的runtime设计架构的。也无法在Aptos进行编译集成。

再看看Moonbeam runtime部分的代码:

```rust
impl pallet_evm::Config for Runtime {
	type FeeCalculator = FixedGasPrice;
	type GasWeightMapping = MoonbeamGasWeightMapping;
	type BlockHashMapping = pallet_ethereum::EthereumBlockHashMapping<Self>;
	type CallOrigin = EnsureAddressRoot<AccountId>;
	type WithdrawOrigin = EnsureAddressNever<AccountId>;
	type AddressMapping = moonbeam_runtime_common::IntoAddressMapping;
	type Currency = Balances;
	type Event = Event;
	type Runner = pallet_evm::runner::stack::Runner<Self>;
	type PrecompilesType = MoonbeamPrecompiles<Self>;
	type PrecompilesValue = PrecompilesValue;
	type ChainId = EthereumChainId;
	type OnChargeTransaction = OnChargeEVMTransaction<DealWithFees<Runtime>>;
	type BlockGasLimit = BlockGasLimit;
	type FindAuthor = FindAuthorAdapter<AuthorInherent>;
}

...
	// Ethereum compatibility.
		EthereumChainId: pallet_ethereum_chain_id::{Pallet, Storage, Config} = 50,
		EVM: pallet_evm::{Pallet, Config, Call, Storage, Event<T>} = 51,
		Ethereum: pallet_ethereum::{Pallet, Call, Storage, Event, Origin, Config} = 52,
		BaseFee: pallet_base_fee::{Pallet, Call, Storage, Config<T>, Event} = 53,

...
```

可以看到，moonbeam的runtime中引入了如此大量的EVM相关的适配组件。也就是说，如果从现有的Moonbeam的substrate的EVM机制移植到Aptos相当于是不可能的。如果要移植，其实相当于我们自己做一套Aptos的“frontier“出来，而这套“frontier”还不是模块化的，也就是我们对Aptos做大规模的链改。这样的工程量非常巨大。

### Moonbeam EVM机制的底层探究

好了，从上述的章节我们了解到，substrate要兼容EVM，必须引入frontier，下面我们来探索下frontier仓库中的几个重要的模块，其实README的文档中已经提到: `fp-consensus` `fp-evm` `fp-rpc` `fp-storage` `pallet-evm` `pallet-ethereum` 

重要的就是上面提到的6个模块。下面我们需要探索下这几个模块的底层实现，了解大概的实现逻辑。

##### fp-consensus

很奇怪，为什么引入EVM兼容层，会有共识的概念引入，看`lib.rs`实现文件，你可以看到大部分都是以太坊日志log，区块Hash和交易Hash。因为这些概念，在EVM里面是与ETH的概念耦合的。所以猜测需要做适配。这个模块主要提供查找日志log。

```rust
...

pub fn find_pre_log(digest: &Digest) -> Result<PreLog, FindLogError> {
	_find_log(digest, OpaqueDigestItemId::PreRuntime(&FRONTIER_ENGINE_ID))
}

pub fn find_post_log(digest: &Digest) -> Result<PostLog, FindLogError> {
	_find_log(digest, OpaqueDigestItemId::Consensus(&FRONTIER_ENGINE_ID))
}

...

pub fn find_log(digest: &Digest) -> Result<Log, FindLogError> {
	let mut found = None;

	for log in digest.logs() {
		let pre_log = log.try_to::<PreLog>(OpaqueDigestItemId::PreRuntime(&FRONTIER_ENGINE_ID));
		match (pre_log, found.is_some()) {
			(Some(_), true) => return Err(FindLogError::MultipleLogs),
			(Some(pre_log), false) => found = Some(Log::Pre(pre_log)),
			(None, _) => (),
		}

		let post_log = log.try_to::<PostLog>(OpaqueDigestItemId::Consensus(&FRONTIER_ENGINE_ID));
		match (post_log, found.is_some()) {
			(Some(_), true) => return Err(FindLogError::MultipleLogs),
			(Some(post_log), false) => found = Some(Log::Post(post_log)),
			(None, _) => (),
		}
	}

	found.ok_or(FindLogError::NotFound)
}

pub fn ensure_log(digest: &Digest) -> Result<(), FindLogError> {
	find_log(digest).map(|_log| ())
}
```

主要是查找，前序日志(prevLog)和后续日志(postLog)。熟悉EVM，或者写过ETH合约的人都知道，EVM是直接支持日志指令的，用于合约发射event。支持LOG0，LOG1,LOG2,LOG3,LOG4 这五个EVM操作码(opCode)。所以要兼容EVM的Dapps生态，肯定是要把EVM上的日志操作码，实现在substrate上，并存储起来了，要对这些日志opCode做出重新解释其定义，存储在特定的substrate的底层存储上。[Understanding event logs on the Ethereum blockchain | by Luit Hollander | MyCrypto | Medium](https://medium.com/mycrypto/understanding-event-logs-on-the-ethereum-blockchain-f4ae7ba50378)

也就是你移植到Aptos上的EVM不仅要运行，对比substrate这里的实现，还要可以查看合约发射出来的日志，日志肯定是存储在Aptos上的，而Aptos上的存储机制与ETH肯定不一样，又要对存储上做适配。

ETH的核心开发者回答了相关的问题，在ETH上，EVM的event log是存放在哪里的？ [solidity - Where do contract event logs get stored in the Ethereum architecture? - Ethereum Stack Exchange](https://ethereum.stackexchange.com/questions/1302/where-do-contract-event-logs-get-stored-in-the-ethereum-architecture)

> 回答是这样的：日志是交易receipts的一部分。它们由客户在执行交易时产生，并与区块链一起存储，以便检索它们。日志本身不是区块链的一部分，因为共识不需要它们（它们只是历史数据），但是它们被区块链验证，因为交易收据的哈希值存储在区块内。

根据回答，相当于要移植EVM event log部分，也要想办法把日志存储在Aptos上，并让Aptos交易这个日志的root hash，同步的时候，不然一些安全机制无法得到保证。

##### fp-evm

这里就是真正的EVM核心之一了，首先，这个模块依赖了Rust实现的一个EVM的package :

```toml
evm = { git = "https://github.com/rust-blockchain/evm", rev = "51b8c2ce3104265e1fd5bb0fe5cdfd2e0938239c", default-features = false, features = ["with-codec"] }
serde = { version = "1.0.144", features = ["derive"], optional = true }
```

从实现代码上看，这个模块实现了EVM相关的交易校验，gas费用计算，费用校验。相当于要对传给EVM的代码进行gas预估，对合约的输出进行执行结果的保存，记录合约日志:

```rust
...
#[derive(Clone, Eq, PartialEq, Encode, Decode)]
#[cfg_attr(feature = "std", derive(Debug, Serialize, Deserialize))]
pub struct ExecutionInfo<T> {
	pub exit_reason: ExitReason,
	pub value: T,
	pub used_gas: U256,
	pub logs: Vec<Log>,
}
...
/// Trait that outputs the current transaction gas price.
pub trait FeeCalculator {
	/// Return the minimal required gas price.
	fn min_gas_price() -> (U256, Weight);
}

impl FeeCalculator for () {
	fn min_gas_price() -> (U256, Weight) {
		(U256::zero(), 0u64)
	}
}
```

相当于如果要把EVM迁移到Aptos也是需要对EVM gas费用这些计算和检验进行中间层的适配包装，不仅仅是依赖一个Rust实现的EVM就可以了。因为一些部署ETH合约的接口的返回参数，也有类似的gas费用预估，要兼容整个EVM生态的工具链，必须这么做。

##### fp-rpc

先分析依赖:

```toml
ethereum = { version = "0.12.0", default-features = false, features = ["with-codec"] }

# Parity
codec = { package = "parity-scale-codec", version = "3.0.0", default-features = false }
ethereum-types = { version = "0.13.1", default-features = false }
scale-info = { version = "2.1.2", default-features = false, features = ["derive"] }
...
# Frontier
fp-evm = { version = "3.0.0-dev", path = "../../primitives/evm", default-features = false }
```

首先就依赖了之前小节提到的`fp-evm` 也就是RPC模块需要这些EVM相关的gas费用的计算预估结果。

然后就是依赖了`ethereum` 和`ethereum-types`这个两个crate [ethereum - crates.io: Rust Package Registry](https://crates.io/crates/ethereum)
这个crate主要提供ETH区块，交易，trie，RLP的类型和方法。也就是这个RPC模块要对这些数据做解码，编码的解析，才可以适配以太坊的EVM相关的RPC。以下就是用`sp-api`对ETH EVM相关的RPC（gas_price，Storage_At）:

https://github.com/paritytech/frontier/blob/a8d4dd55643d324dbbf49be7f4e4e10494f56d39/primitives/rpc/src/lib.rs#L40-L167

实现代码还写了，以太坊交易到substrate的Extrinsic转换trait接口。这些都是对以太坊EVM相关的RPCs做的适配。

##### fp-storage

这个crate的依赖并不多，只是`serde` `codec` 一些序列化和反序列化的crates。

看代码实现 : https://github.com/paritytech/frontier/blob/a8d4dd55643d324dbbf49be7f4e4e10494f56d39/primitives/storage/src/lib.rs#L20-L57

从实现上看，这里就定义了EVM相关的存储项的前缀，因为存储最终会落盘到levelDB或者RocksDB这样的KV存储上，所以一般来说，涉及到存储，都会定义一些业务相关的键的前缀字符串。

```rust
/// Current version of pallet Ethereum's storage schema is stored under this key.
pub const PALLET_ETHEREUM_SCHEMA: &[u8] = b":ethereum_schema";
/// Cached version of pallet Ethereum's storage schema is stored under this key in the AuxStore.
pub const PALLET_ETHEREUM_SCHEMA_CACHE: &[u8] = b":ethereum_schema_cache";

/// Pallet Evm storage items
pub const PALLET_EVM: &[u8] = b"EVM";
pub const EVM_ACCOUNT_CODES: &[u8] = b"AccountCodes";
pub const EVM_ACCOUNT_STORAGES: &[u8] = b"AccountStorages";

/// Pallet Ethereum storage items
pub const PALLET_ETHEREUM: &[u8] = b"Ethereum";
pub const ETHEREUM_CURRENT_BLOCK: &[u8] = b"CurrentBlock";
pub const ETHEREUM_CURRENT_RECEIPTS: &[u8] = b"CurrentReceipts";
pub const ETHEREUM_CURRENT_TRANSACTION_STATUS: &[u8] = b"CurrentTransactionStatuses";

/// Pallet BaseFee storage items
pub const PALLET_BASE_FEE: &[u8] = b"BaseFee";
pub const BASE_FEE_PER_GAS: &[u8] = b"BaseFeePerGas";
pub const BASE_FEE_ELASTICITY: &[u8] = b"Elasticity";
```

**看到这里，这些逻辑也需要在Aptos实现，定义存储项和方案，又有不少工作量**

##### pallet-evm

EVM pallet允许未经修改的EVM代码(opCode)在基于substrate的区块链中执行。关于EVM就是使用了之前提到的Rust实现的EVM。从依赖上看:

```toml
...

evm = { git = "https://github.com/rust-blockchain/evm", rev = "51b8c2ce3104265e1fd5bb0fe5cdfd2e0938239c", default-features = false, features = ["with-codec"] }

...

# Frontier
fp-evm = { version = "3.0.0-dev", path = "../../primitives/evm", default-features = false }
```

从以上可以看出，这个模块本身直接依赖了Rust版的EVM [rust-blockchain/evm: Pure Rust implementation of Ethereum Virtual Machine (github.com)](https://github.com/rust-blockchain/evm) ，这个模块会直接调用EVM。并且也依赖了`fp-evm`， `fp-evm`主要是声明了一些gas费用计算，校验的接口。估计会做一些gas费用计算的实现。

这个Rust版本的EVM经过大幅度修改，修改以后，才变成了模块化的EVM，模块化的方案设计在这里描述 [corepaper/evm: The Core Paper Project of EVM (github.com)](https://github.com/corepaper/evm)

这个pallet EVM执行的生命周期:

**有一组单独的账户由这个EVM pallet管理。基于substrate的账户可以调用这个EVM pallet，将余额从substrate基础货币存入或提取到EVM pallet管理和使用的不同余额中。一旦用户填充了他们的余额，他们可以使用这个pallet EVM创建和调用智能合约。相当于这个pallet EVM做了substrate账户到EVM账户使用创建合约的适配，做了一个中间适配层。**

substrate账户和EVM外部账户之间有一对一的映射，该映射由一个转换函数定义。

从上面来看，其实如果要做Aptos的EVM，初步来看，仅仅只可以使用Rust版这个模块化的EVM，但是还是需要做类似的Aptos的账户到EVM账户的适配，如果不能吃透，哪里细节有误差，可能在Aptos上跑EVM合约都跑不起来，无法完美利用EVM的生态。

同时从文档上来看，为了让模块化的EVM集成进substrate，Parity的技术人员对其做了深思熟虑的考量和取舍

![[a46aeac2d5754c46e7428baeaacfeb0.png]]

上面文档上说了，这个pallet EVM与ETH主网产生的结果差距不大，包括gas花费和余额变化，为了达到这个目的，做了不少工作。但是为了使实现更简单，Parity目前并不需要让不可观察的行为，如状态根，变得相同。他们也不打算遵循完全相同的交易/收据(receipts)格式。然而，给定一个以太坊交易和一个Substrate账户的私钥，人们应该能够将任何以太坊交易转换成与该模块兼容的交易。ETH的交易和这个pallet EVM模块兼容的交易还是做了转换。

接下来看看代码实现, 主要对外实现了取款withdraw, 调用call，合约创建create，还有ETH gas到substrate weight单位的转换等等适配工作:

https://github.com/paritytech/frontier/blob/a8d4dd55643d324dbbf49be7f4e4e10494f56d39/frame/evm/src/lib.rs#L157-L383

**这个模块的写法完全就是substrate的pallet的模式约定写的，完全不可以照抄到Aptos上，所以在Aptos上做同样类似的工作，工作量很大（还需要在很熟悉的前提下）。**

##### pallet-ethereum

首先看看依赖:

```toml
ethereum = { version = "0.12.0", default-features = false, features = ["with-codec"] }
evm = { git = "https://github.com/rust-blockchain/evm", rev = "51b8c2ce3104265e1fd5bb0fe5cdfd2e0938239c", features = ["with-codec"], default-features = false }
serde = { version = "1.0.144", optional = true }

...

# Parity
codec = { package = "parity-scale-codec", version = "3.0.0", default-features = false }
ethereum-types = { version = "0.13.1", default-features = false }

...

# Frontier
fp-consensus = { version = "2.0.0-dev", path = "../../primitives/consensus", default-features = false }
fp-ethereum = { version = "1.0.0-dev", path = "../../primitives/ethereum", default-features = false }
fp-evm = { version = "3.0.0-dev", path = "../../primitives/evm", default-features = false }
fp-rpc = { version = "3.0.0-dev", path = "../../primitives/rpc", default-features = false }
fp-self-contained = { version = "1.0.0-dev", path = "../../primitives/self-contained", default-features = false }
fp-storage = { version = "2.0.0", path = "../../primitives/storage", default-features = false }
pallet-evm = { version = "6.0.0-dev", path = "../evm", default-features = false }
```

这个相关的EVM和以太坊依赖有点多，我们之前提到的crate和pallet，这个模块都依赖了。这些依赖的模块就不详细介绍了，其中一部分之前都提到过。

下面主要看源码，看看它干了什么:

[frontier/lib.rs at master · paritytech/frontier (github.com)](https://github.com/paritytech/frontier/blob/master/frame/ethereum/src/lib.rs)

看了大部分大致的函数名和一些逻辑，主要是实现以太坊交易的执行和校验(EIP2930 , EIP 1559的交易都支持)，交易执行当然就包括合约的调用和创建了，调用的是pallet-evm。包括在Hooks的时候，初始化和finalize块的时候读取一些log信息。还有各种调用的Origin校验。这里可以简单认为通过hooks回调，处理每一个ETH区块。

**这个模块同上，也是substrate runtime pallet的框架约束的写法，完全不能照搬到Aptos上，又是一大笔工作量和复杂度** 

###### 其他

当然，还有其他很多的EVM相关的模块，还有很多工作量没有调研，这里只调研几个重要的模块。


### 调研结论

移植EVM到Aptos的工作，即使要做一个Demo级别，可以做基本功能演示的，都需要巨大成本，现有团队配置，无法胜任。除非可以找到一个业内有丰富经验的区块链架构师。

### 使用frontier的教程

- [How to add Frontier code as a dependency to my substrate-parachain-template based parachain? · Issue #811 · paritytech/frontier (github.com)](https://github.com/paritytech/frontier/issues/811)
- [How to make a Frontier demo for running Ethereum smart contracts · Issue #774 · paritytech/frontier (github.com)](https://github.com/paritytech/frontier/issues/774)

- [frontier/template at master · paritytech/frontier (github.com)](https://github.com/paritytech/frontier/tree/master/template)