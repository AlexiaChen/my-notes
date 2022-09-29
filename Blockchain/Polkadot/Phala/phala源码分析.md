# pRuntime节点源码分析

## 概览

从cargo的依赖上看，pruntime就是主要由rust的rocket web框架开发出来的RPC服务器。区块链
节点通过protobuf协议序列化消息与pruntime通信。

其实中`pruntime` 项目中的`main.rs`中主要就是解析命令参数并且启动基于RPC的API server。

其中API的名称如下:

```rust
match method {
        GetInfo => 1.kibibytes(),
        SyncHeader => 100.mebibytes(),
        SyncParaHeader => 100.mebibytes(),
        SyncCombinedHeaders => 100.mebibytes(),
        DispatchBlocks => 100.mebibytes(),
        InitRuntime => 10.mebibytes(),
        GetRuntimeInfo => 1.kibibytes(),
        GetEgressMessages => 1.kibibytes(),
        ContractQuery => 500.kibibytes(),
        GetWorkerState => 1.kibibytes(),
        AddEndpoint => 10.kibibytes(),
        RefreshEndpointSigningTime => 10.kibibytes(),
        GetEndpointInfo => 1.kibibytes(),
        DerivePhalaI2pKey => 10.kibibytes(),
        Echo => 1.mebibytes(),
        HandoverCreateChallenge => 10.kibibytes(),
        HandoverStart => 10.kibibytes(),
        HandoverAcceptChallenge => 10.kibibytes(),
        HandoverReceive => 10.kibibytes(),
        SignEndpointInfo => 32.kibibytes(),
        ConfigNetwork => 10.kibibytes(),
        HttpFetch => 100.mebibytes(),
}
```

如果启动本地开发环境，什么也不操作，自然可以看到区块链的节点在不停的运行，高度增长，同时pruntime节点的日志会看到不断地接收者几个RPC，也就是以下API随着链的运行，被不停地被调用:

```txt
rocket::server] POST /prpc/PhactoryAPI.GetInfo:
rocket::server] POST /prpc/PhactoryAPI.GetEndpointInfo:
rocket::server] POST /prpc/PhactoryAPI.GetEgressMessages:
rocket::server] POST /prpc/PhactoryAPI.GetInfo:
rocket::server] POST /prpc/PhactoryAPI.SyncHeader:
rocket::server] POST /prpc/PhactoryAPI.DispatchBlocks:

.... // repeat pattern in ordered

rocket::server] POST /prpc/PhactoryAPI.GetInfo:
rocket::server] POST /prpc/PhactoryAPI.GetEndpointInfo:
rocket::server] POST /prpc/PhactoryAPI.GetEgressMessages:
rocket::server] POST /prpc/PhactoryAPI.GetInfo:
rocket::server] POST /prpc/PhactoryAPI.SyncHeader:
rocket::server] POST /prpc/PhactoryAPI.DispatchBlocks:
```

其中RPC/API的业务实现在crates目录下的phactory的crate中。API的入口在`bin_api_service.rs`中，具体业务实现，api service会调用 `prpc_service.rs`里面的函数。这些API涉及的一些元数据，比如runtime state是从`lib.rs`那个结构体的字段来的。 比如`runtime_state`这个字段，这些元数据，大部分都是从`deserialize`函数调用的`load_state`里面来的，其用vistor pattern读取了一个sequence，应该sequence就是反序列化出来的数据，这个sequence类似链表，不停地被回调，读取next element。这些元数据都在一个叫`Phactory<Platform>`的结构体中。这个结构体可被serde序列化和反序列化。

## RPC业务实现

在分析业务之前，需要分析`Phactory<Platform>`这个结构体的整体设计，它到底是干什么的。

### Phactory\<Platform\>结构体

这个结构体里面就表达了这个phactory crate所有需要的平台特定的信息。下面只讲解大概几几个主要字段:

