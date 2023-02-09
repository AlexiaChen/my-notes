
## 开发团队最好知道的东西

#### 轻客户端守护进程(LCD)

与全节点相比，轻客户端仅跟踪区块链上的某些信息。轻客户端不跟踪区块链的整个状态，也不包含链的每个交易/块。 

在 Tendermint 共识中，轻客户端协议允许客户端受益于与全节点受益的相同程度的安全性，同时带宽需求被最小化。客户端可以接收区块链状态和交易的加密证明，而无需同步所有区块甚至它们的标头。 

> 查看 Ethan Frey 撰写的 Tendermint 共识中的轻客户端 [Light Clients in Tendermint Consensus | by Ethan Frey | Cosmos Blog](https://blog.cosmos.network/light-clients-in-tendermint-consensus-1237cfbda104) ，了解更多关于如何在 Tendermint 共识中使用轻客户端的信息。 

因此，轻客户端对于区块链间通信协议 (IBC) 如何跟踪时间戳、root hash和下一个validators set哈希等信息也至关重要。这节省了空间并提高了状态更新处理的效率。

轻客户端守护进程（LCD）是一个由 Cosmos SDK 暴露的 HTTP1.1 服务器，它的默认端口是 `1317`。它为链暴露了一个 REST API——这个 API 允许与RESTful 网络服务。传统上，每个查询都为 LCD 重新实现并在幕后路由到 RPC。

> 为什么叫轻客户端守护进程？
> 
> 在 SDK v0.40 之前，要获得 REST API，必须运行另一个后端服务（或守护进程 [Daemon (computing) - Wikipedia](https://en.wikipedia.org/wiki/Daemon_(computing)) ，一个从 Unix 继承的术语），例如使用 `gaiacli rest-server --laddr 0.0.0.0:1317 --nodelocalhost：26657`。 在 Cosmos SDK v0.40 中，REST 被移动到节点服务中，使其成为 Cosmos SDK 的一部分，但术语“守护进程”仍然存在，导致名称为轻客户端守护进程（LCD）。

#### RPC

远程过程调用 (RPC) 是一种软件通信协议。该术语经常出现在分布式计算中，因为 RPC 是一种通过允许程序引发在不同地址空间（不同机器）中执行的子程序过程来实现进程间通信（IPC）的技术。

RPC可以理解为一种客户端-服务器交互，其中“调用者”是客户端，更具体地说是请求程序，“执行者”是服务器，更具体地说是提供服务的程序。交互是通过请求-响应消息传递系统实现的。

简而言之，RPC 是一种请求-响应协议，由客户端发起，向远程服务器发送请求以执行子程序。

RPC 允许调用不同地址空间中的函数。通常，被调用函数运行在与调用它们的计算机不同的计算机上。但是，使用 RPC，开发人员编写代码时就好像子例程是本地的一样；开发人员不必为远程交互编写详细代码。因此，对于 RPC，它暗示所有调用过程基本上是相同的，与它们是本地调用还是远程调用无关。

> 由于 RPC 实现了远程请求-响应协议，因此请务必注意远程过程调用可能会在出现网络问题时失败。

#### 一个RPC请求是怎样工作的

通常，当调用远程过程调用时，过程参数通过网络传输到执行过程的执行环境。完成后，调用过程调用的结果将传输到调用环境。然后在调用环境中继续执行，就像在常规本地过程调用中一样。

RPC 一步一步的请求可能如下所示：

- 客户端调用客户端存根(stub)——一段代码转换在 RPC 期间在客户端和服务器之间传递的参数。该调用是本地过程调用。

> 存根是替代较长程序的小程序例程。这允许机器的行为就像远程机器上的程序在本地运行一样。客户端有一个与远程过程接口的存根，而服务器有一个与原始请求过程接口的存根。
> 
> 在 RPC 中，客户端的存根替代了提供请求过程的程序。存根接受请求并将其转发到远程过程。一旦远程过程完成请求，它将结果返回给存根，存根又将结果传递给请求过程。
> 
> 服务器还有一个与远程过程接口的存根。

- 客户端存根将过程参数打包到消息中。

> 打包过程参数称为编组(marshaling)。
> 
> 具体来说，这是从一个或多个应用程序收集数据，将数据块放入消息缓冲区，并将数据组织成规定的数据格式的过程。
> 
> marshaling处理对于将以一种语言编写的程序的输出参数作为输入传递给以不同语言编写的程序至关重要。

- 客户端存根然后进行系统调用以发送消息。
- 客户端的本地操作系统（OS）通过相应的传输层将消息从客户端（机器A）发送到服务器（机器B）。
- 服务器操作系统将传入的数据包传递给服务器存根。
- 服务器存根将消息及其包含的过程参数解包——这称为解组(unmarshaling)。
- 服务器存根调用服务器过程并执行该过程。
- 一旦过程完成，输出将返回到服务器存根。
- 服务器存根将返回值打包到消息中。
- 消息被发送到传输层，传输层将消息发送到客户端的传输层。
- 客户端存根解组返回参数并将它们返回给原始调用客户端。

> 在开放系统互连 (OSI) 模型中，RPC 涉及传输层和应用程序层。
> 
> 传输层的任务是通过网络可靠地发送和接收消息。它需要错误检查机制、数据流控制、数据完整性保证、拥塞避免、多路复用和相同订单交付。
> 
> 应用层的任务是确保不同计算机系统和网络上的应用程序之间的有效通信。它是应用程序的一个组件，用于控制与其他设备的通信方法。

#### RPC和cosmos

在 Cosmos 中，命令行界面 (CLI) 使用 RPC 来访问链。 一个节点公开多个端点——`gRPC`、`REST` 和 `Tendermint` 端点。

由 Tendermint 公开，Tendermint RPC 端点是一个 HTTP1.1 服务器。 默认端口为`26657`。gRPC服务器默认端口为`9090`，REST服务器默认端口为`1317`。Tendermint RPC独立于Cosmos SDK，可以配置。 它使用 HTTP `POST` 和 `JSON-RPC 2.0` 进行数据编码。

> 有关 Tendermint RPC、gRPC 和 REST 服务器的更多信息，建议仔细查看 Cosmos SDK 文档（[gRPC, REST, and Tendermint Endpoints | Cosmos SDK](https://docs.cosmos.network/main/core/grpc_rest.html) 。

Cosmos 公开了 Tendermint RPC 和 Cosmos LCD。 例如，CosmJS https://tutorials.cosmos.network/tutorials/6-cosmjs/1-cosmjs-intro.html 使用 RPC 来实现一个 JSON-RPC API。

#### Protobuf

Protobuf（“协议缓冲区”）是一种由 Google 开发的开源跨平台数据格式。它有助于序列化结构化数据，并在网络中或存储数据时协助程序通信。


> 如果您想更熟悉 Protobuf，请查看官方文档  [Protocol Buffers  |  Google Developers](https://developers.google.com/protocol-buffers)  以帮助深入了解并提供指南和教程。
> 
> 另请查看 Protobuf 上有关此平台的部分。[[cosmos的核心概念#Protobuf]]

在 Cosmos 中，Protobuf 是开发者用来描述消息格式的一种数据序列化方法。 Cosmos 应用程序中有很多内部通信，而 Protobuf 是如何完成通信的核心。

在 Cosmos SDK v0.40 中，Protobuf 开始取代 Amino 作为链状态和交易的数据编码格式，部分原因是 Protobuf 的编码/解码性能优于 Amino。此外，开发者工具也更适合 Protobuf。切换的另一个好处是促进了 gRPC 的使用，因为 Protobuf 自动定义和生成 gRPC 函数。因此，开发人员不再需要为 RPC、LCD 和 CLI 实现相同的查询。

#### gRPC

gRPC 是一个开源、高性能的远程过程调用 (RPC) 框架。 它由 Google 开发，用于处理 RPC，并于 2016 年发布。gRPC 可以在任何环境中运行，并支持多种编程语言。

> 有关 gRPC 的更多信息和非常有用的入门信息，请查看 gRPC 文档 [gRPC](https://grpc.io/) 。

gRPC 使用 `HTTP2` 进行传输，使用协议缓冲区 (Protobuf) 对数据进行编码。 gRPC 具有单一规范，这使得所有 gRPC 实现都保持一致。

#### gRPC和cosmos

在 Cosmos 中，gRPC 是带有 Protobuf 的传输控制协议 (TCP) 服务器，用于数据编码。 默认端口为 `9090`。

> 传输控制协议 (TCP) 是主要的互联网协议之一，它允许在客户端和服务器之间建立连接以发送数据。 TCP 使应用程序和互联网协议 (IP) 之间的通信成为可能。

在 Cosmos SDK 中，Protobuf 是主要的编码库。

> cosmos SDK中的编码
> 
> 有线编码协议（wire encoding protocol）是定义数据如何从一个点传输到另一个点的协议。有线协议描述了在应用层交换信息的方式。因此它是应用层协议的通信协议而不是传输协议。要定义数据交换，有线协议需要以下特定属性：
> 
> - 数据类型——数据单位、消息格式等。
> - 通信端点
> - 能力——交付保证、沟通方向等。
> 
> 有线协议可以是基于文本的或二进制协议。
> 
> 在 Cosmos SDK 中，有两类二进制线编码类型：客户端编码和存储编码。客户端编码处理交易处理和签名交易，而存储编码处理状态机交易以及存储在 Merkle 树中的内容。
> 
> Cosmos SDK 使用两种二进制编码协议：
> 
> - Amino：一种对象编码规范。每个 Cosmos SDK 模块都使用 Amino 编解码器来序列化类型和接口。
> - Protocol Buffers（Protobuf）：一种数据序列化方法，开发者用来描述消息格式。
> 
> 由于性能缺陷和缺少跨语言/客户端支持等原因，Protocol Buffers 比 Amino 的使用越来越多
> 
> 有关 Cosmos SDK 中编码的更多信息，请参阅 Cosmos SDK 文档 https://docs.cosmos.network/main/core/encoding.html 。

#### gRPC-web

gRPC 支持跨不同的软件和硬件平台。 gRPC-web 是用于浏览器客户端的 gRPC 的 JavaScript 实现。 gRPC-web 客户端通过特殊代理连接到 gRPC 服务。


> 有关 gRPC-web 的更多信息，建议仔细查看 gRPC 存储库 https://github.com/grpc/grpc-web 。
> 
> 要深入研究 gRPC-web 的开发，文档的快速入门 [Quick start | Web | gRPC](https://grpc.io/docs/platforms/web/quickstart/) 和基础教程[Basics tutorial | Web | gRPC](https://grpc.io/docs/platforms/web/basics/)是非常有价值的资源。

与一般的 gRPC 一样，gRPC-web 使用 `HTTP2` 和 Protobuf 进行数据编码。 默认端口为 `9091`。

> Secret.js 是一个 JavaScript SDK，用于编写与使用 gRPC-web 的 Secret Network [Secret Network - Bringing Privacy to Blockchains, Smart Contracts & Web3 (scrt.network)](https://scrt.network/) 交互的应用程序。

#### gRPC-gateway

gRPC-gateway 是一种将 gRPC 端点公开为 REST 端点的工具。它有助于提供 gRPC 和 RESTful 风格的 API，读取 gRPC 服务定义并生成可以将 RESTful JSON API 转换为 gRPC 的反向代理服务器。对于 Protobuf 查询服务中定义的每个 gRPC 端点，Cosmos SDK 提供相应的 REST 端点。

gRPC-Gateway 的目标是“为您的 gRPC 服务提供 HTTP+JSON 接口” [Background | gRPC-Gateway (grpc-ecosystem.github.io)](https://grpc-ecosystem.github.io/grpc-gateway/docs/overview/background/) 。有了它，开发人员可以从 gRPC 的所有优势中受益，同时仍然提供 RESTful API——当您想要开发 Web 应用程序但浏览器不支持 HTTP2 时，这是一个非常有用的工具。这有助于确保向后兼容性以及多语言、多客户端支持。

> 如果您想探索 gRPC-Gateway，建议仔细查看 gRPC-Gateway 文档 [gRPC-Gateway | gRPC-Gateway Documentation Website (grpc-ecosystem.github.io)](https://grpc-ecosystem.github.io/grpc-gateway/) 。

在 Cosmos SDK 中，gRPC-Gateway 提供了带有 REST API 的 HTTP1.1 服务器和用于数据编码的 base64 编码的 Protobuf；它将 gRPC 端点公开为 REST 端点。它在服务器上路由到 gRPC 并搭载 LCD，因此它也在端口 `1317` 上。

例如，如果由于浏览器不支持 HTTP2 而无法在您的应用程序中使用 gRPC，您仍然可以使用 Cosmos SDK。 SDK 通过 gRPC-Gateway 提供 REST 路由。

> Terra.js [Terra.js (terra-money.github.io)](https://terra-money.github.io/terra.js/) 是一个用于与 Terra 区块链交互的应用程序的 JavaScript SDK，它使用 gRPC-gateway。

#### Amino

Amino 是一种对象编码规范。在 Cosmos SDK 中，每个模块都使用 Amino 编解码器来帮助序列化类型和接口。 Amino 通过在具体类型之前添加字节前缀来处理接口。

通常，Amino 编解码器类型和接口在模块的域中注册。

> 具体类型是实现已注册接口的非接口类型。类型在存储在接口类型字段中或在具有接口元素的列表中时需要注册。
> 
> 作为最佳实践，在初始化时确保：
> 
> - 注册接口。
> - 实现具体类型。
> - 检查问题，例如前缀字节冲突。

每个模块都有一个函数 `RegisterLegacyAminoCodec`。有了它，用户可以提供编解码器并注册所有类型。应用程序为必要的模块调用此方法。

使用 Amino，当模块没有基于 Protobuf 的类型定义时，原始线字节(raw wire bytes)被编码和解码为具体类型或接口。

> 有关 Go 的 Amino 规范和实施的更多信息，请参阅 Tendermint Go Amino 文档 https://github.com/tendermint/go-amino 。


> Amino 基本上是经过一些修改的 JSON。例如，JSON 规范没有定义大于 2^53 的数字，因此在编码大于 uint64/int64 的类型时，Amino 中使用字符串。
> 
> 有关 Amino 类型及其在 JSON 中的表示的更多信息，请参阅 Secret.js 文档 [secret.js/DEVELOPERS.md at master · scrtlabs/secret.js (github.com)](https://github.com/scrtlabs/secret.js/blob/master/DEVELOPERS.md#amino-types-and-how-theyre-represented-in-json) 。

## 设置你的工作环境

- 安装Docker  有助于后面提供的上手练习
- 安装Go  因为cosmos SDK主要就是用go开发的，所以推荐用go的最新稳定版
- 如果要用CosmJS那么需要Node.js，所以安装Node.js的LTS的最新版本
- 如果你不喜欢用go，那么用rust就需要安装rust。
- 因为可能你以后是用Cosmos SDK和CosmJS这样的继承混合环境，所有推荐用VSCode进行开发

## 运行一个节点，API和CLI

> 有不同的方式来运行 Cosmos 区块链的节点。 您将探索如何使用 simapp  https://github.com/cosmos/cosmos-sdk/tree/master/simapp  执行此操作。

在开始使用 `simapp` 之前，先看看您要做什么：

https://youtu.be/qzUgh8mvyJE 

#### 编译simapp

Cosmos SDK 存储库包含一个名为 simapp  [cosmos-sdk/simapp at main · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/tree/main/simapp) 的文件夹。 在此文件夹中，您可以找到运行 Cosmos SDK 模拟版本的代码，因此您可以在不实际与您的链交互的情况下测试命令。 该二进制文件称为 `simd`，您将使用它与您的节点进行交互。

首先，创建目录并将其更改为 cosmos 文件夹，然后将 cosmos-sdk 存储库克隆到该文件夹中：

```bash
mkdir cosmos
cd cosmos
git clone git@github.com:cosmos/cosmos-sdk.git
cd cosmos-sdk
# 这个版本号看后面变化
git checkout v0.42.6
make build
ls build
```

你就可以看到有一个新的`build`目录被创建了，里面有`simd` 二进制文件。

#### 初始化simapp

现在重置数据库。 不仅在数据库已经初始化时运行此步骤，即使这是您第一次测试 `simapp`：

```bash
cd build
./simd unsafe-reset-all
```

命令输出列出了所有设置为初始状态的文件及其位置。

```txt
8:43AM INF Removed all blockchain history dir=/home/mathxh/.simapp/data  
8:43AM INF Generated private validator file keyFile=/home/mathxh/.simapp/config/priv_validator_key.jso  
n stateFile=/home/mathxh/.simapp/data/priv_validator_state.json
```

是时候初始化应用程序了。 初始化创建创世块和初始链状态：

```bash
./simd init demo
```

会打印如下:

```json
{

    "app_message": {

        "auth": {

            "accounts": [],

            "params": {

                "max_memo_characters": "256",

                "sig_verify_cost_ed25519": "590",

                "sig_verify_cost_secp256k1": "1000",

                "tx_sig_limit": "7",

                "tx_size_cost_per_byte": "10"

            }

        },

        "bank": {

            "balances": [],

            "denom_metadata": [],

            "params": {

                "default_send_enabled": true,

                "send_enabled": []

            },

            "supply": []

        },

        "capability": {

            "index": "1",

            "owners": []

        },

        "crisis": {

            "constant_fee": {

                "amount": "1000",

                "denom": "stake"

            }

        },

        "distribution": {

            "delegator_starting_infos": [],

            "delegator_withdraw_infos": [],

            "fee_pool": {

                "community_pool": []

            },

            "outstanding_rewards": [],

            "params": {

                "base_proposer_reward": "0.010000000000000000",

                "bonus_proposer_reward": "0.040000000000000000",

                "community_tax": "0.020000000000000000",

                "withdraw_addr_enabled": true

            },

            "previous_proposer": "",

            "validator_accumulated_commissions": [],

            "validator_current_rewards": [],

            "validator_historical_rewards": [],

            "validator_slash_events": []

        },

        "evidence": {

            "evidence": []

        },

        "genutil": {

            "gen_txs": []

        },

        "gov": {

            "deposit_params": {

                "max_deposit_period": "172800s",

                "min_deposit": [

                    {

                        "amount": "10000000",

                        "denom": "stake"

                    }

                ]

            },

            "deposits": [],

            "proposals": [],

            "starting_proposal_id": "1",

            "tally_params": {

                "quorum": "0.334000000000000000",

                "threshold": "0.500000000000000000",

                "veto_threshold": "0.334000000000000000"

            },

            "votes": [],

            "voting_params": {

                "voting_period": "172800s"

            }

        },

        "ibc": {

            "channel_genesis": {

                "ack_sequences": [],

                "acknowledgements": [],

                "channels": [],

                "commitments": [],

                "next_channel_sequence": "0",

                "receipts": [],

                "recv_sequences": [],

                "send_sequences": []

            },

            "client_genesis": {

                "clients": [],

                "clients_consensus": [],

                "clients_metadata": [],

                "create_localhost": false,

                "next_client_sequence": "0",

                "params": {

                    "allowed_clients": [

                        "06-solomachine",

                        "07-tendermint"

                    ]

                }

            },

            "connection_genesis": {

                "client_connection_paths": [],

                "connections": [],

                "next_connection_sequence": "0"

            }

        },

        "mint": {

            "minter": {

                "annual_provisions": "0.000000000000000000",

                "inflation": "0.130000000000000000"

            },

            "params": {

                "blocks_per_year": "6311520",

                "goal_bonded": "0.670000000000000000",

                "inflation_max": "0.200000000000000000",

                "inflation_min": "0.070000000000000000",

                "inflation_rate_change": "0.130000000000000000",

                "mint_denom": "stake"

            }

        },

        "params": null,

        "slashing": {

            "missed_blocks": [],

            "params": {

                "downtime_jail_duration": "600s",

                "min_signed_per_window": "0.500000000000000000",

                "signed_blocks_window": "100",

                "slash_fraction_double_sign": "0.050000000000000000",

                "slash_fraction_downtime": "0.010000000000000000"

            },

            "signing_infos": []

        },

        "staking": {

            "delegations": [],

            "exported": false,

            "last_total_power": "0",

            "last_validator_powers": [],

            "params": {

                "bond_denom": "stake",

                "historical_entries": 10000,

                "max_entries": 7,

                "max_validators": 100,

                "unbonding_time": "1814400s"

            },

            "redelegations": [],

            "unbonding_delegations": [],

            "validators": []

        },

        "transfer": {

            "denom_traces": [],

            "params": {

                "receive_enabled": true,

                "send_enabled": true

            },

            "port_id": "transfer"

        },

        "upgrade": {},

        "vesting": {}

    },

    "chain_id": "test-chain-eNZSDb",

    "gentxs_dir": "",

    "moniker": "demo",

    "node_id": "edc6225a9cbca84c4601910550090535afb1dedd"

}
```

您可以在输出中找到您的 `chain_id`，在我们的构建中恰好称为 `test-chain-eNZSDb`。 记下您的输出名称，因为稍后您将需要它来确定链 ID，方法是通过标志 `--chain-id` 将其传递给 `simapp`。

您可以使用以下命令检查初始配置：

```bash
cat ~/.simapp/config/genesis.json
```

#### 准备你的账号

如果不了解账户的概念，先看 [[cosmos的核心概念#账户]]

您还可以检查您的密钥。 这些保存在后端密钥环中，默认情况下是操作系统的密钥环：

```bash
./simd keys list
```

可能正为你所预期的，现在没有账户，所以输出 `[]`

所以添加key,并输入钥匙环的密码后:

```bash
./simd keys add mathxh
```

然后输出:

```txt
Enter keyring passphrase:  
password must be at least 8 characters  
Enter keyring passphrase:  
Re-enter keyring passphrase:  
  
- name: mathxh  
type: local  
address: cosmos14zy3l08ajyaq58l3lfpuxapytqh3v2xpsw0dun  
pubkey: cosmospub1addwnpepqfvezgxs05qrm6mexmtk8mjg72vw5uhlf24s5khryytpzzxgajsvxr57t0w  
mnemonic: ""  
threshold: 0  
pubkeys: []  
  
  
**Important** write this mnemonic phrase in a safe place.  
It is the only way to recover your account if you ever forget your password.  
  
govern choose ostrich weird clap endorse summer eight because ignore worry warm van young multiply end  
less fly story talent flip suggest tired notable easy
```

您可以在上述输出的末尾看到助记词。 这个单词序列是一个助记符，您可以使用它来恢复您的公钥和私钥。 在生产环境中，助记符必须以可靠和保密的方式存储，作为密钥管理基础设施的一部分。

确认密钥已添加,也要输入钥匙环密码：

```bash
./simd keys list
./simd keys show mathxh
```

#### 让你自己成为一个合适的validator

如前所述，Cosmos SDK 区块链依赖于已识别的validator来生成块。 最初没有validator来生成块。 您处于第 22 条军规的情况：您的初始化和未启动的链需要一个创世帐户和validator来进行引导。

使您的密钥（也称为帐户）在创世文件中具有初始余额：

```bash
./simd add-genesis-account mathxh 100000000stake
```

此处附加到金额的是`stake`后缀。 根据创世文件，该`stake`代表该链中代币的单位。 因此，此命令会向您的帐户添加 100000000 `stake`。 如果有疑问，您可以在 genesis.json 文件中确认正确的后缀：

```bash
grep -A 2 -B 2 denom ~/.simapp/config/genesis.json
```

您还可以在创世文件本身中确认您有一个初始余额：

```bash
grep -A 10 balances ~/.simapp/config/genesis.json
```

尽管存在初始余额，但在运行区块链之前，您仍然需要避开第 22 条军规并将引导交易包含在创世文件中。

> 在这种情况下，为了让您的网络正常运行，您必须满足加权validators的 2/3 阈值。
> 
> 然而，您将独自在网络上，因此您可以在强制执行的最低金额 [cosmos-sdk/staking.go at 66ca3e88aaf20d00a8d3074ba786c057fb7ade98 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/66ca3e8/types/staking.go#L21-L22) 或以上，即 `1000000` stake 上抵押任何数量。 但是，为了提醒自己，诚实的节点大量投入是很重要的，您在刚刚创建的 mathxh 帐户中投入了 `100000000` `stake`中的 `70000000` `stake`。 确保不要用完你所有的代币，这样你仍然可以支付gas费，这样你以后就不会用完代币。
> 
> 不要忘记使用您自己的 `--chain-id`。

```bash
./simd gentx mathxh 70000000stake --chain-id test-chain-eNZSDb
```

可以确认行为已经写入配置:

```txt
Genesis transaction written to "/home/mathxh/.simapp/config/gentx/gentx-edc6225a9cbca84c46019105500905  
35afb1dedd.json"
```

在您在自己的文件中创建此创世交易后，使用 `collect-gentxs` 收集所有创世交易以将其包含在您的创世文件中：

```bash
./simd collect-gentxs
```


#### 创建区块（也就是运行区块链）

现在你可以启动单节点区块链了:

```bash
./simd start
```

在运行命令的终端窗口中，您可以看到正在生成和验证的块：

```txt
9:27AM INF starting ABCI with Tendermint  
9:27AM INF Starting multiAppConn service impl=multiAppConn module=proxy  
9:27AM INF Starting localClient service connection=query impl=localClient module=abci-client  
9:27AM INF Starting localClient service connection=snapshot impl=localClient module=abci-client  
9:27AM INF Starting localClient service connection=mempool impl=localClient module=abci-client  
9:27AM INF Starting localClient service connection=consensus impl=localClient module=abci-client  
9:27AM INF Starting EventBus service impl=EventBus module=events  
9:27AM INF Starting PubSub service impl=PubSub module=pubsub  
9:27AM INF Starting IndexerService service impl=IndexerService module=txindex  
9:27AM INF ABCI Handshake App Info hash= height=0 module=consensus protocol-version=0 software-version  
=  
9:27AM INF ABCI Replay Blocks appHeight=0 module=consensus stateHeight=0 storeHeight=0  
9:27AM INF asserting crisis invariants inv=0/11 module=x/crisis name=staking/module-accounts  
9:27AM INF asserting crisis invariants inv=1/11 module=x/crisis name=staking/nonnegative-power  
9:27AM INF asserting crisis invariants inv=2/11 module=x/crisis name=staking/positive-delegation  
9:27AM INF asserting crisis invariants inv=3/11 module=x/crisis name=staking/delegator-shares  
9:27AM INF asserting crisis invariants inv=4/11 module=x/crisis name=distribution/nonnegative-outstand  
ing  
9:27AM INF asserting crisis invariants inv=5/11 module=x/crisis name=distribution/can-withdraw  
9:27AM INF asserting crisis invariants inv=6/11 module=x/crisis name=distribution/reference-count  
9:27AM INF asserting crisis invariants inv=7/11 module=x/crisis name=distribution/module-account  
9:27AM INF asserting crisis invariants inv=8/11 module=x/crisis name=bank/nonnegative-outstanding  
9:27AM INF asserting crisis invariants inv=9/11 module=x/crisis name=bank/total-supply  
9:27AM INF asserting crisis invariants inv=10/11 module=x/crisis name=gov/module-account  
9:27AM INF asserted all invariants duration=4.7752 height=0 module=x/crisis  
9:27AM INF created new capability module=ibc name=ports/transfer  
9:27AM INF port binded module=x/ibc/port port=transfer  
9:27AM INF claimed capability capability=1 module=transfer name=ports/transfer  
9:27AM INF Completed ABCI Handshake - Tendermint and App are synced appHash= appHeight=0 module=consen  
sus  
9:27AM INF Version info block=11 p2p=8 tendermint_version=0.34.11  
9:27AM INF This node is a validator addr=48B7071171137E47CB81679B4B3D10A3DB4124F9 module=consensus pub  
Key=QPhU/5FDZM5vYGqe3vZzUcu+CZYCQSIxYNIti3YuYQA=  
9:27AM INF P2P Node ID ID=edc6225a9cbca84c4601910550090535afb1dedd file=/home/mathxh/.simapp/config/no  
de_key.json module=p2p  
9:27AM INF Adding persistent peers addrs=[] module=p2p  
9:27AM INF Adding unconditional peer ids ids=[] module=p2p  
9:27AM INF Add our address to book addr={"id":"edc6225a9cbca84c4601910550090535afb1dedd","ip":"0.0.0.0  
","port":26656} book=/home/mathxh/.simapp/config/addrbook.json module=p2p  
9:27AM INF Starting Node service impl=Node  
9:27AM INF Starting pprof server laddr=localhost:6060  
9:27AM INF Starting RPC HTTP server on 127.0.0.1:26657 module=rpc-server  
9:27AM INF Starting P2P Switch service impl="P2P Switch" module=p2p  
9:27AM INF Starting Evidence service impl=Evidence module=evidence  
9:27AM INF Starting StateSync service impl=StateSync module=statesync  
9:27AM INF Starting PEX service impl=PEX module=pex  
9:27AM INF Starting AddrBook service book=/home/mathxh/.simapp/config/addrbook.json impl=AddrBook modu  
le=p2p  
9:27AM INF Starting Mempool service impl=Mempool module=mempool  
9:27AM INF Starting BlockchainReactor service impl=BlockchainReactor module=blockchain  
9:27AM INF Starting Consensus service impl=ConsensusReactor module=consensus  
9:27AM INF Reactor module=consensus waitSync=false  
9:27AM INF Starting State service impl=ConsensusState module=consensus  
9:27AM INF Ensure peers module=pex numDialing=0 numInPeers=0 numOutPeers=0 numToDial=10  
9:27AM INF No addresses to dial. Falling back to seeds module=pex  
9:27AM INF Starting baseWAL service impl=baseWAL module=consensus wal=/home/mathxh/.simapp/data/cs.wal  
/wal  
9:27AM INF Starting Group service impl=Group module=consensus wal=/home/mathxh/.simapp/data/cs.wal/wal  
9:27AM INF Searching for height height=1 max=0 min=0 module=consensus wal=/home/mathxh/.simapp/data/cs  
.wal/wal  
9:27AM INF Searching for height height=0 max=0 min=0 module=consensus wal=/home/mathxh/.simapp/data/cs  
.wal/wal  
9:27AM INF Found height=0 index=0 module=consensus wal=/home/mathxh/.simapp/data/cs.wal/wal  
9:27AM INF Catchup by replaying consensus messages height=1 module=consensus  
9:27AM INF Replay: Done module=consensus  
9:27AM INF Starting TimeoutTicker service impl=TimeoutTicker module=consensus  
9:27AM INF Saving AddrBook to file book=/home/mathxh/.simapp/config/addrbook.json module=p2p size=0  
9:27AM INF Timed out dur=4922.375899 height=1 module=consensus round=0 step=1  
9:27AM INF received proposal module=consensus proposal={"Type":32,"block_id":{"hash":"54C61648181FEACF  
82B362B4F4E1ACC2406D87F273B85EBA27AD39BD5BFCA5F3","parts":{"hash":"FAB5D4D216F3DDDE97D1528F12C1F630AEA  
8CD330DE1A71E0BD72732FA1288C9","total":1}},"height":1,"pol_round":-1,"round":0,"signature":"QVaVuhpiIb  
mpOe4rqm7ZdN67BVd+cxtP9cFQxoOk+eZ2t72ZR+fM/ZYF1c9GQajjmF7uEb9wfyQVEFODgucRBA==","timestamp":"2022-11-2  
4T01:27:27.818594296Z"}  
9:27AM INF received complete proposal block hash=54C61648181FEACF82B362B4F4E1ACC2406D87F273B85EBA27AD3  
9BD5BFCA5F3 height=1 module=consensus  
9:27AM INF finalizing commit of block hash=54C61648181FEACF82B362B4F4E1ACC2406D87F273B85EBA27AD39BD5BF  
CA5F3 height=1 module=consensus num_txs=0 root=E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991  
B7852B855  
9:27AM INF minted coins from module account amount=2stake from=mint module=x/bank  
9:27AM INF executed block height=1 module=state num_invalid_txs=0 num_valid_txs=0  
9:27AM INF commit synced commit=436F6D6D697449447B5B37382032323220333820313920323530203139312031333920  
393220313820313338203137203233342031332031393020313738203636203632203438203137322032313720313234203138  
3820323231203634203139362031372031373020323533203436203137342032382038385D3A317D  
9:27AM INF committed state app_hash=4EDE2613FABF8B5C128A11EA0DBEB2423E30ACD97CBCDD40C411AAFD2EAE1C58 h  
eight=1 module=state num_txs=0  
9:27AM INF indexed block height=1 module=txindex  
9:27AM INF Timed out dur=4742.505696 height=2 module=consensus round=0 step=1  
9:27AM INF received proposal module=consensus proposal={"Type":32,"block_id":{"hash":"DE448B274EF8907D  
FBB86F80AB5769062564BC138D865AFCB3F4526ADCD26502","parts":{"hash":"EEF521D8DCC8DFF5690CBD702CC2662B0EE  
91B8C9816D2E92E7BE647F1C74313","total":1}},"height":2,"pol_round":-1,"round":0,"signature":"i09Mle3Zou  
hoS1gsNsdacF47icNC1hvF91VserVQg4NYJzlXB/yK7UCipxcaD24WY2SEeM0gB3Tcr8Xy1guyBg==","timestamp":"2022-11-2  
4T01:27:33.352228931Z"}  
9:27AM INF received complete proposal block hash=DE448B274EF8907DFBB86F80AB5769062564BC138D865AFCB3F45  
26ADCD26502 height=2 module=consensus  
9:27AM INF finalizing commit of block hash=DE448B274EF8907DFBB86F80AB5769062564BC138D865AFCB3F4526ADCD  
26502 height=2 module=consensus num_txs=0 root=4EDE2613FABF8B5C128A11EA0DBEB2423E30ACD97CBCDD40C411AAF  
D2EAE1C58  
9:27AM INF minted coins from module account amount=2stake from=mint module=x/bank  
9:27AM INF executed block height=2 module=state num_invalid_txs=0 num_valid_txs=0  
9:27AM INF commit synced commit=436F6D6D697449447B5B37203237203135203137372033203530203136302031333420  
313938203230362035372036342031353120343920323431203133342031373620363420313635203134352031353620313332  
20313530203134352032303220313838203233302035372032323920323238203230203136335D3A327D  
9:27AM INF committed state app_hash=071B0FB10332A086C6CE39409731F186B040A5919C849691CABCE639E5E414A3 h  
eight=2 module=state num_txs=0  
9:27AM INF indexed block height=2 module=txindex
```

在同一文件夹中打开一个新终端并检查余额：

```bash
./simd query bank balances $(./simd keys show mathxh -a)
```

会打印:

```txt
balances:  
- amount: "30000000"  
denom: stake  
pagination:  
next_key: null  
total: "0"
```

可以看到`mathxh`账户还剩`30000000` `stake`， 因为抵押了`70000000` `stake` 给validator。

#### 发送交易

练习发送交易。 为此，您将创建另一个名为“student”的帐户并将一些代币转移到该帐户：

```bash
./simd keys add student
```

会输出:

```txt
- name: student  
type: local  
address: cosmos1kwlpt622qclkvqqmgn9547wczqd2zchlu8zy39  
pubkey: cosmospub1addwnpepqdatf270n672k5cp4eklmpxqc7ww4c3jtps3pfe2khdp8cv84pxf7lfr5ax  
mnemonic: ""  
threshold: 0  
pubkeys: []  
  
  
**Important** write this mnemonic phrase in a safe place.  
It is the only way to recover your account if you ever forget your password.  
  
transfer six pause leaf engine sign gaze noodle obey abstract hundred salad pond quote finish season m  
ixed photo damp carry blossom once deer marriage
```

在发送任何代币之前确认新账户的余额不存在：

```bash
./simd query bank balances $(./simd keys show student -a)
```

此帐户没有余额。 您的区块链中尚不存在新帐户。 只有密钥对已生成并存储在您的密钥环中：

```txt
balances: []  
pagination:  
next_key: null  
total: "0"
```

您需要发送交易来更改此新帐户的余额：

```bash
./simd tx bank send $(./simd keys show mathxh -a) $(./simd keys show student -a) 10stake --chain-id test-chain-eNZSDb
```

在签名和广播之前，您应该被提示确认交易：

```txt
{"body":{"messages":[{"@type":"/cosmos.bank.v1beta1.MsgSend","from_address":"cosmos14zy3l08ajyaq58l3lf  
puxapytqh3v2xpsw0dun","to_address":"cosmos1kwlpt622qclkvqqmgn9547wczqd2zchlu8zy39","amount":[{"denom":  
"stake","amount":"10"}]}],"memo":"","timeout_height":"0","extension_options":[],"non_critical_extensio  
n_options":[]},"auth_info":{"signer_infos":[],"fee":{"amount":[],"gas_limit":"200000","payer":"","gran  
ter":""}},"signatures":[]}  
  
confirm transaction before signing and broadcasting [y/N]: y  
code: 0  
codespace: ""  
data: ""  
gas_used: "0"  
gas_wanted: "0"  
height: "0"  
info: ""  
logs: []  
raw_log: '[]'  
timestamp: ""  
tx: null  
txhash: 23D4B4438D1D5BCBAC9CC771795007AEEB5B798998604FBCFC1DA1C0ECCC65BA
```

最后可以发现student账户下多了10个stake:

```bash
./simd query bank balances $(./simd keys show student -a)
```

#### CLI路由

现在是时候编写一些 Go 代码了。 simd 如何通过命令行界面进行交互？ 检查 `cosmos-sdk/simapp/simd/main.go` [cosmos-sdk/main.go at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/simapp/simd/main.go) 文件：

```go
package main

import (
  "os"

  "github.com/cosmos/cosmos-sdk/server"
  svrcmd "github.com/cosmos/cosmos-sdk/server/cmd"
  "github.com/cosmos/cosmos-sdk/simapp"
  "github.com/cosmos/cosmos-sdk/simapp/simd/cmd"
)

func main() {
  rootCmd, _ := cmd.NewRootCmd()

  if err := svrcmd.Execute(rootCmd, simapp.DefaultNodeHome); err != nil {
    switch e := err.(type) {
    case server.ErrorCode:
      os.Exit(e.Code)

    default:
      os.Exit(1)
    }
  }
}
```

`cmd.NewRootCmd()` [cosmos-sdk/main.go at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/simapp/simd/main.go#L13) 函数是 CLI 处理程序。 它是通过`“github.com/cosmos/cosmos-sdk/simapp/simd/cmd”` [cosmos-sdk/main.go at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/simapp/simd/main.go#L9) 行导入的。 它可以在 `cosmos-sdk/simapp/simd/cmd/root.go`  [cosmos-sdk/root.go at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/simapp/simd/cmd/root.go#L39) 文件中找到：

```go
func NewRootCmd() (*cobra.Command, params.EncodingConfig)
```

其中定义了应用程序名称等基本属性  [cosmos-sdk/root.go at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/simapp/simd/cmd/root.go#L51-L53) ：

```go
rootCmd := &cobra.Command{
    Use:   "simd",
    Short: "simulation app",
```

此外，观察 Cobra 被导入并用于 CLI [cosmos-sdk/root.go at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/simapp/simd/cmd/root.go#L144-L155)  重定向：

```go
rootCmd.AddCommand(
    genutilcli.InitCmd(simapp.ModuleBasics, simapp.DefaultNodeHome),
    genutilcli.CollectGenTxsCmd(banktypes.GenesisBalancesIterator{}, simapp.DefaultNodeHome),
    genutilcli.MigrateGenesisCmd(),
    genutilcli.GenTxCmd(simapp.ModuleBasics, encodingConfig.TxConfig, banktypes.GenesisBalancesIterator{}, simapp.DefaultNodeHome),
    genutilcli.ValidateGenesisCmd(simapp.ModuleBasics),
    AddGenesisAccountCmd(simapp.DefaultNodeHome),
    tmcli.NewCompletionCmd(rootCmd, true),
    NewTestnetCmd(simapp.ModuleBasics, banktypes.GenesisBalancesIterator{}),
    debug.Cmd(),
    config.Cmd(),
)
```

另外，查看 `simapp/app.go`  [cosmos-sdk/app.go at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/simapp/app.go) ，其中将导入每个模块和密钥管理器。 您将看到的第一件事是大多数 Cosmos-sdk 应用程序使用的大量模块 [cosmos-sdk/app.go at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/simapp/app.go#L33-L44) ：

```go
...
  "github.com/cosmos/cosmos-sdk/x/auth"
  "github.com/cosmos/cosmos-sdk/x/auth/ante"
  authkeeper "github.com/cosmos/cosmos-sdk/x/auth/keeper"
  authsims "github.com/cosmos/cosmos-sdk/x/auth/simulation"
  authtx "github.com/cosmos/cosmos-sdk/x/auth/tx"
  authtypes "github.com/cosmos/cosmos-sdk/x/auth/types"
  "github.com/cosmos/cosmos-sdk/x/auth/vesting"
  "github.com/cosmos/cosmos-sdk/x/bank"
  bankkeeper "github.com/cosmos/cosmos-sdk/x/bank/keeper"
  banktypes "github.com/cosmos/cosmos-sdk/x/bank/types"
  "github.com/cosmos/cosmos-sdk/x/capability"
  capabilitykeeper "github.com/cosmos/cosmos-sdk/x/capability/keeper"
  capabilitytypes "github.com/cosmos/cosmos-sdk/x/capability/types"
...
```

`/cosmos-sdk/x/` 文件夹中的模块由几个致力于 Cosmos 堆栈的组织维护。 要了解一个模块，最好的方法是查看相应的 `spec` 文件夹。 例如，查看 `cosmos-sdk/x/bank/spec/01_state.md`  [cosmos-sdk/01_state.md at d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/d83a3bf92c9a84bddf3f5eb6692a1101c18b42f1/x/bank/spec/01_state.md) 以了解您在本节中使用的`bank`模块的状态。

> 要了解更多关于模块的知识，请看 [[cosmos的核心概念#模块(Modules)]]

## Golang的基础知识

[Chapter Overview - Introduction to Go | Developer Portal (cosmos.network)](https://tutorials.cosmos.network/tutorials/4-golang-intro/#)   这里直接看其他的资料也可以，比如go by examples啥的。资源很多，就不列出来了

## IBC开发者

> 理解IBC Denoms

最初的 Cosmos 白皮书 [Whitepaper - Resources - Cosmos Network](https://v1.cosmos.network/resources/whitepaper)  提出的 Interchain 愿景是主权的、特定于应用程序的PoS区块链之一。这一愿景的一个重要组成部分是区块链间通信协议或简称为 IBC。有了 IBC，链可以维护它们的主权，同时仍然能够无需许可地与其他链（也支持 IBC）进行互操作，从而为区块链互联网铺平道路。

听起来不错，对吧？但是等等，这到底是什么意思？

好吧，IBC 支持在链之间传递任意消息（事实上，甚至更通用的状态机，如单机 [410 Deleted by author — Medium](https://interchain-io.medium.com/ibc-beyond-light-clients-solo-machine-fb55ba0b0234) ），因此开发人员可以继续创建各种 IBC 应用程序，这些应用程序通过 IBC 交换数据包以启用应用逻辑。

然而，迄今为止第一个也是最主要的例子是将（可替代的）代币从源链转移到目标链。

举个例子：你在 Cosmos Hub 上有一些 ATOM，但想在 DEX（去中心化交易所）如 Osmosis（[Swap (osmosis.zone)](https://app.osmosis.zone/) 上用它交换一些其他代币。这可以通过使用流行的区块浏览器 Mintscan 在 Hub 和 Osmosis 之间进行随机 IBC 传输来说明。

![[Pasted image 20221124104616.png]]

进行以下交易 [COSMOS Tx F7196B37828BAAF5C55E499D62A58E2927542CB2FB57B587BA77BF5BB044FFBF (mintscan.io)](https://www.mintscan.io/cosmos/txs/F7196B37828BAAF5C55E499D62A58E2927542CB2FB57B587BA77BF5BB044FFBF) 。 在那里您可以看到有关交易的一些一般信息以及数据，特别是关于交易中包含的 IBC 传输消息的数据。 在发件人和收件人的下面，您会发现：

![[Pasted image 20221124104902.png]]

如果你熟悉 [[IBC介绍#什么是IBC]] , 那么你会知道以上这些概念。

现在，来看看，把ATOM从Osmosis发送回cosmos Hub会是如何呢？ 请看这个交易 [OSMOSIS Tx 9721FE816ABEE87D25259F87BA816EB53651194DD871C3F6B0A5B00434429A80 (mintscan.io)](https://www.mintscan.io/osmosis/txs/9721FE816ABEE87D25259F87BA816EB53651194DD871C3F6B0A5B00434429A80) 

![[Pasted image 20221124105219.png]]

现在，您在 `Origin Denom` 字段中看到的不是 `uatom`，而是：`IBC/27394FB092D2ECCD56123C74F36E4C1F926001CEADA9CA97EA622B25F41E5EB2`。

这就是所谓的 IBC 命名（IBC denoms），这就是通过 IBC 发送的资产将在链上表示的方式。

#### ICS-20 代币转账

> IBC 部分详细讨论了代币传输或 ICS-20  [[IBC介绍#IBC token转移]] 。 ICS-20 中的“ICS”是跨链标准的简写。 在本节中，您可以深入了解 IBC 如何实现跨链（可替代）代币的传输。 出于本教程的目的，这里有一个简短的总结。

想象两个区块链，区块链 A 和区块链 B。作为起点，您在区块链 A 上有一些要发送到区块链 B 的代币。您可以按照下图中的步骤操作：

![[Pasted image 20221124104616.png]]

使用 IBC 将代币发送到另一个区块链时：

- 数据包commitment存储在区块链 A 上，要发送的代币托管在链 A 上（图片左上角）。
- 中继器记录要发送的数据包并在目标链上提交 `MsgRecvPacket` 以及链 B 上的链 A 轻客户端验证的证明（图像中间）。
- 使用 IBC，代币所代表的价值可以跨链转移，但代币本身不能。因此，区块链 B 以凭证(vouncher)替代代币的形式铸造自己的代表代币。这些将以 IBC 名称 `IBC/...`（图像右下角）为特征。

> 请注意，这仅考虑代币转移成功且可以验证相应证明的happy path，也就是不考虑错误情况。

当使用 IBC 将代币发送回源区块链时：

- 区块链 B 将凭证代币（voucher tokens）发送回区块链 A。
- 凭证代币在区块链 B 上被销毁（销毁）。
- 区块链 A 上锁定的代币被解锁。

> 解锁区块链 A 上锁定的代币的唯一方法是从区块链 B 发送回凭证代币。结果是区块链 B 上的凭证代币被销毁。 销毁过程有目的地使凭证停止流通。

> 所以无论是substrate的XCM还是各种跨链桥，还是cosmos的IBC，跨链转移资产的原理都是mint-burn这样的模型。

#### IBC denoms是怎样被继承(Derived)的?

IBC 是一种协议，允许中继器无需许可地创建客户端、连接和通道。 同样，请参阅 [[IBC介绍#IBC token转移]] 以获取更深入的信息。 正如那里所解释的那样，未经许可创建客户端、连接和通道的结果是，经过不同path的token具有不同的安全保证。 为了解决这个问题，IBC 协议确保在通过 IBC 传输代币时表示在目标链上铸造的凭证时将path信息添加到基本面额/命名(base denomination)。

再次以 Osmosis 和 Cosmos Hub 为例，当从 Osmosis 向 Hub 发送一些 OSMO 时，Hub 端的 channelEnd 具有以下特征：

```go
channelEnd{
  ...
  counterpartyChannelIdentifier: 'channel-141',
  counterpartyPortIdentifier: 'transfer'
  ...
}
```

	这是根据以下格式添加的：`{portID}/{channelID}/base_denom`

在此特定示例中：`transfer/channel-141/uosmo`。

IBC 资产的这种表示（其中路径信息被添加到资产所表示的基本面额之前）是资产的有效表示，并且可以在某些用户界面上找到。 然而，它并不完全是之前引用的 IBC 名称 `IBC/....`

事实上，还需要一个步骤才能达到 IBC 面额。 您使用 SHA256 散列函数获取在路径信息前加上 `base_denom` 的散列值。 这给出了 IBC 名称的以下内容：

```go
// hash() representing a SHA256 hashing function returning a string
ibc_denom := 'ibc/' + hash('path' + 'base_denom')
```

> 在前面的示例中，对于 `transfer/channel-141/uosmo`，对应的 IBC denom是：`ibc/14F9BC3E44B8A9C1BE1FB08980FAB87034C9905EF17CF2F5008FC085218811CC`。

请注意，通过 IBC 转移的资产以 IBC 面额（IBC denom）存储在链上。 然而，由前端和用户界面的开发人员决定他们是否会使用人类可读的形式来满足他们的用户体验需求。

可以使用查询来根据 IBC 资产的路径信息找到哈希，这将在后面进行描述； 但是，您始终也可以使用 SHA256 哈希生成器  [SHA-256 hash calculator | Xorbin](https://xorbin.com/tools/sha256-hash-calculator) 来计算它。

> 那么...为什么要使用哈希？
> 
> 散列函数有许多理想的特性，使它们经常用于密码学。 在此讨论中最有用的属性是，无论输入的长度如何，散列输出总是减少到固定长度（在 SHA256 的情况下为 256 位）。
> 
> 考虑以下：
> - 散列可以包含在链与链之间的多个跃点上跟踪令牌的路径。
> - 当直接打印路径时，这可能会非常长。
> - Cosmos SDK 对代币的denom有 64 个字符的限制。
> 
> 这就是首选散列的原因。 有关设计决策的更多信息，请参见此处 [ADR 001: Coin Source Tracing | IBC-Go (cosmos.network)](https://ibc.cosmos.network/main/architecture/adr-001-coin-source-tracing.html) 。

使用散列时的权衡是您无法在给定输出的情况下计算输入（散列是一种不可逆的操作）。 因此，ICS-20 模块保留它遇到的 IBC 面额（IBC denominations）的映射，以便查找原始路径和 `base_denom`。 因此，您需要查询一个节点以找出实际路径和面额是什么。 此查询称为 `denomtrace`。


> 对于 IBC denoms，`/` 字符有特殊含义。 它用于从基本命名（base denom）中解析端口和通道标识符。 因此，最初禁止在代币的基本命名中使用正斜杠。
> 
> 但是，由于 Evmos 链的要求（在基于合约的denoms中使用正斜杠），添加了对包含 `/` 的base denoms的支持。 有关详细信息，请查看 ibc-go 文档 [Migrating from not supporting base denoms with slashes to supporting base denoms with slashes | IBC-Go (cosmos.network)](https://ibc.cosmos.network/main/migrations/support-denoms-with-slashes.html) 。

#### 实战例子： `denom_trace`

您可以区分两种与 IBC denoms交互的情况：

- 计算给定路径的 IBC denom。
- 遇到IBC denom时跟踪路径信息和base denom。

`transfer` IBC 模块公开了对这两种情况的查询。 为了查询，您将必须与区块链网络的节点进行交互。

这些查询是：

```bash
 <binary> query ibc-transfer denom-hash [trace] [flags]
 <binary> query ibc-transfer denom-trace [hash] [flags]
```

本教程使用来自 Cosmos Hub 的 gaiad 二进制文件，继续前面的示例，其中一些 OSMO 从 Osmosis 传输到 Hub。


> 尽管这里使用 `gaiad` 作为示例，但您应该使用您感兴趣的资产所在链的链二进制文件。 `transfer` IBC 模块需要在查询 `denom_trace` 时查找它存储的映射。

安装 Gaia 二进制文件：

```bash
git clone --depth 1 --branch v7.0.0 git@github.com:cosmos/gaia.git
cd gaia
git checkout v7.0.0
make install
make build
cd build
gaiad version
```

按照 gaiad 子命令查询 denom 并了解代币来自的通道：

```bash
gaiad query ibc-transfer denom-trace 14F9BC3E44B8A9C1BE1FB08980FAB87034C9905EF17CF2F5008FC085218811CC --node https://rpc.cosmos.network:443
```

输出

```txt
denom_trace:  
  base_denom: uosmo  
  path: transfer/channel-141
```

从终端输出，您现在知道有一个 IBC 端口`transfer`和通道 `channel-141` 对应于 Hub 和 Osmosis 之间的 IBC 连接。 要了解端口和通道背后的 IBC 轻客户端，您需要执行另一个查询。


> 轻客户端是如何得名的？ 这些是链上客户端，仅跟踪交易对手链的块哈希。 这允许链在 IBC 上创建不信任的连接，而无需客户端完全复制交易对手链，从而使此类客户端“轻”而不是“重”。

ibc 通道客户端状态传输查询列出指定路径的客户端详细信息：

```bash
# 就是以上查出来的path `transfer/channel-141`
gaiad query ibc channel client-state transfer channel-141 --node https://rpc.cosmos.network:443
```

输出:

```txt
client_id: 07-tendermint-259  
client_state:  
'@type': /ibc.lightclients.tendermint.v1.ClientState  
allow_update_after_expiry: true  
allow_update_after_misbehaviour: true  
chain_id: osmosis-1  
frozen_height:  
revision_height: "0"  
revision_number: "0"  
latest_height:  
revision_height: "7030941"  
revision_number: "1"  
max_clock_drift: 600s  
proof_specs:  
- inner_spec:  
child_order:  
- 0  
- 1  
child_size: 33  
empty_child: null  
hash: SHA256  
max_prefix_length: 12  
min_prefix_length: 4  
leaf_spec:  
hash: SHA256  
length: VAR_PROTO  
prefix: AA==  
prehash_key: NO_HASH  
prehash_value: SHA256  
max_depth: 0  
min_depth: 0  
- inner_spec:  
child_order:  
- 0  
- 1  
child_size: 32  
empty_child: null  
hash: SHA256  
max_prefix_length: 1  
min_prefix_length: 1  
leaf_spec:  
hash: SHA256  
length: VAR_PROTO  
prefix: AA==  
prehash_key: NO_HASH  
prehash_value: SHA256  
max_depth: 0  
min_depth: 0  
trust_level:  
denominator: "3"  
numerator: "1"  
trusting_period: 1206000s  
unbonding_period: 1209600s  
upgrade_path:  
- upgrade  
- upgradedIBCState
```

> 本教程只讨论单跳的 `denom_trace`。 有关多跳和前端服务后果的信息，请参阅 ibc-go 文档https://ibc.cosmos.network/main/apps/transfer/overview.html#multiple-hops 。

这是很多信息，但它没有回答这个问题：你怎么知道这个 IBC 客户是否值得信赖？

#### chain ID和client ID

花点时间考虑一下这个问题：您将如何识别一个链？

初始回答可能是chain ID。毕竟，这就是字面上的链标识符。但是，链 ID 不是唯一标识符。任何人都可以使用相同的链 ID 启动一条链，因此这不是一个很好的参数来验证您要连接的链的身份。

但是，IBC client ID 是由 Cosmos SDK IBC Keeper 模块生成的  [ibc-go/keeper.go at e012a4af5614f8774bcb595962012455667db2cf · cosmos/ibc-go (github.com)](https://github.com/cosmos/ibc-go/blob/e012a4af5614f8774bcb595962012455667db2cf/modules/core/02-client/keeper/keeper.go#L56) （ICS-02 没有指定 IBC client ID 的标准）。这意味着在IBC看来，一条链是凭借`client_id`来标识的。

一种链名称服务可以验证链 ID 和客户端 ID 的组合。目前正在使用或正在开发中的有几个选项：

- 链名服务（链上、去中心化）：

   CNS [tendermint/cns: Chain Name System (github.com)](https://github.com/tendermint/cns) 旨在成为 Cosmos Hub 有一天会运行的 Cosmos SDK 模块。作为跨链交易的hub，Cosmos hub只有托管有关如何到达其他chain ID 的关键信息才有意义。 CNS 目前正在开发中，后续会发布更多信息。

- 链注册表（链下，半去中心化）：

   链注册表 [cosmos/chain-registry (github.com)](https://github.com/cosmos/chain-registry) 是权宜之计。每个chain ID 都有一个文件夹来描述它的起源和一个节点列表。要声明他们的chain ID，区块链运营商必须分叉注册表存储库，使用他们的chain ID 创建一个分支，并提交拉取请求以将他们的chain ID 包含在chain ID 的官方 `cosmos/registry` 表中。

   每个chain ID 都由一个文件夹表示，在该文件夹中，`peers.json` 文件包含您可以连接到的节点列表。

能够列出所有可能的区块链路径仍然是一个未解决的问题。一些生态系统的努力已经在发展，以帮助弥合这一差距。以 Pulsar [PulsarDefi/IBC-Token-Data-Cosmos: A package containing all IBC Token Data, across the whole Cosmos Ecosystem. (github.com)](https://github.com/PulsarDefi/IBC-Token-Data-Cosmos) 的这个 IBC-Cosmos 存储库为例：它试图在所有 IBC 连接的链上聚合所有已知的 IBC denoms。他们使用以下数据模式（data schema）：

```json
{
    "ibc/HASH__CHAIN": {
        "chain": String,
        "hash": String,
        "supply": String,
        "path": String,
        "origin": {
            "denom": String,
            "chain": String | List[String] | null
            // null if we cannot find this denom on native_token_data.json
            // list if we could not pick correct chain e.g: [terra, terra2] for uluna
        }
    }
}
```

使用此数据作为源，可以编写一个 API，允许在不查询节点的情况下查询path、base denom。

中继器可以创建从新创建的区块链（没有确定的身份）到另一个区块链的新通道，而不会泄露太多信息。 在 IBC denoms中存储路径信息，您可以追溯并从通道检查关联的客户端，使我们能够估计资产的安全保证。

#### 确保IBC客户端没有过期

除了验证链（或者更确切地说是轻客户端）的身份之外，另一件需要考虑的事情是轻客户端是否过期。

如果轻客户端在 `TrustingPeriod` 内没有得到更新，或者在提交了不当行为的证据时，它们可能会过期或被冻结。例如，如果 Tendermint 共识失败（如果超过 1/3 的validators产生了冲突块，也称为双重签名），并且在链上提交了这种共识失败的证明，则 IBC 客户端将被冻结，具有非零的 `frozen_height`。

在前面的示例中，`gaiad query ibc channel client-state` 的输出确认了客户端状态，因此您知道 IBC 客户端没有过期。


要了解如何通过提交治理提案来恢复已过期的客户端，请查看 ibc-go 文档 [How to recover an expired client with a governance proposal | IBC-Go (cosmos.network)](https://ibc.cosmos.network/main/ibc/proposals.html) 。

`latest_height.revision_height`是IBC客户端最后一次更新时的区块高度。在前面的例子中，为了确保区块高度仍然是最新的，你必须查询区块链本身的区块高度 `5901208`（或执行查询时的最新区块高度），并确保该时间戳`block + 1209600s/336h/14d`的`trusting_period`在当前时间之后。

例如，您可以使用查询验证 IBC 客户端状态：

```bash
gaiad query block 5901208 --node https://rpc.cosmos.network:443
```

```json
{"block_id":{"hash":"2C419625E506D928D55D1818FC8F16CCAEBD87F5EA66D40F511EF5E70BEB6471","parts":{"total":1,"hash":"AEFAB923462B6F8D5C7FB9D1A450F1DC6B28A4CC7376390A761CA93337280A25"}},"block":{"header":{"version":{"block":"11"},"chain_id":"cosmoshub-4","height":"5901208","time":"2021-04-18T01:22:14.006419503Z","last_block_id":{"hash":"008884190C21C603E758FD8271946C3DD5E8A2A9F74F20ED00F4FA2376425C37","parts":{"total":1,"hash":"7B3DAFB0A30F043CCF9D4D74A6EEDAAB5CC52A2E7E4BF8113E8EA2A1FED59F68"}},"last_commit_hash":"BCFF383EC45F0454A17B2A4F08DF71C8A630A567B9C69170E647E53E3E8E4957","data_hash":"E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855","validators_hash":"B2BFC39012E931F7980FBCDFE52D4461EA2D6387180F44120B0B0D129919F5C8","next_validators_hash":"B2BFC39012E931F7980FBCDFE52D4461EA2D6387180F44120B0B0D129919F5C8","consensus_hash":"0F2908883A105C793B74495EB7D6DF2EEA479ED7FC9349206A65CB0F9987A0B8","app_hash":"44CCA5932610E4790081F4C39AFD72999ED5BBFBE5E24AA9D35422F369A85C7F","last_results_hash":"3F51C34758AEB30A69294CD60758BB6F93FB5663DEA74B3A4B0D484184234941","evidence_hash":"E3B0C44298FC1C149AFBF4C8996FB92427AE41E4649B934CA495991B7852B855","proposer_address":"A6935D877B9776C45B96EEAE526959A3B9A5AB1A"},"data":{"txs":null},"evidence":{"evidence":null},"last_commit":{"height":"5901207","round":0,"block_id":{"hash":"008884190C21C603E758FD8271946C3DD5E8A2A9F74F20ED00F4FA2376425C37","parts":{"total":1,"hash":"7B3DAFB0A30F043CCF9D4D74A6EEDAAB5CC52A2E7E4BF8113E8EA2A1FED59F68"}},"signatures":[{"block_id_flag":2,"validator_address":"83F47D7747B0F633A6BA0DF49B7DCF61F90AA1B0","timestamp":"2021-04-18T01:22:08.047075563Z","signature":"XepBuFkrPTq9SHUWOMLSsMqNxT45I47G8EAac4q1uz6xNqCahFVfCvIOl6MXfYlbtS+DK5s1MjBQC/EXbVAfBA=="},{"block_id_flag":2,"validator_address":"68F27079590DCACC3AC1C4892A1A008F348F596F","timestamp":"2021-04-18T01:22:14.006419503Z","signature":"MNpUqrfo+hkw2WQD/Q22uAYATnBB1YW1GKbf+GwLHpzi8CYY/wPJrbmKl4nAmHRTUYu8AW44I1Lr2/v0ZsfZBg=="},{"block_id_flag":2,"validator_address":"2199EAE894CA391FA82F01C2C614BFEB103D056C","timestamp":"2021-04-18T01:22:13.969246339Z","signature":"Nptl71NWmvYUXuNrC+zS5eXiC9T8aI1x1PGqAW1dU0o3QrKVYAb5M+1LlgXyZs4RvPTm1Eoz7zB93NpPZyPaDw=="},{"block_id_flag":2,"validator_address":"AC2D56057CD84765E6FBE318979093E8E44AA18F","timestamp":"2021-04-18T01:22:14.082255452Z","signature":"OA0km/EHI8P0TJVF6kEYJeDsEbOC4sZQHb/n5nBCB+gYqu7uLVnydAGHKkz+KxpX69hnBhn+yyc1zO5iBpUtDA=="},{"block_id_flag":2,"validator_address":"F8C01C0681578AA700D736D675C9992065F65E3E","timestamp":"2021-04-18T01:22:13.941031374Z","signature":"rqBEP0CUJxuzxrxFT1VvAdemeTxHXrh9oJn9A1V2CYYkpirz0pefduZx54rgts4Rq57wWuEWhdW6IoZKMoapDg=="},{"block_id_flag":2,"validator_address":"F59734A896A7689436BC3422244FD862AE189C5C","timestamp":"2021-04-18T01:22:14.163141298Z","signature":"JiynnzStJp8ur5bY0N99lKxkRYD2KUZyJH7q4IyVwU/KlAI3NcM4AiKyiEEJsZzUwpG++t7p/ZSXrgtHMcdwDA=="},{"block_id_flag":2,"validator_address":"679B89785973BE94D4FDF8B66F84A929932E91C5","timestamp":"2021-04-18T01:22:14.047519114Z","signature":"EvHT6dqShMfPuR082e0t9RXdcuHxPls+SPllXJqkXRF2+yioeoi6a9G0aCHldA4h9OtII94p/NTrpYtFTbWsCg=="},{"block_id_flag":2,"validator_address":"ED509E78097E1306A91FEDE8E85B75D06BDDF6E3","timestamp":"2021-04-18T01:22:14.097547913Z","signature":"P8rI7ZaYOAoVO1gBQybmsN/M1AbfhzuTGGEcmICnFAMbLUizeHiwjDlimNLtiQhzJ8hO2ib4wLDHEOn4KUexAQ=="},{"block_id_flag":2,"validator_address":"33C99B2E79B887ADA2228E53FB6EA000AB983337","timestamp":"2021-04-18T01:22:14.084834888Z","signature":"YW4zeUlhAaPvTFy9PA44hz3EWrzgzA+Nie8QqZtgeyLiVTJn7JMEUkIPqpZlHtTgsUJxi5QkSU8fc7YrWug1AA=="},{"block_id_flag":2,"validator_address":"31920F9BC3A39B66876CC7D6D5E589E10393BF0E","timestamp":"2021-04-18T01:22:13.973663864Z","signature":"UHix2qaDby4l5wrdlmWsMiLGXe3TRB+X5ixJrt716/TjzuhzQQddW4t5++KMdXZzl/JKVFsOGPwCRk+YQmOzCg=="},{"block_id_flag":2,"validator_address":"51205659A717DFFB96E054F8BD1108730E17AEA7","timestamp":"2021-04-18T01:22:14.063902574Z","signature":"lvsSCyO0ApalKxOXVUBYd/pKxRnSAYXRu2DL5D2FPMocyr8+6FhdEbL+B8uKWek0OtUoCCjue9vCuL9Es5ftAw=="},{"block_id_flag":2,"validator_address":"099E2B09583331AFDE35E5FA96673D2CA7DEA316","timestamp":"2021-04-18T01:22:13.925955591Z","signature":"eU1cfKpa0ebvPdzcN0vBm7y7eZ60y1Hbglvo35oJn3J2in+OXckBGS6NxJ76ymbZYTJwFXZMwX9y+pYnkYp0BQ=="},{"block_id_flag":2,"validator_address":"95E060D07713070FE9822F6C50BD76BCCBF9F17A","timestamp":"2021-04-18T01:22:09.357898414Z","signature":"HgMxfo1lrm6L7Rs92FHB3+lHUQq4a8K4bEWklqH9F/PErPH5ICMRzNa3+jeHXAzNuTvZCdfomhkUDyKISIEnAg=="},{"block_id_flag":2,"validator_address":"D9F8A41B782AA6A66ADC81F953923C7DCE7B6001","timestamp":"2021-04-18T01:22:13.967389923Z","signature":"ZUHILXFFDBIZZdp8Qt7f26kv0YtCKEi3LE08AFmcsk1pA4Jho59bJE/dnkEoy6XDpt7EJOX77j0AbECBzjMyDQ=="},{"block_id_flag":2,"validator_address":"B1167D0437DB9DF0D533EE2ACDE48107139BDD2E","timestamp":"2021-04-18T01:22:13.979141963Z","signature":"VEy1mvcPo2jtVw/YztP4f0CSAzVvBi1wb9+ZZxziv5uWcVtaKe4fKu7RNBmZ2ABU5J5oSep++FwEznW1EYLHBg=="},{"block_id_flag":2,"validator_address":"DA6AAAA959C9EF88A3EB37B1F107CB2667EBBAAB","timestamp":"2021-04-18T01:22:13.98176581Z","signature":"LuBYt4s+SemqA8mqyP+GXnSslpPiwWpWerifKn4tgfTXmNKguOeIwv/ch+RWKagYFl+30SKnw0K17VDfH3jlAA=="},{"block_id_flag":2,"validator_address":"B00A6323737F321EB0B8D59C6FD497A14B60938A","timestamp":"2021-04-18T01:22:13.805669443Z","signature":"9pMeMRdqCC6U99AeMTFrNy9RKGGeB/y3IqaSlInwB8MpnWVTZ9r/eR1mzA8SRdDlL37cAU0I4P9AfFVzSl6wBQ=="},{"block_id_flag":2,"validator_address":"A6935D877B9776C45B96EEAE526959A3B9A5AB1A","timestamp":"2021-04-18T01:22:14.317397227Z","signature":"Rtqv6RwKuO1A+L2e/Wxnu1uro1+XHSWwPBMS7yOfgGotZXBu3RBRWbDl8W5f3fVfjNfZZ4qdszm1dv6XhIx0CA=="},{"block_id_flag":2,"validator_address":"05859E9862A6680DBEAB510362C10B1661B9743A","timestamp":"2021-04-18T01:22:14.129623405Z","signature":"tJ02A9ZqeuBN6kWNEd0EuLbKqQ4uvhUczsWYOitIM/th8pQKtiNq5pPfr9qu8u/MSbHksICWh1l1Oy2MG2jUCA=="},{"block_id_flag":2,"validator_address":"D2D458F9209ECB8CA2AAB1D99E06611B812A8797","timestamp":"2021-04-18T01:22:13.95433751Z","signature":"s0xb3GF/MNrGsExjx5U6mvjoOrKVn2SoPY+YsA4wCLssVWGq28n8TqGFNrn4Fvze+fzx0KDiw9ozNaOZXHPgBA=="},{"block_id_flag":2,"validator_address":"1CED30733D1625C89AB698677606D0E37B3676A9","timestamp":"2021-04-18T01:22:13.958606829Z","signature":"cySc1i/5W+3lc4QFYIQm95TCZ3WXQQh8UUroXwyrRAVdR2c3HEtmcwGo/+kbN2OXf3T5BD2z7aTfFxS2ETEIDQ=="},{"block_id_flag":2,"validator_address":"9C17C94F7313BB4D6E064287BEEDE5D3888E8855","timestamp":"2021-04-18T01:22:13.979484236Z","signature":"LSghajsKHPleBdAbXVmYX7CSk8mDz3nMF1bEDeUKEl1fE8EcSabZ4P0xdJneg5f6jsU62iXlPOFfuc1whZ4vBg=="},{"block_id_flag":2,"validator_address":"CC05882978FC5FDD6A7721687E14C0299AE004B8","timestamp":"2021-04-18T01:22:14.037811426Z","signature":"x6EZTvrlzXqlskNsxmxCIAuKLyqfiFNNe0Rppg0ZKhqHObKQPWxssj43wuDPbXPnJRXc9qXRcD0afrd7g9EpDw=="},{"block_id_flag":2,"validator_address":"696ABC95186FD65A07050C28AB00C9358A315030","timestamp":"2021-04-18T01:22:14.084703223Z","signature":"IUHS7aYqHklmQbuipAdhQRxhRuPzcMIfvubtup1kVGpSpRZcskcugluWITiI+djtDN1JtTErrChwUCvfOshxAw=="},{"block_id_flag":2,"validator_address":"81965FE8A15FA8078C9202F32E4CFA72F85F2A22","timestamp":"2021-04-18T01:22:14.021819383Z","signature":"Cyb4K2KUp9eYQrbBRMGTV38Qsgcs+e41+tq9Ff2ZeECQ9QAvUX/5NYsBCh2Pte9wh/MpGQ902iPE8COh5DU+CQ=="},{"block_id_flag":2,"validator_address":"57713BB7421C7FEB381B863FC87DED5E829AA961","timestamp":"2021-04-18T01:22:14.045708198Z","signature":"1N8wBQVit0v0zvQh1mR7WKeqXrIHM08wPogh+Uf7DGnyB3a2bfDzFRmuF00i/tjuqDvtTEI0Ahhhh2uX6STmDA=="},{"block_id_flag":2,"validator_address":"671460930CCDC9B06C5D055E4D550EB8DAF2291E","timestamp":"2021-04-18T01:22:14.092963567Z","signature":"u8RV+QiBe65r5dUzrtkwPfcPWIvdClg5UZmLbHzxB6fH7BHaKrCshiM5otPpwXtyFrMORh71gBzqet1ffHQGAA=="},{"block_id_flag":2,"validator_address":"75DAB316F4CA1367F532AB71A80B7FA65AB69039","timestamp":"2021-04-18T01:22:14.058823302Z","signature":"8J74y/cfdm5c8dnpmP4VDIGVuXL94nibdgkAr4AO0XixYp1fuwkAZbK1wMeefAStjiwaOh+HAEo8rjFcZefODw=="},{"block_id_flag":2,"validator_address":"E800740C68C81B30345C3AE2BA638FA56FF67EEF","timestamp":"2021-04-18T01:22:13.955801894Z","signature":"FwlIX6JqtKzLRV1zk+6+FO71E5ZWDeR8Iy0EPz74W/UTRntp3uT7aC9kEIuDDHSVnp/+AHFLbX9i9qR8AJrtBQ=="},{"block_id_flag":2,"validator_address":"8FE3CFFA6A07B093E441BB84DA1B6DABF53AFA2D","timestamp":"2021-04-18T01:22:14.005328546Z","signature":"jNxl16sxxxA0L0T7oh7subP6pI6/5C8O7t6QYowVaVfIOYw7jNYYBe/tUt0RrkuvTWZX7yF28QkTnOMJao1hBg=="},{"block_id_flag":2,"validator_address":"D14A542E8756C3A942D9FD8873DC2E9A7798A17F","timestamp":"2021-04-18T01:22:14.052800231Z","signature":"6bOlv96qqnFwsdB2Rh14pQfYMPsnGRixAFrGwrfU4GqI+s2sHjB3bc2vlymkAZlQs6/5Bqj+TyeqxrcrVjHPDw=="},{"block_id_flag":2,"validator_address":"FD5D54E0D9E4768FEA4C0DFFDC89FA96B6657F32","timestamp":"2021-04-18T01:22:14.116094181Z","signature":"Uj7ZxfQriNHHRs8Ym4Drnu3tdp0cC6NXoAY/D0fO5BmJPezTyBdLQNkH6JJKSG+yN7JkcElW2bURfO5N45t4Dw=="},{"block_id_flag":2,"validator_address":"EE73A19751D58C5EC044C11E3FB7AE685A10D2C1","timestamp":"2021-04-18T01:22:14.014072057Z","signature":"bIzcjiQdNDHm1vbW1DrmEjjAUgW0r7uA0wGHgUKJ4w7HvM9ks72yk8RVQA5yEDfHaHaIw9c9LXfWM5GSYCjnAw=="},{"block_id_flag":2,"validator_address":"019B9CA2944D3CC36C7C73283EF3D58E56C8A5D4","timestamp":"2021-04-18T01:22:14.000141344Z","signature":"DxI9k433GrZ1HTlB7UmrsXE8rHgjlAAGZ1suy6SvFXCL/PO6mJNOBNGStWZ+UjS3y7dpWQDrdSSBg5OdF92pDQ=="},{"block_id_flag":2,"validator_address":"49BBFB1BA1A75052E3226E8E1E0EFEB33918B8B2","timestamp":"2021-04-18T01:22:14.010348812Z","signature":"3RAcE6Q/ygxmoaU+/Y8oBT/OU1eoLPHVFgf4FkblgQLA/FLsK3vmDKclOTVfqvusoBMaLagoS01uKMTmUHxdBw=="},{"block_id_flag":2,"validator_address":"BAC33F340F3497751F124868F049EC2E8930AC2F","timestamp":"2021-04-18T01:22:13.980777669Z","signature":"jpR36L7mnye4477/9jXMWym5RY2eaoa/xnMBRuylOxvVw2/NoEYuNGMy/l4+jqRCeSuHkiIXuuWQQeuHaa0yCQ=="},{"block_id_flag":2,"validator_address":"C2356622B495725961B5B201A382DD57CD3305EC","timestamp":"2021-04-18T01:22:13.974348109Z","signature":"Io3PMqXdJdu29HDzLd8vMhuWhHtKuX82njijpWgg1+/S2QhwCfByCHKBEOuNKl5lFqi0Sv7iXqR40eFddvRyDw=="},{"block_id_flag":2,"validator_address":"7B3A2EFE5B3FCDF819FCF52607314CEFE4754BB6","timestamp":"2021-04-18T01:22:14.054451607Z","signature":"9Echbdq1uLdiBh/K/xL4TY0j3d/NbiiFNB0OrBgTY+WJM/LpPrB0UZt6rASSUOJoSwD2YP23qrSeG6+tIfPFCw=="},{"block_id_flag":2,"validator_address":"B4E1085F1C9EBB0EA994452CB1B8124BA89BED1A","timestamp":"2021-04-18T01:22:13.928297112Z","signature":"css2cBJ3qjuCswL/klgoSvzkmTTNFBK3Yz0yVFBacc76Zu7AJ2YTPyJQcZvemD1jkzAGKEIEyh+WrS6SGwKjAA=="},{"block_id_flag":2,"validator_address":"9DC4012099BE743189074B85E49891AE3B3FEE9B","timestamp":"2021-04-18T01:22:14.031924823Z","signature":"nhC/vbxoEfrFrpIHWVMB+AJqkx1858aJDHGCXRLQgbEOrCBGNajdXtTh1MDA37jQNbVtwTYA6yc7rm7LCUJlBA=="},{"block_id_flag":2,"validator_address":"8E0EE37B7B1A038DD145E30F1EF97DF3619EF429","timestamp":"2021-04-18T01:22:14.236981546Z","signature":"W4wwgBV4Xaz4KwCC8x9GJ/nv9iM7ejBwDzn24F0O3oynTs716DDf/PMJvdvC7yV8cTS2Z5lMc4htJ+2CNgOMCg=="},{"block_id_flag":2,"validator_address":"8CE843E04C48B7864F8568FA39E90EA13FE6586F","timestamp":"2021-04-18T01:22:14.100641765Z","signature":"CgsRuEfQiTyFmaK3PsilMt4v57SIeAwA7nZON8rJYquY8zW00pnc0VlUDGiVztwc3H+ev+3NNgw9LqV/z5+qDg=="},{"block_id_flag":2,"validator_address":"4AF69D6A5436C30E3584C1628433DE55E758BCCA","timestamp":"2021-04-18T01:22:14.113875428Z","signature":"CiqjyslzYXOMFvXw9WszTcxEZR5fzTuVqLts5Ar++306sPzgbOloRttiFLSS9qYcxF26lyerPO+xSF+uDOskDg=="},{"block_id_flag":2,"validator_address":"4EB1282675F724B59026F2173C23F0DC9936F118","timestamp":"2021-04-18T01:22:14.065699233Z","signature":"WXE4hnB+wtefKdQOeBYv2aOAL60Bl1mGWGtUIXMgTB4SPFECFRkluyf8CXhA9bQit8W/x5jw5RxMnjldJX+WAQ=="},{"block_id_flag":2,"validator_address":"91C823A744DE50F91C17A46B624EDF8F7150A7DD","timestamp":"2021-04-18T01:22:14.050997785Z","signature":"UOpRVVgefcnN6shakFVuv396gtxbY2doDUQ84awtPGEwVAsqnZJQioLkIpLiNOL7rt7WyvucvwmsBTtaHdRVBQ=="},{"block_id_flag":2,"validator_address":"46A3F8B8393BAA153C40E5722EAE82EA0D48B32D","timestamp":"2021-04-18T01:22:14.153133495Z","signature":"58ITKszqF5l8+uqeWPiT1RHVQlE2zsecb0dlMbYEnG3a7jCIUsyb/BkV15NaRGXgBkfBb2o8lig5xsZyxWGHCg=="},{"block_id_flag":2,"validator_address":"B0155252D73B7EEB74D2A8CC814397E66970A839","timestamp":"2021-04-18T01:22:14.13666307Z","signature":"XJtSxE0XdBMQKNXqd5wVnZl7/K3H+k6BkZcyLDiPDjzRMXcdK0pSSJAl+8oywQcUMBxyTRt3U3NX5AaQZ/A1BQ=="},{"block_id_flag":2,"validator_address":"1AE0BD432F9A5122474A646325D1AFA6068692E9","timestamp":"2021-04-18T01:22:14.143443845Z","signature":"lhFfpAClIUf2wX2EM+G7/vycn6cl1ylwwgO3adTeyy4TX18iRTMYmj6P6DBiseySnoccyuHJA2UNosetGt1lAQ=="},{"block_id_flag":2,"validator_address":"2C9CCC317FB283D54AC748838A64F29106039E51","timestamp":"2021-04-18T01:22:14.120082038Z","signature":"dqFvUp07DiYRDUAe2XduSx0YjQvpW3VLYPu2SLj/n8u3l+Mbxdp2rJ9zuR8GDZR/Xs1ylTa3BMyFCHUI5YBJAw=="},{"block_id_flag":2,"validator_address":"9DF8E338C85E879BC84B0AAA28A08B431BD5B548","timestamp":"2021-04-18T01:22:14.034467313Z","signature":"gfYqJHl72T0JGPYtcf5CYa9ogSKAwdpwT3Dmd7HxGlTi0f3Cum3qKKsNcbNu1fobCHVbQqdIW1dR0TPwTyZ2DA=="},{"block_id_flag":2,"validator_address":"7B3D01F754DFF8474ED0E358812FD437E09389DC","timestamp":"2021-04-18T01:22:14.012714104Z","signature":"4nb6JYBHlbumIba8AxrrXHosq5NhRZgCeyZDWkg+oqH0VQgHznd2MPvKd4CMi5tv3ZNFAVKiETJxADi86AmZAg=="},{"block_id_flag":2,"validator_address":"B4999CD535E4CD32B590BEB47020A724F40B65E5","timestamp":"2021-04-18T01:22:14.01538347Z","signature":"vLkf/GImwksgan6467gaox+FL47aNLunmbeX8yQvNzqkzgASCgsy8zneWTOQlKAt+lTaav8eV/TLo0ijtayWBQ=="},{"block_id_flag":2,"validator_address":"E83BFC436D2CE8DCC9EC0589B2E5B735E37FB85C","timestamp":"2021-04-18T01:22:13.932227896Z","signature":"rbGGpUQ32V0NhFwqg9H6/TPli00QsTal4IPk0dq6emJ+ZUItTicXM8QtKxuEHvuXkcqf6S9bPGBB9vorwwDYAQ=="},{"block_id_flag":2,"validator_address":"C52ACDB32057F5C731BBDD48460B93C3500DD324","timestamp":"2021-04-18T01:22:14.143671392Z","signature":"IqdG2oFTSVht2B4nDvvQ7T06RavvzlDyLjZ+KxoBL09e43qQGIhmj2f1iID5oZDq+RFsT+O4UJoiYiJOpHD3CQ=="},{"block_id_flag":2,"validator_address":"52E1646134432BF9532B4881C6ED32E40AE5A2DD","timestamp":"2021-04-18T01:22:13.966498952Z","signature":"6GUwrKqyxk9Cp3WqSMaWGaR9s2bRwxr2s7LTCc0jp4CsJLuVnaCpUJ38xJwNoxYZJHdeysA1E7xbATS35W/YAA=="},{"block_id_flag":2,"validator_address":"CC87F56B58621811E2B5A47F38C6166E295CE36E","timestamp":"2021-04-18T01:22:13.935741303Z","signature":"WxylWGRkWLxUKbFRihnqtWdN7JWDyJDwHej8ws12MgkWbJQw68aoClokNuzJLzWNBmhzgyaFPkqwhKsHnTy4AA=="},{"block_id_flag":2,"validator_address":"21223475CE86F3C7CD5E985AA88FC24A29C97813","timestamp":"2021-04-18T01:22:14.112625163Z","signature":"1BGeLPL7F6R+FuXfxY3J6rA7XQ73Xg0KbaEjkroyCOGEz6Tko9lYfy4qp+0aJP+jlN49PaDv1l1rhPoHxPZVAw=="},{"block_id_flag":2,"validator_address":"399671C2FE4B2714EC6E87D4EE454EF15F33AA2A","timestamp":"2021-04-18T01:22:14.134866819Z","signature":"fePLWfJLrusdaliEYuCAA0K/Rn5BIgy+br13F48MihCcvexWsDSq9TX8nNrNCZjjyjAVV0xg6n8pAMl0+YliCg=="},{"block_id_flag":2,"validator_address":"B543A7DF48780AEFEF593A003CD060B593C4E6B5","timestamp":"2021-04-18T01:22:13.960011524Z","signature":"Xg+oaNmsRu+m1U25T5f1gBO7O43YTVqZqf+gLTO5/o/UqL9FdV3eitCm8EOwqCMX2GHAK/w+eGCrfD0jljfnCQ=="},{"block_id_flag":2,"validator_address":"D540AB022088612AC74B287D076DBFBC4A377A2E","timestamp":"2021-04-18T01:22:14.596777286Z","signature":"Eo1BArAN2QIDYlvBUz4PIGIN9d14j1jEX3Vzdr9V1cZyiT81+WNXMdRdpzNm3+j/ssLEpzuDqRzFNL3f2Jg4Bg=="},{"block_id_flag":2,"validator_address":"C2DDD9700CF5DEC0457DC423829B31EA8FD4F9D4","timestamp":"2021-04-18T01:22:13.958645686Z","signature":"Yf5JnURrODyKXITb6g81Wm4NckZuWqvFxa4lz9f3mnTvNh+4AIQ3awTvMk1b1qRDnfBloAluRQglTPBrar0sAA=="},{"block_id_flag":2,"validator_address":"808D6B054A0B6D3FF5F5EAF0A65CFC64C543F833","timestamp":"2021-04-18T01:22:14.037969149Z","signature":"fZwwvyEfVmqW09V95RfhupTWcG9zrQi+NMLf1fpoX9e1b17CJU2ohLTCgCknYQAW68w5SffsVBttC/ILXO9WCg=="},{"block_id_flag":2,"validator_address":"51DB2566204EE266427EA8A6CB719835AB170BE9","timestamp":"2021-04-18T01:22:13.950229906Z","signature":"m8t5A6koWkyMcJwfJpI89MI6F6gC0atBj+/AVVXcD7GH8PS4/dUZ/0rC2eNmPgx/tJPghQBGK2EGinsQgDG6DQ=="},{"block_id_flag":2,"validator_address":"68F5BBEACEF114C720EA9C98BFA2FFDE01C54FD1","timestamp":"2021-04-18T01:22:14.120978359Z","signature":"VVQVk0CUiv1b8WCmUV0CMqsJH0+ifmxveUc2wFd0INl6hcjOYcuCvwQ2ZLiCV63ejTD8CiHe4BcvwqIZqV1yDw=="},{"block_id_flag":2,"validator_address":"BF4CB4D59D19D451CF5E7BC49349DF4AA222D78B","timestamp":"2021-04-18T01:22:13.925122486Z","signature":"Q9jDEaz69l6x7+UaYGAjjN9VI4iNtWPQCohso4+c18bIg/dC1DBsAaj6YDyDLn6fyh6mkjF6UU1QgfUbw1kOBA=="},{"block_id_flag":2,"validator_address":"D7F7C79487C10A5CF1ABEB1DBD81E8D49757C422","timestamp":"2021-04-18T01:22:14.162299864Z","signature":"f6z0ekW1cFJMdF4dn56PrLa/l3ol3QJRVhR0OsBHJ3f6+eeA20/mJ6yLmdnWn0+d6Q0AcUdMIktJoicYchH3Dg=="},{"block_id_flag":2,"validator_address":"42D6705E716616B4A5442BDAA050B7C6E9FDDE43","timestamp":"2021-04-18T01:22:14.076294385Z","signature":"u3bKWBUviw11UGhtQdMyIIH2sDX595ZwCK6xEbMIYFXJQxkbCsmg+ztN0NP74NoeNyadG7ZzrNK0k2wuRrTDBw=="},{"block_id_flag":2,"validator_address":"2A9135E88160AF052C9ABBB3F6301D7D99E23848","timestamp":"2021-04-18T01:22:14.012721168Z","signature":"pHkCuwNlmYARafSvVQGN0l3F0j4NORzuL71a9y0lAlWnWsU/zWelno8PWWRMrkKXnKey827rem9au93qkVeGCg=="},{"block_id_flag":2,"validator_address":"B0765A2F6FCC11D8AC46275FAC06DD35F54217C1","timestamp":"2021-04-18T01:22:14.149639366Z","signature":"FGWAcWykfnk3wUqkeIEZIRDtHbNMelchV01IJBASfuk6Em5bYuR7YuN0W/j3//mEx42qrCKGH4noY2NPehWBAg=="},{"block_id_flag":2,"validator_address":"3363E8F97B02ECC00289E72173D827543047ACDA","timestamp":"2021-04-18T01:22:13.985549531Z","signature":"ikgF2CKXqK/tZOqwjAe+GCCgKS5NeZhF0fpglc0zha5DgNr3PPdcqAMI+RNES/Ds95Sa5VAcFUqAMtKSgaN3BQ=="},{"block_id_flag":2,"validator_address":"000AA5ABF590A815EBCBDAE070AFF50BE571EB8B","timestamp":"2021-04-18T01:22:14.097161407Z","signature":"RBgEcWpHMmY36Fj/OB2kMXDLfWgPZ+Qr0ODS+0ISb0X/vU8GCduutNGVAaDWiPdqFC9jMkRld3kVXvrL9NlDDA=="},{"block_id_flag":2,"validator_address":"25A9D452D35F12050ADEE6B31934BB85C2817D76","timestamp":"2021-04-18T01:22:14.013673775Z","signature":"63UKwvzAGgHWs6GTOj5vD6i1TynbGWEhdJMA4Yn5AqiVIzRWhe/QudvlzEKak1U35Q/yVkX9AqZ9QOAWctXdBw=="},{"block_id_flag":2,"validator_address":"25445D0EB353E9050AB11EC6197D5DCB611986DB","timestamp":"2021-04-18T01:22:14.041950424Z","signature":"xF5m3DkZdoUfiOexSpu8CtSYjOPcqQFES7bg2a9D9AVTU7rMABL9nHMOaWVnDB/0wloLZ0pj+FsWZ6+5QDApCQ=="},{"block_id_flag":2,"validator_address":"33B3E03704B6EDA0B8EBBD8F96E210D363447719","timestamp":"2021-04-18T01:22:14.100876888Z","signature":"F7QlQooOPIG10YjsRpC1YI+awZsao/wKxU9oq3ssMiVrDPd+Y7MvJ1p3KUC9Dc1AaRSR0cetxPATohT9W7ZjCA=="},{"block_id_flag":2,"validator_address":"2C2A5E9B9EF903164FA2BD3F140571DDF02A00CC","timestamp":"2021-04-18T01:22:14.034698592Z","signature":"J/aoiXx7T/cs8r3xL51lcohm6G0gJ+YiQ5GYS/rNeFPTtxdkisuggY8oFt3bVBMVRC0/3z6BI3LR1BTbaqdpCA=="},{"block_id_flag":2,"validator_address":"5AB353B748D45F20DFCE19D73BA89F26E1C34CF7","timestamp":"2021-04-18T01:22:14.047438866Z","signature":"0r9+NKOUxvA+VBHkDEGhaGqbhRmjB3V4cmiNkVE0erNbcnmdCZW6VtpozUqxOskaQU3lrgOoldM4aKRjAJ6CBw=="},{"block_id_flag":2,"validator_address":"2C528D2345ED6E953C1C0819E2C3A01ABFCBA557","timestamp":"2021-04-18T01:22:08.047075563Z","signature":"dhjEgN65RdEkmtjiwzIsCgJq3s9szkANDY8lP5zQ0xmT6OIP6wmrCeNoH69t6na2R+rUVCQ9St7BtS/OGFWUBA=="},{"block_id_flag":2,"validator_address":"F80083B183C4019925C72B5F482F2246AC2D4394","timestamp":"2021-04-18T01:22:14.068149914Z","signature":"dRtGsOdF0xQ10mdrnjokK9pwO0ZNN6lOORn+OmJA948BBI/Bp+qTPpZGu2SQg2WdnvB6Z/APky9G9Iy2QjTPDA=="},{"block_id_flag":2,"validator_address":"5731848E19257705AA28CC7EFAA8C708EE014D52","timestamp":"2021-04-18T01:22:13.764444918Z","signature":"1e5dJz3FRCaiWjug8FNaQXFdM5Hpz8GG6ptd4yQ9Ai9EEVq+sZ/LI0YZx+5ELcMsc6Wzdx/O+NmqevVMUMAEBQ=="},{"block_id_flag":2,"validator_address":"70C5B4E6779C59A24CFD9146581E27021C2AEC26","timestamp":"2021-04-18T01:22:13.927288629Z","signature":"ZNWJcO4B9DmELmb4Lhpmdxs53ekYC9bUIPWoG+VzYh7iM6GCSRoXAfPM4aObayPXULOIn29lr4CzvT88M+q+DQ=="},{"block_id_flag":2,"validator_address":"B34591DA79AAD0213534E2E915F50DE5CDBDF250","timestamp":"2021-04-18T01:22:14.058157561Z","signature":"FY8SMdh7lPz/bVLIfk86SppNkOmpfHmr1+o1/K+SmWWHzGd7cjyZW3FcfcEkMHD5z2rX4droG9Ce320/ZqyYAA=="},{"block_id_flag":2,"validator_address":"000001E443FD237E4B616E2FA69DF4EE3D49A94F","timestamp":"2021-04-18T01:22:13.868503628Z","signature":"jlSE2tg8wn0IiexV6Jg6JLT2bG0FIEzPUodE/hGm1kwgm3pgF0c/HV4Np4fLTnHzhw+NIPliHj0adNA76qYaBg=="},{"block_id_flag":2,"validator_address":"B2BF68AD4CED6FE8F71AACAD01003436EBE0729F","timestamp":"2021-04-18T01:22:14.153138558Z","signature":"MhvSvXHrHZfd0HD67InfyPCoo6OO0mmYtsuL0MkPYpFmWcp/zXuYbhzs5JdZZqvY2pjqBDViRBLhyXjpD5uSBA=="},{"block_id_flag":2,"validator_address":"6F322BAF5D73D5C77271941901B1D866476D5C90","timestamp":"2021-04-18T01:22:14.052806915Z","signature":"5aNeiFZQdH75PTMZDXEr8GLgB19UBKYevHq+0dEexOOORrcA6cxbJpBXgbRJqMbfXR0hDzslz0QTomm7lCTaCw=="},{"block_id_flag":2,"validator_address":"732CEEF54C374DDC6ADECBFD707AEFD07FEDC143","timestamp":"2021-04-18T01:22:13.979072215Z","signature":"DQhRmYKBNJMzJZhOND0Tpr9GUatZa2bBXXgQvC3A74JLG+JWGI4AkdX30dYJqFnxvo9QURDQmMIOcFLceoHMBQ=="},{"block_id_flag":2,"validator_address":"4C92230FAC162303D981C06DD22663A4FC7622BC","timestamp":"2021-04-18T01:22:13.950298292Z","signature":"SFANzePH1WehGB2OSyJg9Ktp/hrNmu/vjI/xitqIJ8+CKfDjaC0fjlTJf3H2H+FYhoTiA4p7ASR42BvYUA6bAg=="},{"block_id_flag":2,"validator_address":"FA0E5DFACCDCF74957A742144FE55BE61D433377","timestamp":"2021-04-18T01:22:13.971577473Z","signature":"5m7odbxeWoBvYm1eMGF5f0WwHj2MFQ5JiwN1fn7ohte1fcCx6VeyIXV+CDEtdr7kBLS91jtUPnhjb0AGH4QBAw=="},{"block_id_flag":2,"validator_address":"1E9CE94FD0BA5CFEB901F90BC658D64D85B134D2","timestamp":"2021-04-18T01:22:13.855764108Z","signature":"bYNqpra4QzYLkxvWTXAijy3JTTd75Bo5dyYIXh1YGG5NDSz3LlDwcd+oC0jMkfE8ThoGpwu+VJO3TElsUBPgAg=="},{"block_id_flag":2,"validator_address":"9EE94DBB86F72337192BF291B0E767FD2729F00A","timestamp":"2021-04-18T01:22:13.997832784Z","signature":"NHQ2G6UI2KL+g+sQz0svr8fQZBB66tysLB4MgtBRkV6FlTMl9LX0smFtYEtiVbMcHU4JEuvoTW0gxTHIeJ9VBQ=="},{"block_id_flag":2,"validator_address":"818964B4FB36D28109C3E853778B33231B27C5FC","timestamp":"2021-04-18T01:22:14.048525449Z","signature":"ks1flqItUd3XmSOtD29CALFxqfdhN7UoTQOTXmNgqTQZhLoFVD4a4ItzZM3PJqBFIgPrr9ntuxPPuadaPhhQAQ=="},{"block_id_flag":2,"validator_address":"EBED694E6CE1224FB1E8A2DD8EE63A38568B1E2B","timestamp":"2021-04-18T01:22:14.17993875Z","signature":"SwPUGQXRKZQQ8txRDu1K2bxn4UYMJbtQycQw1hSW0+IJCRAuN51/Ww02vX+UjRTPjyrWDbotl1WDmjHHyW38Ag=="},{"block_id_flag":2,"validator_address":"BB76BC6322C7533A7CCD3AF1C089C7B3971FB012","timestamp":"2021-04-18T01:22:13.978047809Z","signature":"mAESlf0pTmigBKPPdrp9gYtgcKsybogX5RbVBRXITFb1+IoX1IlwvJIymYIG38MADGfwFjBe9kKXiTDZnpkVDw=="},{"block_id_flag":2,"validator_address":"955A47C8AC8632825DD475E90913D40AB09D3FB4","timestamp":"2021-04-18T01:22:14.12904704Z","signature":"MXPYj6BFmicy95+LNm19VLVHfZOuymirLVF5uWdjALXFL/QfbFicjLsDVPYH3EFudiNyTjJj1uWe7pYgEGmaCQ=="},{"block_id_flag":2,"validator_address":"6CB47D786B2F350C13A60BB77D398AC82E900985","timestamp":"2021-04-18T01:22:14.080944557Z","signature":"Y5Ct3CV9hgipjDnM2+8UC8UZBLzuPiwdeWR9Rt6aC06kVds+Cm2VLfneHnEyf2yr91LrmEWw0OyQLiNrc1+UAg=="},{"block_id_flag":2,"validator_address":"2B9A55D3BF93D7375DD207B75C5ED4D2B91D9146","timestamp":"2021-04-18T01:22:13.999813043Z","signature":"EgBHfgJl4HXZkqJqSkfRbvfz25zf7AlBvuo+yu9+aNgLBu6zJhn8zDAtNi6ESpRxNpzTspXPPz7oYuLK7OaZBg=="},{"block_id_flag":2,"validator_address":"846BE4F39E3122D2A2D3FE5454E2561073E95538","timestamp":"2021-04-18T01:22:14.200965584Z","signature":"7EVRhqsf7VNTZ2fSIqcpy8p6LPI2qB5APQolHlfzqRuZcqPbw4T+AUAxMw3+yJpNDqyAKxYS0o+qXLubyrTFCw=="},{"block_id_flag":2,"validator_address":"0D2186876D26D882F7BE50DED92BD3CB53838143","timestamp":"2021-04-18T01:22:14.035402563Z","signature":"ZF4mW1yQI6AYgQnDLscBrFWCnlLa12ZOVxuYGUTIo7nmZx9cvz6OygKqLyo4r731vzagsxS0gmMuF9UeCp0IDw=="},{"block_id_flag":2,"validator_address":"A4F1D5534F3FA905A4DA606E8A10834976511FF7","timestamp":"2021-04-18T01:22:14.038092076Z","signature":"tYTkK3SUQ+CQhGW2VYd2K11Zbs+CleoTnZcHEG/zEWMdsjYQVWddv42urOAojZ6dk6f5NJiSwLZ68zndhyrkBQ=="},{"block_id_flag":2,"validator_address":"9E14352CB5293C6073D61280A197085C6748DAFA","timestamp":"2021-04-18T01:22:13.98446573Z","signature":"2IunEoL59UKOJTJ/JKgndnRjFUQXp3/Dtd5KxU3DKJY6+B6o0mTjrl9fpblQWrtcyGoM3loA+StN2Y9oBCC5DA=="},{"block_id_flag":2,"validator_address":"CC5CE2418DA85C78D7F89FECAAB08DDCE314C0CC","timestamp":"2021-04-18T01:22:14.193082664Z","signature":"TnHShH7wPw3tzqYZwJ9Li06d9WIOAOhLh+VBxDjX9dXVdzmkTXveDuAPfwY3C7JUgQjH9hLINHZ2Q3YFTA66Dw=="},{"block_id_flag":2,"validator_address":"1ED471CB34E38311145098D80361434558F2F1C7","timestamp":"2021-04-18T01:22:14.067366744Z","signature":"y61n5hczzrvcD42RCIRsTqMnV1FRaGZJ/cSi0u6wiMTJRBq5cKkDgf4zMMVwUiOpUwaY7Ptv85Bo6gO2eHnWDw=="},{"block_id_flag":2,"validator_address":"407F144D1C9DEA4EE6A8CBC2D4C022A657506B83","timestamp":"2021-04-18T01:22:14.006539322Z","signature":"/CB7xC9T+Q73sTRwR2mjtRi4nwFLA9IDT0yeS5IUuzLruPT5UU3HS87YYGHuANuf+SgayXVq90aC0Al44FlDDw=="},{"block_id_flag":2,"validator_address":"F55714243D32FB65B6D95A29D0350EA0CABBA8EA","timestamp":"2021-04-18T01:22:13.961778569Z","signature":"Eeqk+ytR6IOiOApGX/yneyBAYuUxLAcTERea1TS2l4PNFQ7GMUp6+hjRojKWeXiX9AqBTSNeb/sZqJ+5s7t8AQ=="},{"block_id_flag":2,"validator_address":"874D6AA838384A79A3D4D062541F91BF7E31BDBA","timestamp":"2021-04-18T01:22:14.041999Z","signature":"Bqg6excMBe/M0Ysk6q/YvYLsp4Gw9sSNxSgaS4YOK3dPWNQTQrlW96zZDBqd24CqtclLUEfI/afn5E4GMN7sDg=="},{"block_id_flag":2,"validator_address":"24935D59FAA94E793652CBF4716C6041CD7AA400","timestamp":"2021-04-18T01:22:14.069673829Z","signature":"rXVlMdvyHzScvBYretoSBoHLLPXgiV2thkCTPiOLIYIdXBmMo/xO/NsuFnBpQT0WMqRVEhc6gKX26gAVCq7aAQ=="},{"block_id_flag":2,"validator_address":"C4903229B9EAD415C79E8FA69D2BBA6117617C41","timestamp":"2021-04-18T01:22:13.913916007Z","signature":"D6YDHWQWDB03IOvEo3aNOXTAzEITEKr5R1Amtybych8/lpUpUi6hTqZnp4u8+2Y8OI8JgaJEYWZXZThoqI4NCQ=="},{"block_id_flag":2,"validator_address":"2A5FECF26C3FB43426AC6B3DB58A5ABC5800F2A0","timestamp":"2021-04-18T01:22:14.0292223Z","signature":"b5NYxwvv7H+e3IEhkW40KEulXaAVs/YXVVmcNVxneti9PXUzFWsmytWv/DPnHVJRV3wxwbfkGWUmAlGBkEZeDg=="},{"block_id_flag":2,"validator_address":"708FDDCE121CDADA502F2B0252FEF13FDAA31E50","timestamp":"2021-04-18T01:22:13.88578415Z","signature":"I9Cx3VHhgtNBx+lWZsEkoMaH6cKQLfQqblXhaVDEkcSJ/ZeR2ah9ViHFEg+R9KanryYJMgoDl7iOnH6m7OnRBg=="},{"block_id_flag":2,"validator_address":"8011772ED7DDF2CC9CD7A48C8C0AA2486E9F4E97","timestamp":"2021-04-18T01:22:14.006391343Z","signature":"9giM+gEbp7AEUYRFmMS9Ie6l7Rbk6WeMn5TXBcgUpWkxooGvtti2nX25AnQ7D/3cfcsPGOfTigcfYf2M6MY2Ag=="},{"block_id_flag":2,"validator_address":"2C882D57565E86A9EEC3433B15025F8F96318453","timestamp":"2021-04-18T01:22:14.04553832Z","signature":"9I7qeLZUtjpJ9MFF58WPTtnuY/JJh17g9JgrOxbwkd3VUSwDoSuj6crQfJ0xloKrANq6xMLRwJdHWxq83JA9Dg=="},{"block_id_flag":2,"validator_address":"0ABBA36C54DD0CA6A790AEF96A01D4392E36345F","timestamp":"2021-04-18T01:22:14.06038166Z","signature":"mvZoJepyQjdMKIKn+dF4YSGiIIEbhD/dp3uy7j2PhYvNx4/mipBgpU4LM7yGw0eEH8UPeBdxuNkkQT8llwKiCw=="},{"block_id_flag":2,"validator_address":"EB1DF22507B79CE700F86C4C8B13D7DF01DFDA9C","timestamp":"2021-04-18T01:22:14.108863816Z","signature":"fubeov80Q7cmlWolByJioP6L9JveLXBJyZTu8muAs7W5M0Yr0quWRBLD58LIus3xgw6x7YCr+0lCz31H/8tjAA=="},{"block_id_flag":2,"validator_address":"48FD560D3CB0B552924CBC0F9C2BA68883FA1135","timestamp":"2021-04-18T01:22:13.935957015Z","signature":"55vH3KEIsxk56T4ZOBJAAfgPpr+HKYKxIoTN8kKr/eNAssYEYXkw0o5Yj6JqNWllXzdop4P7ROfsOckFgk7dCQ=="},{"block_id_flag":2,"validator_address":"A7D9E6DB8CA5E46A61AC36235D4C8185F7BF11A4","timestamp":"2021-04-18T01:22:14.036024492Z","signature":"RNj1FAijuVJjCny4cqvUi0S/b5ylE1plzb1OInUwKGiWN3fHWvT4uvbtbwxMfdNcChxo9L0vb6v1AIbSKn3GAQ=="},{"block_id_flag":2,"validator_address":"F919902709B7482F01C030E8B57BF93B8D87043B","timestamp":"2021-04-18T01:22:14.076932918Z","signature":"rcSwxMEFsgHZQG8g/tIlFbSvaU2opWgYLkX9Hyyu7cO0YvncdrrexmpwVT+dhwnJlKN1ePOTX1KodoIKdqIaAQ=="},{"block_id_flag":2,"validator_address":"569B09EA3D9607C5F1A38DED81CED264C7AA94CB","timestamp":"2021-04-18T01:22:14.00827439Z","signature":"/GyAqlnl6AwRKhpO8O/Wop+lOI1p0lkYiFWszgdcxqwok75759AiJsXdLj4YiUYL3zasGqZ+WM2lEtLvAVD+BQ=="},{"block_id_flag":2,"validator_address":"8C8802A921114169D2581CD46E3CA6853F6F2A7F","timestamp":"2021-04-18T01:22:13.952350605Z","signature":"+lpzTXQs/VXBUxYxnAhSqTjVPfQ6kOX9LUNYx1004hgNjzdATMz81o3QU1TdcoPZWi46myOcVWhtYSxmcjljAw=="},{"block_id_flag":2,"validator_address":"8FA2380FA64B122427202970CF12BD4991B4C6D2","timestamp":"2021-04-18T01:22:13.956069004Z","signature":"y/Wc8LfteFBdDMjbo09IFKId4C9d3Q/mmgj/zjIQtDYDzUf1o/YTFhPkWQMvJUDMZxMj2h/0q97/mcTOLf//Ag=="},{"block_id_flag":2,"validator_address":"00CCEECFD01B2D333A1B56ABF7CFF8A6B8ED25B1","timestamp":"2021-04-18T01:22:14.030330329Z","signature":"lsuZCkpPJ6ete0f3f5P6uGP8Gu32g2lM5hml55/BaALjnf+mgcEXjf5Wu0zvJDNzSeO/KZ5v4Xwhn/yPL+/BDw=="},{"block_id_flag":2,"validator_address":"D269A81C7741F458C31DB7FB14C58176421FB2F8","timestamp":"2021-04-18T01:22:14.061399157Z","signature":"YrXKgw1nO1x2Pt6NcP3CaQwno8O5b4YEQuT85idfaoPJaMDe/DZtJbTBKagD2v+gP8KcNNFLw9AwkypdCU0+AA=="},{"block_id_flag":2,"validator_address":"B31581E9FF571045531370E2D4895A2D6FEDECFB","timestamp":"2021-04-18T01:22:13.974658818Z","signature":"JR8fWNcJKPEgM8gd28mafnci1NWLv3jGBTZDXhrOofuO7RhmN0WnkP3yM94ujphBb+tGnE2iOCIKHQGbFpvqCQ=="},{"block_id_flag":2,"validator_address":"F6CA0B14D98D69D1FAA1B19E250D03BEF7DD9541","timestamp":"2021-04-18T01:22:13.959225884Z","signature":"K3cJZQUX1YbGlEZloOxotphme9o4El0xE/CP7L6vC8IrLezTXSgdq7Av71hCx9wGxXwJd6836gUE3U9rIF7xCg=="},{"block_id_flag":2,"validator_address":"B724CDAA69A47B90B18D0EA7FD5B046D537DA64A","timestamp":"2021-04-18T01:22:13.965853657Z","signature":"gHdduPugO20eha09ySQqKnZ75m0sb2eFQjm15Ag3uDuO7h60cxXOEfoG1liC+mdiqKvOLqWtav+YWPukd6sHAQ=="},{"block_id_flag":2,"validator_address":"2CD6A3523F5D61573EF1E3654B421AD211CFE8DC","timestamp":"2021-04-18T01:22:14.08142515Z","signature":"WsmXTjF4ka6INguJi4ABdMqT7AxKHVp3oBzrbmv/UwaJDKC7aDtNC/Jbnb7ENUmgkwyDe/TQJCsKZy95J2dLAw=="},{"block_id_flag":2,"validator_address":"99304AA9AEEC13FCEF0FF72DFC8953273FE559BC","timestamp":"2021-04-18T01:22:14.042678804Z","signature":"6+4OEOxGtSBGu9FS6sWP0QDUNHVkeXJdauPa2BWBIW5vLEhI/mgRaawbb4tRO7BIfcPTXldAgWzi0Hk/HJtrDQ=="}]}}}
```

## CosmJS介绍

### 什么是cosmJS

去中心化应用程序 (dApp) 是在去中心化网络上运行的软件应用程序。区块链提供持久数据或状态，以及持久流程和逻辑。 Cosmos SDK 可帮助开发人员创建此类应用程序。在大多数情况下，用户界面很重要，在许多情况下，服务器交互也很重要。这就是 CosmJS https://github.com/cosmos/CosmJS  派上用场的地方。

顾名思义，CosmJS 是一个 TypeScript/JavaScript 库。它帮助开发人员将前端用户界面和后端服务器与实现去中心化的应用程序的 Cosmos 区块链相集成。对于许多用户来说，“dApp”是用户界面，尽管它通常以集中的、传统的方式交付给浏览器——依赖于 DNS 基础设施和集中式网络服务器。

这种重新引入一定程度的集中化通常被大多数团队认为是可以接受的，前提是系统的重要业务逻辑由区块链强制执行，并且并非严格必须使用提供的用户界面来使用 dApp .作为一般启发式方法，提供用户界面是为了方便而不是必需。

一般来说，用户界面帮助用户解释区块链状态、编写和签署交易并发送它们——所有这些都可能通过其他不太方便的方法来完成。用户界面由也与区块链交互的服务器或微服务支持。

> 所以CosmJS就是波卡substrate中的polkadot.js，EVM-based chain的ethers.js和web3.js

#### 一些例子

从事开发直观和连贯的用户界面的开发人员需要在浏览器级别完成某些事情：

- 帮助用户创建未签名的 Cosmos SDK 交易。
- 让用户用他们的钱包签署一个未签名的交易。
- 帮助用户将已签名的交易提交到 Cosmos SDK 端点。
- 使用旧版 REST 端点从 Cosmos Hub 或自定义模块查询状态。
- 使用 gRPC 端点从 Cosmos Hub 或自定义模块查询状态。
- 帮助用户在单个交易中提交多条消息。

后端系统通常是整体设计的有用组件：

- 出于性能原因缓存复杂状态。
- 最大限度地减少基本匿名浏览的客户端要求。
- 监控区块链的变化并通知客户。
- 提供 API 端点和 WebSocket。

开发人员需要一个工具包来完成解决这些基本问题的事情：

- 当助记词已知时签署交易。
- 在私钥已知时签署交易。
- 将多条消息分组到一个交易中。
- 在 Keplr 等钱包的帮助下请求用户签名。
- 查询区块链状态。
- 监听 Cosmos SDK 模块发出的事件。

CosmJS 协助完成这些任务以及更多。

CosmJS 的模块化结构允许开发人员只导入需要的部分，这有助于减少下载payload。由于该库是独立的，因此它与 Vue、React 和 Express 等流行的 JavaScript 框架兼容。

#### 包

CosmJS 是一个库，由 @cosmjs 命名空间 [npm (npmjs.com)](https://www.npmjs.com/org/cosmjs) 中的许多较小的 npm 包组成，即所谓的“monorepo”。

通常人们只需要 `stargate` 和 `encoding` 包，因为它们包含与 Cosmos SDK 链 0.40 及更高版本交互的主要功能。

其中，这里有一些示例包：

![[Pasted image 20221124163049.png]]


#### 模块化

我们为这个 monorepo 中的模块化和干净的依赖树感到自豪。 这确保了我们这边的软件质量，并让用户准确地选择他们需要的东西，并且只选择他们需要的东西。 下图显示了所有内容是如何组合在一起的（每个项目都是一个 npm 包；右边依赖于左边）：

![[Pasted image 20221124163139.png]]

在下一节中我们会上手操作。

- [HackAtom HCMC Workshop - CosmWasm/CosmJS: from zero to hero - YouTube](https://www.youtube.com/watch?v=VTjiC4wcd7k)
- [cosmos/cosmjs: The Swiss Army knife to power JavaScript based client solutions ranging from Web apps/explorers over browser extensions to server-side clients like faucets/scrapers. (github.com)](https://github.com/cosmos/CosmJS) 

### 你的第一次CosmJS行动

```bash
git clone git@github.com:b9lab/cosmjs-sandbox.git
cd cosmjs-sandbox
npm install
```

然后你就可以看到这个项目下没有任何一个ts/js源文件，所以添加一个`experiment.ts`的源文件，加入以下代码:

```ts
const runAll = async(): Promise<void> => {
    console.log("TODO")
}
runAll()
```

编译运行:

```bash
npm run experiment
```

你就会发现输出了`TODO`。

#### 测试网准备

Cosmos 生态系统有许多测试网在运行。 Cosmos Hub 目前正在运行一个公共测试网（[testnets/public at master · cosmos/testnets (github.com)](https://github.com/cosmos/testnets/tree/master/public) ，用于您正在连接并运行脚本的 Theta 升级。 您需要连接到公共节点，以便查询信息和广播交易。 可用节点之一是：

```txt
RPC: https://rpc.sentry-01.theta-testnet.polypore.xyz
```

您需要一个测试网上的钱包地址，并且您必须创建一个 24 字的助记词才能这样做。 CosmJS 可以为您生成一个。 使用以下脚本创建一个新文件 `generate_mnemonic.ts`：[cosmjs-sandbox/generate_mnemonic.ts at 723d2a9d0b08f3208164fb414d578188f7a71eaa · b9lab/cosmjs-sandbox (github.com)](https://github.com/b9lab/cosmjs-sandbox/blob/723d2a9/generate_mnemonic.ts#L1-L10) 

```ts
import { DirectSecp256k1HdWallet } from "@cosmjs/proto-signing"

const generateKey = async (): Promise<void> => {
    const wallet: DirectSecp256k1HdWallet = await DirectSecp256k1HdWallet.generate(24)
    process.stdout.write(wallet.mnemonic)
    const accounts = await wallet.getAccounts()
    console.error("Mnemonic with 1st account:", accounts[0].address)
}

generateKey()

```

生成一个叫Alice的秘钥对:

```bash
npx ts-node generate_mnemonic.ts > testnet.alice.mnemonic.key
```

上面写入的文件是暂存助记词。

输出:

```txt
Mnemonic with 1st account: cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y
```

> 如果上面运行失败，那么请升级nodejs版本 [How to Update Node.js to Latest Version {Linux, Windows, and MacOS} (phoenixnap.com)](https://phoenixnap.com/kb/update-node-js-version) 

这节的全部代码在这里 [b9lab/cosmjs-sandbox at file-preparation (github.com)](https://github.com/b9lab/cosmjs-sandbox/tree/file-preparation) 

#### 添加你的导入

你需要一个小而简单的区块链接口，一个最终可能有用户的接口。 好的做法是在必要之前不要请求用户地址（例如，当用户单击相关按钮时）。 因此，在 `experiment.ts` 中，您首先使用只读客户端。 在文件顶部导入它：

```ts
import { StargateClient } from "@cosmjs/stargate"
```

#### 定义你的连接

接下来你该告诉你的客户端如何连接之前的RPC节点:

```ts
const rpc = "rpc.sentry-01.theta-testnet.polypore.xyz:26657"
```

在`runAll`函数中初始化连接和马上检查是否连接到了正确的地方:

```ts
const runAll = async(): Promise<void> => {
    const client = await StargateClient.connect(rpc)
    console.log("With client, chain id:", await client.getChainId(), ", height:", await client.getHeight())
}
```

运行 `npm run experiment`， 输出:

```txt
With client, chain id: theta-testnet-001 , height: 13288982
```

#### 获取余额

通常您还无法访问您的用户地址。 但是，对于本练习，您需要知道 Alice 有多少代币，因此在 `runAll` 中添加一个临时的新命令：

```ts
console.log(
    "Alice balances:",
    await client.getAllBalances("cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y"), // <-- replace with your generated address
)
```

运行后，你会发现Alice的地址下的所有代币的余额都是空的，那是必然的，你生成一个新的地址，一般不会有钱在里面。

如果您刚刚创建了这个帐户，Alice 的余额为零。 Alice 需要代币才能发送交易并参与网络。 测试网的一个常见做法是公开水龙头（在限制范围内免费向您发送测试代币的服务）。

Cosmos Hub Testnet Faucet有一个专用的 Discord 频道 https://discord.com/channels/669268347736686612/953697793476821092/958291295741313024 ，您可以在其中为每个 Discord 用户每天索取一次代币。

通过在频道中输入以下命令，转到faucet频道并为 Alice 请求代币：

```bash
$request cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y theta
```

[Transaction Details | Testnet Block Explorer (polypore.xyz)](https://explorer.theta-testnet.polypore.xyz/transactions/465156F8512B25CB806AF59EE36ACBC84779066BFE331DC954E819BA98BAC0B8) 这里是请求代币的交易

这下，你再次运行`npm run experiment` 就会输出:

```json
Alice balances: [ { denom: 'uatom', amount: '10000000' } ]
```

`uatom` 是测试网上不可分割的代币单元(类似于以太坊上的wei，polkadot上的Plank)。 它是微型 ATOM 或 µ-ATOM 的缩写。所以这里发了0.001 ATOM。 所以确认后，您可以注释掉余额查询。

#### 获取水龙头(faucet)地址

作为练习，您希望Alice将一些代币发送回Faucet，因此您需要它的地址。 您可以从Faucet机器人请求它，但也可以使用 experiment.ts 中的交易哈希来获取它。

首先，您需要获得交易。

在顶部添加必要的导入：

```ts
import { IndexedTx, StargateClient } from "@cosmjs/stargate"
```

```ts
const faucetTx: IndexedTx = (await client.getTx(
    "465156F8512B25CB806AF59EE36ACBC84779066BFE331DC954E819BA98BAC0B8",
))!
```

以上的代码就获取了水龙头发送的交易了

#### 反序列化交易

可以这样输出交易:

```ts
console.log("Faucet Tx:", faucetTx)
```

```json
Faucet Tx: {  
height: 13289139,  
hash: '465156F8512B25CB806AF59EE36ACBC84779066BFE331DC954E819BA98BAC0B8',  
code: 0,  
rawLog: '[{"events":[{"type":"coin_received","attributes":[{"key":"receiver","value":"cosmos1ykt8lw8  
6us7d93de4gqr4pg0z3ujse6vjjtx3y"},{"key":"amount","value":"10000000uatom"}]},{"type":"coin_spent","att  
ributes":[{"key":"spender","value":"cosmos15aptdqmm7ddgtcrjvc5hs988rlrkze40l4q0he"},{"key":"amount","v  
alue":"10000000uatom"}]},{"type":"message","attributes":[{"key":"action","value":"/cosmos.bank.v1beta1  
.MsgSend"},{"key":"sender","value":"cosmos15aptdqmm7ddgtcrjvc5hs988rlrkze40l4q0he"},{"key":"module","v  
alue":"bank"}]},{"type":"transfer","attributes":[{"key":"recipient","value":"cosmos1ykt8lw86us7d93de4g  
qr4pg0z3ujse6vjjtx3y"},{"key":"sender","value":"cosmos15aptdqmm7ddgtcrjvc5hs988rlrkze40l4q0he"},{"key"  
:"amount","value":"10000000uatom"}]}]}]',  
tx: Uint8Array(323) [  
10, 148, 1, 10, 145, 1, 10, 28, 47, 99, 111, 115,  
109, 111, 115, 46, 98, 97, 110, 107, 46, 118, 49, 98,  
101, 116, 97, 49, 46, 77, 115, 103, 83, 101, 110, 100,  
18, 113, 10, 45, 99, 111, 115, 109, 111, 115, 49, 53,  
97, 112, 116, 100, 113, 109, 109, 55, 100, 100, 103, 116,  
99, 114, 106, 118, 99, 53, 104, 115, 57, 56, 56, 114,  
108, 114, 107, 122, 101, 52, 48, 108, 52, 113, 48, 104,  
101, 18, 45, 99, 111, 115, 109, 111, 115, 49, 121, 107,  
116, 56, 108, 119,  
... 223 more items  
],  
gasUsed: 76728,  
gasWanted: 200000  
}
```

但是，这样输出对人可读性不是那么好，所以需要反序列化:

```ts
import { Tx } from "cosmjs-types/cosmos/tx/v1beta1/tx"

const decodedTx: Tx = Tx.decode(faucetTx.tx)
console.log("DecodedTx:", decodedTx)
console.log("Decoded messages:", decodedTx.body!.messages)

```

```json
DecodedTx: {  
signatures: [  
Uint8Array(64) [  
23, 12, 13, 119, 226, 105, 171, 112, 56, 161, 29,  
195, 109, 239, 214, 228, 43, 131, 234, 79, 247, 163,  
155, 206, 2, 102, 238, 87, 117, 50, 116, 51, 113,  
156, 39, 112, 227, 190, 230, 101, 239, 235, 205, 8,  
15, 41, 211, 55, 37, 144, 1, 177, 26, 255, 97,  
210, 91, 110, 57, 28, 208, 9, 81, 49  
]  
],  
body: {  
memo: '',  
timeoutHeight: Long { low: 0, high: 0, unsigned: true },  
messages: [ [Object] ],  
extensionOptions: [],  
nonCriticalExtensionOptions: []  
},  
authInfo: {  
signerInfos: [ [Object] ],  
fee: { gasLimit: [Long], payer: '', granter: '', amount: [Array] }  
}  
}  
Decoded messages: [  
{  
typeUrl: '/cosmos.bank.v1beta1.MsgSend',  
value: Uint8Array(113) [  
10, 45, 99, 111, 115, 109, 111, 115, 49, 53, 97, 112,  
116, 100, 113, 109, 109, 55, 100, 100, 103, 116, 99, 114,  
106, 118, 99, 53, 104, 115, 57, 56, 56, 114, 108, 114,  
107, 122, 101, 52, 48, 108, 52, 113, 48, 104, 101, 18,  
45, 99, 111, 115, 109, 111, 115, 49, 121, 107, 116, 56,  
108, 119, 56, 54, 117, 115, 55, 100, 57, 51, 100, 101,  
52, 103, 113, 114, 52, 112, 103, 48, 122, 51, 117, 106,  
115, 101, 54, 118, 106, 106, 116, 120, 51, 121, 26, 17,  
10, 5, 117, 97,  
... 13 more items  
]  
}  
]
```

反序列化交易并没有完全反序列化它包含的任何消息，也没有完全反序列化它们的值，这又是一个 `Uint8Array`。 交易反序列化器知道如何正确解码任何交易，但它不知道如何对消息执行相同的操作。 消息实际上可以是任何类型，并且每种类型都有自己的反序列化器。 这不是 `Tx.decode` 交易反序列化器函数所知道的。因为消息可以自定义格式。是一个二进制data字段。

#### 这个长字符串是什么

请注意 `typeUrl: "/cosmos.bank.v1beta1.MsgSend"` 字符串。 这来自 Protobuf 定义，是以下各项的混合：

最初声明 `MsgSend` 的包：[cosmos-sdk/tx.proto at 3a1027c74b15ad78270dbe68b777280bde393576 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/3a1027c/proto/cosmos/bank/v1beta1/tx.proto#L2)  而消息本身的名字 `MsgSend`在这里 [cosmos-sdk/tx.proto at 3a1027c74b15ad78270dbe68b777280bde393576 · cosmos/cosmos-sdk (github.com)](https://github.com/cosmos/cosmos-sdk/blob/3a1027c/proto/cosmos/bank/v1beta1/tx.proto#L22) 

您在解码消息中看到的这个 `typeUrl` 字符串是在`value`数组下序列化的消息类型的规范标识符。 有许多不同的消息类型，每一种都来自 Cosmos SDK 的不同模块或基础层，您可以在此处找到它们的概述 [Docs · cosmos/cosmos-sdk (buf.build)](https://buf.build/cosmos/cosmos-sdk) 。

区块链客户端本身知道如何序列化或反序列化它只是因为这个`“/cosmos.bank.v1beta1.MsgSend”`字符串被传递了。 使用此 `typeUrl`，区块链客户端和 CosmJS 能够选择正确的反序列化器。 这个对象在 `cosmjs-types` 中也被命名为 `MsgSend`。 但在本教程中，您手动选择了反序列化器。

> 要了解如何为您自己的区块链项目制作您自己的类型，请前往创建  自定义 CosmJS 接口。[[#创建自定义的CosmJS接口]]

#### 反序列化消息

下面就是手动选择消息的反序列化器。现在您知道交易中的唯一消息是 `MsgSend`，您需要反序列化它。 首先在顶部添加必要的导入：

```ts
import { MsgSend } from "cosmjs-types/cosmos/bank/v1beta1/tx"


const sendMessage: MsgSend = MsgSend.decode(decodedTx.body!.messages[0].value)
console.log("Sent message:", sendMessage)

```

然后`npm run experiment`

```json
Sent message: {  
fromAddress: 'cosmos15aptdqmm7ddgtcrjvc5hs988rlrkze40l4q0he',  
toAddress: 'cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y',  
amount: [ { denom: 'uatom', amount: '10000000' } ]  
}
```

所以水龙头的地址在sendMessage里面。

```ts
const faucet: string = sendMessage.fromAddress
console.log("Faucet balances:", await client.getAllBalances(faucet))
```

```json
Faucet balances: [ { denom: 'uatom', amount: '852330414566' } ]
```

> 还有一种获得水龙头地址的方法
> 
> 您可以通过其他方式自行处理数据，而不是使用 `Tx` 和 `MsgSend` 导入附带的解码函数。 如果您想进行更多试验，请手动解析 `rawLog`，而不是像之前建议的那样反序列化交易。
> 
> 请注意 `Tx` 和 `rawLog` 之间的概念差异。 `Tx` 或 `MsgSend` 对象是当交易包含在块中时发生的计算的输入。 `rawLog` 是上述计算的结果输出，其内容取决于执行交易时区块链代码发出的内容。
> 
> 从 `IndexedTx` 中，您可以看到有一个 `rawLog`  [cosmjs/stargateclient.ts at 13ce43c1220c9d94e5d8c444bfc74d7e72d7e254 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/13ce43c/packages/stargate/src/stargateclient.ts#L64) ，它恰好是一个字符串化的 JSON。

```ts
const rawLog = JSON.parse(faucetTx.rawLog)
console.log("Raw log:", JSON.stringify(rawLog, null, 4))

const faucet_another: string = rawLog[0].events
    .find((eventEl: any) => eventEl.type === "coin_spent")
    .attributes.find((attribute: any) => attribute.key === "spender").value

    console.log("Faucet balances in another way:", await client.getAllBalances(faucet_another))
```

> 虽然这是从交易消息中提取特定值的完全有效的方法，但不推荐这样做。 导入消息类型的好处是您不必手动挖掘您希望在应用程序中使用的每个 `Tx` 的原始日志。

#### 准备一个签名客户端

如果您查看 `StargateClient`  [cosmjs/stargateclient.ts at 0f0c9d8a754cbf01e17acf51d3f2dbdeaae60757 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/0f0c9d8/packages/stargate/src/stargateclient.ts#L139) 中的方法，您会发现它只包含查询类型的方法，没有用于发送交易的方法。

现在，为了让Alice发送交易，她需要能够签署交易。 为了能够签署交易，她需要访问她的私钥或助记词。 或者更确切地说，她需要一个可以访问这些的客户端。 这就是 `SigningStargateClient`  [cosmjs/signingstargateclient.ts at 0f0c9d8a754cbf01e17acf51d3f2dbdeaae60757 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/0f0c9d8/packages/stargate/src/signingstargateclient.ts#L147) 的用武之地。方便的是，`SigningStargateClient` 继承自 `StargateClient`。

更新您的导入行：

```ts
import { IndexedTx, SigningStargateClient, StargateClient } from "@cosmjs/stargate"
```

当您使用 `connectWithSigner`  [cosmjs/signingstargateclient.ts at 0f0c9d8a754cbf01e17acf51d3f2dbdeaae60757 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/0f0c9d8/packages/stargate/src/signingstargateclient.ts#L156) 方法实例化 `SigningStargateClient` 时，您需要向其传递一个`Signer` [cosmjs/signingstargateclient.ts at 0f0c9d8a754cbf01e17acf51d3f2dbdeaae60757 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/0f0c9d8/packages/stargate/src/signingstargateclient.ts#L158) 。 在这种情况下，请使用 `OfflineDirectSigner`  [cosmjs/signer.ts at 0f0c9d8a754cbf01e17acf51d3f2dbdeaae60757 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/0f0c9d8/packages/proto-signing/src/signer.ts#L21-L24)  接口。


> 推荐的消息编码方式是使用 `OfflineDirectSigner`，它使用 Protobuf。 然而，诸如 Ledger 之类的硬件钱包不支持这一点，并且仍然需要遗留的 Amino 编码器。 如果您的应用需要 Amino 支持，您必须使用 `OfflineAminoSigner`。
> 
> 在此处阅读有关编码的更多信息 [Encoding | Cosmos SDK](https://docs.cosmos.network/main/core/encoding.html) 。

signer需要访问 Alice 的私钥，有几种方法可以实现这一点。 在这个例子中，使用Alice保存的助记词。 要在您的代码中将助记符作为文本加载，您需要导入：

```ts
import { readFile } from "fs/promises"
```

有多种可用的 `OfflineDirectSigner` 实现。 由于其 `fromMnemonic`  [cosmjs/directsecp256k1hdwallet.ts at 0f0c9d8a754cbf01e17acf51d3f2dbdeaae60757 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/0f0c9d8/packages/proto-signing/src/directsecp256k1hdwallet.ts#L140-L141) 方法，`DirectSecp256k1HdWallet` [cosmjs/directsecp256k1hdwallet.ts at 0f0c9d8a754cbf01e17acf51d3f2dbdeaae60757 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/0f0c9d8/packages/proto-signing/src/directsecp256k1hdwallet.ts#L133) 实现与我们最为相关。 添加导入：

```ts
import { DirectSecp256k1HdWallet, OfflineDirectSigner } from "@cosmjs/proto-signing"

const getAliceSignerFromMnemonic = async (): Promise<OfflineDirectSigner> => {
    return DirectSecp256k1HdWallet.fromMnemonic((await readFile("./testnet.alice.mnemonic.key")).toString(), {
        prefix: "cosmos",
    })
}
 
 const aliceSigner: OfflineDirectSigner = await getAliceSignerFromMnemonic()
    const alice = (await aliceSigner.getAccounts())[0].address
    console.log("Alice's address from signer", alice)
    const signingClient = await SigningStargateClient.connectWithSigner(rpc, aliceSigner)
    console.log(
        "With signing client, chain id:",
        await signingClient.getChainId(),
        ", height:",
        await signingClient.getHeight()
    )
```

```txt
Alice's address from signer cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y  
With signing client, chain id: theta-testnet-001 , height: 13292751
```


#### 发送代币

```ts
console.log("Gas fee:", decodedTx.authInfo!.fee!.amount)
    console.log("Gas limit:", decodedTx.authInfo!.fee!.gasLimit.toString(10))
    console.log("Alice balance before:", await client.getAllBalances(alice))
    console.log("Faucet balance before:", await client.getAllBalances(faucet))
    const result = await signingClient.sendTokens(alice, faucet, [{ denom: "uatom", amount: "100000" }], {
        amount: [{ denom: "uatom", amount: "500" }],
        gas: "200000",
    })
    console.log("Transfer result:", result)
    console.log("Alice balance after:", await client.getAllBalances(alice))
    console.log("Faucet balance after:", await client.getAllBalances(faucet))
```

> 您已连接到公开运行的测试网。 因此，您依赖于其他人来让区块链运行在开放且公开可用的 RPC 端口和水龙头上。 如果您想尝试连接到您自己的本地运行的区块链怎么办？

`experiment.ts`的全部代码:

```ts
import { readFile } from "fs/promises"
import { DirectSecp256k1HdWallet, OfflineDirectSigner } from "@cosmjs/proto-signing"
import { IndexedTx, SigningStargateClient, StargateClient } from "@cosmjs/stargate"
import { Tx } from "cosmjs-types/cosmos/tx/v1beta1/tx"
import { MsgSend } from "cosmjs-types/cosmos/bank/v1beta1/tx"



const rpc = "rpc.sentry-01.theta-testnet.polypore.xyz:26657"

const getAliceSignerFromMnemonic = async (): Promise<OfflineDirectSigner> => {
    return DirectSecp256k1HdWallet.fromMnemonic((await readFile("./testnet.alice.mnemonic.key")).toString(), {
        prefix: "cosmos",
    })
}

const runAll = async(): Promise<void> => {
    console.log("TODO")
    const client = await StargateClient.connect(rpc)
    console.log("With client, chain id:", await client.getChainId(), ", height:", await client.getHeight())

    console.log(
        "Alice balances:",
        await client.getAllBalances("cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y"), // <-- replace with your generated address
    )
    const faucetTx: IndexedTx = (await client.getTx(
        "465156F8512B25CB806AF59EE36ACBC84779066BFE331DC954E819BA98BAC0B8",
    ))!

    console.log("Faucet Tx:", faucetTx)

    const decodedTx: Tx = Tx.decode(faucetTx.tx)
    console.log("DecodedTx:", decodedTx)
    console.log("Decoded messages:", decodedTx.body!.messages)

    const sendMessage: MsgSend = MsgSend.decode(decodedTx.body!.messages[0].value)
    console.log("Sent message:", sendMessage)

    const faucet: string = sendMessage.fromAddress
    console.log("Faucet balances:", await client.getAllBalances(faucet))

    const rawLog = JSON.parse(faucetTx.rawLog)
    console.log("Raw log:", JSON.stringify(rawLog, null, 4))

    const faucet_another: string = rawLog[0].events
    .find((eventEl: any) => eventEl.type === "coin_spent")
    .attributes.find((attribute: any) => attribute.key === "spender").value

    console.log("Faucet balances in another way:", await client.getAllBalances(faucet_another))

    const aliceSigner: OfflineDirectSigner = await getAliceSignerFromMnemonic()
    const alice = (await aliceSigner.getAccounts())[0].address
    console.log("Alice's address from signer", alice)
    const signingClient = await SigningStargateClient.connectWithSigner(rpc, aliceSigner)
    console.log(
        "With signing client, chain id:",
        await signingClient.getChainId(),
        ", height:",
        await signingClient.getHeight()
    )

    console.log("Gas fee:", decodedTx.authInfo!.fee!.amount)
    console.log("Gas limit:", decodedTx.authInfo!.fee!.gasLimit.toString(10))
    console.log("Alice balance before:", await client.getAllBalances(alice))
    console.log("Faucet balance before:", await client.getAllBalances(faucet))
    const result = await signingClient.sendTokens(alice, faucet, [{ denom: "uatom", amount: "100000" }], {
        amount: [{ denom: "uatom", amount: "500" }],
        gas: "200000",
    })
    console.log("Transfer result:", result)
    console.log("Alice balance after:", await client.getAllBalances(alice))
    console.log("Faucet balance after:", await client.getAllBalances(faucet))
}


runAll()

```

```json
Alice's address from signer cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y  
With signing client, chain id: theta-testnet-001 , height: 13292795  
Gas fee: [ { denom: 'uatom', amount: '1000' } ]  
Gas limit: 200000  
Alice balance before: [ { denom: 'uatom', amount: '10000000' } ]  
Faucet balance before: [ { denom: 'uatom', amount: '852330414566' } ]  
Transfer result: {  
code: 0,  
height: 13292796,  
rawLog: '[{"events":[{"type":"coin_received","attributes":[{"key":"receiver","value":"cosmos15aptdqm  
m7ddgtcrjvc5hs988rlrkze40l4q0he"},{"key":"amount","value":"100000uatom"}]},{"type":"coin_spent","attri  
butes":[{"key":"spender","value":"cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y"},{"key":"amount","val  
ue":"100000uatom"}]},{"type":"message","attributes":[{"key":"action","value":"/cosmos.bank.v1beta1.Msg  
Send"},{"key":"sender","value":"cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y"},{"key":"module","value  
":"bank"}]},{"type":"transfer","attributes":[{"key":"recipient","value":"cosmos15aptdqmm7ddgtcrjvc5hs9  
88rlrkze40l4q0he"},{"key":"sender","value":"cosmos1ykt8lw86us7d93de4gqr4pg0z3ujse6vjjtx3y"},{"key":"am  
ount","value":"100000uatom"}]}]}]',  
transactionHash: '582AADA2C70E5F5C317F65F34E24A405773DB51CC5B123C54C502462DBBC9805',  
gasUsed: 74190,  
gasWanted: 200000  
}  
Alice balance after: [ { denom: 'uatom', amount: '9899500' } ]  
Faucet balance after: [ { denom: 'uatom', amount: '852330514566' } ]
```

#### 本地启动一个链

也就是通过`CosmJS` 连接本地用`simapp` 启动的链。创建两个账户 Alice和Bob。运行`simd`：

```bash
./build/simd init demo
# chain id test-chain-joEf8H
./build/simd keys add alice
./build/simd keys add bob
./build/simd start
```

端口 `26657` 是使用 SDK 构建的 RPC 端点的默认端口，除非在` ~/.simapp/config/config.toml `中另有配置。 `127.0.0.1:26657` 是您稍后需要添加到脚本中的 URL。

然后在sanbox中添加`experiment-local.ts`的脚本文件。

#### 准备你的秘钥

虽然你有 Alice 的地址，但你可能没有她的助记词或私钥。 私钥存储在操作系统的密钥环后端中。 出于本练习的目的，提取它——通常这是不安全的操作：

```bash
./build/simd keys export alice --unsafe --unarmored-hex
```

您将获得一个 64 位长的十六进制值。 将其复制并粘贴到 `cosmjs-sandbox` 文件夹中的新 `simd.alice.private.key` 文件中。 .gitignore 之前已经配置为忽略它，从而降低了风险。

#### 更新你的脚本

```ts
const rpc = "http://127.0.0.1:26657"
// replace bob's address
const faucet: string = "cosmos1umpxwaezmad426nt7dx3xzv5u0u7wjc0kj7ple"
```

接下来，您需要替换创建 Alice 的签名者的函数，因为您使用的是私钥而不是助记词，所以 `DirectSecp256k1HdWallet` 自带的 `fromMnemonic` 方法不起作用。 `DirectSecp256k1Wallet`自带的`fromKey` [cosmjs/directsecp256k1wallet.ts at 0f0c9d8a754cbf01e17acf51d3f2dbdeaae60757 · cosmos/cosmjs (github.com)](https://github.com/cosmos/cosmjs/blob/0f0c9d8/packages/proto-signing/src/directsecp256k1wallet.ts#L21) 方法是这次比较合适的选择。

调整import：

```ts
import { DirectSecp256k1Wallet, OfflineDirectSigner } from "@cosmjs/proto-signing"
```

在 `DirectSecp256k1Wallet` 中，`fromKey` 工厂函数需要一个 `Uint8Array`。 幸运的是，CosmJS 包含一个将十六进制字符串转换为 Uint8Array 的实用程序。 导入它：

```ts
import { fromHex } from "@cosmjs/encoding"
```

```ts
const getAliceSignerFromPriKey = async(): Promise<OfflineDirectSigner> => {
    return DirectSecp256k1Wallet.fromKey(
        fromHex((await readFile("./simd.alice.private.key")).toString()),
        "cosmos",
    )
}
```

```ts
const aliceSigner: OfflineDirectSigner = await getAliceSignerFromPriKey()
```

然后运行`npm run experiment-local` ，记住在此之前要运行`./simd start` 确保本地的区块链运行，将Alice的代币转到bob的地址上。

```txt
With signing client, chain id: test-chain-joEf8H , height: 52  
Alice balance before: [ { denom: 'stake', amount: '30000000' } ]  
Faucet balance before: []  
Transfer result: {  
code: 0,  
height: 53,  
rawLog: '[{"msg_index":0,"events":[{"type":"coin_received","attributes":[{"key":"receiver","value":"  
cosmos133k4rczl05g9wust8fw4a6l6t4hy8clkuzamuv"},{"key":"amount","value":"100000stake"}]},{"type":"coin  
_spent","attributes":[{"key":"spender","value":"cosmos1gcqf8cp925wsk7h6muyxxcuphrrjsm6y3jryrg"},{"key"  
:"amount","value":"100000stake"}]},{"type":"message","attributes":[{"key":"action","value":"/cosmos.ba  
nk.v1beta1.MsgSend"},{"key":"sender","value":"cosmos1gcqf8cp925wsk7h6muyxxcuphrrjsm6y3jryrg"},{"key":"  
module","value":"bank"}]},{"type":"transfer","attributes":[{"key":"recipient","value":"cosmos133k4rczl  
05g9wust8fw4a6l6t4hy8clkuzamuv"},{"key":"sender","value":"cosmos1gcqf8cp925wsk7h6muyxxcuphrrjsm6y3jryr  
g"},{"key":"amount","value":"100000stake"}]}]}]',  
transactionHash: '6AEC6BA2D344694F7C41270AFC2B93A69629B46A7EB5B80715058A0AE83D4D69',  
gasUsed: 90821,  
gasWanted: 200000  
}  
Alice balance after: [ { denom: 'stake', amount: '29899500' } ]  
Faucet balance after: [ { denom: 'stake', amount: '100000' } ]
```

看以上输出，转账成功。

`experiment-local.ts`文件的源码为:

```ts
import { readFile } from "fs/promises"
import { fromHex } from "@cosmjs/encoding"
import { DirectSecp256k1Wallet, OfflineDirectSigner } from "@cosmjs/proto-signing"
import { SigningStargateClient, StargateClient } from "@cosmjs/stargate"

const rpc = "http://127.0.0.1:26657"

const getAliceSignerFromPriKey = async (): Promise<OfflineDirectSigner> => {
    return DirectSecp256k1Wallet.fromKey(
        fromHex((await readFile("./simd.alice.private.key")).toString()),
        "cosmos"
    )
}

const runAll = async (): Promise<void> => {
    const client = await StargateClient.connect(rpc)
    console.log("With client, chain id:", await client.getChainId(), ", height:", await client.getHeight())
    console.log(
        "Alice balances:",
        await client.getAllBalances("cosmos17tvd4hcszq7lcxuwzrqkepuau9fye3dal606zf")
    )
    const faucet: string = "cosmos133k4rczl05g9wust8fw4a6l6t4hy8clkuzamuv"

    const aliceSigner: OfflineDirectSigner = await getAliceSignerFromPriKey()
    const alice = (await aliceSigner.getAccounts())[0].address
    console.log("Alice's address from signer", alice)
    const signingClient = await SigningStargateClient.connectWithSigner(rpc, aliceSigner)
    console.log(
        "With signing client, chain id:",
        await signingClient.getChainId(),
        ", height:",
        await signingClient.getHeight()
    )

    console.log("Alice balance before:", await client.getAllBalances(alice))
    console.log("Faucet balance before:", await client.getAllBalances(faucet))
    const result = await signingClient.sendTokens(alice, faucet, [{ denom: "stake", amount: "100000" }], {
        amount: [{ denom: "stake", amount: "500" }],
        gas: "200000",
    })
    console.log("Transfer result:", result)
    console.log("Alice balance after:", await client.getAllBalances(alice))
    console.log("Faucet balance after:", await client.getAllBalances(faucet))
}

runAll()
```

### 组合复杂的交易

TODO

### 学习集成Keplr

TODO

### 创建自定义的CosmJS接口

TODO

## 理解SDK的一些内建模块

主要就是熟悉和用命令行操作cosmosSDK中自带的重要的几个模块。

### 理解Authz模块

authz  [Authz Overview | Cosmos SDK](https://docs.cosmos.network/v0.46/modules/authz/)  模块使一个用户（授予者，the granter）能够授权另一个用户（被授予者，the grantee）代表他们执行消息。 authz 模块不同于负责指定基本交易和帐户类型的 auth  [Auth Overview | Cosmos SDK](https://docs.cosmos.network/v0.46/modules/auth/) (authentication) 模块。

#### 使用authz来授权认证

#### 需求

选择0.46.x

#### 配置

> 如果您以前使用过 `simd`，您的主目录中可能已经有一个 `.simapp` 目录。 要保留以前的数据，请将目录保存到另一个位置或使用 `--home` 标志并在以下说明中为每个命令指定不同的目录。 如果你不想保留以前的数据，删除以前的目录（`rm -rf ~/.simapp`）。

```bash
simd config chain-id demo
simd config keyring-backend test
```

#### Key建立

```bash
# plunge hundred health electric victory foil marine elite shiver tonight away verify vacuum giant pencil ocean nest pledge okay endless try spirit special start
simd keys add alice --recover
# shuffle oppose diagram wire rubber apart blame entire thought firm carry swim old head police panther lyrics road must silly sting dirt hard organ
simd keys add bob --recover
```
以上加入了`--recover` 参数，所以可以定制自己的助记词, 是BIP39助记词。

#### 链的建立

```bash
simd init test --chain-id demo
simd add-genesis-account alice 5000000000stake --keyring-backend test
simd add-genesis-account bob 5000000000stake --keyring-backend test
simd gentx alice 1000000stake --chain-id demo
simd collect-gentxs
```

#### 启动链

```bash
simd start
```

#### 提交一个proposal 提案

要演示对治理提案进行投票的授权，您必须首先创建一个治理提案。 以下命令创建一个包含最低保证金的文本提案，它允许治理提案立即进入投票期。 有关命令和标志选项的更多信息，请运行 `simd tx gov submit-proposal --help`。

创建提案

```bash
# 这个命令0.46.x的版本不支持，需要有一个提案的json文件。0.44.x支持
simd tx gov submit-proposal --title="Test Authorization" --description="Is Bob authorized to vote?" --type="Text" --deposit="10000000stake" --from alice
```

查看提案:

```bash
simd query gov proposal 1
```

#### 授权认证(grant authorization)

接下来，授予者（ganter）必须向被授予者(grantee)授予授权。

该认证是一种“通用”认证，它以一种消息类型（例如`MsgVote`）作为参数，并允许被授权者无限制地认证代表授权者执行该消息。 其他认证类型 [Concepts | Cosmos SDK](https://docs.cosmos.network/v0.46/modules/authz/01_concepts.html#built-in-authorizations) 包括“send”、“delegate”、“unbond”和“redelegate”，在这种情况下，granter可以设置代币数量限制。 在这种情况下，被授予者可以任意多次投票，直到授予者撤销授权。

创建认证:

```bash
simd tx authz grant cosmos1khljzagdncfs03x5g6rf9qp5p93z9qgc3w5dwt generic --msg-type /cosmos.gov.v1beta1.MsgVote --from alice
```

查看认证:

```bash
simd query authz grants cosmos1jxd2uhx0j6e59306jq3jfqs7rhs7cnhvey4lqh cosmos1khljzagdncfs03x5g6rf9qp5p93z9qgc3w5dwt /cosmos.gov.v1beta1.MsgVote
```

#### 生成交易

为了让被授予者代表授予者执行消息，被授予者必须首先生成一个未签名的交易，其中交易作者（`--from` 地址）是授予者。

创建未签名交易:

```bash
simd tx gov vote 1 yes --from cosmos1jxd2uhx0j6e59306jq3jfqs7rhs7cnhvey4lqh --generate-only > tx.json
```

查看交易:

```bash
cat tx.json
```

#### 执行交易

最后，被授权者可以使用 `exec` 命令签署并发送交易。 交易的作者（`--from` 地址）是被授权者。

签名和发送交易:

```bash
simd tx authz exec tx.json --from bob
```

查看投票:

```bash
simd query gov vote 1 cosmos1jxd2uhx0j6e59306jq3jfqs7rhs7cnhvey4lqh
```

#### 吊销认证

授予者可以使用 revoke 命令撤销对被授予者的认证。

吊销认证:

```bash
simd tx authz revoke cosmos1khljzagdncfs03x5g6rf9qp5p93z9qgc3w5dwt /cosmos.gov.v1beta1.MsgVote --from alice
```

查看认证:

```bash
simd query authz grants cosmos1jxd2uhx0j6e59306jq3jfqs7rhs7cnhvey4lqh cosmos1khljzagdncfs03x5g6rf9qp5p93z9qgc3w5dwt /cosmos.gov.v1beta1.MsgVote
```

### 理解Feegrant模块

TODO

### 理解Group模块

TODO

### 理解Gov模块

TODO