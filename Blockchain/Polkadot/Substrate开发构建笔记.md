# Substrate开发构建笔记

## libraries简介

在使用node template时，你不需要知道任何关于底层架构或正在使用的库的信息，因为基本组件已经组装好，可以使用。
然而，如果你想设计和构建一个自定义的区块链，你可能想熟悉可用的库，并知道这些不同的库有什么作用。

在架构中，你了解了一个substrate节点的核心组件，以及该节点的不同部分如何承担不同的责任。在更多的技术层面上，节点不同层之间的职责分离反映在用于构建基于Substrate的区块链的核心库中。下图说明了库如何反映外层节点和runtime的职责，以及primitives库如何提供两者之间的通信层。

![[Pasted image 20220831145529.png]]

### 核心节点库

使Substrate节点能够处理其网络责任的库，包括共识和区块执行，是在库名中使用`sc_`前缀的Rust crates。例如，`sc_service`库 [sc_service - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_service/index.html) 负责为Substrate区块链建立网络层，管理网络参与者和交易池之间的通信。`sc`的`c`就是`client`的缩写，也有网络通信和客户端的意思。

提供外层节点和runtime之间的通信层的库是在crate名称中使用`sp_`前缀。这些库协调了需要外层节点和runtime互动的活动。例如，`sp_std`库 [sp_std - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_std/index.html) 从Rust的标准库中提取有用的primitives，并使其可用于任何依赖runtime的代码。`sp`的`p`就是`primitives`的缩写

使你能够构建runtime逻辑并对传入和传出runtime的信息进行编码和解码的库是在crate名称中使用`frame_`前缀。`frame_*`库提供了runtime的基础设施。例如，`frame_system`库 [frame_system - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_system/index.html) 提供了一套与其他Substrate组件交互的基本函数，`frame_support` [frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/index.html) 使你能够声明runtime存储项、错误和事件。

除了`frame_*`库提供的基础设施，runtime还可以包括一个或多个`pallet_*`库。每个使用`pallet_`前缀的Rust crate代表一个单独的FRAME模块。在大多数情况下，你使用`pallet_*`库来组装你想纳入区块链的功能以适应你的项目。

你可以在不使用`frame_*`或`pallet_*`库的情况下，使用`sp_*`核心库所暴露的primitives来构建一个Substrate runtime。然而，`frame_*`或`pallet_*`库提供了组成Substrate runtime的最有效途径。

如果你去substrate仓库下面的`frame文件夹`下，可以发现各种frame模块，大部分的模块的package name 都是`pallet-`前缀打头的。