```rust
platform---是平台相关的
args----是初始化这个应用的参数，包括pRuntime的持久化路径，GateKeeper的主密钥加密保存路径，版本，git的commit id，公有端口，运行合约的CPU核心数量，检查点间隔（这个字段也可序列化和反序列化）
dev_mode-----是否是开发模式运行，在init_rumtime里面设置
machine_id----这个暂时没有开出来哪里设置的，机器ID，猜想与CPU ID会绑定之类的
runtime_info---里面有运行时的基本信息，创世块hash(中继链的header？)，worker的公钥，ECDH公钥，还有XGS attestation证明。也是在init_time里面被设置的，作为一个worker的注册信息被设置
runtime_state----RuntimeState这个结构体，里面有消息发送和消息接收队列，存储同步器，创世块hash
。这个runtime_state也是在init_runtime里面被设置。创世块hash是RPC的caller传递过来的，其他都是根据传递过来的RPC参数进行默认或者判别式的初始化
side_task_man----是一个边任务管理器，这个管理器的实现就是一个Task的Vec
system---平台特定的系统数据，这个System是一个大结构体，里面很多字段，暂时跳过。里面有各种事件和channel。这个字段在init_runtime和load_state里面被设置
handover_ecdh_key---- 交接的ECDH协商的秘钥
last_checkpoint----最近的检查点，创建检查点文件(take_checkpoint)的时候被设置,检查点文件名字是根据当前区块高度，真正的检查点其实是吧，Phactory<Platform>整个结构体的可序列化字段全部dump到文件里面。last_checkpoint被更新为当前时间。检查点之间都有固定的间隔。更新检查点和保存检查点是由RPC PhactoryAPI.DispatchBlocks驱动的
query_scheduler-----查询调度器，主要是针对合约的查询请求，因为pRuntime是运行合约的地方。针对某个特定合约ID的查询，合约ID是一个256位的hash值。是由RPC hactoryAPI.ContractQuery调用的来查询的
netconfig-----网络配置，除了new出来的默认的，就是由PhactoryAPI.ConfigNetwork RPC主动设置的
```

init_runtime这个函数是`PhactoryAPI.InitRuntime`这个RPC内部调用的，这个RPC就是pRuntime节点的RPC了。

再说一下检查点，从检查点中恢复Phactory\<Platform\>这个结构体（函数 `restore_from_checkpoint`），是pRuntime节点 main函数调用的，也就是初始化pRuntime的时候会尝试从检查点恢复最近的结构体状态。

### get_info

这个RPC就是获取当前pRuntime节点的一些状态，比如初始化参数：git commit id, 是否是开发模式，是否初始化，平台运行中的内存使用情况，runtime_state这个运行时状态，未处理的pending消息。因为这个RPC获取的参数主要是来自于runtime_state和system这个两个字段，所以我们需要重点了解下这两个结构体的具体字段:


- RuntimeState结构体

```txt
send_mq---在init_runtime中默认初始化，目的是向链上发送的消息buffer，发一个删除一个消息。内部实现是一个BTreeMap<SenderId,Channel> checkpoint中都恢复这个状态
recv_mq---在init_runtime中默认初始化,BTreeMap<Path,Vec<Sender<Message>>>的订阅者结构。checkpoint中会恢复这个状态。DispatchBlock RPC也会触发这个recv_mq的方法，设置为local index 0.
storage_synchronizer---存储同步器，用来同步存储的，改同步器会随着块的增长，逐渐地把状态同步到pRuntime中的chain_storage字段，也就是下面这个字段
chain_storage---
genesis_block_hash---这个是从init_runtim这个RPC的caller里面传递过来的
```

说下这个recv_mq，其在handle_inbound_messages函数中被调用，一个块调用一次，DispatchBlock一次RPC可能会来多个块。首先从runtime_state中拿到链的存储状态，从存储状态中拿到PhalaMq的OutboundMessages，这些消息拿到以后形成Vec\<Message\>, recv_mq会被封装进BlockInfo中，是以一个可变引用。消息Vec被循环依次处理，每个消息都有sender和destination。它被BlockInfo类型的block的recv_mq的成员方法dispatch出去。

dispatch函数实际上就是给recv_mq内部的订阅者BTreeMap相关的消息目标路径，发送给对应目标的接收者channel。并更新local index累加。处理完消息后，就清空recv_mq。

本质上以上过程就是从存储状态中拿到消息列表，发送到recv_mq中。

说下storage_synchronizer，在init_runtime中，会根据是否是平行链来做相应的初始化，里面有一个区块的Validator和Validator初始化的main bridge。初始化完成后，在sync_header(PhactoryAPI.SyncHeader)，sync_para_header(PhactoryAPI.SyncParaHeader)，sync_combined_headers(PhactoryAPI.SyncCombinedHeaders)，dispatch_block(PhactoryAPI.DispatchBlocks)中都会用到，他们上层都有对应的RPC。

可以看到同步的主要是区块头，因为是轻量级的，当然，在dispath_block中，其调用了其成员函数feed_block，一个区块调用一次，来通过block带来的changes来apply这些变换到RuntimeState结构体里面的chain_storage中。

- System结构体

```txt

```

TODO