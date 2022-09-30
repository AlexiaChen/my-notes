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

在 [[Substrate学习笔记#Substrate架构]] 中，你了解到Substrate节点由一个外层节点host和一个runtime执行环境组成。这些节点组件通过runtime API调用和host函数调用相互通信。在本节中，你将了解更多关于Substrate runtime如何被编译成平台Native可执行文件和存储在区块链上的WebAssembly（Wasm）二进制文件。在你看到二进制文件是如何被编译的内部工作后，你将了解更多关于为什么有两个二进制文件，它们何时被使用，以及如果你需要，你如何改变执行策略。

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

Substrate的runtime存储API包含了在你的区块链的创世区块中初始化存储项的功能。创世存储配置API暴露了许多初始化存储的机制，所有这些机制都在`#[pallet::genesis_config]`中有入口点。`GenesisConfig`数据类型定义`在#[pallet::genesis_config]`属性下，`#[pallet::genesis_build]` 属性用于构建genesis配置。  
  
要消耗一个pallet的创世配置能力，你必须在将pallet添加到runtime时包含`config`元素。所有为runtime提供信息的pallet的`GenesisConfig`类型将被聚合到该runtime的一个`GenesisConfig`类型中，该类型实现了`BuildStorage trait` [BuildStorage in sp_runtime - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sp_runtime/trait.BuildStorage.html)。例如，在`node_template_runtime::GenesisConfig struct` [GenesisConfig in node_template_runtime - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/node_template_runtime/struct.GenesisConfig.html)中，该类型的每个属性都对应于runtime的pallet中具有`Config`元素的`GenesisConfig`。最终，runtime的`GenesisConfig`通过`ChainSpec trait` [ChainSpec in sc_chain_spec - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_chain_spec/trait.ChainSpec.html) 的方式暴露出来。  
  
关于使用Substrate的创世存储配置能力的一个完整而具体的例子，请参考Substrate代码库中的Chain Spec中的society pallet的创世存储配置。继续阅读以获得对这些能力的更详细的描述。  [substrate/chain_spec.rs at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/blob/master/bin/node/cli/src/chain_spec.rs)

#### `genesis_config`

`#[pallet::genesis_config]` [pallet in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/attr.pallet.html#genesis-config-palletgenesis_config-optional) 宏提供了一个扩展，将为pallet的`GenesisConfig`数据类型添加一个属性。这个属性的值将被用作你的链的创世块中的存储项的初始值。`config`扩展需要一个参数，它将决定`GenesisConfig`数据类型的属性名称--如果提供get方法，这个参数是可选的。

下面是一个例子，演示了使用`config`扩展和一个名为`MyVal`的存储值，在`GenesisConfig`数据类型上为存储值的pallet创建一个名为`init_val`的属性。然后，这个属性被用于演示使用`GenesisConfig`类型在你的链的创世块中设置存储值的初始值。

在`my_pallet/src/lib.rs` :

```rust
#[pallet::genesis_config]
pub struct GenesisConfig<T: Config> {
		pub init_val: u64,
	}
```

在 `chain_spec.rs`: 

```rust
GenesisConfig {
    my_pallet: MyPalletConfig {
        init_val: 221u64 + SOME_CONSTANT_VALUE,
    },
}
```

#### `gensis_build`

`#[pallet::genesis_build]` [pallet in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/attr.pallet.html#genesis-build-palletgenesis_build-optional) 属性允许你定义`genesis_configuration`如何在pallet本身中构建（这让你可以访问pallet的私有函数）。

这里有一个例子，演示了使用`#[pallet::genesis_config]`和`#[pallet::genesis_build]`来设置一个存储项的初始值。在这种情况下，这个例子涉及两个存储项：一个表示成员账户ID的列表，另一个指定列表中的一个特殊成员（首要成员）。

在`my_pallet/src/lib.rs`：

```rust
#[pallet::genesis_config]
struct GenesisConfig {
    members: Vec<T::AccountId>,
    prime: T::AccountId,
}

#[pallet::genesis_build]
impl<T: Config> GenesisBuild<T> for GenesisConfig {
    fn build(&self) {
        Pallet::<T>::initialize_members(&self.members);
        SomeStorageItem::<T>::put(self.prime);
    }
}
```

在`chain_spec.rs`中

```rust
GenesisConfig {
    my_pallet: MyPalletConfig {
        members: LIST_OF_IDS,
        prime: ID,
    },
}
```

你也可以使用`genesis_build`来定义一个`GenesisConfig`属性，该属性不与特定的存储项绑定。如果你想在你的pallet中调用一个私有的helper函数来设置几个存储项，或者调用一个在你的pallet中包含的其他pallet中定义的函数，这可能是可取的。例如，使用一个名为`intitialize_members`的假想的私有函数，这将看起来像:

在`my_pallet/src/lib.rs`:

```rust
#[pallet::genesis_config]
struct GenesisConfig {
    members: Vec<T::AccountId>,
    prime: T::AccountId,
}

#[pallet::genesis_build]
impl<T: Config> GenesisBuild<T> for GenesisConfig {
    fn build(&self) {
        Pallet::<T>::initialize_members(&config.members);
        SomeStorageItem::<T>::put(self.prime);
    }
}
```

在 `chain_spec.rs`

```rust
GenesisConfig {
    my_pallet: MyPalletConfig {
        members: LIST_OF_IDS,
        prime: ID,
    },
}
```



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

当交易被执行或数据被存储在链上时，该活动改变了链的状态并消耗了区块链资源。因为区块链可用的资源是有限的，所以管理链上的操作如何消耗这些资源很重要。除了在实际方面受到限制--如存储容量--区块链资源代表了恶意用户的潜在攻击媒介。例如，一个恶意的用户可能会试图用信息使网络超载，以阻止网络产生新的区块。为了保护区块链资源不被耗尽或超载，你需要管理它们如何被提供以及如何被消耗。需要注意的资源包括。

- 内存使用
- 存储输入和输出
- 计算
- 交易和区块大小
- 状态数据库大小

substrate为区块的编写者（生成者）提供了几种方法来管理对资源的访问，并防止链上的个别组件消耗过多的任何单一资源。区块编写者可用的两个最重要的机制是权重和交易费用。

权重是用来管理验证一个区块所需的时间。一般来说，权重是用来描述执行区块body中的calls所需的时间。通过控制一个区块所能消耗的执行时间，权重对存储输入和输出以及计算设定了限制。

一个区块所允许的一些权重是作为区块初始化和finalization的一部分而消耗的。权重也可能被用来执行强制性的 inherent extrinsic calls。为了帮助确保区块不消耗过多的执行时间，并防止恶意用户用不必要的calls使系统超载，权重与交易费用结合使用。

交易费用提供了一种经济激励，以限制执行时间、计算和执行操作所需的calls数量。交易费也被用来使区块链在经济上可持续发展，因为它们通常适用于由用户发起的交易，并在执行交易请求之前扣除。

### 费用怎么被计算

一项交易的最终费用是用以下参数计算的。

- 基本费用。这是用户为一项交易支付的最低金额。它在runtime被声明为一个基本权重，并使用`WeightToFee`转换为费用。
- 权重费。与交易消耗的执行时间（输入和输出以及计算）成比例的费用。
- 长度费。与交易的编码长度成比例的费用。
- tip：一个可选的提示，用于提高交易的优先级，使其有更大的机会被交易队列所包含。

基本费用和按比例计算的权重和长度费用构成了inclusion fee(基本费用+权重费用+长度费用)。inclusion fee是交易被包含在一个区块中必须具备的最低费用。

### 使用transaction payment pallet

交易支付pallet [pallet_transaction_payment - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_transaction_payment/index.html) 提供了计算inclusion费用的基本逻辑。

你也可以使用交易支付pallet来做：

- 使用`Config::WeightToFee`将权重值转换为基于货币类型的可扣除费用。
- 使用`Config::FeeMultiplierUpdate`，根据上一个区块结束时链的最终状态，通过定义一个乘数（倍率）来更新下一个区块的费用。
- 使用`Config::OnChargeTransaction`管理交易费用的提取、退款和存款。

你可以在交易支付文档中了解更多关于这些configuration trait [pallet_transaction_payment - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_transaction_payment/index.html) 的信息。

你应该注意，交易费用是在交易执行前提取的。在交易执行后，交易权重可以被调整以反映交易使用的实际资源。如果交易使用的资源比预期的少，交易费会被修正，调整后的交易费会被存入。

#### 仔细看一下inclusion fee

计算最终费用的公式是这样的:

$$

inclusion fee = base fee + length fee + （targeted fee adjustment * weight fee）

$$

$$
final fee = inclusion fee + tip
$$

在这个公式中，`targeted_fee_adjustment` 是一个乘数，可以根据网络的拥堵情况来调整最终的费用。

- 从基本权重(base weight)得出的基本费用(`base_fee`)涵盖了签名验证等包容开销。
- `length_fee`是一个按字节计算的费用，它乘以编码的extrinsic的长度。
- `weight_fee`费用是用两个参数计算的。
  `ExtrinsicBaseWeight`，该参数在runtime声明，适用于所有的extrinsis。
  `#[pallet::weight]`注解，该注解说明了一个extrinsic的复杂性。

为了将权重转换为货币，runtime必须定义一个`WeightToFee`结构，实现一个转换函数`Convert<Weight,Balance>`。

请注意，在调用extrinsic之前，extrinsic的发送者会被收取inclusion fee。即使交易执行失败，该费用也会从发送者的余额中扣除。

#### 余额不足的账户

如果一个账户没有足够的余额来inclusion fee并保持活力--也就是说，没有足够的余额来支付inclusion fee并保持最低存在的存款(deposit)--那么你应该确保交易被取消，这样就不会被扣除费用，交易也不会开始执行。

Substrate并不强制执行这种回滚行为。然而，这种情况会很少发生，因为交易队列和区块制作逻辑在向区块添加extrinsic之前会执行检查以防止这种情况。

#### 费用倍率(乘法因子)

inclusion fee公式总是对相同的input产生相同的费用。然而，权重可以是动态的，根据`WeightToFee`的定义方式 [Config in pallet_transaction_payment::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_transaction_payment/pallet/trait.Config.html#associatedtype.WeightToFee) ，最终的费用可以包括一定程度的变化。

为了说明这种变化，交易支付pallet提供了`FeeMultiplierUpdate`可配置参数。[Config in pallet_transaction_payment::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_transaction_payment/pallet/trait.Config.html#associatedtype.FeeMultiplierUpdate)


默认的更新功能受到Polkadot网络的启发，实现了有针对性的调整，其中定义了块权重的目标饱和度(saturation level)。如果前一个区块的饱和度较高，那么费用就会略有增加。同样地，如果前一个区块的交易量少于目标值，那么费用就会小幅减少。有关费用乘数调整的更多信息，请参见Web3研究页面。https://research.web3.foundation/en/latest/polkadot/overview/2-token-economics.html#relay-chain-transaction-fees-and-per-block-transaction-limits


### 有特殊要求的交易

inclusion fee在执行前必须是可计算的，因此只能代表固定的逻辑。有些交易需要用其他策略来限制资源。比如说。

- 保证金（Bonds）是一种费用，在链上发生某些事件后可能会被退回或削减。

例如，你可能想要求用户在参与投票时缴纳保证金(bond)。在投票结束后，保证金可能会被返还，或者在投票者试图进行恶意行为时被砍掉。

- 存款(Deposit)是以后可能被退回的费用。

例如，你可能要求用户支付押金（deposit）来执行一个使用存储的操作。如果随后的操作释放了存储，用户的押金就可以被返还。

- 燃烧操作是用来根据一个交易的内部逻辑来支付的。

例如，如果一个交易创建了新的存储项目来支付增加的状态大小，那么该交易可能会从发送方烧掉资金。

- 限制使你能够对某些操作执行恒定或可配置的限制。

例如，默认的Staking pallet只允许提名人提名16个validators，以限制validator选举过程的复杂性。

需要注意的是，如果你查询链上的交易费用，它只返回inclusion fee。

### 默认权重注解

Substrate中的所有dispatchable functions必须指定一个权重。这样做的方法是使用基于注解的系统，该系统可以让你将数据库读/写权重的固定值和/或基于基准的固定值结合起来。最基本的例子是这样的。

```rust
#[pallet::weight(100_000)]
fn my_dispatchable() {
    // ...
}
```

请注意，`ExtrinsicBaseWeight`会自动添加到声明的权重中，以考虑到简单地将一个空的extrinsic纳入一个块中的成本。

#### 权重和数据库读写操作

为了使权重注解独立于所部署的数据库后端，它们被定义为常数，然后在表达由dispatchable程序执行的数据库访问时在注解中使用。

```rust
#[pallet::weight(T::DbWeight::get().reads_writes(1, 2) + 20_000)]
fn my_dispatchable() {
    // ...
}
```

这个dispatchable程序除了做一个数据库的读取和两个数据库的写入之外，还做了其他的事情，增加了可增加的20,000。一般来说，每次访问`#[pallet::storage]`块内声明的值时，就会有一次数据库访问。然而，只有唯一的访问被计算在内，因为一个值被访问后，它被缓存起来，再次访问它不会导致数据库操作。就是说。

- 同一个值的多次读取算作一次读取。
- 同一个值的多次写入算作一次写入。
- 多次读同一个值，然后写这个值，算作一次读和一次写。
- 写入后再读，只算作一次写入。

#### 分发类(Dispacth classes)

分发(dispatch)被分成三类。

- 正常
- 操作性
- 强制性

如果一个分发在权重注释中没有被定义为操作性或强制性，那么该调度默认被识别为正常。你可以指定该dispatchable使用另一个类，比如这样。

```rust
#[pallet::weight((100_000, DispatchClass::Operational))]fn my_dispatchable() {
    // ...
}
```

这个元组符号还允许你指定一个最终参数，决定是否根据注释的权重向用户收费。如果你不另行指定，则假定`Pays::Yes`。

```rust
#[pallet::weight(100_000, DispatchClass::Normal, Pays::No)]
fn my_dispatchable() {
    // ...
}
```

##### 普通分发

这类分发代表了正常的用户触发的交易。这些类型的分发只消耗一个块的总权重限制的一部分。关于普通分发可消耗的块的最大部分的信息，见`AvailableBlockRatio`。[BlockLength in frame_system::limits - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_system/limits/struct.BlockLength.html#method.max_with_normal_ratio) 普通分发被发送到交易池中。[Glossary | Substrate_ Docs](https://docs.substrate.io/reference/glossary/#transaction-pool)

##### 操作性分发

与代表网络能力使用的普通分发不同，操作性分发是那些提供网络能力的分发。操作性分发可以消耗一个区块的全部权重限制。它们不受`AvailableBlockRatio`的约束。这类调度被赋予最大的优先权，并免于支付length fee。

##### 强制分发

强制性分发被包含在一个区块中，即使它们导致区块超过其权重限制。你只能对区块作者提交的inherent 交易 [Glossary | Substrate_ Docs](https://docs.substrate.io/reference/glossary/#inherent-transactions) 使用强制派分发。这个分发类旨在表示作为区块验证过程一部分的功能。因为这些分发总是包含在区块中，无论函数的权重如何，关键是验证过程要防止恶意节点滥用该函数来制作有效但不可能重的区块。你通常可以通过确保以下几点来实现这一点。

- 执行的操作始终是轻的。
- 该操作只能包含在一个区块中一次。

为了使恶意节点更难滥用强制分发，它们不能被包含在返回错误的块中。这个分发类的存在是为了服务于这样的假设：允许创建一个超重的块比不允许创建任何块要好。

#### 动态权重

除了纯粹的固定权重和常数之外，权重的计算还可以考虑dispatchable的输入参数。权重应该是可以通过一些基本的算术从输入参数中计算出来的。

```rust
#[pallet::weight(FunctionOf(
  |args: (&Vec<User>,)| args.0.len().saturating_mul(10_000),
  DispatchClass::Normal,
  Pays::Yes,
))]
fn handle_users(origin, calls: Vec<User>) {
    // Do something per user
}
```


### 分发后的权重校正

根据执行逻辑，一个dispatchable的函数所消耗的权重可能比分发前规定的要少。为了纠正权重，该函数声明一个不同的返回类型并返回其实际重量。

```rust
#[pallet::weight(10_000 + 500_000_000)]
fn expensive_or_cheap(input: u64) -> DispatchResultWithPostInfo {
    let was_heavy = do_calculation(input);

    if (was_heavy) {
        // None means "no correction" from the weight annotation.
        Ok(None.into())
    } else {
        // Return the actual weight consumed.
        Ok(Some(10_000).into())
    }
}
```


### 自定义费用

你还可以通过自定义权重函数或inclusion fee函数来定义自定义收费系统。

#### 自定义权重

你可以使用权重模块 [frame_support::weights - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/weights/index.html) 创建一个自定义的权重计算类型，而不是使用默认的权重注解。自定义权重计算类型必须实现以下trait。  
  
- `WeighData<T>`来确定分发的权重。  
- `ClassifyDispatch<T>`来确定分发的类别。  
- `PaysFee<T>`确定分发的发送者是否支付费用。

然后，Substrate 将这三个trait的输出信息捆绑到 `DispatchInfo` struct中，并通过为所有 Call 变体和不透明的extrinsic类型实现` GetDispatchInfo` 来提供这些信息。这被System和Excutive模块内部使用。  
  
`ClassifyDispatch`、`WeighData` 和 `PaysFee` 在 T 上是通用的，T 被解析为除origin外的所有分发参数的元组。下面的例子说明了一个计算权重的结构，即`m * len(args)`，其中m是一个给定的乘数，`args`是所有分发参数的连接元组。在这个例子中，如果交易的参数长度超过100字节，分发类就会被操作（操作性分发），如果编码的长度大于10字节，就会支付费用。  
  
```rust
struct LenWeight(u32);
impl<T> WeighData<T> for LenWeight {
    fn weigh_data(&self, target: T) -> Weight {
        let multiplier = self.0;
        let encoded_len = target.encode().len() as u32;
        multiplier * encoded_len
    }
}

impl<T> ClassifyDispatch<T> for LenWeight {
    fn classify_dispatch(&self, target: T) -> DispatchClass {
        let encoded_len = target.encode().len() as u32;
        if encoded_len > 100 {
            DispatchClass::Operational
        } else {
            DispatchClass::Normal
        }
    }
}

impl<T> PaysFee<T> {
    fn pays_fee(&self, target: T) -> Pays {
        let encoded_len = target.encode().len() as u32;
        if encoded_len > 10 {
            Pays::Yes
        } else {
            Pays::No
        }
    }
}
```

一个权重计算器函数也可以被胁迫为参数的最终类型，而不是将其定义为一个可以被编码的模糊类型。代码大致会是这样的。

```rust
struct CustomWeight;
impl WeighData<(&u32, &u64)> for CustomWeight {
    fn weigh_data(&self, target: (&u32, &u64)) -> Weight {
        ...
    }
}

// given a dispatch:
#[pallet::call]
impl<T: Config<I>, I: 'static> Pallet<T, I> {
    #[pallet::weight(CustomWeight)]
    fn foo(a: u32, b: u64) { ... }
}
```

在这个例子中，`CustomWeight`只能与具有特定签名`（u32, u64）`的分发一起使用，而`LenWeight`则可以与任何东西一起使用，因为对`<T>`没有任何假设。

#### 自定义inclusion fee

下面的例子说明了如何定制你的inclusion fee。你必须在相应的模块中配置适当的关联类型(associaated type)。

```rust
// Assume this is the balance type
type Balance = u64;

// Assume we want all the weights to have a `100 + 2 * w` conversion to fees
struct CustomWeightToFee;
impl Convert<Weight, Balance> for CustomWeightToFee {
    fn convert(w: Weight) -> Balance {
        let a = Balance::from(100);
        let b = Balance::from(2);
        let w = Balance::from(w);
        a + b * w
    }
}

parameter_types! {
    pub const ExtrinsicBaseWeight: Weight = 10_000_000;
}

impl frame_system::Config for Runtime {
    type ExtrinsicBaseWeight = ExtrinsicBaseWeight;
}

parameter_types! {
    pub const TransactionByteFee: Balance = 10;
}

impl transaction_payment::Config {
    type TransactionByteFee = TransactionByteFee;
    type WeightToFee = CustomWeightToFee;
    type FeeMultiplierUpdate = TargetedFeeAdjustment<TargetBlockFullness>;
}

struct TargetedFeeAdjustment<T>(sp_std::marker::PhantomData<T>);
impl<T: Get<Perquintill>> Convert<Fixed128, Fixed128> for TargetedFeeAdjustment<T> {
    fn convert(multiplier: Fixed128) -> Fixed128 {
        // Don't change anything. Put any fee update info here.
        multiplier
    }
}
```


## 自定义pallet

构建自定runtime最常见的方法是以现有的pallets开始 [FRAME pallets | Substrate_ Docs](https://docs.substrate.io/reference/frame-pallets/) 。例如，你可以开始建立一个特定于应用的staking pallet，使用现有的collective和balances pallet中暴露的类型，但包括你的应用及其staking规则所需的自定义runtime逻辑。

### Pallet宏和属性

FRAME广泛使用Rust宏来封装复杂的代码块。构建自定义pallet的最重要的宏是pallet宏。pallet宏定义了一个pallet必须提供的核心属性集。比如说  
  
- `#[pallet::pallet]`是一个强制性的pallet属性，它使你能够为pallet定义一个结构（struct），这样pallet它就可以存储易于检索的数据。  
- `#[pallet::config]`是一个强制性的Pallet属性，使你能够为Pallet定义configuration trait。  

pallet宏还定义了pallet通常提供的核心属性集。例如。  
  
- `#[pallet::call]` 是使你能够为pallet实现dispatchable函数calls的属性。  
- `#[pallet::error]` 是使你能够产生dispatchable的errors的属性。  
- `#[pallet::event]` 是使你能够产生dispatchable的events的属性。  
- `#[pallet::storage]` 是使你能够在runtime生成一个存储实例及其元数据的属性。  

这些核心属性与你在编写自定义pallet时需要做出的决定一致。例如，你需要考虑。  
  
- 存储。你的pallet要存储什么数据？数据是存储在链上还是链外？  
- 函数。你的pallet暴露的可调用函数是什么？  
- 交易性。你的函数调用是否被设计为原子化地修改存储？  
- 钩子。你的pallet是否会调用任何runtime钩子？  

	宏简化了你为实现自定义runtime逻辑所需编写的代码。然而，有些宏对函数声明有特殊要求。例如，`Config trait`必须被`fram_system::Config` 约束，`#[pallet::pallet]` struct必须被声明为`pub struct Pallet<T>(_)`;。关于FRAME pallets中使用的宏的概述，见FRAME宏。[FRAME macros | Substrate_ Docs](https://docs.substrate.io/reference/frame-macros/)  
  

### 有用的FRAME traits

- Pallet Origin
- Origins: EnsureOrigin, EnsureOneOf ...

### runtime实现

编写一个pallet和为runtime实现它是相辅相成的。你的pallet的`Config trait`是为runtime实现的，而runtime是一个特殊的结构，用于编译`construct_runtime`宏中的所有实现的pallets。

- `parameter_types` [parameter_types in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/macro.parameter_types.html) 和`ord_parameter_types` [ord_parameter_types in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/macro.ord_parameter_types.html) 宏对于向可配置的pallet常量传递数值非常有用。
- \[ 其他考虑因素，如no_std \]
- 最小化的runtime引用
- 侧链架构引用
- Api端点：oninitialize, offchain workers ?


## Pallets之间耦合(coupling)

耦合一词经常被用来描述两个软件模块相互依赖的程度。例如，在面向对象的编程中，紧耦合和松耦合被用来描述对象类之间的关系。

- 紧密耦合是指两组类之间相互依赖。
- 松散耦合是指一个类使用另一个类所暴露的接口。

在Substrate中，pallet紧耦合和pallet松耦合被用来描述一个托盘如何调用另一个托盘的函数。这两种技术以不同的方式实现同样的事情，各自有一定的权衡。相当于有松耦合调用和紧耦合调用。

### 紧耦合pallets

因为紧耦合使pallets的工作不那么灵活和可扩展，所以只有当一个pallet需要整体继承其耦合的对应物而不是特定的类型或方法时，你才会使用pallet紧耦合。

当编写一个需要紧耦合的pallet时，你要明确地指定pallet的`Config trait`被要耦合的pallet的配`Config trait`所约束。

所有FRAME pallets都与`frame_system` pallet紧密耦合。下面的例子说明了如何使用名为`some_pallet`的pallet的`Config trait`来与`frame_system` pallet紧密耦合。

```rust
pub trait Config: frame_system::Config + some_pallet::Config {
    // --snip--
}
```

这与面向对象编程中使用类的继承非常相似。提供这个trait bound意味着这个pallet只能安装在同时安装了`some_pallet` pallet的runtime中。与`frame_system`类似，这个例子中的紧耦合要求你在耦合的pallet的Cargo.toml文件中指定`some_pallet`。

紧密耦合有几个缺点，开发者应该考虑到。

- 可维护性：一个pallet的改变往往会导致需要修改另一个pallet。
- 可重用性：两个模块都必须包含其中一个才能使用，这使得紧密耦合的pallets更难集成。

### 松耦合pallets

在松散的pallet耦合中，你可以指定某些类型需要被绑定的trait和函数接口。

这些类型的实际实现是在runtime配置中在pallet之外进行的--通常是在`/runtime/src/lib.rs`文件中定义的代码。通过松散耦合，你可以使用另一个已经实现了trait的pallet中的类型和接口，或者你可以声明一个全新的结构，实现这些trait，并在runtime实现pallet时将其分配。

举个例子，假设你有一个可以访问账户余额并向另一个账户转账的pallet。这个pallet定义了一个 "`Currency`" trait，它有一个抽象的函数接口，以后会实现实际的转账逻辑。

```rust
pub trait Currency<AccountId> {
    // -- snip --
    fn transfer(
        source: &AccountId,
        dest: &AccountId,
        value: Self::Balance,
        // don't worry about the last parameter for now
        existence_requirement: ExistenceRequirement,
    ) -> DispatchResult;
}
```

在第二个pallet中，你定义了`MyCurrency`的关联类型，并通过`Currency<Self::AccountId>`trait将其绑定，这样你就可以通过调用`T::MyCurrency::transfer(...)`来使用余额转移逻辑。

```rust
pub trait Config: frame_system::Config {
    type MyCurrency: Currency<Self::AccountId>;
}

impl<T: Config> Pallet<T> {
    pub fn my_function() {
        T::MyCurrency::transfer(&buyer, &seller, price, ExistenceRequirement::KeepAlive)?;
    }
}
```

注意，在这一点上，你还没有指定如何实现`Currency::transfer()`逻辑。只是约定了它将在某处实现。

现在，你可以使用runtime配置-`runtime/src/lib.rs`来实现pallet并指定类型为`Balances`。

```rust
impl my_pallet::Config for Runtime {
    type MyCurrency = Balances;
}
```

`Balances`类型在`construct_runtime！`宏中被指定为实现`Currency trait` [pallet_balances - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_balances/index.html#implementations-1)的`pallet_balances` [pallet_balances - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_balances/index.html)的一部分。

通过runtime提供的实现，你可以在你松散耦合的pallet中使用`Currency<AccountId>` trait。

许多FRAME pallets都以这种方式耦合到这个`Currency trait`。

### 选择一个pallet的耦合策略

一般来说，松耦合比紧耦合提供更多的灵活性，从系统设计的角度来看，松耦合被认为是一种更好的做法。它能保证你的代码有更好的可维护性、可重用性和可扩展性。然而，紧耦合对于那些不那么复杂的pallet或者方法和类型的重叠多于差异的pallet来说是有用的。

在FRAME中，有两个pallets与`pallet_treasury` [substrate/frame/treasury at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/treasury) 紧密耦合。

- 赏金pallet [substrate/frame/bounties at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/bounties)
- 小费pallet [substrate/frame/tips at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/tips)

一般来说，一个pallet越复杂，就越不应该将其紧密耦合。这让人想起计算机科学中的一个概念--内聚性 [Cohesion (computer science) - Wikipedia](https://en.wikipedia.org/wiki/Cohesion_(computer_science)) ，这是一个用来考察软件系统整体质量的指标。


### How-to

- [Use loose pallet coupling | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/pallet-design/use-loose-coupling/)
- [Use tight pallet coupling | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/pallet-design/use-tight-coupling/)

## 事件和错误

当一个pallet想要向外部实体如用户、区块链浏览器或dApps发送关于runtime的变化或条件的通知时，它可以emit事件。

在自定义pallet中，你可以定义。

- 你想发射什么类型的事件
- 这些事件中包含哪些信息
- 这些事件何时被发射出来

### 声明一个事件

runtime事件是使用`#[pallet::event]`宏创建的。比如说。

```rust
#[pallet::event]
#[pallet::metadata(u32 = "Metadata")]
pub enum Event<T: Config> {
	/// Set a value.
	ValueSet(u32, T::AccountId),
}
```

`Event` enum需要在你的runtime的`Config` trait中声明。

```rust
#[pallet::config]
	pub trait Config: frame_system::Config {
		/// The overarching event type.
		type Event: From<Event<Self>> + IsType<<Self as frame_system::Config>::Event>;
	}
```


### 向你的runtime暴露这些事件

你在pallet中定义的任何事件必须在`/runtime/src/lib.rs` 文件中暴露给runtime。

要将事件暴露给runtime。

- 在一个文本编辑器中打开`/runtime/src/lib.rs` 文件。
- 在你的pallet的`Config` trait中实现`Event`类型。

```rust
impl template::Config for Runtime {
	 type Event = Event;
}
```

- 添加`Event`类型到 `construct_runtime!`宏中

```rust
construct_runtime!(
	 pub enum Runtime where
 	 Block = Block,
	   NodeBlock = opaque::Block,
	   UncheckedExtrinsic = UncheckedExtrinsic
	 {
    // --snip--
	   TemplateModule: template::{Pallet, Call, Storage, Event<T>},
	   //--add-this------------------------------------->
		 }
);
```

在这个例子中，事件是一个泛型，需要`<T>`参数。如果你的事件不使用泛型，就不需要`<T>`参数。

### 存放一个事件

Substrate提供了一个关于如何使用宏来存放事件的默认实现。存放一个事件的结构如下。

```rust
// 1. Use the `generate_deposit` attribute when declaring the Events enum.
#[pallet::event]
	#[pallet::generate_deposit(pub(super) fn deposit_event)] // <------ here ----
	#[pallet::metadata(...)]
	pub enum Event<T: Config> {
		// --snip--
	}

// 2. Use `deposit_event` inside the dispatchable function
#[pallet::call]
	impl<T: Config> Pallet<T> {
		#[pallet::weight(1_000)]
		pub(super) fn set_value(
			origin: OriginFor<T>,
			value: u64,
		) -> DispatchResultWithPostInfo {
			let sender = ensure_signed(origin)?;
			// --snip--
			Self::deposit_event(RawEvent::ValueSet(value, sender));
		}
	}
```

这个函数的默认行为是在FRAME system中调用`deposit_event` [Pallet in frame_system::pallet - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_system/pallet/struct.Pallet.html#method.deposit_event) ，它将事件写入存储。

这个函数将事件放在system pallet的runtime存储中，用于该块。在一个新的区块开始时，system pallet会自动删除所有从上一个区块存储的事件。

使用默认实现存入的事件可以直接被下游库支持，比如Polkadot-JS API [polkadot-js/api: Promise and RxJS APIs around Polkadot and Substrate based chains via RPC calls. It is dynamically generated based on what the Substrate runtime provides in terms of metadata. Full documentation & examples available (github.com)](https://github.com/polkadot-js/api)。然而，如果你想以不同的方式处理事件，你可以实现你自己的 `deposit_event` 函数。

### 支持的类型

事件可以发射任何支持使用SCALE编解码器 [Type encoding (SCALE) | Substrate_ Docs](https://docs.substrate.io/reference/scale-codec/) 进行类型编码的类型。

在你想使用runtime 泛型的情况下，如`AccountId`或`Balances`，你需要包括一个where子句 [Where clauses - Rust By Example (rust-lang.org)](https://doc.rust-lang.org/rust-by-example/generics/where.html) 来定义这些类型，如上面的例子所示。

### 监听事件

Substrate RPC并没有直接暴露查询事件的endpoints。如果你使用默认的实现，你可以通过查询System pallet的存储来查看当前块的事件列表。否则，Polkadot-JS API支持对runtime事件的WebSocket订阅。[polkadot-js/api: Promise and RxJS APIs around Polkadot and Substrate based chains via RPC calls. It is dynamically generated based on what the Substrate runtime provides in terms of metadata. Full documentation & examples available (github.com)](https://github.com/polkadot-js/api)

### 错误

runtime代码应该明确地、优雅地处理所有错误情况。runtime代码中的函数必须是不抛出(non-throwing)的函数，永远不会引起编译器的panic [To panic! or Not to panic! - The Rust Programming Language (rust-lang.org)](https://doc.rust-lang.org/book/ch09-03-to-panic-or-not-to-panic.html) 。编写non-throwing的Rust代码的一个常见习惯是编写返回Result类型[Result in frame_support::dispatch::result - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/dispatch/result/enum.Result.html) 的函数。`Result` enum类型拥有一个`Err`变量，它允许一个函数在不需要panic的情况下表示它未能成功执行。在FRAME开发环境中，可以被派发到runtime的函数调用必须返回一个`DispatchResult`类型 [DispatchResult in frame_support::dispatch - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/dispatch/type.DispatchResult.html) ，这个类型表示，如果函数遇到了错误，该类型可能是`DispatchError`变量。[DispatchError in frame_support::dispatch - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/dispatch/enum.DispatchError.html)

每个FRAME pallet可以通过使用`#[pallet::error]`宏定义一个自定义的`DispatchError`。例如。

```rust
#[pallet::error]
pub enum Error<T> {
		/// Error names should be descriptive.
		InvalidParameter,
		/// Errors should have helpful documentation associated with them.
		OutOfSpace,
	}
```

FRAME support模块还包括一个有用的 `ensure!` 宏 [ensure in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/macro.ensure.html) ，可以用来检查预设条件，如果不符合，就会发出错误。

```rust
frame_support::ensure!(param < T::MaxVal::get(), Error::<T>::InvalidParameter);
```

## 随机

随机性在计算机科学和其他领域有许多应用。例如，它被用于在一些治理系统中选择理事，进行统计或科学分析，进行加密操作，以及博彩和赌博。许多需要随机性的应用已经在区块链网络上找到了应用指出。这篇文章描述了随机性是如何在Substrate runtime中产生和使用的。

### 确定性随机

在传统的非区块链计算中，需要随机性的应用程序可以选择使用从硬件中提取的真实随机值，或者使用实际上是确定的、但由于加密技术而无法预测的伪随机值。

在区块链上运行的应用程序受到更严格的限制，因为网络中的所有当局必须同意任何链上的值，包括注入的任何随机性数据。由于这种约束，直接使用真正的随机性是不可能的。

幸运的是，像可验证随机函数(VRF) https://en.wikipedia.org/wiki/Verifiable_random_function 这样的密码学基础的进步和多方随机性计算的发展，使得需要随机性的应用仍然可以在区块链上使用。

### substrate的随机trait

Substrate提供了一个名为`Randomness`的trait https://paritytech.github.io/substrate/master/frame_support/traits/trait.Randomness.html ，它编码了生成随机性的逻辑和消耗随机性的逻辑之间的接口。这个trait允许这两块逻辑被独立地实现。

#### 消费随机性

开发者在编写一个需要随机性的pallet时，不需要担心提供随机性的问题。相反，pallet可以简单地要求一个实现该trait的随机性源。randomness trait提供了两种获得随机性的方法。

第一个方法叫做`random_seed`。它不需要任何参数，并给出一个原始的随机性。在一个块中多次调用这个方法，每次都会返回相同的值。因此，不建议直接使用这个方法。

第二个方法叫做`random`。它接收一个作为上下文标识符的字节数，并返回一个对该上下文来说独一无二的结果，并且在底层随机性源允许的范围内独立于其他上下文。

#### 生成随机性

有许多不同的方法来实现`Randomness` trait，每一种方法都代表了性能、复杂性和安全性之间的不同权衡。Substrate提供了两种实现，如果开发者想做出不同的权衡，他们可以提供自己的实现。

Substrate提供的第一个实现是随机性集体翻转pallet [pallet_randomness_collective_flip - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_randomness_collective_flip/index.html) 。这个托盘是基于集体掷硬币的。它具有相当高的性能，但不是很安全。这个pallet应该只在测试消耗随机性的pallet时使用，而不是在生产中使用。

第二个实现是`BABE pallet` [pallet_babe - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_babe/index.html) ，它使用可验证的随机函数(VRF)。这个pallet提供了生产级的随机性，并在Polkadot中使用。选择这个随机性源决定了你的区块链使用Babe共识。

### 安全属性

`Randomness` trait为Substrate runtime中的随机性源提供了一个方便而有用的抽象。但该特性本身并没有做出任何安全保证。runtime开发者必须确保所使用的随机性源满足所有消耗其随机性的pallet的安全要求。

### 其他

- [Incorporate randomness | Substrate_ Docs](https://docs.substrate.io/reference/how-to-guides/pallet-design/incorporate-randomness/)


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

runtime origin被可分发的函数用来检查一个调用来自哪里。

### 原始 origins

Substrate定义了三个原始origins，可以在你的runtime pallet中使用。

```rust
pub enum RawOrigin<AccountId> {
	Root,
	Signed(AccountId),
	None,
}
```


- Root。一个系统级的origin。这是最高的权限级别，可以被认为是runtime origin的超级用户。
- 签名的。一个交易origin。这是由一些链上账户的私钥签署的，包括签署者的账户ID。这使得runtime可以验证dispatch的来源，并随后向相关账户收取交易费用。
- None。缺乏Origin。这需要由validator同意或由模块验证后才能包括。这种origin类型的性质更复杂，因为它被设计为绕过某些runtime机制。这种origin类型的一个使用案例是允许validator直接将数据插入到一个块中。

### Origin调用

你可以在你的runtime中用任何origin构建calls。比如说。

```rust
// Root
proposal.dispatch(system::RawOrigin::Root.into())

// Signed
proposal.dispatch(system::RawOrigin::Signed(who).into())

// None
proposal.dispatch(system::RawOrigin::None.into())
```

你可以看一下Sudo模块的源代码 [pallet_sudo - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/pallet_sudo/index.html) ，了解它的实际实现。


### 自定义Origins

除了三种核心origin类型外，runtime开发人员还可以定义自定义origin。这些可以被用作runtime特定模块函数内的授权检查，或围绕runtime请求的origin定义自定义访问控制逻辑。

自定义origin允许runtime开发人员根据他们的runtime逻辑指定有效的origin。例如，可能需要将某些函数的访问限制在特殊的自定义origin上，并且只授权从一个集体（collective）[substrate/frame/collective at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/collective) 的成员中进行dispatch调用。使用自定义origin的好处是，它为runtime开发人员提供了一种方法，以配置对runtime的dispatch call的特权访问。

### 下一步

#### 学习更多的

- 学习关于origin怎样在`#[pallet::call]`宏中被使用 [pallet in frame_support - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_support/attr.pallet.html#call-palletcall-optional)

#### 例子

- 查看`Sudo pallet`，看看它是如何允许用户用Root和Signed origin调用的。[substrate/frame/sudo at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/sudo)
- 查看`Timestamp pallet`，看看它是如何验证一个带有None origin的调用。[substrate/frame/timestamp at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/timestamp)
- 查看`Collective pallet`，看看它是如何构建一个自定义的Member origin。[substrate/frame/collective at master · paritytech/substrate (github.com)](https://github.com/paritytech/substrate/tree/master/frame/collective)
- 查看我们关于创建和使用自定义origin的配方。

#### 参考

- [RawOrigin in frame_system - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/frame_system/enum.RawOrigin.html)


## RPCs

## 应用开发

在substrate区块链上运行的大多数应用程序需要某种形式的前端或面向用户的界面--如浏览器、桌面、移动或硬件客户端--使用户或其他程序能够访问和修改区块链存储的数据。例如，你可以开发一个基于浏览器的应用程序，用于互动游戏，或者开发一个特定的硬件应用程序来实现硬件钱包。根据你的需要，存在不同的库来构建这些类型的应用程序。本文解释了查询Substrate节点和使用其暴露的metadata的过程，以帮助你了解在创建前端客户端应用程序和使用客户端特定库时如何使用metadata。

### Metadata系统

substrate节点提供一个RPC调用，`state_getMetadata`，它返回当前runtime中所有类型的完整描述。客户端应用程序使用元数据与节点交互，解析响应，并格式化发送到节点的消息的payload。这个元数据包括关于一个pallet的存储项、交易、事件、错误和常量的信息。当前的元数据版本（V14）与之前的版本有很大不同，因为它包含了更丰富的类型信息。如果一个runtime包括一个具有自定义类型的pallet，那么类型信息将作为元数据的一部分返回。Polkadot使用V14元数据，从runtime spec版本9110 [Polkascan · Explorer](https://explorer.polkascan.io/polkadot/runtime/9110) 的块号7229126开始[Subscan | Polkadot Block Detail: 7229126](https://polkadot.subscan.io/block/7229126) ，Kusama从runtime spec版本9111 [Polkascan · Explorer](https://explorer.polkascan.io/kusama/runtime/9111) 的块号9625129开始 [Subscan | Kusama Block Detail: 9625129](https://kusama.subscan.io/block/9625129) 。这对于打算与使用旧版元数据的runtime进行交互的开发者来说是非常有用的。请参考这份文件，了解从V13到V14的迁移指南。  [Migration guide for Substrate Metadata V14 (github.com)](https://gist.github.com/ascjones/0d81a4c44e84cacd9f714cd34a6de823)

当前的元数据模式使用`scale-info` crate [scale_info - Rust (docs.rs)](https://docs.rs/scale-info/latest/scale_info/) 来获取runtime中的pallet的类型信息，当你编译一个节点时。  
  
目前的元数据实现需要前端API使用SCALE编解码库 [Type encoding (SCALE) | Substrate_ Docs](https://docs.substrate.io/reference/scale-codec/) 来编码和解码RPC payload以发送和接收交易。下面的步骤总结了元数据是如何生成、暴露以及用于从runtiime进行和接收调用(calls)。  

- 可调用的pallet函数，以及类型、参数和文档都由runtime公开暴露出来。
- `frame-metadata` crate描述了关于如何与runtime通信的信息的结构。这些信息的形式是由`scale-info`提供的类型注册表，以及关于哪些pallet存在的信息（以及注册表中每个pallet的相关类型）。
- `scale-info` crate被用来标注整个runtime的类型，并使得建立一个runtime类型注册表成为可能。这些类型信息足够详细，我们可以用它来找出如何正确地对给定类型进行SCALE编码或解码的一些值。
- `frame-metadata`中描述的结构被填充了来自runtime的信息，然后这被SCALE编码并通过`state_getMetadata` RPC调用提供。
- 自定义RPC APIs使用元数据接口，并提供方法来调用runtime。需要一个SCALE编解码库来编码和解码进出API的调用和数据。

每个substrate链都存储了它们所使用的metadata系统的版本号，这使得应用程序知道如何处理某个区块所暴露的元数据变得非常有用。如前所述，最新的元数据版本（V14）为一个链能够生成的元数据提供了一个重要的增强。但是，如果一个应用程序想要与用比V14更早的版本创建的区块进行交互呢？那么，这将需要设置一个遵循旧的metadata系统的前端界面，据此，自定义类型将需要被识别，并作为前端代码的一部分手动包含。如果你需要的话，可以学习如何使用desub工具 [paritytech/desub: Decode Substrate with Backwards-Compatible Metadata (github.com)](https://github.com/paritytech/desub) 来完成这个任务。  
  
捆绑在元数据中的类型信息使应用程序有能力与不同链上的节点进行通信，每个链上的节点可能各自暴露不同的调用、事件、类型和存储。它还允许库(libraries)生成与给定的Substrate节点通信所需的几乎所有代码，使`subxt`等库有可能生成特定于目标链的前端接口。  [paritytech/subxt: Submit extrinsics (transactions) to a substrate node via RPC (github.com)](https://github.com/paritytech/subxt)
  
有了这个系统，任何runtime都可以被查询到其可用的runtime调用、类型和参数。  
元数据还暴露了一个类型将如何被解码，使外部应用程序更容易检索和处理这些信息。  
  
### Metadata格式

查询`state_getMetadata` RPC函数将返回一个SCALE编码的字节向量，该向量使用`frame-metadata`和`parity-scale-codec`库进行解码。

由`state_getMetadata` RPC返回的hex blob取决于元数据的版本，但是通常会有以下结构。

- 一个硬编码的magic number，0x6d657461，它代表纯文本中的 "meta"。
- 一个32位的整数，代表正在使用的元数据格式的版本，例如`14`或十六进制的`0x0e`。
- 十六进制编码的类型和元数据信息。在V14中，这部分包含一个类型信息的注册表（由`scale-info` crate生成）。在以前的版本中，这部分包含pallet的数量，然后是每个pallet暴露的元数据。

下面是一个使用V14 metadata系统的runtime的元数据解码的浓缩版本（使用subxt生成 [subxt | Substrate_ Docs](https://docs.substrate.io/reference/command-line-tools/subxt/) ）。

```json
[
  1635018093, // the magic number
  {
    "V14": {
      // the metadata version
      "types": {
        // type information
        "types": []
      },
      "pallets": [
        // metadata exposes by pallets
      ],
      "extrinsic": {
        // the format of an extrinsic  and its signed extensions
        "ty": 111,
        "version": 4, // the transaction version used to encode and decode an extrinsic
        "signed_extensions": []
      },
      "ty": 125 // the type ID for the system pallet
    }
  }
]
```

如上所述，整数`1635018093`是一个 "magic number"，用纯文本表示 "meta"。元数据的其余部分有两个部分：pallets和extrinsic。pallets部分包含关于runtime的pallets的信息，而extrinsic部分描述了runtime使用的extrinsics的版本。不同的extrinsic版本可能有不同的格式，特别是在考虑签名交易时。[[Substrate学习笔记#已签名交易]]

#### Pallets

这里有一个在metadata返回的json中一个浓缩版的pallets数组的单个元素的样例:

```json
{
  "name": "System", // name of the pallet, the System pallet for example
  "storage": {
    // storage entries
  },
  "calls": [
    // index for this pallet's call types
  ],
  "event": [
    // index for this pallet's event types
  ],
  "constants": [
    // pallet constants
  ],
  "error": [
    // index for this pallet's error types
  ],
  "index": 0 // the index of the pallet in the runtime
}
```

每个元素都包含它所代表的pallet的名称，以及一个存储对象、调用数组、事件数组和错误数组。如果调用或事件是空的，它们将被表示为空，如果常量或错误是空的，它们将被表示为一个空数组。

数组中每个元素的类型索引只是u32整数，用于访问该项目的类型信息。例如，`system pallet`中的调用的类型ID是145。查询类型ID将给你提供system pallet的可用调用信息，包括每个调用的文档。对于每个字段，你可以访问以下的类型信息和元数据。

- 存储元数据 [PalletStorageMetadata in frame_metadata::v14 - Rust (docs.rs)](https://docs.rs/frame-metadata/latest/frame_metadata/v14/struct.PalletStorageMetadata.html) ：为区块链客户端提供查询storage RPC [StateApiServer in sc_rpc::state - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_rpc/state/trait.StateApiServer.html#tymethod.storage) 以获得特定存储项的信息所需的信息。
- 调用元数据 [PalletCallMetadata in frame_metadata::v14 - Rust (docs.rs)](https://docs.rs/frame-metadata/latest/frame_metadata/v14/struct.PalletCallMetadata.html) ：包括`#[pallet]`宏所定义的runtime调用的信息，包括调用名称、参数和文档。
- 事件元数据 [PalletEventMetadata in frame_metadata::v14 - Rust (docs.rs)](https://docs.rs/frame-metadata/latest/frame_metadata/v14/struct.PalletEventMetadata.html) ：提供`#[pallet::event]`宏生成的元数据，包括pallet事件的名称、参数和文档。
- 常量元数据：提供`#[pallet::constant]`宏生成的元数据，包括常量的名称、类型和十六进制编码的值。
- 错误元数据 [PalletErrorMetadata in frame_metadata::v14 - Rust (docs.rs)](https://docs.rs/frame-metadata/latest/frame_metadata/v14/struct.PalletErrorMetadata.html) ：提供由`#[pallet::error]`宏生成的元数据，包括该pallet中每个错误类型的名称和文档。

请注意，所使用的IDs在一段时间内并不稳定：它们很可能会从一个版本跳转到下一个版本，这意味着开发者应该避免依赖固定的类型ID来证明他们的应用程序的未来。（也就是说这个ID不固定，未来会变）

#### Extrinsic

Extrinsic metadata [ExtrinsicMetadata in frame_metadata::v14 - Rust (docs.rs)](https://docs.rs/frame-metadata/latest/frame_metadata/v14/struct.ExtrinsicMetadata.html) 是由runtime产生的，并提供了关于交易如何格式化的有用信息。返回的解码元数据包含交易版本和签名扩展，看起来像这样。

```json
      "extrinsic": {
        "ty": 111,
        "version": 4,
        "signed_extensions": [
          {
            "identifier": "CheckSpecVersion",
            "ty": 117,
            "additional_signed": 4
          },
          {
            "identifier": "CheckTxVersion",
            "ty": 118,
            "additional_signed": 4
          },
          {
            "identifier": "CheckGenesis",
            "ty": 119,
            "additional_signed": 9
          },
          {
            "identifier": "CheckMortality",
            "ty": 120,
            "additional_signed": 9
          },
          {
            "identifier": "CheckNonce",
            "ty": 122,
            "additional_signed": 34
          },
          {
            "identifier": "CheckWeight",
            "ty": 123,
            "additional_signed": 34
          },
          {
            "identifier": "ChargeTransactionPayment",
            "ty": 124,
            "additional_signed": 34
          }
        ]
      }
```

类型系统是复合的，这意味着每个类型的ID都包含对某些类型的引用，或者对另一个类型的ID的引用，从而可以访问相关的原始类型。例如，我们可以编码的一个类型是`BitVec<Order, Store>`类型：为了正确解码，我们需要知道使用的`Order`和`Store`类型是什么，这可以使用解码后的JSON中的 "path "来访问该类型ID。

### RPC APIs

Substrate带有以下API来与节点进行交互。

- AuthorApi [AuthorApiServer in sc_rpc::author - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_rpc/author/trait.AuthorApiServer.html) 。一个用于对全节点(full node)进行调用的API，包括编写extrinsic和验证会话密钥。
- ChainApi [ChainApiServer in sc_rpc::chain - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_rpc/chain/trait.ChainApiServer.html)。一个用于检索块头和区块finality信息的API。
- OffchainApi [OffchainApiServer in sc_rpc::offchain - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_rpc/offchain/trait.OffchainApiServer.html) 。一个为offchain worker进行RPC调用的API。
- StateApi [StateApiServer in sc_rpc::state - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_rpc/state/trait.StateApiServer.html) 。用于查询链上状态信息的API，如runtime版本、存储项和证明。
- SystemApi [SystemApiServer in sc_rpc::system - Rust (paritytech.github.io)](https://paritytech.github.io/substrate/master/sc_rpc/system/trait.SystemApiServer.html) : 检索网络状态信息的API，如连接的P2P对端Peers和节点角色。

### 连接到一个节点

查询一个Substrate节点可以通过使用超文本传输协议（HTTP）或基于WebSocket（WS）的JSON-RPC客户端来完成。WS（在大多数应用中使用）的主要优点是，一个单一的连接可以重复用于许多进出节点的消息(全双工通信)，而典型的HTTP连接一次只允许来自客户端的单一消息，然后响应。出于这个原因，如果你想订阅一些可能导致多个消息返回给客户端的RPC端点，你必须使用websocket连接而不是HTTP连接。通过HTTP连接通常用于在offchain woker中获取数据--在链下操作中了解更多信息。[[Substrate学习笔记#链下操作]]

连接到Substrate节点的另一种方式（仍在试验中）是使用`Substrate Connect`，它允许应用程序生成自己的轻型客户端并直接连接到暴露的JSON-RPC端点。这些应用程序将依靠浏览器内的本地内存来建立与轻型客户端的连接。

### 开始构建

Parity维护以下建立在JSON-RPC API之上的库，用于与Substrate节点进行交互。

- subxt [paritytech/subxt: Submit extrinsics (transactions) to a substrate node via RPC (github.com)](https://github.com/paritytech/subxt) 提供了一种为特定链构建的静态前端创建接口的方法。
Polkadot JS API [polkadot{.js}](https://polkadot.js.org/) 提供了一个库，为任何Substrate构建的区块链构建动态接口。
Substrate Connect [paritytech/substrate-connect: Run Wasm Light Clients of any Substrate based chain directly in your browser. (github.com)](https://github.com/paritytech/substrate-connect) 提供了一个库和一个浏览器扩展，用于构建直接与为其目标链创建的浏览器内轻客户端连接的应用程序。作为一个使用Polkadot JS API的库，Connect对于需要连接到多个链的应用程序非常有用，在同一应用程序与多个链互动时，为终端用户提供单一一致的体验。

### 前端使用场景

![[d922567170347a412bd7dcefec40fd7.png]]

## 构建一个确定性的runtime

TODO

## 升级Runtime

TODO