substrate有官方的pallets商城，可以复用别人开发好的pallet [Home | Substrate_ Marketplace](https://marketplace.substrate.io/)


### 模块化架构

核心库的分离为编写区块链逻辑提供了一个灵活和模块化的架构。primitives库提供了一个基础，外层节点和runtime都可以在此基础上构建，而不需要彼此直接沟通。primitives类型和traits在他们自己的独立的crates中被暴露出来，所以他们可以供外层节点和runtime组件使用，而不会引入循环的依赖问题。

### 前端库

除了使你能够建立基于Substrate的区块链的核心库之外，还有一些客户端库，你可以用来与Substrate节点互动。你可以使用客户端库来构建特定于应用程序的前端。一般来说，客户端库所暴露的功能是在Substrate远程过程调用（RPC）API之上实现的。关于使用元数据和前端库来构建应用程序的更多信息，请参见 https://docs.substrate.io/build/application-dev/#rpc-apis


### 深入substrate的Rust API

请看文档 [sc_service - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_service/index.html)


## 构建流程

在《架构》[Architecture | Substrate_ Docs](https://docs.substrate.io/fundamentals/architecture/) 中，你了解到Substrate节点由一个外层节点host和一个runtime执行环境组成。这些节点组件通过runtime API调用和host函数调用相互通信。在本节中，你将了解更多关于Substrate runtime如何被编译成平台Native可执行文件和存储在区块链上的WebAssembly（Wasm）二进制文件。在你看到二进制文件是如何被编译的内部工作后，你将了解更多关于为什么有两个二进制文件，它们何时被使用，以及如果你需要，你如何改变执行策略。

### 编译一个optimized artifact

你可能已经知道，你可以通过在 Substrate 节点项目的根目录下运行 cargo build --release 命令来编译 Substrate 节点。该命令为项目构建特定平台的可执行文件和WebAssembly二进制文件，并产生一个优化的可执行工件。生成优化的可执行工件包括一些编译后的处理。

作为优化过程的一部分，WebAssembly runtime二进制文件通过一系列内部步骤进行编译和压缩，然后被纳入链的创世状态中。为了让你更好地了解这个过程，下面的图总结了这些步骤。

![[Pasted image 20220901090207.png]]


#### 构建WASM binary

`wasm-builder`是一个工具，它将为你的项目构建WebAssembly二进制文件的过程整合到主要的`cargo build`过程中。这个工具被发布在`substrate-wasm-builder`  crate中。

当你开始构建过程时，cargo会从项目中所有的Cargo.toml构建一个依赖图。然后，runtime `build.rs`模块使用`substrate-wasm-builde`r crate将runtime的Rust代码编译成WebAssembly二进制文件，创建初始二进制artifact。

##### WASM里面包含的功能特性

默认情况下，`wasm-builder`在WebAssembly二进制文件和平台Native可执行文件中都会启用为项目定义的所有功能，除了`default`和`std`功能，这些功能只在Native构建中启用。

##### 用环境变量自定义构建过程

你可以使用以下环境变量来定制WebAssembly二进制文件的构建方式。

![[4e41b0bdf9ed9ba24104f58851674cd.png]]

##### 压缩优化WASM binary

`substrate-wasm-builder` crate使用较低级别的程序来优化指令序列，并删除任何不必要的代码，如用于调试的代码，以创建一个紧凑的WebAssembly二进制文件。然后，二进制文件被进一步压缩，以减少最终WebAssembly二进制文件的大小。当编译器处理节点的`runtime/src/lib.rs`文件时，它看到需求包含了生成的WebAssembly二进制文件。

```bash
include!(concat!(env!("OUT_DIR"), "/wasm_binary.rs"));
```

这段代码在其编译结果中包括紧凑的WebAssembly二进制文件（`WASM_BINARY`）和由编译器生成的未压缩的WebAssembly二进制文件（`WASM_BINARY_BLOATY`），并生成项目的最终可执行二进制文件。

在构建过程的每个阶段，WebAssembly二进制文件被压缩到越来越小的尺寸。例如，你可以比较Polkadot的每个WebAssembly二进制artifact的大小。

```bash
.rw-r--r-- 1.2M pep  1 Dec 16:13 │  ├── polkadot_runtime.compact.compressed.wasm
.rw-r--r-- 5.1M pep  1 Dec 16:13 │  ├── polkadot_runtime.compact.wasm
.rwxr-xr-x 5.5M pep  1 Dec 16:13 │  └── polkadot_runtime.wasm
```

你应该始终使用完全压缩的运行时（`*_runtime.compact.compressed.wasm`）WebAssembly二进制文件，用于链上升级和中继链验证。在大多数情况下，没有必要使用最初的WebAssembly二进制文件或compact工件。

### 执行策略

在你用Native和WebAssembly runtime编译了节点后，你使用命令行选项来指定节点应该如何操作。关于你可以用来启动节点的命令行选项的细节，请参见node-template命令行参考。[node-template | Substrate_ Docs](https://docs.substrate.io/reference/command-line-tools/node-template/)


当你启动节点时，节点可执行文件使用你指定的命令行选项来初始化链并生成创世块。作为这个过程的一部分，节点将WebAssembly runtime作为一个存储项的value和一个相应的`:code` key添加。

在你启动节点后，运行中的节点会选择要使用的runtime(Native runtimeh或者WASM runtime)。默认情况下，节点总是使用WebAssembly runtime进行所有操作，包括。

- 同步
- 编写生成新块
- 导入区块
- 与offchain工作者互动

#### WASM runtime选择

使用WebAssembly runtime是很重要的，因为WebAssembly和Native rungime可能会出现分歧。例如，如果你对runtinme做了修改，你必须生成一个新的WebAssembly blob，并更新链以使用新版本的WebAssembly runtime。更新后，WebAssembly runtime与native runtime不同。为了说明这种差异，所有的执行策略都把runtime的WebAssembly表示作为canonical runtime。如果native runtime和WebAssembly runtime的版本不同，WebAssembly runtime总是被选中。

由于WebAssembly runtime被存储为区块链状态的一部分，网络必须就这个二进制的表示方法达成共识。为了达成关于二进制的共识，代表WebAssembly runtime的blob必须在所有同步节点上完全相同。

#### WASM 执行环境

WebAssembly的执行环境可能比Rust的执行环境更具限制性。例如，WebAssembly执行环境是一个32位的架构，最大内存为4GB。可以在WebAssembly runtime上执行的逻辑，总是可以在Rust执行环境中执行。然而，并非所有能在Rust native runtime执行的逻辑都能在WebAssembly runtime执行。区块生成节点通常使用WebAssembly执行环境，以帮助确保他们产生有效的块。

#### Native执行环境

虽然WebAssembly runtime是默认选择的，但你可以通过指定一个执行策略作为命令行选项来覆盖所有或特定操作的runtime。  
  
如果native runtime和WebAssembly runtime有相同的版本，你可以有选择地使用native来代替WebAssembly runtime，也可以在WebAssembly runtime之外使用，或者在使用WebAssembly runtime失败时作为退路。一般来说，你只会因为性能的原因选择使用native runtime，或者因为它是一个比WebAssembly runtime更少限制的环境。例如，你可能想使用native runtime进行初始同步。为了使用native runtime来完成这个同步块，你可以使用`--execution-syncing native`或`--execution-syncing native-else-wasm`这个命令行选项来启动节点。  
  
关于使用命令行选项为所有或特定操作指定执行策略的信息，见node-template。[node-template | Substrate_ Docs](https://docs.substrate.io/reference/command-line-tools/node-template/) 关于执行策略变体的信息，见ExecutionStrategy  [ExecutionStrategy in sp_state_machine - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_state_machine/enum.ExecutionStrategy.html)


### 构建WASM时排除Native runtime

开始一个新的链需要一个WebAssembly runtime。在提供了一个初始的WebAssembly runtime后，代表WebAssembly runtime的blob可以作为chain spec的一部分传递给其他节点。在一些罕见的情况下，你可能想在没有native runtime的情况下编译WebAssembly target。例如，如果你正在测试一个WebAssembly runtime时，以准备一个无分叉升级，你可能想只编译新的WebAssembly二进制。

尽管这是一个罕见的使用情况，你可以使用`build-only-wasm.sh` [substrate/build-only-wasm.sh at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/blob/master/.maintain/build-only-wasm.sh) 脚本来构建`no_std` WebAssembly二进制，而不编译native runtime。

你也可以使用`wasm-runtime-overrides`命令行选项，从文件系统中加载WebAssembly。

### 编译Rust时排除WASM

如果你想为一个节点编译Rust代码而不构建新的WebAssembly runtime，你可以使用`SKIP_WASM_BUILD`作为构建选项。这个选项主要用于在不需要更新WebAssembly的情况下加快编译时间。

## Runtime存储

runtime存储允许你在区块链中存储数据，这些数据在区块之间永久存在，并可以从runtine逻辑中访问。存储应该是区块链runtime开发人员最关键的关注点之一。设计良好的存储系统可以减少网络中节点的负载，最终降低你的区块链参与者的开销成本 换句话说，区块链runtime存储的基本原则是尽量减少其使用。

Substrate暴露了一组分层、模块化的存储API，允许runtime开发人员做出最适合他们的存储决定。本文件旨在提供有关Substrate runtime存储接口的信息和最佳实践。

### 存储项

在Substrate中，任何pallet都可以引入新的存储项，这些项目将成为你区块链状态的一部分。这些存储项可以是简单的单值项目，也可以是更复杂的存储maps。你选择实现的存储项的类型完全取决于它们在你的runtime逻辑中的预期作用。

FRAME的存储模块 [frame_support::storage - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/) 使runtime开发人员能够访问Substrate的灵活存储API，它可以支持任何可由SCALE编解码 [Type encoding (SCALE) | Substrate_ Docs](https://docs.substrate.io/reference/scale-codec/) 的value。这些包括：

- 存储value [StorageValue in frame_support::storage - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageValue.html) -  用于存储任何单一的值，如u64。
- 存储map [StorageMap in frame_support::storage - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageMap.html) --用于存储KV map，如账户到余额的映射。
- 双存储 Map [StorageDoubleMap in frame_support::storage - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageDoubleMap.html) - 用来作为具有两个Key的存储map的实现，以提供有效移除具有共同第一个Key的所有条目的能力。
- N存储Map [StorageNMap in frame_support::storage - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageNMap.html) - 用于存储具有任意数量Key的map，它可以作为建立三重存储map、四重存储map等的基础。是双存储Map的扩展。

#### 存储value

这种类型的存储项应该被用于被runtime视为单一单位的值。这可能是单个原始值、单个`struct`或单个相关项目的collection。如果一个存储项被用于存储项目的`lists`，runtime开发者应该注意他们使用的`lists`的大小。大的`lists`会产生存储成本，就像大的`structs`一样。此外，在你的runtime中对一个大的`lists`进行迭代可能会导致超过块生产时间。如果这种情况发生在主权链上，区块链会变慢。如果这种情况发生在parachain [Glossary | Substrate_ Docs](https://docs.substrate.io/reference/glossary/#parachain) 上，区块链将停止生产区块并停止运作。因为相当于破坏了共识，出块是由间隔的。

虽然将相关项目包裹在共享`struct`中是减少存储读取次数的绝佳方式，但在某些时候，对象的大小将开始产生成本，可能超过存储读取的优化。阅读关于基准测试的内容 [Benchmark | Substrate_ Docs](https://docs.substrate.io/test/benchmark/) ，了解如何优化执行时间。

请参考Storage Value文档，了解Storage Value所暴露的方法的全部清单。[StorageValue in frame_support::storage - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageValue.html#required-methods)

#### 存储map

map数据结构是管理项目集合的理想选择，这些项目集合的元素将被随机访问，而不是按顺序对它们进行整体迭代。Substrate中的存储map被实现为KV映射，它提供了与传统hash map [Hash table - Wikipedia](https://en.wikipedia.org/wiki/Hash_table) 类似的接口，以实现随机查找。为了给runtime工程师更多的控制权，Substrate允许开发者选择最适合其使用情况的散列算法来生成map的key。这在关于Hash算法的章节中有所涉及。

请参考Storage Map文档，以获得Storage Map所暴露的方法的全面列表。[StorageMap in frame_support::storage - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageMap.html#required-methods)

####  Double Storage Map

双存储map与单一存储map非常相似，只是它们包含两个Key，这对于查询具有共同Key的值很有用。

#### N storage map

N存储地map也与它的兄弟姐妹非常相似，即存储map和双存储map，但能够容纳任何任意数量的Keys。

要在FRAME v2中指定N存储map的key，在声明`StorageNMap`时，必须提供一个包含特殊`NMapKey`struct的tuple作为Key（即第二个）类型参数的类型。

关于使用N存储map的语法的更多细节，请参考N存储map文档。[StorageNMap in frame_support::storage - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/trait.StorageNMap.html)

#### 迭代遍历storage map

substrate 存储map在其Key和Value方面是可迭代的。因为map经常被用来跟踪无界的数据集（如账户余额），在runtime不谨慎地对它们进行迭代可能会导致块不能及时产生。此外，由于访问map的元素比访问本地列表的元素需要更多的数据库读取，因此就执行时间而言，map迭代的成本明显高于list迭代。

总的来说，Substrate专注于根据原则和最佳实践进行编程，而不是硬性的对错规则。这里的信息旨在帮助你理解Substrate的所有存储功能，以及如何在尊重其设计原则的情况下使用它们。例如，在你的runtime中对存储map进行迭代既不对也不错--然而，就最佳实践而言，避免这样做会被认为是更好的方法。

Substrate的Iterable Storage Map接口定义了以下方法:

- `iter()`- 枚举map中的所有元素，没有特定的顺序。如果你在做这个的时候改变map，你会得到未定义的结果。
    -  [`IterableStorageMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageMap.html#tymethod.iter) 
    -  [`IterableStorageDoubleMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageDoubleMap.html#tymethod.iter)
    -   [`IterableStorageNMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageNMap.html#tymethod.iter).
- `drain()` -  从map中移除所有的元素，并不按特定顺序遍历它们。如果你在做这个的时候向map添加元素，你会得到未定义的结果。
    -   [`IterableStorageMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageMap.html#tymethod.drain)
    -   [`IterableStorageDoubleMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageDoubleMap.html#tymethod.drain)
    -    [`IterableStorageNMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageNMap.html#tymethod.drain)
- `translate()` 使用提供的函数来translate map的所有元素，没有特定的顺序。要从map上删除一个元素，请从translate函数中返回None。
    -    [`IterableStorageMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageMap.html#tymethod.translate)
    -    [`IterableStorageDoubleMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageDoubleMap.html#tymethod.translate)
    -   [`IterableStorageNMap`](https://paritytech.github.io/substrate/master/frame_support/storage/trait.IterableStorageNMap.html#tymethod.translate)

### 声明存储项

runtime的存储项是在任何基于FRAME的pallet中用`#[pallet::storage]` [pallet in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/attr.pallet.html#storage-palletstorage-optional) 创建的。下面是一个声明四种不同类型存储项的例子。

```rust
#[pallet::storage]
type SomePrivateValue<T> = StorageValue<_, u32, ValueQuery>;

#[pallet::storage]
#[pallet::getter(fn some_primitive_value)]
pub(super) type SomePrimitiveValue<T> = StorageValue<_, u32, ValueQuery>;

#[pallet::storage]
pub(super) type SomeComplexValue<T: Config> = StorageValue<_, T::AccountId, ValueQuery>;

#[pallet::storage]
#[pallet::getter(fn some_map)]
pub(super) type SomeMap<T: Config> = StorageMap<_, Blake2_128Concat, T::AccountId, u32, ValueQuery>;

#[pallet::storage]
pub(super) type SomeDoubleMap<T: Config> = StorageDoubleMap<_, Blake2_128Concat, u32, Blake2_128Concat, T::AccountId, u32, ValueQuery>;

#[pallet::storage]
#[pallet::getter(fn some_nmap)]
pub(super) type SomeNMap<T: Config> = StorageNMap<
    _,
    (
        NMapKey<Blake2_128Concat, u32>,
        NMapKey<Blake2_128Concat, T::AccountId>,
        NMapKey<Twox64Concat, u32>,
    ),
    u32,
    ValueQuery,
>;
```

请注意，map的存储项指定了将被使用的Hash算法。

#### QueryKindTrait

传递给存储项的`QueryKindTrait` [QueryKindTrait in frame_support::storage::types - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/types/trait.QueryKindTrait.html) 的impl决定了当存储中没有value时应该如何处理。对于`OptionQuery` [OptionQuery in frame_support::storage::types - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.OptionQuery.html)，当存储中没有值时，`get`法将返回`None`。对于`ValueQuery` [ValueQuery in frame_support::storage::types - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/storage/types/struct.ValueQuery.html) ，当存储中没有value时，`get`方法将返回`OnEmpty ` generic。对于有特定默认值需要配置的情况，建议使用`ValueQuery`。

#### 可见性

在上面的例子中，除了`SomePrivateValue`之外，所有的存储项都通过`pub`关键字的方式被公开。区块链存储从runtime外部的来看，总是公开可见的 [Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/#accessing-storage-items) ；substrate 存储项的可见性只影响runtime内的其他pallet是否能够访问一个存储项。

#### Getter方法

``#[pallet::getter(..)]`` 宏提供了一个可选的`get`扩展，可以用来在包含存储项的模块上实现该存储项的getter方法；该扩展将getter函数的期望名称作为参数。如果你省略了这个可选的扩展，你仍然能够访问存储项的值，但你不能通过在模块上实现的getter方法来实现；相反，你将需要使用存储项的get方法。

可选的`getter`扩展只影响到从Substrate代码中访问存储项的方式--你将始终能够查询你的runtime的存储，以获得存储项的值。

下面是一个例子，它为一个名为`SomeValu`e的存储值实现了一个名为`some_value`的getter方法。这个pallet现在除了`SomeValue::get()`方法之外，还可以访问`Self::some_value()`方法。

```rust
#[pallet::storage]
#[pallet::getter(fn some_value)]
pub(super) type SomeValue = StorageValue<_, u64, ValueQuery>;
```

#### 默认值

substrate允许你指定一个默认值，当一个存储项的值未被设置时，该值将被返回。尽管默认值实际上并不占用runtime存储，但runtime逻辑会在执行期间看到这个值。

下面是一个在存储中指定默认值的例子。

```rust
#[pallet::type_value]
pub(super) fn MyDefault<T: Config>() -> T::Balance { 3.into() }
#[pallet::storage]
pub(super) type MyStorageValue<T: Config> =
    StorageValue<Value = T::Balance, QueryKind = ValueQuery, OnEmpty = MyDefault<T>>;
```

注意，为了增加每个存储字段的清晰度，上面的语法是声明存储项的非简略版本。

### 访问存储项

用Substrate构建的区块链暴露了一个远程程序调用（RPC）服务器，可用于查询runtime存储。你可以使用像Polkadot JS [polkadot{.js}](https://polkadot.js.org/) 这样的软件库，从你的代码中轻松与RPC服务器互动，并访问存储项。Polkadot JS团队还维护Polkadot Apps UI [Polkadot/Substrate Portal](https://polkadot.js.org/apps/) ，这是一个功能齐全的网络应用程序，用于与基于Substrate的区块链互动，包括查询存储。

### Hash算法

Substrate中的存储map的一个新特点是，它们允许开发者指定用于生成map的key的散列算法。一个用于封装Hash逻辑的Rust对象被称为 "Hasher"。广义上讲，Substrate开发者可用的Hash算法可以用两种方式描述：（1）它们是否是加密的；以及（2）它们是否产生透明的输出。

为了完整起见，非透明Hash算法的特征描述如下，但请记住，任何不产生透明输出的Hash算法在基于FRAME的区块链中已经被废弃。

#### 密码学Hash算法

密码学Hash算法使我们能够建立工具，使操纵Hash算法的输入来影响其输出变得极其困难。例如，即使输入是数字1到10，密码学Hash算法也会产生一个广泛的输出分布。当用户能够影响存储map的Key时，使用密码学Hash算法是至关重要的。如果不这样做，就会产生一个攻击矢量，使恶意行为者很容易降低你的区块链网络的性能（大量让Hash产生Key碰撞）。一个应该使用密码学Hash算法来生成其密钥的map的例子是用于跟踪账户余额的map。在这种情况下，使用密码学Hash算法是很重要的，这样攻击者就不能用许多小的转账来轰击你的系统，让你的账户号码顺序化。如果没有适当的密码学Hash算法，这将创造一个不平衡的存储结构，从而影响性能。阅读更多关于公共的substrate哈希算法。

密码学Hash算法比它们的非密码学Hash算法更加复杂和资源密集，这就是为什么runtime工程师必须了解它们的适当用法，以便最好地利用Substrate提供的灵活性。

#### 透明Hash算法

一个透明的Hash算法是一个能让人很容易发现和验证用于生成特定输出的输入的算法。在Substrate中，Hash算法通过将算法的输入与输出相连接而变得透明。这使得用户可以很容易地检索到一个密钥的原始未散列值，并在他们愿意的情况下验证它（通过重新Hash它）。Substrate的创建者已经放弃了在基于FRAME的runtime中使用非透明的Hasher，所以提供这个信息主要是为了完整性。事实上，如果你想访问 iterable map的功能，就必须使用透明Hash算法。

#### 公共的Substrate Hasher

本表列出了Substrate中使用的一些常见的哈希算法，并表示那些是密码学的，那些是透明的。

![[1662104693473.png]]

注意，上图打叉的反而是支持这个属性的意思，因为显然Blake2是密码学Hash算法。

- [Blake2_128Concat in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/struct.Blake2_128Concat.html)
- [Twox64Concat in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/struct.Twox64Concat.html)
- [Identity in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/struct.Identity.html)

indentity Hasher封装了一种Hash算法，其输出等于其输入（身份函数）。这种类型的哈希算法应该只在起始密钥已经是一个密码学哈希完后的值的情况下使用。

### 创世配置



### 最佳实践

Substrate旨在提供一个灵活的框架，使你能够建立适合你的需求的区块链。然而，Substrate代码库坚持了一些最佳实践，以促进创建安全、性能和可长期维护的区块链网络。以下各节概述了使用Substrate存储的最佳实践，同时也描述了促使它们产生的重要的第一原则。

#### 存储什么

记住，区块链runtime存储的基本原则是尽量减少其使用。只有关键的共识数据应该存储在你的runtime中。在可能的情况下，使用Hash等技术来减少你必须存储的数据量。例如，Substrate的许多治理能力--如Democracy pallet的propose功能 [Call in pallet_democracy::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_democracy/pallet/enum.Call.html#variant.propose) --允许网络参与者对dispatchable call的Hash值进行投票，其大小总是有限制的，而不是调用本身其长度可能是无限制的。这在runtime升级的情况下尤其如此，在这种情况下，dispatchable call需要整个runtime Wasm blob作为其参数。因为这些治理机制是在链上实现的，所有需要就特定提案的状态达成共识的信息也必须存储在链上--这包括正在投票的内容。然而，通过将链上提案与其哈希值绑定，Substrate的治理机制允许这样做，将与提案相关的所有数据推迟到提案被批准后再上链。这意味着存储不会被浪费在失败的提案上。  
  
一旦提案通过，有人就可以启动实际的dispatchable call（包括其所有参数），这将被Hash并与提案中的Hash进行比较。另一个使用哈希值来最小化链上存储的数据的常见模式是在IPFS https://docs.ipfs.io 中存储与一个对象相关的pre-image；这意味着只有IPFS的位置（一个大小受限的哈希值）需要在链上存储。  
  
哈希值只是一种机制，可以用来控制runtime存储的大小。另一个机制的例子是边界。  


#### 先校验，最后写

Substrate在extrinsic dispatch之前不缓存状态。相反，它在extrinsic被执行调用时直接apply变化。如果一个extrinsic失败了，任何状态变化都会持续存在。正因为如此，在确定所有前提条件得到满足之前，不要进行任何存储突变，这一点很重要。一般来说，可能导致存储突变的代码块应该有如下结构。

```
{
  // all checks and throwing code go here

  // ** no throwing code below this line **

  // all event emissions & storage writes go here
}
```

不要使用runtime存储来存储逻辑上是原子性的操作范围内的中间数据或瞬时数据，或在操作失败时将不需要的数据。这并不意味着runtime存储不应该被用来跟踪需要多个原子操作的正在进行的操作的状态，如实用Utility pallet中的多签功能 [Call in pallet_utility::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_utility/pallet/enum.Call.html#variant.as_multi) 的情况。在这种情况下，runtime存储被用来跟踪dispatchable call的签名者，即使一个给定的调用可能永远不会收到足够的签名而被实际调用。在这种情况下，每个签名被认为是正在进行的多签操作中的一个原子事件；记录一个签名所需的数据在与该签名相关的所有前提条件得到满足后才会被存储。


#### 创建边界

为存储项的大小创建界限是控制runtime存储使用的一种极其有效的方法，也是整个Substrate代码库中反复使用的一种方法。一般来说，任何由用户行为决定大小的存储项都应该有一个界限。上面描述的Multisig pallet [Config in pallet_multisig::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_multisig/pallet/trait.Config.html#associatedtype.MaxSignatories) 的多签名功能就是这样一个例子。在这种情况下，与多重签名操作相关的签名者名单是由多重签名参与者提供的。因为这个签名人名单对于就多签名操作的状态达成共识是必要的，所以它必须被存储在runtime。然而，为了让runtime开发人员控制这些列表可能占用的存储空间，Utility pallet要求用户对这个数字配置一个边界，这个边界将在任何东西被写入存储之前作为一个前提条件。

## 交易，权重和费用

## 自定义pallet

## Pallets之间联接

## 事件和错误

## 随机

## Chain Spec

在Substrate中，chain spec是描述基于Substrate的区块链网络的信息集合。例如，chain spec确定了一个区块链节点所连接的网络，它最初与之通信的其他节点，以及节点为产生区块必须同意的初始状态。

chain spec是用ChainSpec结构定义的 [GenericChainSpec in sc_service - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_service/struct.GenericChainSpec.html) 。ChainSpec结构将一条链所需的信息分成两部分。

- 客户端规范(client spec)，包含底层外层节点用来与网络参与者进行通信并向遥测端点(telemetry endpoints)发送数据的信息。许多这些链的chain spec设置可以在启动节点时被命令行选项覆盖，也可以在区块链启动后改变。
- 网络中所有节点同意的初始创世状态。创世状态必须在区块链首次启动时建立，此后如果不启动一个全新的区块链，就不能改变它。

### 自定义外层节点设置

对于外层节点，chain spec控制的信息有：。

- 节点与之通信的启动节点(bootnodes)。
- 节点发送遥测数据的服务器端点。
- 节点所连接的网络的人和机器可读的名称。

因为Substrate框架是可扩展的，你也可以自定义chain spec，以包括额外的信息。例如，你可以配置外层节点连接到特定高度的特定区块，以防止在从创世块开始同步一个新节点时进行长程攻击(long range attacks)。

注意，你可以在创世之后定制外层节点设置。然而，节点只添加使用相同`protocolId`的P2P对端Peers。[ProtocolId in sc_network::config - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_network/config/struct.ProtocolId.html)

### 自定义创世状态配置

网络中的所有节点必须在创世状态上达成一致，然后才能就任何后续区块达成一致。在chain spec的创世部分配置的信息被用来创建创世块。它在你启动第一个节点时生效，不能用命令行选项来覆盖（也就是硬编码在代码里面的）。然而，你可以在chain spec的创世部分配置一些信息。例如，你可以自定义chain spec的创世部分，包括以下信息。

- 初始代币持有人余额，比如Alice Bob这样的账户余额。
- 最初属于治理委员会的账户。
- 控制sudo密钥的管理账户。

substrate节点还包括链上runtime逻辑的编译后的WebAssembly字节码，所以初始runtime也必须在chain spec中提供。
https://raw.githubusercontent.com/substrate-developer-hub/substrate-docs/main/static/assets/tutorials/cumulus/chain-specs/rococo-custom-2-plain.json


### 保存chain spec信息

chain spec中的信息可以以Rust代码或JSON文件的形式存储。susbtrate节点通常包括至少一个，也可能有多个硬编码的chain specs。将这些信息作为Rust代码直接包含在节点中，确保节点可以连接到至少一条链，而不需要节点操作者提供任何额外的信息。如果你在构建区块链时打算定义一个主网络，那么这个主网络规范通常是在外层节点中硬编码的。

或者，你可以使用`build-spec`子命令将chain spec序列化为一个JSON文件。在启动测试网络或私有链时，通常会将JSON编码的链规范与节点二进制文件一起分发。

### 提供一个chain spec给一个启动节点

每次启动一个节点时，你都会提供节点应该使用的chain spec。在最简单的情况下，节点使用一个默认的chain spec，它被硬编码到节点的二进制文件中。你可以在启动节点时使用 --chain 命令行选项来选择另一种硬编码的chain spec。例如，你可以通过指定 --chain local 作为命令行选项，指示节点使用与 "local "字符串相关的链规范。

如果你不想用硬编码的chain spec来启动节点，你可以把它作为一个JSON文件来提供。例如，你可以通过指定-chain=someCustomSpec.json作为命令行选项，指示节点使用someCustomSpec.json文件中的chain spec。如果你指定一个JSON文件，节点会尝试对提供的JSON chain进行反序列化进节点的`Chain Spec struct`中，然后使用它。

### 为runtime声明存储项

在大多数情况下，substrate runtime将要求存储项在创世时被配置。例如，如果你正在用FRAME开发 runtime，任何用Config trait声明的存储项都需要在创世时配置。这些存储值是在chain spec的创世部分配置的。

#### 创建自定义的chain spec

如果你正在为开发、测试或演示目的创建一个一次性网络，你可能需要一个完全定制的chain spec。要创建一个完全定制的chain spec，你可以将默认的chain spec导出为JSON格式，然后编辑JSON文件中的字段。例如，你可以使用build-spec子命令将链规范导出为JSON文件。


```bash
substrate build-spec > myCustomSpec.json
```

在你导出chain spec后，你可以在文本编辑器中修改其任何字段。例如，你可能想改变网络名称、bootnodes和任何genesis 存储项，如token余额。编辑完JSON文件后，你可以使用定制的JSON启动节点。比如说。

```bash
substrate --chain=myCustomSpec.json
```

### Raw chain spec

substrate节点支持runtime升级。通过runtime升级，区块链的runtime可以与链开始时不同。chain spec包含的信息结构可以被节点的runtime理解。例如，考虑一下默认Substrate节点的chain spec .json文件的这个摘录。

```json
"sudo": {
  "key": "5GrwvaEF5zXb26Fz9rcQpDWS57CtERHpNehXCPcNoHGKutQY"
}
```

在这个chain spec被用来初始化节点的创世存储之前，人类可读的key必须被转换为storage trie [Runtime storage | Substrate_ Docs](https://docs.substrate.io/build/runtime-storage/) 的实际存储键。这种转换是直截了当的，但它要求节点的runtime 能够理解chain spec。

如果一个拥有升级的runtime的节点试图从创世块开始同步一个链，它将无法理解这个人类可读的chain spec中的信息。出于这个原因，有一个chain spec的第二个编码。这第二个编码创建了一个raw版本的chain spec。

在分发JSON格式的chan spec时，你应该以raw格式分发，以确保所有节点在runtime升级后也能同步链。基于substrate的节点支持--raw标志，以产生raw chain spec。

```bash
substrate build-spec --chain=myCustomSpec.json --raw > customSpecRaw.json
```

转换为raw格式后，sudo key片段看起来像这样。

```json
"0x50a63a871aced22e88ee6466fe5aa5d9": "0xd43593c715fdd31c61141abd04a99fd6822c8558854ccde39a5684e7a56da27d",
```

### 一些参考

- 怎样配置创世状态 [Configure genesis state | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/basics/configure-genesis-state/)
- 怎样自定义chain spec [Customize a chain specification | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/basics/customize-a-chain-specification/)
- substrate node template的chain spec [substrate-node-template/chain_spec.rs at main · substrate-developer-hub/substrate-node-template (github.com)](https://github.com/substrate-developer-hub/substrate-node-template/blob/main/node/src/chain_spec.rs)

## 特权调用(privileged calls)和起源(origins)

## RPCs

## 应用开发

## 升级Runtime

