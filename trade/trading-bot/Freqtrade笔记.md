
[freqtrade/freqtrade: Free, open source crypto trading bot (github.com)](https://github.com/freqtrade/freqtrade)

https://www.freqtrade.io/

## 介绍

一个自动化的加密货币交易工具，它是免费的开源软件，使用 Python 编写。它的优点有以下几点：

- 支持主流的交易所：你可以用 Freqtrade 连接你常用的加密货币交易所进行交易。
- 远程操控：你可以通过 Telegram 或网页界面来控制 Freqtrade 的运行，方便进行管理。
- 模拟测试和数据分析：Freqtrade 提供模拟测试功能，你可以不用投入真实资金来评估交易策略的有效性。同时也有图表等工具帮助你分析交易数据。
- 资金管理和策略优化：Freqtrade 具备资金管理功能，可以帮助你控制风险。此外，它还能利用机器学习来优化你的交易策略，使其更加有效。

需要注意的是，使用 Freqtrade 有一定的门槛，建议具备一定编程基础（Python）并熟悉加密货币交易的人使用。同时， Freqtrade 的开发者也强调，该软件仅供学习目的，使用前请充分了解风险，不对交易结果负责。

> 本软件仅用于教育目的。不要拿你害怕失去的钱冒险。使用该软件的风险由您自行承担。作者和所有关联公司对您的交易结果不承担任何责任。
> 
> 始终从在 Dry-run 中运行交易机器人开始，在您了解它的工作原理以及您应该预期的利润/损失之前不要参与资金。
> 
> 我们强烈建议您具备基本的编码技能和 Python 知识。不要犹豫，阅读源代码并了解该机器人的机制、其中实现的算法和技术。

![[Pasted image 20240626154755.png]]

### 功能

- 制定你的策略：使用 [pandas](https://pandas.pydata.org/) 用 python 编写你的策略。[策略存储库](https://github.com/freqtrade/freqtrade-strategies)中提供了激励您的示例策略。
- 下载市场数据：下载交易所和您可能想要交易的市场的历史数据。
- 回溯测试：根据下载的历史数据测试您的策略。
- 优化：使用采用加工学习方法的超优化为您的策略找到最佳参数。您可以为您的策略优化买入、卖出、止盈 （ROI）、止损和追踪止损参数。
- 选择市场：创建静态列表或使用基于最高交易量和/或价格的自动列表（在回测期间不可用）。您还可以明确地将您不想交易的市场列入黑名单。
- 运行：使用模拟货币（Dry run模式）测试您的策略或使用真实货币（实时交易模式）进行部署。
- 使用 Edge（可选模块）运行：该概念是根据止损的变化找到市场的最佳历史[交易预期](https://www.freqtrade.io/en/stable/edge/#expectancy)，然后允许/拒绝市场交易。交易的规模是基于您资本的一定百分比的风险。
- 控制/监控：使用 Telegram 或 WebUI（启动/停止机器人、显示盈亏、每日摘要、当前未平仓交易结果等）。
- 分析：可以对回测数据或Freqtrade交易历史（SQL数据库）进行进一步分析，包括自动标准图，以及将数据加载到[交互式环境中](https://www.freqtrade.io/en/stable/data-analysis/)的方法。

支持主流交易所，特别是Binance和OKX。

### 安装

#### 脚本安装

```bash
# update repository 
sudo apt-get update 
# install packages 
sudo apt install -y python3-pip python3-venv python3-dev python3-pandas git curl

# Download `develop` branch of freqtrade repository
git clone https://github.com/freqtrade/freqtrade.git

# Enter downloaded directory
cd freqtrade

# your choice (1): novice user
git checkout stable

# your choice (2): advanced user
git checkout develop

# --install, Install freqtrade from scratch 
./setup.sh -i

# activate virtual environment Each time you open a new terminal, you must run `source .venv/bin/activate` to activate your virtual environment.
source ./.venv/bin/activate
```

#### 手动安装

```bash
# This will use the ta-lib tar.gz included in this repository.
sudo ./build_helpers/install_ta-lib.sh

wget http://prdownloads.sourceforge.net/ta-lib/ta-lib-0.4.0-src.tar.gz
tar xvzf ta-lib-0.4.0-src.tar.gz
cd ta-lib
sed -i.bak "s|0.00000001|0.000000000000000001 |g" src/ta_func/ta_utility.h
./configure --prefix=/usr/local
make
sudo make install
# On debian based systems (debian, ubuntu, ...) - updating ldconfig might be necessary.
sudo ldconfig  
cd ..
rm -rf ./ta-lib*

# create virtualenv in directory /freqtrade/.venv
python3 -m venv .venv

# run virtualenv
source .venv/bin/activate

python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
python3 -m pip install -e .
```

#### 用Conda安装

```bash
# download freqtrade
git clone https://github.com/freqtrade/freqtrade.git

# enter downloaded directory 'freqtrade'
cd freqtrade

conda create --name freqtrade python=3.12

conda env list

# enter conda environment
conda activate freqtrade

# exit conda environment - don't do it now
conda deactivate

python3 -m pip install --upgrade pip
python3 -m pip install -r requirements.txt
python3 -m pip install -e .

# Linux Only
# Ensure that the environment is active!
conda activate freqtrade

# Linux only
cd build_helpers
bash install_ta-lib.sh ${CONDA_PREFIX} nosudo
```

#### 初始化

```bash
# Step 1 - Initialize user folder
freqtrade create-userdir --userdir user_data

# Step 2 - Create a new configuration file
freqtrade new-config --config user_data/config.json

freqtrade trade --config user_data/config.json --strategy SampleStrategy
```

> 记住，以上运行最好先了解Dry run是什么，也就是模拟模式运行。不要无脑启动trade bot，先了解原理和配置


## 基本原理

#### 术语

- **Strategy**: 你的策略，告诉交易机器人要做什么.
- **Trade**: Open position. 也就是未平仓头寸。头寸的意思就是你买或卖的位置，对于买，就是多头头寸。对于卖，就是空投头寸。平仓的意思就是你这个对应买单的卖单。
- **Open Order**: 就是尚未执行完成的订单，（未平仓）
- **Open Trade**: 未执行完成的交易 (未平仓)
- **Pair**: 交易对，也就是BTC/USDT   ETH/USDT等
- **Timeframe**:  蜡烛的时间周期，相当于K线图的周期，分时线，日线，周线。
- **Indicators**: 技术指标等(SMA, EMA, RSI, ...).
- **Limit order**: 限价单是指示以指定价格或更优惠的价格执行订单的指令。
- **Market order**: 市价单是一种指令以市场上可用的最佳价格执行订单的指令。这意味着交易所尽最大努力以提交订单时的市场价格成交订单。除非市场条件发生极端变化，例如交易所暂停交易或证券突然退市，否则市价单通常都能成交。当然也会影响市场价格
- **Current Profit**: 对于本次交易，当前未实现(未落袋)的盈利. 主要用在机器人的UI显示上
- **Realized Profit**: 已经实现/落袋的盈利.在 Freqtrade 交易策略中，“Realized Profit” 仅与**部分退出**（Partial Exits）相关。部分退出是指您仅出售部分持仓并保留剩余部分。这与完全退出（Full Exit）不同，完全退出是指您出售所有持仓。
- **Total Profit**: 未实现和已实现盈利的综合. 在 Freqtrade 交易策略中，“Total Profit” 的相对百分比（%）是根据该交易的**总投资**（Total Investment）计算出来的。总投资是指您为该交易投入的所有资金。`Total Profit (%) = (Total Profit / Total Investment) * 100`

#### 手续费处理(fee handing)

Freqtrade 中的所有利润计算都包含手续费。**回测 / 超参数优化 / 模拟运行模式** 下，Freqtrade 使用交易所的默认手续费 (通常是最低等级的手续费)。**实盘交易** 中，Freqtrade 使用交易所实际收取的手续费 (这可能包括 BNB 抵扣等折扣)。

#### 交易对命名

Fretrade的交易对遵循 [ccxt - documentation](https://docs.ccxt.com/#/README?id=consistency-of-base-and-quote-currencies) 的标准，如果写错，那么机器人将报错: his pair is not available

#### 现货交易对命名

For spot pairs, naming will be `base/quote` (e.g. `ETH/USDT`).

#### 期货交易对命名

For futures pairs, naming will be `base/quote:settle` (e.g. `ETH/USDT:USDT`).

#### 机器人执行逻辑

用dry-run模式启动或者实盘模式启动，启动机器人的同时会启动迭代循环，也就是运行了机器人启动的回调函数。默认是每隔几秒钟运行以下动作:

- 从持久化存储中获取未执行完成的交易
- 计算当前的可交易的交易对列表
- 下载交易对列表的 OHLCV 数据， 数据包括所有的不可交易的交易对[informative pairs](https://www.freqtrade.io/en/stable/strategy-customization/#get-data-for-non-tradeable-pairs)  
    这个步骤只会在每个蜡烛柱上执行一次，来避免不必要的网络流量.
- 调用 `bot_loop_start()` 策略回调.
- 对每个交易对进行策略分析.
    - 调用 `populate_indicators()`
    - 调用 `populate_entry_trend()`
    - 调用 `populate_exit_trend()`
- 从交易所拿到并更新交易的未执行的订单的状态.
    - 调用 `order_filled()` 已完成订单的策略回调  .
    - 检查未执行完成订单的超时时间.
        - 调用 `check_entry_timeout()`  入场(买)方向的未完成的订单的策略回调.
        - 调用 `check_exit_timeout()` 出场(卖)方向的未完成的订单的策略回调
        - 调用 `adjust_entry_price()` 入场(买)方向的未完成的订单的策略回调.
- 检验已存在的仓位(头寸) ，然后最终给这些仓位下对应的出场卖单.
    - 检测 止损, ROI 和 退出信号, `custom_exit()` and `custom_stoploss()`.
    - 确定 出场卖出价格，基于 `exit_pricing` 配置项 或者 通过使用 `custom_exit_price()` 回调.
    - 在一个卖单被下之前， `confirm_trade_exit()` 策略回调会被调用.
- 为未执行完成的订单检查仓位调整，如果这个被打开的话，打开是通过调用 `adjust_trade_position()` ， 如果有需要，会下其他额外的订单
- 检查是否还有可用的交易槽位(trade-slots) (如果 `max_open_trades`配置项设置的值达到).
- 检测入场信号，试图在一个新的位置上买入入场.
    - 确定入场买入价格，基于 `entry_pricing` 配置项, 或者通过使用  `custom_entry_price()` 回调.
    - 在保证金和期货模式下，调用`leverage()` 策略回调以确定所需的杠杆率。
    - 确定入场的代币数量 ，通过调用 `custom_stake_amount()` 回调.
    - Before an entry order is placed, `confirm_trade_entry()` strategy callback is called.

以上循环会一直重复

#### 回测和超参数的执行逻辑

其实也差不多

回测[backtesting](https://www.freqtrade.io/en/stable/backtesting/) 或 超参数 [hyperopt](https://www.freqtrade.io/en/stable/hyperopt/) 只会运行以上一部分逻辑, 因为大部分的交易操作是完全模拟化的.

- 从配置的交易对列表中加载历史数据.
- 只会调用 `bot_start()` 一次.
- 计算指标 (每个交易对调用 `populate_indicators()` 一次).
- 计算入场退场信号 (每个交易对调用 `populate_entry_trend()` 和 `populate_exit_trend()` 一次).
- 在每根蜡烛上循环模拟入场和出场的点位 .
    - 调用 `bot_loop_start()` 策略回调.
    - 检测订单的超时, 要不通过 `unfilledtimeout` 配置, 或者通过 `check_entry_timeout()` / `check_exit_timeout()` 策略回调.
    - 为尚未执行完成的入场订单(买单)，调用 `adjust_entry_price()` 策略回调
    - 检测交易的入场信号 (`enter_long` / `enter_short` columns列).
    - 确定交易的入场和推出 (调用 `confirm_trade_entry()` 和 `confirm_trade_exit()` 如果在策略中被实现).
    - 调用 `custom_entry_price()` (如果在策略中被实现) 来决定入场价格 (价格被调整至开盘蜡烛内).
    - 在保证金和期货模式下, 调用`leverage()` s策略回调以确定所需的杠杆率 
    - 决定质押大小， 通过调用`custom_stake_amount()` 回调.
    - 检查未完成交易的位置调整， 如果打开和调用了 `adjust_trade_position()`来确定是否需要额外的订单来调整
    - 为已经完成的入场订单(买单)调用 `order_filled()` 策略回调.
    - 调用 `custom_stoploss()` 和 `custom_exit()` 来找到自定义的退出点位置
    - 对于退出来说，基于退出信号, 自定义退出和部分退出: 调用 `custom_exit_price()` 来确定退出价格 (价格被调整到收盘蜡烛内).
    - 为已完成的退出订单(卖单)调用 `order_filled()` 策略回调.
- 生成并输出回测报告

> 对于回测和 超参数 在计算中包括交易所默认费用。通过指定“--fee”参数，可以将自定义费用传递给回测和hyperopt.


> 回测会尽最大努力在每个蜡烛上调用一次每个回调函数. (`--timeframe-detail` 这个参数会修改这个行为到 每个更小周期的蜡烛一次). 在实盘中大多数回调在每一次循环迭代中会被调用一次，（大约每5秒一次循环） - 这样可能会导致与回测不一致，不匹配.


## 配置


[Configuration - Freqtrade](https://www.freqtrade.io/en/stable/configuration/)

Freqtrades有狠毒可配置的特性和可能性。默认，这些设置都被配置在配置文件中。

#### Freqtrade的配置文件

机器人在操作过程中使用一组配置参数，这些参数都符合机器人配置。它通常从文件（Freqtrade配置文件）中读取其配置。

默认情况下，机器人从位于当前工作目录中的`config.json`文件加载配置

你可以指定不同的配置文件给机器人使用， 通过 `-c/--config` 命令行选项.

如果你是使用 [Quick start](https://www.freqtrade.io/en/stable/docker_quickstart/#docker-quick-start) 方法来安装的机器人,安装脚本应该给你创建了一个默认的配置文件了(`config.json`) .

如果默认的配置文件不存在，我们推荐使用 `freqtrade new-config --config user_data/config.json` 来生成一个基本的配置文件

配置文件是JSON格式的.

我们对JSON的标准语法做了提高，可以加入这样的单行注释 `// ...` 和 多行注释 `/* ... */` 

配置文件很简单，就是变更配置项，重启机器人 或者,如果之前机器人就是停止的, 那么再次运行，就是使用的变更后的配置. 机器人在启动时会校验JSON的格式，如果有错误，会直接警告输出报错

##### 环境变量

通过环境变量在Freqtrade配置中设置选项。 这优先于配置文件或策略中的相应值。也就是环境变量的优先级更高。

环境变量必须以前缀`FREQTRADE__`为前缀才能加载到 freqtrade 配置中。

`__` 作为一个缩进级别的分隔符, 所以格式应该是这样 `FREQTRADE__{section}__{key}`. 比如 - 一个环境变量被定义成 `export FREQTRADE__STAKE_AMOUNT=200` 就相当于修改配置文件中的 `{stake_amount: 200}`.

一个更复杂点的例子是 `export FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>` 来保存你的交易所密钥. 这个就会把密钥值移动到 配置文件的 `exchange.key`区域中 . 使用这种模式,  所有的配置设置都可以被设置为环境变量。

 请一定要注意，环境变量会覆盖配置文件的配置，但是命令行参数比环境变量优先级更高，是最高的优先级。 

以下是常见的环境变量例子

```bash
FREQTRADE__TELEGRAM__CHAT_ID=<telegramchatid>
FREQTRADE__TELEGRAM__TOKEN=<telegramToken>
FREQTRADE__EXCHANGE__KEY=<yourExchangeKey>
FREQTRADE__EXCHANGE__SECRET=<yourExchangeSecret>
```

> 环境变量在机器人启动的时候会被探测到 - 所以如果你不能找到一个值在配置文件中是否生效，那么请你确认它不是从环境变量中加载的 

> 环境变量其实是在初始化配置之后被加载的，这样 如果你使用环境变量你可以就不提供配置文件的路径了. 请使用 `--config path/to/config.json`  这在某种程度上也适用于user_dir。虽然可以通过环境变量设置用户目录，但不会从该位置加载配置。
##### 多个配置文件

机器人可以指定和使用多个配置文件，或者机器人可以从进程标准输入流中读取其配置参数。

你可以用`add_config_files`来指定额外的配置文件. 这个参数指定的文件会被加载合并到最初始的配置文件中. 该文件相对于初始配置文件进行解析（类似于C语言的include）用命令行 `--config` 参数也可以指定多个配置文件, 但是更简单的使用是你不必为所有的命令指定所有的文件。

> 可以使用多个配置文件来解决你保存敏感信息的问题，也就是敏感信息的配置和一般性的配置解耦。`freqtrade trade --config user_data/config1.json --config user_data/config-private.json <...>`

> ```json
> "add_config_files": [
    "config1.json",
    "config-private.json"
]
> ```

> 配置文件的配置项覆盖问题，如果哪个配置文件最后被加载，那么最后被加载的值会覆盖之前加载的值。

#### 配置参数

以下表格会列出所有的可配置参数

Freqtrade 也可以通过命令行参数加载许多选项， 请看 `--help` .
##### 配置选项的优先级

所有配置选项的优先级从高到低如下:

- CLI命令行参数会覆盖所有配置
- 环境变量
- 配置文件被使用的顺序，最后一个配置文件的优先级相对最高，会覆盖之前的配置文件的冲突项
- 策略配置只会在没有通过通过配置文件或命令行设置配置的时候生效，这些策略配置项已经在下表以策略覆盖标记出来了

##### 参数列表

必需参数标记为“Required”，这意味着需要以一种可能的方式设置它们。


| Parameter                                                | Description                                                                                                                                                                                                                                                                                                                                                                              |
| -------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `max_open_trades`                                        | **Required.**  你的机器人被允许的未完成（未平仓）的交易数量 ， 每一个交易对只能进行一次未平仓交易, 因此交易对列表的长度是另一个可以应用的限制. 如果设置未-1就忽略这个参数 (i.e. 不对未平仓交易数量设限, 被交易对列表长度设限). [More information below](https://www.freqtrade.io/en/stable/configuration/#configuring-amount-per-trade). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Positive integer or -1. |
| `stake_currency`                                         | **Required.** 用来做交易的加密货币，比如你投资USDT或者是BTC，或者ETH来做这个机器人的量化交易.  <br>**Datatype:** String                                                                                                                                                                                                                                                                                                    |
| `stake_amount`                                           | **Required.** 加密货币每次交易的数量. 设置为 `"unlimited"` 允许机器人使用所有的可用余额. [More information below](https://www.freqtrade.io/en/stable/configuration/#configuring-amount-per-trade).  <br>**Datatype:** Positive float or `"unlimited"`.                                                                                                                                                               |
| `tradable_balance_ratio`                                 | 允许机器人可以交易的账户所有余额的比例. [More information below](https://www.freqtrade.io/en/stable/configuration/#configuring-amount-per-trade).  <br>_Defaults to `0.99` 99%)._  <br>**Datatype:** Positive float between `0.1` and `1.0`.                                                                                                                                                                |
| `available_capital`                                      | 机器人的可用启动资金。在同一个交易所帐户上运行多个机器人时很有用.  [More information below](https://www.freqtrade.io/en/stable/configuration/#configuring-amount-per-trade).  <br>**Datatype:** Positive float.                                                                                                                                                                                                          |
| `amend_last_stake_amount`                                | 如有必要，使用减少的最后质押（投资）金额. [More information below](https://www.freqtrade.io/en/stable/configuration/#configuring-amount-per-trade).  <br>_Defaults to `false`._  <br>**Datatype:** Boolean                                                                                                                                                                                                   |
| `last_stake_amount_min_ratio`                            | 定义必须保留和执行的最小质押(投资)金额。仅适用于修改为降低值的最后一个质押（投资）金额 (i.e. if `amend_last_stake_amount` is set to `true`). [More information below](https://www.freqtrade.io/en/stable/configuration/#configuring-amount-per-trade).  <br>_Defaults to `0.5`._  <br>**Datatype:** Float (as ratio)                                                                                                               |
| `amount_reserve_percent`                                 | 在最小交易对金额中保留一些金额 . 机器人会保留 `amount_reserve_percent` + stoploss 值 当计算最小交易对质押(投资) 以避免可能的交易拒绝.  <br>_Defaults to `0.05` (5%)._  <br>**Datatype:** Positive Float as ratio.                                                                                                                                                                                                                    |
| `timeframe`                                              | 时间级别周期 (e.g `1m`, `5m`, `15m`, `30m`, `1h` ...). 一般不在配置文件中指定, 而是在策略源码文件中指定. [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** String                                                                                                                                                                                 |
| `fiat_display_currency`                                  | 用于显示您的利润的法定货币. [More information below](https://www.freqtrade.io/en/stable/configuration/#what-values-can-be-used-for-fiat_display_currency).  <br>**Datatype:** String                                                                                                                                                                                                                  |
| `dry_run`                                                | **Required.** 定义机器人是否用实盘还是Dry -Run模式运行，默认是Dry run模式跑.  <br>_Defaults to `true`._  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                                              |
| `dry_run_wallet`                                         | 定义Dry run模式下的钱包投资金额.  <br>_Defaults to `1000`._  <br>**Datatype:** Float                                                                                                                                                                                                                                                                                                                 |
| `cancel_open_orders_on_exit`                             | 取消未平仓订单 当 `/stop` RPC 指令被发出, `Ctrl+C` 被按下 或者机器人不在预期内死亡 . 当设置未 `true`, 这个允许你使用 `/stop` 来取消 未平仓 和部分平仓的交易 在一个市场崩溃 的事件下，不会影响开仓的仓位  <br>_Defaults to `false`._  <br>**Datatype:** Boolean                                                                                                                                                                                                     |
| `process_only_new_candles`                               | 仅当新蜡烛到达时才启用指标处理。如果 false 每个循环填充指标，这将意味着同一根蜡烛被多次处理，从而产生系统负载，但您的策略可能有用，具体取决于分时数据，而不仅仅是蜡烛. 一般是写在策略源码中 [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `true`._  <br>**Datatype:** Boolean                                                                                                                              |
| `minimal_roi`                                            | **Required.** 将阈值设置为机器人将用于退出交易的比率，一般写在策略源码中  [More information below](https://www.freqtrade.io/en/stable/configuration/#understand-minimal_roi). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Dict                                                                                                              |
| `stoploss`                                               | **Required.** 机器人用于止损的比率，一般写在策略源码中. More details in the [stoploss documentation](https://www.freqtrade.io/en/stable/stoploss/). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Float (as ratio)                                                                                                                   |
| `trailing_stop`                                          | 启用追踪止损 （根据你的止损配置是写在策略源码中还是配置文件中，与止损的配置位置保持一致就行）  More details in the [stoploss documentation](https://www.freqtrade.io/en/stable/stoploss/#trailing-stop-loss). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Boolean                                                                                            |
| `trailing_stop_positive`                                 | 达到盈利后更改止损. More details in the [stoploss documentation](https://www.freqtrade.io/en/stable/stoploss/#trailing-stop-loss-custom-positive-loss). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Float                                                                                                               |
| `trailing_stop_positive_offset`                          | 当设置了`trailing_stop_positive`的偏移 .  是正数，比率偏移 More details in the [stoploss documentation](https://www.freqtrade.io/en/stable/stoploss/#trailing-stop-loss-only-once-the-trade-has-reached-a-certain-offset). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `0.0` (no offset)._  <br>**Datatype:** Float            |
| `trailing_only_offset_is_reached`                        | 仅在达到偏移量时应用追踪止损. [stoploss documentation](https://www.freqtrade.io/en/stable/stoploss/). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `false`._  <br>**Datatype:** Boolean                                                                                                                                        |
| `fee`                                                    | 回溯测试/试运行期间使用的费用。通常不应配置，这有 freqtrade 回退到交易所默认费用.. 设置比率 (e.g. 0.001 = 0.1%). 一次交易会运用两次手续费, 一次是买, 一次是卖.  <br>**Datatype:** Float (as ratio)                                                                                                                                                                                                                                                 |
| `futures_funding_rate`                                   | 当交易所无法提供历史资金利率时，将使用用户指定的资金利率。这不会覆盖实际的历史利率。建议将其设置为 0，除非您正在测试特定的代币并且您了解资金费率将如何影响 freqtrade 的利润计算. [More information here](https://www.freqtrade.io/en/stable/leverage/#unavailable-funding-rates)  <br>_Defaults to `None`._  <br>**Datatype:** Float                                                                                                                                      |
| `trading_mode`                                           | 指定您是要定期交易、使用杠杆交易，还是交易价格来自匹配加密货币价格的合约. [leverage documentation](https://www.freqtrade.io/en/stable/leverage/).  <br>_Defaults to `"spot"`._  <br>**Datatype:** String                                                                                                                                                                                                                     |
| `margin_mode`                                            | 当使用杠杆进行交易时，这决定了交易者拥有的抵押品是共享还是隔离到每个交易对 [leverage documentation](https://www.freqtrade.io/en/stable/leverage/).  <br>**Datatype:** String                                                                                                                                                                                                                                                  |
| `liquidation_buffer`                                     | 指定在强制平仓价格和止损之间设置多大的安全网以防止头寸达到强制平仓价格的比率 [leverage documentation](https://www.freqtrade.io/en/stable/leverage/).  <br>_Defaults to `0.05`._  <br>**Datatype:** Float                                                                                                                                                                                                                       |
|                                                          | **Unfilled timeout**                                                                                                                                                                                                                                                                                                                                                                     |
| `unfilledtimeout.entry`                                  | **Required.** 机器人将等待未平仓的入场订单完成多长时间（以分钟或秒为单位），之后只要有信号，订单将被取消并以当前（新）价格重复. [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Integer                                                                                                                                                                                    |
| `unfilledtimeout.exit`                                   | **Required.** 机器人将等待未平仓的出场订单完成多长时间（以分钟或秒为单位），之后只要有信号，订单将被取消并以当前（新）价格重复. [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Integer                                                                                                                                                                                    |
| `unfilledtimeout.unit`                                   | 在未平仓超时设置中使用的单位。注意：如果将 unfilledtimeout.unit 设置为“seconds”，则“internals.process_throttle_secs”必须低于或等于超时 [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `"minutes"`._  <br>**Datatype:** String                                                                                                                         |
| `unfilledtimeout.exit_timeout_count`                     | 退出订单可以超时多少次。一旦达到此超时次数，就会触发紧急退出。0 禁用并允许无限制的订单取消 [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `0`._  <br>**Datatype:** Integer                                                                                                                                                                                     |
|                                                          | **Pricing**                                                                                                                                                                                                                                                                                                                                                                              |
| `entry_pricing.price_side`                               | 选择机器人应查看的点差一侧以获取入场率. [More information below](https://www.freqtrade.io/en/stable/configuration/#entry-price).  <br>_Defaults to `"same"`._  <br>**Datatype:** String (either `ask`, `bid`, `same` or `other`).                                                                                                                                                                           |
| `entry_pricing.price_last_balance`                       | **Required.** 插值出价. More information [below](https://www.freqtrade.io/en/stable/configuration/#entry-price-without-orderbook-enabled).                                                                                                                                                                                                                                                   |
| `entry_pricing.use_order_book`                           | 使用订单簿的项作为入场  [Order Book Entry](https://www.freqtrade.io/en/stable/configuration/#entry-price-with-orderbook-enabled).  <br>_Defaults to `true`._  <br>**Datatype:** Boolean                                                                                                                                                                                                             |
| `entry_pricing.order_book_top`                           | Bot will use the top N rate in Order Book "price_side" to enter a trade. I.e. a value of 2 will allow the bot to pick the 2nd entry in [Order Book Entry](https://www.freqtrade.io/en/stable/configuration/#entry-price-with-orderbook-enabled).  <br>_Defaults to `1`._  <br>**Datatype:** Positive Integer                                                                             |
| `entry_pricing. check_depth_of_market.enabled`           | 如果订单簿中满足买单和卖单的差额，则不要输入. [Check market depth](https://www.freqtrade.io/en/stable/configuration/#check-depth-of-market).  <br>_Defaults to `false`._  <br>**Datatype:** Boolean                                                                                                                                                                                                            |
| `entry_pricing. check_depth_of_market.bids_to_ask_delta` | 订单簿中买入订单和卖出订单的差额比率。值低于 1 表示卖单规模更大，而值大于 1 表示买入订单规模更大. [Check market depth](https://www.freqtrade.io/en/stable/configuration/#check-depth-of-market)  <br>_Defaults to `0`._  <br>**Datatype:** Float (as ratio)                                                                                                                                                                           |
| `exit_pricing.price_side`                                | 选择机器人应查看的点差一侧以获取出场率. [More information below](https://www.freqtrade.io/en/stable/configuration/#exit-price-side).  <br>_Defaults to `"same"`._  <br>**Datatype:** String (either `ask`, `bid`, `same` or `other`).                                                                                                                                                                       |
| `exit_pricing.price_last_balance`                        | 插值出场价格. More information [below](https://www.freqtrade.io/en/stable/configuration/#exit-price-without-orderbook-enabled).                                                                                                                                                                                                                                                                |
| `exit_pricing.use_order_book`                            | 使用订单簿退出来开启未平仓交易的退出 [Order Book Exit](https://www.freqtrade.io/en/stable/configuration/#exit-price-with-orderbook-enabled).  <br>_Defaults to `true`._  <br>**Datatype:** Boolean                                                                                                                                                                                                         |
| `exit_pricing.order_book_top`                            | Bot will use the top N rate in Order Book "price_side" to exit. I.e. a value of 2 will allow the bot to pick the 2nd ask rate in [Order Book Exit](https://www.freqtrade.io/en/stable/configuration/#exit-price-with-orderbook-enabled)  <br>_Defaults to `1`._  <br>**Datatype:** Positive Integer                                                                                      |
| `custom_price_max_distance_ratio`                        | 配置当前价格和自定义入场或出场价格之间的最大距离比.  <br>_Defaults to `0.02` 2%)._  <br>**Datatype:** Positive float                                                                                                                                                                                                                                                                                              |
|                                                          | **TODO**                                                                                                                                                                                                                                                                                                                                                                                 |
| `use_exit_signal`                                        | 使用策略产生的退出信号（除了minimal_roi）.  <br>设置这个未false会关闭 exit_long 和exit_short的使用 . 不会对其他的退出方法产生影响(Stoploss, ROI, callbacks). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `true`._  <br>**Datatype:** Boolean                                                                                                             |
| `exit_profit_only`                                       | 等到直到机器人达到 `exit_profit_offset` 配置的盈利偏移才开始做退出决策. [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `false`._  <br>**Datatype:** Boolean                                                                                                                                                                                |
| `exit_profit_offset`                                     | 只在此值之上 ，退出信号才被激活， 只有exit_profile_only打开时，这个配置才生效 . [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `0.0`._  <br>**Datatype:** Float (as ratio)                                                                                                                                                                      |
| `ignore_roi_if_entry_signal`                             | 如果入场信号一直处于激活状态，那么就不要退出 这个设置优先于   `minimal_roi` 和 `use_exit_signal`配置组合. [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `false`._  <br>**Datatype:** Boolean                                                                                                                                                        |
| `ignore_buying_expired_candle_after`                     | 指定秒数，直到一个买入信号不再被使用.  <br>**Datatype:** Integer                                                                                                                                                                                                                                                                                                                                           |
| `order_types`                                            | 根据行为动作，配置订单类型，行为 (`"entry"`, `"exit"`, `"stoploss"`, `"stoploss_on_exchange"`). [More information below](https://www.freqtrade.io/en/stable/configuration/#understand-order_types). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Dict                                                                           |
| `order_time_in_force`                                    | 为买单和卖单配置强制配置时间 [More information below](https://www.freqtrade.io/en/stable/configuration/#understand-order_time_in_force). [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>**Datatype:** Dict                                                                                                                                    |
| `position_adjustment_enable`                             | 开启策略使用仓位调整 （买和卖. [More information here](https://www.freqtrade.io/en/stable/strategy-callbacks/#adjust-trade-position).  <br>[Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `false`._  <br>**Datatype:** Boolean                                                                                                   |
| `max_entry_position_adjustment`                          | 每个未平仓交易（在订单簿买入方向的第一个订单之上）的最大额外订单数量 . 设置未 `-1` 是对数量无限制. [More information here](https://www.freqtrade.io/en/stable/strategy-callbacks/#adjust-trade-position).  <br>[Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `-1`._  <br>**Datatype:** Positive Integer or -1                                                 |
|                                                          | **Exchange**                                                                                                                                                                                                                                                                                                                                                                             |
| `exchange.name`                                          | **Required.** 使用的交易所名字.  <br>**Datatype:** String                                                                                                                                                                                                                                                                                                                                        |
| `exchange.key`                                           | 交易所的API Key. 只有使用实盘模式才是必填项.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                                                                                                                              |
| `exchange.secret`                                        | 交易所的API secret. 实盘模式才要求必填.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                                                                                                                               |
| `exchange.password`                                      | 交易所的API密码. 实盘模式才必填，就是API请求的密码.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                                                                                                                           |
| `exchange.uid`                                           | 交易所的API uid. 只有使用实盘模式才必填 API请求的uid.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                                                                                                                      |
| `exchange.pair_whitelist`                                | 机器人用于交易的货币对列表，并在回测期间检查潜在交易. 支持正则表达式的交易对 `.*/BTC`. 没有被 VolumePairList使用. [More information](https://www.freqtrade.io/en/stable/plugins/#pairlists-and-pairlist-handlers).  <br>**Datatype:** List                                                                                                                                                                                         |
| `exchange.pair_blacklist`                                | 机器人在交易和回测期间绝对避免的交易对 . [More information](https://www.freqtrade.io/en/stable/plugins/#pairlists-and-pairlist-handlers).  <br>**Datatype:** List                                                                                                                                                                                                                                           |
| `exchange.ccxt_config`                                   | 传递给两个 ccxt 实例（同步和异步）的其他 CCXT 参数。这通常是其他 ccxt 配置的正确位置。参数可能因交易所而异，并记录在 [ccxt 文档](https://ccxt.readthedocs.io/en/latest/manual.html#instantiation)中。请避免在此处添加交换密钥（请改用专用字段），因为它们可能包含在日志中。<br>**Datatype:** Dict                                                                                                                                                                                |
| `exchange.ccxt_sync_config`                              | 传递给常规（同步）ccxt 实例的其他 CCXT 参数。参数可能因交易所而异，并记录在 [ccxt 文档](https://ccxt.readthedocs.io/en/latest/manual.html#instantiation)中<br>**Datatype:** Dict                                                                                                                                                                                                                                            |
| `exchange.ccxt_async_config`                             | 传递给异步 ccxt 实例的其他 CCXT 参数。参数可能因交易所而异，并记录在 [ccxt 文档](https://ccxt.readthedocs.io/en/latest/manual.html#instantiation)中<br>**Datatype:** Dict                                                                                                                                                                                                                                               |
| `exchange.markets_refresh_interval`                      | 市场的刷新时间间隔  <br>_Defaults to `60` minutes._  <br>**Datatype:** Positive Integer                                                                                                                                                                                                                                                                                                           |
| `exchange.skip_pair_validation`                          | 跳过启动时的交易列表检测.  <br>_Defaults to `false`_  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                                                                                      |
| `exchange.skip_open_order_update`                        | 如果交易所导致问题，则在启动时跳过未平仓单更新。仅与现场条件相关  <br>_Defaults to `false`_  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                                                                   |
| `exchange.unknown_fee_rate`                              | 计算交易费用时使用的回退值。这对于以不可交易货币收取费用的交易所很有用。此处提供的值将乘以“费用成本”。<br>_Defaults to `None`  <br>__Datatype:_* float                                                                                                                                                                                                                                                                                     |
| `exchange.log_responses`                                 | 记录相关的交易所响应。仅适用于调试模式 - 小心使用。<br>_Defaults to `false`_  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                                                                          |
| `experimental.block_bad_exchanges`                       | 禁用已知不适用于 freqtrade 的区块交易所。除非您想测试该交换现在是否有效，否则请保留默认值。<br>_Defaults to `true`._  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                                                  |
|                                                          | **Plugins**                                                                                                                                                                                                                                                                                                                                                                              |
| `edge.*`                                                 | Please refer to [edge configuration document](https://www.freqtrade.io/en/stable/edge/) for detailed explanation of all possible configuration options.                                                                                                                                                                                                                                  |
| `pairlists`                                              | Define one or more pairlists to be used. [More information](https://www.freqtrade.io/en/stable/plugins/#pairlists-and-pairlist-handlers).  <br>_Defaults to `StaticPairList`._  <br>**Datatype:** List of Dicts                                                                                                                                                                          |
| `protections`                                            | Define one or more protections to be used. [More information](https://www.freqtrade.io/en/stable/plugins/#protections).  <br>**Datatype:** List of Dicts                                                                                                                                                                                                                                 |
|                                                          | **Telegram**                                                                                                                                                                                                                                                                                                                                                                             |
| `telegram.enabled`                                       | Enable the usage of Telegram.  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                                                                                                 |
| `telegram.token`                                         | Your Telegram bot token. Only required if `telegram.enabled` is `true`.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                                                                                  |
| `telegram.chat_id`                                       | Your personal Telegram account id. Only required if `telegram.enabled` is `true`.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                                                                        |
| `telegram.balance_dust_level`                            | Dust-level (in stake currency) - currencies with a balance below this will not be shown by `/balance`.  <br>**Datatype:** float                                                                                                                                                                                                                                                          |
| `telegram.reload`                                        | Allow "reload" buttons on telegram messages.  <br>_Defaults to `true`.  <br>__Datatype:_* boolean                                                                                                                                                                                                                                                                                        |
| `telegram.notification_settings.*`                       | Detailed notification settings. Refer to the [telegram documentation](https://www.freqtrade.io/en/stable/telegram-usage/) for details.  <br>**Datatype:** dictionary                                                                                                                                                                                                                     |
| `telegram.allow_custom_messages`                         | Enable the sending of Telegram messages from strategies via the dataprovider.send_msg() function.  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                             |
|                                                          | **Webhook**                                                                                                                                                                                                                                                                                                                                                                              |
| `webhook.enabled`                                        | Enable usage of Webhook notifications  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                                                                                         |
| `webhook.url`                                            | URL for the webhook. Only required if `webhook.enabled` is `true`. See the [webhook documentation](https://www.freqtrade.io/en/stable/webhook-config/) for more details.  <br>**Datatype:** String                                                                                                                                                                                       |
| `webhook.entry`                                          | Payload to send on entry. Only required if `webhook.enabled` is `true`. See the [webhook documentation](https://www.freqtrade.io/en/stable/webhook-config/) for more details.  <br>**Datatype:** String                                                                                                                                                                                  |
| `webhook.entry_cancel`                                   | Payload to send on entry order cancel. Only required if `webhook.enabled` is `true`. See the [webhook documentation](https://www.freqtrade.io/en/stable/webhook-config/) for more details.  <br>**Datatype:** String                                                                                                                                                                     |
| `webhook.entry_fill`                                     | Payload to send on entry order filled. Only required if `webhook.enabled` is `true`. See the [webhook documentation](https://www.freqtrade.io/en/stable/webhook-config/) for more details.  <br>**Datatype:** String                                                                                                                                                                     |
| `webhook.exit`                                           | Payload to send on exit. Only required if `webhook.enabled` is `true`. See the [webhook documentation](https://www.freqtrade.io/en/stable/webhook-config/) for more details.  <br>**Datatype:** String                                                                                                                                                                                   |
| `webhook.exit_cancel`                                    | Payload to send on exit order cancel. Only required if `webhook.enabled` is `true`. See the [webhook documentation](https://www.freqtrade.io/en/stable/webhook-config/) for more details.  <br>**Datatype:** String                                                                                                                                                                      |
| `webhook.exit_fill`                                      | Payload to send on exit order filled. Only required if `webhook.enabled` is `true`. See the [webhook documentation](https://www.freqtrade.io/en/stable/webhook-config/) for more details.  <br>**Datatype:** String                                                                                                                                                                      |
| `webhook.status`                                         | Payload to send on status calls. Only required if `webhook.enabled` is `true`. See the [webhook documentation](https://www.freqtrade.io/en/stable/webhook-config/) for more details.  <br>**Datatype:** String                                                                                                                                                                           |
| `webhook.allow_custom_messages`                          | Enable the sending of Webhook messages from strategies via the dataprovider.send_msg() function.  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                              |
|                                                          | **Rest API / FreqUI / Producer-Consumer**                                                                                                                                                                                                                                                                                                                                                |
| `api_server.enabled`                                     | Enable usage of API Server. See the [API Server documentation](https://www.freqtrade.io/en/stable/rest-api/) for more details.  <br>**Datatype:** Boolean                                                                                                                                                                                                                                |
| `api_server.listen_ip_address`                           | Bind IP address. See the [API Server documentation](https://www.freqtrade.io/en/stable/rest-api/) for more details.  <br>**Datatype:** IPv4                                                                                                                                                                                                                                              |
| `api_server.listen_port`                                 | Bind Port. See the [API Server documentation](https://www.freqtrade.io/en/stable/rest-api/) for more details.  <br>**Datatype:** Integer between 1024 and 65535                                                                                                                                                                                                                          |
| `api_server.verbosity`                                   | Logging verbosity. `info` will print all RPC Calls, while "error" will only display errors.  <br>**Datatype:** Enum, either `info` or `error`. Defaults to `info`.                                                                                                                                                                                                                       |
| `api_server.username`                                    | Username for API server. See the [API Server documentation](https://www.freqtrade.io/en/stable/rest-api/) for more details.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                              |
| `api_server.password`                                    | Password for API server. See the [API Server documentation](https://www.freqtrade.io/en/stable/rest-api/) for more details.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                              |
| `api_server.ws_token`                                    | API token for the Message WebSocket. See the [API Server documentation](https://www.freqtrade.io/en/stable/rest-api/) for more details.  <br>**Keep it in secret, do not disclose publicly.**  <br>**Datatype:** String                                                                                                                                                                  |
| `bot_name`                                               | Name of the bot. Passed via API to a client - can be shown to distinguish / name bots.  <br>_Defaults to `freqtrade`_  <br>**Datatype:** String                                                                                                                                                                                                                                          |
| `external_message_consumer`                              | Enable [Producer/Consumer mode](https://www.freqtrade.io/en/stable/producer-consumer/) for more details.  <br>**Datatype:** Dict                                                                                                                                                                                                                                                         |
|                                                          | **Other**                                                                                                                                                                                                                                                                                                                                                                                |
| `initial_state`                                          | Defines the initial application state. If set to stopped, then the bot has to be explicitly started via `/start` RPC command.  <br>_Defaults to `stopped`._  <br>**Datatype:** Enum, either `stopped` or `running`                                                                                                                                                                       |
| `force_entry_enable`                                     | Enables the RPC Commands to force a Trade entry. More information below.  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                                                      |
| `disable_dataframe_checks`                               | Disable checking the OHLCV dataframe returned from the strategy methods for correctness. Only use when intentionally changing the dataframe and understand what you are doing. [Strategy Override](https://www.freqtrade.io/en/stable/configuration/#parameters-in-the-strategy).  <br>_Defaults to `False`_.  <br>**Datatype:** Boolean                                                 |
| `internals.process_throttle_secs`                        | Set the process throttle, or minimum loop duration for one bot iteration loop. Value in second.  <br>_Defaults to `5` seconds._  <br>**Datatype:** Positive Integer                                                                                                                                                                                                                      |
| `internals.heartbeat_interval`                           | Print heartbeat message every N seconds. Set to 0 to disable heartbeat messages.  <br>_Defaults to `60` seconds._  <br>**Datatype:** Positive Integer or 0                                                                                                                                                                                                                               |
| `internals.sd_notify`                                    | Enables use of the sd_notify protocol to tell systemd service manager about changes in the bot state and issue keep-alive pings. See [here](https://www.freqtrade.io/en/stable/advanced-setup/#configure-the-bot-running-as-a-systemd-service) for more details.  <br>**Datatype:** Boolean                                                                                              |
| `strategy`                                               | **Required** Defines Strategy class to use. Recommended to be set via `--strategy NAME`.  <br>**Datatype:** ClassName                                                                                                                                                                                                                                                                    |
| `strategy_path`                                          | Adds an additional strategy lookup path (must be a directory).  <br>**Datatype:** String                                                                                                                                                                                                                                                                                                 |
| `recursive_strategy_search`                              | Set to `true` to recursively search sub-directories inside `user_data/strategies` for a strategy.  <br>**Datatype:** Boolean                                                                                                                                                                                                                                                             |
| `user_data_dir`                                          | Directory containing user data.  <br>_Defaults to `./user_data/`_.  <br>**Datatype:** String                                                                                                                                                                                                                                                                                             |
| `db_url`                                                 | Declares database URL to use. NOTE: This defaults to `sqlite:///tradesv3.dryrun.sqlite` if `dry_run` is `true`, and to `sqlite:///tradesv3.sqlite` for production instances.  <br>**Datatype:** String, SQLAlchemy connect string                                                                                                                                                        |
| `logfile`                                                | Specifies logfile name. Uses a rolling strategy for log file rotation for 10 files with the 1MB limit per file.  <br>**Datatype:** String                                                                                                                                                                                                                                                |
| `add_config_files`                                       | Additional config files. These files will be loaded and merged with the current config file. The files are resolved relative to the initial file.  <br>_Defaults to `[]`_.  <br>**Datatype:** List of strings                                                                                                                                                                            |
| `dataformat_ohlcv`                                       | Data format to use to store historical candle (OHLCV) data.  <br>_Defaults to `feather`_.  <br>**Datatype:** String                                                                                                                                                                                                                                                                      |
| `dataformat_trades`                                      | Data format to use to store historical trades data.  <br>_Defaults to `feather`_.  <br>**Datatype:** String                                                                                                                                                                                                                                                                              |
| `reduce_df_footprint`                                    | Recast all numeric columns to float32/int32, with the objective of reducing ram/disk usage (and decreasing train/inference timing in FreqAI). (Currently only affects FreqAI use-cases)  <br>**Datatype:** Boolean.  <br>Default: `False`.                                                                                                                                               |

##### 策略的配置参数

可以在配置文件或策略中设置以下参数。 配置文件中设置的值始终覆盖策略中设置的值。

- `minimal_roi`
- `timeframe`
- `stoploss`
- `max_open_trades`
- `trailing_stop`
- `trailing_stop_positive`
- `trailing_stop_positive_offset`
- `trailing_only_offset_is_reached`
- `use_custom_stoploss`
- `process_only_new_candles`
- `order_types`
- `order_time_in_force`
- `unfilledtimeout`
- `disable_dataframe_checks`
- `use_exit_signal`
- `exit_profit_only`
- `exit_profit_offset`
- `ignore_roi_if_entry_signal`
- `ignore_buying_expired_candle_after`
- `position_adjustment_enable`
- `max_entry_position_adjustment`

##### 配置每次交易的金额数量

有几种方法可以配置机器人将用于入场交易的投资货币量。所有方法都遵循[可用的余额配置](https://www.freqtrade.io/en/stable/configuration/#tradable-balance)，如下所述。

###### 最小投资质押交易

最低质押投资金额将取决于交易所和交易对，通常列在交易所支持页面中

假设XRP/USD最小的可交易金额数量 是 20 XRP (由交易所给定), 1个XRP的价格是 0.6\(, 那么对于买单而言，最小的质押投资金额未 `20 * 0.6 ~= 12`. 对于美元也是有限制的- 对于所有的订单必须大于 10\) - 这里只是举例.

为了保证安全的执行, freqtrade 不会允许每次买入交易的投资金额在10.1美元 , 相反,  它会确认是否有放置这个买入交易对应的止损交易的空间(+ an offset, defined by `amount_reserve_percent`, which defaults to 5%).

在保留了5%的前提下, 最小的投资质押金额在12.6美元(`12 * (1 + 0.05)`). If we take into account a stoploss of 10% on top of that - we'd end up with a value of ~14$ (`12.6 / (1 - 0.1)`).

为了在止损值较大的情况下限制此计算，计算出的最低质押限额永远不会超过实际限额的 50%。

> 由于交易所的限额通常是稳定的，并且不经常更新，因此某些货币对可能会显示出相当高的最低限额，这仅仅是因为自交易所上次调整限额以来价格上涨了很多。Freqtrade 将质押金额调整为此值，除非它比计算/期望的质押金额高出 30% > - 在这种情况下，交易将被拒绝。
###### 可交易余额

By default, the bot assumes that the `complete amount - 1%` is at it's disposal, and when using [dynamic stake amount](https://www.freqtrade.io/en/stable/configuration/#dynamic-stake-amount), it will split the complete balance into `max_open_trades` buckets per trade. Freqtrade will reserve 1% for eventual fees when entering a trade and will therefore not touch that by default.

You can configure the "untouched" amount by using the `tradable_balance_ratio` setting.

For example, if you have 10 ETH available in your wallet on the exchange and `tradable_balance_ratio=0.5` (which is 50%), then the bot will use a maximum amount of 5 ETH for trading and considers this as an available balance. The rest of the wallet is untouched by the trades.

> This setting should **not** be used when running multiple bots on the same account. Please look at [Available Capital to the bot](https://www.freqtrade.io/en/stable/configuration/#assign-available-capital) instead.

> The `tradable_balance_ratio` setting applies to the current balance (free balance + tied up in trades). Therefore, assuming the starting balance of 1000, a configuration with `tradable_balance_ratio=0.99` will not guarantee that 10 currency units will always remain available on the exchange. For example, the free amount may reduce to 5 units if the total balance is reduced to 500 (either by a losing streak or by withdrawing balance).
###### 分配可用的启动金

To fully utilize compounding profits when using multiple bots on the same exchange account, you'll want to limit each bot to a certain starting balance. This can be accomplished by setting `available_capital` to the desired starting balance.

Assuming your account has 10000 USDT and you want to run 2 different strategies on this exchange. You'd set `available_capital=5000` - granting each bot an initial capital of 5000 USDT. The bot will then split this starting balance equally into `max_open_trades` buckets. Profitable trades will result in increased stake-sizes for this bot - without affecting the stake-sizes of the other bot.

Adjusting `available_capital` requires reloading the configuration to take effect. Adjusting the `available_capital` adds the difference between the previous `available_capital` and the new `available_capital`. Decreasing the available capital when trades are open doesn't exit the trades. The difference is returned to the wallet when the trades conclude. The outcome of this differs depending on the price movement between the adjustment and exiting the trades.

> Incompatible with `tradable_balance_ratio`
> Setting this option will replace any configuration of `tradable_balance_ratio`.

###### 修正最后一次投资质押金额

Assuming we have the tradable balance of 1000 USDT, `stake_amount=400`, and `max_open_trades=3`. The bot would open 2 trades and will be unable to fill the last trading slot, since the requested 400 USDT are no longer available since 800 USDT are already tied in other trades.

To overcome this, the option `amend_last_stake_amount` can be set to `True`, which will enable the bot to reduce stake_amount to the available balance to fill the last trade slot.

In the example above this would mean:

- Trade1: 400 USDT
- Trade2: 400 USDT
- Trade3: 200 USDT

> This option only applies with [Static stake amount](https://www.freqtrade.io/en/stable/configuration/#static-stake-amount) - since [Dynamic stake amount](https://www.freqtrade.io/en/stable/configuration/#dynamic-stake-amount) divides the balances evenly.

> The minimum last stake amount can be configured using `last_stake_amount_min_ratio` - which defaults to 0.5 (50%). This means that the minimum stake amount that's ever used is `stake_amount * 0.5`. This avoids very low stake amounts, that are close to the minimum tradable amount for the pair and can be refused by the exchange.
###### 静态质押投资金额

The `stake_amount` configuration statically configures the amount of stake-currency your bot will use for each trade.

The minimal configuration value is 0.0001, however, please check your exchange's trading minimums for the stake currency you're using to avoid problems.

This setting works in combination with `max_open_trades`. The maximum capital engaged in trades is `stake_amount * max_open_trades`. For example, the bot will at most use (0.05 BTC x 3) = 0.15 BTC, assuming a configuration of `max_open_trades=3` and `stake_amount=0.05`.

> This setting respects the [available balance configuration](https://www.freqtrade.io/en/stable/configuration/#tradable-balance).
###### 动态质押投资金额

Alternatively, you can use a dynamic stake amount, which will use the available balance on the exchange, and divide that equally by the number of allowed trades (`max_open_trades`).

To configure this, set `stake_amount="unlimited"`. We also recommend to set `tradable_balance_ratio=0.99` (99%) - to keep a minimum balance for eventual fees.

In this case a trade amount is calculated as:

`currency_balance / (max_open_trades - current_open_trades)`

To allow the bot to trade all the available `stake_currency` in your account (minus `tradable_balance_ratio`) set

```json
"stake_amount" : "unlimited",
"tradable_balance_ratio": 0.99,
```

> Compounding profits
> This configuration will allow increasing/decreasing stakes depending on the performance of the bot (lower stake if the bot is losing, higher stakes if the bot has a winning record since higher balances are available), and will result in profit compounding.

> When using Dry-Run Mode
> When using `"stake_amount" : "unlimited",` in combination with Dry-Run, Backtesting or Hyperopt, the balance will be simulated starting with a stake of `dry_run_wallet` which will evolve. It is therefore important to set `dry_run_wallet` to a sensible value (like 0.05 or 0.01 for BTC and 1000 or 100 for USDT, for example), otherwise, it may simulate trades with 100 BTC (or more) or 0.05 USDT (or less) at once - which may not correspond to your real available balance or is less than the exchange minimal limit for the order amount for the stake currency.
######   开启仓位调整下的动态质押投资金额

When you want to use position adjustment with unlimited stakes, you must also implement `custom_stake_amount` to a return a value depending on your strategy. Typical value would be in the range of 25% - 50% of the proposed stakes, but depends highly on your strategy and how much you wish to leave into the wallet as position adjustment buffer.

For example if your position adjustment assumes it can do 2 additional buys with the same stake amounts then your buffer should be 66.6667% of the initially proposed unlimited stake amount.

Or another example if your position adjustment assumes it can do 1 additional buy with 3x the original stake amount then `custom_stake_amount` should return 25% of proposed stake amount and leave 75% for possible later position adjustments.

#### 订单使用的价格

Prices for regular orders can be controlled via the parameter structures `entry_pricing` for trade entries and `exit_pricing` for trade exits. Prices are always retrieved right before an order is placed, either by querying the exchange tickers or by using the orderbook data.

> Orderbook data used by Freqtrade are the data retrieved from exchange by the ccxt's function `fetch_order_book()`, i.e. are usually data from the L2-aggregated orderbook, while the ticker data are the structures returned by the ccxt's `fetch_ticker()`/`fetch_tickers()` functions. Refer to the ccxt library [documentation](https://github.com/ccxt/ccxt/wiki/Manual#market-data) for more details.

> Please read the section [Market order pricing](https://www.freqtrade.io/en/stable/configuration/#market-order-pricing) section when using market orders.

##### 入场价格

###### 入场价格侧

The configuration setting `entry_pricing.price_side` defines the side of the orderbook the bot looks for when buying.

The following displays an orderbook.

```txt
... 
103 
102 
101 # ask 
-------------Current spread 
99 # bid 
98 
97 
...
```

If `entry_pricing.price_side` is set to `"bid"`, then the bot will use 99 as entry price.  
In line with that, if `entry_pricing.price_side` is set to `"ask"`, then the bot will use 101 as entry price.

Depending on the order direction (_long_/_short_), this will lead to different results. Therefore we recommend to use `"same"` or `"other"` for this configuration instead. This would result in the following pricing matrix:

|direction|Order|setting|price|crosses spread|
|---|---|---|---|---|
|long|buy|ask|101|yes|
|long|buy|bid|99|no|
|long|buy|same|99|no|
|long|buy|other|101|yes|
|short|sell|ask|101|no|
|short|sell|bid|99|yes|
|short|sell|same|101|no|
|short|sell|other|99|yes|
Using the other side of the orderbook often guarantees quicker filled orders, but the bot can also end up paying more than what would have been necessary. Taker fees instead of maker fees will most likely apply even when using limit buy orders. Also, prices at the "other" side of the spread are higher than prices at the "bid" side in the orderbook, so the order behaves similar to a market order (however with a maximum price).

###### 订单簿开启的时候的入场价格

When entering a trade with the orderbook enabled (`entry_pricing.use_order_book=True`), Freqtrade fetches the `entry_pricing.order_book_top` entries from the orderbook and uses the entry specified as `entry_pricing.order_book_top` on the configured side (`entry_pricing.price_side`) of the orderbook. 1 specifies the topmost entry in the orderbook, while 2 would use the 2nd entry in the orderbook, and so on.

###### 订单博没开启时候的入场价格

The following section uses `side` as the configured `entry_pricing.price_side` (defaults to `"same"`).

When not using orderbook (`entry_pricing.use_order_book=False`), Freqtrade uses the best `side` price from the ticker if it's below the `last` traded price from the ticker. Otherwise (when the `side` price is above the `last` price), it calculates a rate between `side` and `last` price based on `entry_pricing.price_last_balance`.

The `entry_pricing.price_last_balance` configuration parameter controls this. A value of `0.0` will use `side` price, while `1.0` will use the `last` price and values between those interpolate between ask and last price.

###### 检测市场深度

When check depth of market is enabled (`entry_pricing.check_depth_of_market.enabled=True`), the entry signals are filtered based on the orderbook depth (sum of all amounts) for each orderbook side.

Orderbook `bid` (buy) side depth is then divided by the orderbook `ask` (sell) side depth and the resulting delta is compared to the value of the `entry_pricing.check_depth_of_market.bids_to_ask_delta` parameter. The entry order is only executed if the orderbook delta is greater than or equal to the configured delta value.

> A delta value below 1 means that `ask` (sell) orderbook side depth is greater than the depth of the `bid` (buy) orderbook side, while a value greater than 1 means opposite (depth of the buy side is higher than the depth of the sell side).


##### 出场价格

###### 出场价格侧

The configuration setting `exit_pricing.price_side` defines the side of the spread the bot looks for when exiting a trade.

The following displays an orderbook:

```txt
...
103
102
101  # ask
-------------Current spread
99   # bid
98
97
...
```

If `exit_pricing.price_side` is set to `"ask"`, then the bot will use 101 as exiting price.  
In line with that, if `exit_pricing.price_side` is set to `"bid"`, then the bot will use 99 as exiting price.

Depending on the order direction (_long_/_short_), this will lead to different results. Therefore we recommend to use `"same"` or `"other"` for this configuration instead. This would result in the following pricing matrix:

|Direction|Order|setting|price|crosses spread|
|---|---|---|---|---|
|long|sell|ask|101|no|
|long|sell|bid|99|yes|
|long|sell|same|101|no|
|long|sell|other|99|yes|
|short|buy|ask|101|yes|
|short|buy|bid|99|no|
|short|buy|same|99|no|
|short|buy|other|101|yes|

###### 订单博打开的时候的出场价格

When exiting with the orderbook enabled (`exit_pricing.use_order_book=True`), Freqtrade fetches the `exit_pricing.order_book_top` entries in the orderbook and uses the entry specified as `exit_pricing.order_book_top` from the configured side (`exit_pricing.price_side`) as trade exit price.

1 specifies the topmost entry in the orderbook, while 2 would use the 2nd entry in the orderbook, and so on.

###### 订单博未打开的出场价格

The following section uses `side` as the configured `exit_pricing.price_side` (defaults to `"ask"`).

When not using orderbook (`exit_pricing.use_order_book=False`), Freqtrade uses the best `side` price from the ticker if it's above the `last` traded price from the ticker. Otherwise (when the `side` price is below the `last` price), it calculates a rate between `side` and `last` price based on `exit_pricing.price_last_balance`.

The `exit_pricing.price_last_balance` configuration parameter controls this. A value of `0.0` will use `side` price, while `1.0` will use the last price and values between those interpolate between `side` and last price.


##### 市场订单价格

When using market orders, prices should be configured to use the "correct" side of the orderbook to allow realistic pricing detection. Assuming both entry and exits are using market orders, a configuration similar to the following must be used

```json
  "order_types": {
    "entry": "market",
    "exit": "market"
    // ...
  },
  "entry_pricing": {
    "price_side": "other",
    // ...
  },
  "exit_pricing":{
    "price_side": "other",
    // ...
  },
```

Obviously, if only one side is using limit orders, different pricing combinations can be used.

##### 理解minimal_roi

The `minimal_roi` configuration parameter is a JSON object where the key is a duration in minutes and the value is the minimum ROI as a ratio. See the example below:

```json
"minimal_roi": {
    "40": 0.0,    # Exit after 40 minutes if the profit is not negative
    "30": 0.01,   # Exit after 30 minutes if there is at least 1% profit
    "20": 0.02,   # Exit after 20 minutes if there is at least 2% profit
    "0":  0.04    # Exit immediately if there is at least 4% profit
},
```

Most of the strategy files already include the optimal `minimal_roi` value. This parameter can be set in either Strategy or Configuration file. If you use it in the configuration file, it will override the `minimal_roi` value from the strategy file. If it is not set in either Strategy or Configuration, a default of 1000% `{"0": 10}` is used, and minimal ROI is disabled unless your trade generates 1000% profit.

> A special case presents using `"<N>": -1` as ROI. This forces the bot to exit a trade after N Minutes, no matter if it's positive or negative, so represents a time-limited force-exit.

##### 理解force_entry_enable

The `force_entry_enable` configuration parameter enables the usage of force-enter (`/forcelong`, `/forceshort`) commands via Telegram and REST API. For security reasons, it's disabled by default, and freqtrade will show a warning message on startup if enabled. For example, you can send `/forceenter ETH/BTC` to the bot, which will result in freqtrade buying the pair and holds it until a regular exit-signal (ROI, stoploss, /forceexit) appears.

This can be dangerous with some strategies, so use with care.

See [the telegram documentation](https://www.freqtrade.io/en/stable/telegram-usage/) for details on usage.

##### 忽略过期的蜡烛

When working with larger timeframes (for example 1h or more) and using a low `max_open_trades` value, the last candle can be processed as soon as a trade slot becomes available. When processing the last candle, this can lead to a situation where it may not be desirable to use the buy signal on that candle. For example, when using a condition in your strategy where you use a cross-over, that point may have passed too long ago for you to start a trade on it.

In these situations, you can enable the functionality to ignore candles that are beyond a specified period by setting `ignore_buying_expired_candle_after` to a positive number, indicating the number of seconds after which the buy signal becomes expired.

For example, if your strategy is using a 1h timeframe, and you only want to buy within the first 5 minutes when a new candle comes in, you can add the following configuration to your strategy:

```json
  {
    //...
    "ignore_buying_expired_candle_after": 300,
    // ...
  }
```

> This setting resets with each new candle, so it will not prevent sticking-signals from executing on the 2nd or 3rd candle they're active. Best use a "trigger" selector for buy signals, which are only active for one candle.

##### 理解order_types

The `order_types` configuration parameter maps actions (`entry`, `exit`, `stoploss`, `emergency_exit`, `force_exit`, `force_entry`) to order-types (`market`, `limit`, ...) as well as configures stoploss to be on the exchange and defines stoploss on exchange update interval in seconds.

This allows to enter using limit orders, exit using limit-orders, and create stoplosses using market orders. It also allows to set the stoploss "on exchange" which means stoploss order would be placed immediately once the buy order is fulfilled.

`order_types` set in the configuration file overwrites values set in the strategy as a whole, so you need to configure the whole `order_types` dictionary in one place.

If this is configured, the following 4 values (`entry`, `exit`, `stoploss` and `stoploss_on_exchange`) need to be present, otherwise, the bot will fail to start.

For information on (`emergency_exit`,`force_exit`, `force_entry`, `stoploss_on_exchange`,`stoploss_on_exchange_interval`,`stoploss_on_exchange_limit_ratio`) please see stop loss documentation [stop loss on exchange](https://www.freqtrade.io/en/stable/stoploss/)

策略语法:

```python
order_types = {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": False,
    "stoploss_on_exchange_interval": 60,
    "stoploss_on_exchange_limit_ratio": 0.99,
}
```

配置:

```json
"order_types": {
    "entry": "limit",
    "exit": "limit",
    "emergency_exit": "market",
    "force_entry": "market",
    "force_exit": "market",
    "stoploss": "market",
    "stoploss_on_exchange": false,
    "stoploss_on_exchange_interval": 60
}
```

> Not all exchanges support "market" orders. The following message will be shown if your exchange does not support market orders: `"Exchange <yourexchange> does not support market orders."` and the bot will refuse to start.

> Please carefully read the section [Market order pricing](https://www.freqtrade.io/en/stable/configuration/#market-order-pricing) section when using market orders.

> `order_types.stoploss_on_exchange_interval` is not mandatory. Do not change its value if you are unsure of what you are doing. For more information about how stoploss works please refer to [the stoploss documentation](https://www.freqtrade.io/en/stable/stoploss/).
> 
> If `order_types.stoploss_on_exchange` is enabled and the stoploss is cancelled manually on the exchange, then the bot will create a new stoploss order.

> If stoploss on exchange creation fails for some reason, then an "emergency exit" is initiated. By default, this will exit the trade using a market order. The order-type for the emergency-exit can be changed by setting the `emergency_exit` value in the `order_types` dictionary - however, this is not advised.

##### 理解order_time_in_force

The `order_time_in_force` configuration parameter defines the policy by which the order is executed on the exchange. Three commonly used time in force are:

**GTC (Good Till Canceled):**

This is most of the time the default time in force. It means the order will remain on exchange till it is cancelled by the user. It can be fully or partially fulfilled. If partially fulfilled, the remaining will stay on the exchange till cancelled.

**FOK (Fill Or Kill):**

It means if the order is not executed immediately AND fully then it is cancelled by the exchange.

**IOC (Immediate Or Canceled):**

It is the same as FOK (above) except it can be partially fulfilled. The remaining part is automatically cancelled by the exchange.

**PO (Post only):**

Post only order. The order is either placed as a maker order, or it is canceled. This means the order must be placed on orderbook for at least time in an unfilled state.


###### time_in_force 配置

The `order_time_in_force` parameter contains a dict with entry and exit time in force policy values. This can be set in the configuration file or in the strategy. Values set in the configuration file overwrites values set in the strategy.

The possible values are: `GTC` (default), `FOK` or `IOC`.

```json
"order_time_in_force": {
    "entry": "GTC",
    "exit": "GTC"
},
```

> This is ongoing work. For now, it is supported only for binance, gate and kucoin. Please don't change the default value unless you know what you are doing and have researched the impact of using different values for your particular exchange.

##### 法币转换

Freqtrade uses the Coingecko API to convert the coin value to it's corresponding fiat value for the Telegram reports. The FIAT currency can be set in the configuration file as `fiat_display_currency`.

Removing `fiat_display_currency` completely from the configuration will skip initializing coingecko, and will not show any FIAT currency conversion. This has no importance for the correct functioning of the bot.

######  What values can be used for fiat_display_currency

The `fiat_display_currency` configuration parameter sets the base currency to use for the conversion from coin to fiat in the bot Telegram reports.

The valid values are:

```python
"AUD", "BRL", "CAD", "CHF", "CLP", "CNY", "CZK", "DKK", "EUR", "GBP", "HKD", "HUF", "IDR", "ILS", "INR", "JPY", "KRW", "MXN", "MYR", "NOK", "NZD", "PHP", "PKR", "PLN", "RUB", "SEK", "SGD", "THB", "TRY", "TWD", "ZAR", "USD"
```

In addition to fiat currencies, a range of crypto currencies is supported.

The valid values are:

```python
"BTC", "ETH", "XRP", "LTC", "BCH", "BNB"
```

######  Coingecko Rate limit problems

On some IP ranges, coingecko is heavily rate-limiting. In such cases, you may want to add your coingecko API key to the configuration.

```json
{
    "fiat_display_currency": "USD",
    "coingecko": {
        "api_key": "your-api",
        "is_demo": true
    }
}
```

Freqtrade supports both Demo and Pro coingecko API keys.

The Coingecko API key is NOT required for the bot to function correctly. It is only used for the conversion of coin to fiat in the Telegram reports, which usually also work without API key.

##### 使用Dry-run模式

We recommend starting the bot in the Dry-run mode to see how your bot will behave and what is the performance of your strategy. In the Dry-run mode, the bot does not engage your money. It only runs a live simulation without creating trades on the exchange.

1. Edit your `config.json` configuration file.
2. Switch `dry-run` to `true` and specify `db_url` for a persistence database.

```json
"dry_run": true,
"db_url": "sqlite:///tradesv3.dryrun.sqlite",
```

1. Remove your Exchange API key and secret (change them by empty values or fake credentials):

```json
"exchange": {
    "name": "binance",
    "key": "key",
    "secret": "secret",
    ...
}
```

Once you will be happy with your bot performance running in the Dry-run mode, you can switch it to production mode.

> A simulated wallet is available during dry-run mode and will assume a starting capital of `dry_run_wallet` (defaults to 1000).

###### 使用dry-run的考量

- API-keys may or may not be provided. Only Read-Only operations (i.e. operations that do not alter account state) on the exchange are performed in dry-run mode.
- Wallets (`/balance`) are simulated based on `dry_run_wallet`.
- Orders are simulated, and will not be posted to the exchange.
- Market orders fill based on orderbook volume the moment the order is placed.
- Limit orders fill once the price reaches the defined level - or time out based on `unfilledtimeout` settings.
- Limit orders will be converted to market orders if they cross the price by more than 1%.
- In combination with `stoploss_on_exchange`, the stop_loss price is assumed to be filled.
- Open orders (not trades, which are stored in the database) are kept open after bot restarts, with the assumption that they were not filled while being offline.

##### 切换到实盘模式

In production mode, the bot will engage your money. Be careful, since a wrong strategy can lose all your money. Be aware of what you are doing when you run it in production mode.

When switching to Production mode, please make sure to use a different / fresh database to avoid dry-run trades messing with your exchange money and eventually tainting your statistics.

###### 设置你的交易所账号

You will need to create API Keys (usually you get `key` and `secret`, some exchanges require an additional `password`) from the Exchange website and you'll need to insert this into the appropriate fields in the configuration or when asked by the `freqtrade new-config` command. API Keys are usually only required for live trading (trading for real money, bot running in "production mode", executing real orders on the exchange) and are not required for the bot running in dry-run (trade simulation) mode. When you set up the bot in dry-run mode, you may fill these fields with empty values.

###### 切换你的机器人到实盘模式

**Edit your `config.json` file.**

**Switch dry-run to false and don't forget to adapt your database URL if set:**

```json
"dry_run": false,
```

**Insert your Exchange API key (change them by fake API keys):**

```json
{
    "exchange": {
        "name": "binance",
        "key": "af8ddd35195e9dc500b9a6f799f6f5c93d89193b",
        "secret": "08a9dc6db3d7b53e1acebd9275677f4b0a04f1a5",
        //"password": "", // Optional, not needed by all exchanges)
        // ...
    }
    //...
}
```

You should also make sure to read the [Exchanges](https://www.freqtrade.io/en/stable/exchanges/) section of the documentation to be aware of potential configuration details specific to your exchange.

> To keep your secrets secret, we recommend using a 2nd configuration for your API keys. Simply use the above snippet in a new configuration file (e.g. `config-private.json`) and keep your settings in this file. You can then start the bot with `freqtrade trade --config user_data/config.json --config user_data/config-private.json <...>` to have your keys loaded.
> 
> **NEVER** share your private configuration file or your exchange keys with anyone!

###### 让freqtrade使用代理

To use a proxy with freqtrade, export your proxy settings using the variables `"HTTP_PROXY"` and `"HTTPS_PROXY"` set to the appropriate values. This will have the proxy settings applied to everything (telegram, coingecko, ...) **except** for exchange requests.

```bash
export HTTP_PROXY="http://addr:port"
export HTTPS_PROXY="http://addr:port"
freqtrade
```

###### 代理交易所请求

To use a proxy for exchange connections - you will have to define the proxies as part of the ccxt configuration.

```json
{ 
  "exchange": {
    "ccxt_config": {
      "httpsProxy": "http://addr:port",
    }
  }
}
```

For more information on available proxy types, please consult the [ccxt proxy documentation](https://docs.ccxt.com/#/README?id=proxy).

下一步，阅读[[#启动机器人]]

## 自定义策略

这节主要教你如何定制你的策略，添加新的指标和设置交易规则，在阅读这节，希望你熟悉了freqtrade的 [[#基本原理]]

#### 开发你自己的策略

The bot includes a default strategy file. Also, several other strategies are available in the [strategy repository](https://github.com/freqtrade/freqtrade-strategies).

You will however most likely have your own idea for a strategy. This document intends to help you convert your strategy idea into your own strategy.

To get started, use `freqtrade new-strategy --strategy AwesomeStrategy` (you can obviously use your own naming for your strategy). This will create a new strategy file from a template, which will be located under `user_data/strategies/AwesomeStrategy.py`.

> This is just a template file, which will most likely not be profitable out of the box.

> `freqtrade new-strategy` has an additional parameter, `--template`, which controls the amount of pre-build information you get in the created strategy. Use `--template minimal` to get an empty strategy without any indicator examples, or `--template advanced` to get a template with most callbacks defined.


##### 一个策略的剖析

A strategy file contains all the information needed to build a good strategy:

- Indicators
- Entry strategy rules
- Exit strategy rules
- Minimal ROI recommended
- Stoploss strongly recommended

The bot also include a sample strategy called `SampleStrategy` you can update: `user_data/strategies/sample_strategy.py`. You can test it with the parameter: `--strategy SampleStrategy`

Additionally, there is an attribute called `INTERFACE_VERSION`, which defines the version of the strategy interface the bot should use. The current version is 3 - which is also the default when it's not set explicitly in the strategy.

Future versions will require this to be set.

```bash
freqtrade trade --strategy AwesomeStrategy
```

**For the following section we will use the [user_data/strategies/sample_strategy.py](https://github.com/freqtrade/freqtrade/blob/develop/freqtrade/templates/sample_strategy.py) file as reference.**

> To avoid problems and unexpected differences between Backtesting and dry/live modes, please be aware that during backtesting the full time range is passed to the `populate_*()` methods at once. It is therefore best to use vectorized operations (across the whole dataframe, not loops) and avoid index referencing (`df.iloc[-1]`), but instead use `df.shift()` to get to the previous candle.

> Since backtesting passes the full time range to the `populate_*()` methods, the strategy author needs to take care to avoid having the strategy utilize data from the future. Some common patterns for this are listed in the [Common Mistakes](https://www.freqtrade.io/en/stable/strategy-customization/#common-mistakes-when-developing-strategies) section of this document.

##### 数据帧

Freqtrade uses [pandas](https://pandas.pydata.org/) to store/provide the candlestick (OHLCV) data. Pandas is a great library developed for processing large amounts of data.

Each row in a dataframe corresponds to one candle on a chart, with the latest candle always being the last in the dataframe (sorted by date).

```python
> dataframe.head()
                       date      open      high       low     close     volume
0 2021-11-09 23:25:00+00:00  67279.67  67321.84  67255.01  67300.97   44.62253
1 2021-11-09 23:30:00+00:00  67300.97  67301.34  67183.03  67187.01   61.38076
2 2021-11-09 23:35:00+00:00  67187.02  67187.02  67031.93  67123.81  113.42728
3 2021-11-09 23:40:00+00:00  67123.80  67222.40  67080.33  67160.48   78.96008
4 2021-11-09 23:45:00+00:00  67160.48  67160.48  66901.26  66943.37  111.39292
```

Pandas provides fast ways to calculate metrics. To benefit from this speed, it's advised to not use loops, but use vectorized methods instead.

Vectorized operations perform calculations across the whole range of data and are therefore, compared to looping through each row, a lot faster when calculating indicators.

As a dataframe is a table, simple python comparisons like the following will not work

```python
    if dataframe['rsi'] > 30:
        dataframe['enter_long'] = 1
```

The above section will fail with `The truth value of a Series is ambiguous. [...]`.

This must instead be written in a pandas-compatible way, so the operation is performed across the whole dataframe.

```python
    dataframe.loc[
        (dataframe['rsi'] > 30)
    , 'enter_long'] = 1
```

With this section, you have a new column in your dataframe, which has `1` assigned whenever RSI is above 30.

##### 自定义指标

Buy and sell signals need indicators. You can add more indicators by extending the list contained in the method `populate_indicators()` from your strategy file.

You should only add the indicators used in either `populate_entry_trend()`, `populate_exit_trend()`, or to populate another indicator, otherwise performance may suffer.

It's important to always return the dataframe without removing/modifying the columns `"open", "high", "low", "close", "volume"`, otherwise these fields would contain something unexpected.


```python
def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Adds several different TA indicators to the given DataFrame

    Performance Note: For the best performance be frugal on the number of indicators
    you are using. Let uncomment only the indicator you are using in your strategies
    or your hyperopt configuration, otherwise you will waste your memory and CPU usage.
    :param dataframe: Dataframe with data from the exchange
    :param metadata: Additional information, like the currently traded pair
    :return: a Dataframe with all mandatory indicators for the strategies
    """
    dataframe['sar'] = ta.SAR(dataframe)
    dataframe['adx'] = ta.ADX(dataframe)
    stoch = ta.STOCHF(dataframe)
    dataframe['fastd'] = stoch['fastd']
    dataframe['fastk'] = stoch['fastk']
    dataframe['blower'] = ta.BBANDS(dataframe, nbdevup=2, nbdevdn=2)['lowerband']
    dataframe['sma'] = ta.SMA(dataframe, timeperiod=40)
    dataframe['tema'] = ta.TEMA(dataframe, timeperiod=9)
    dataframe['mfi'] = ta.MFI(dataframe)
    dataframe['rsi'] = ta.RSI(dataframe)
    dataframe['ema5'] = ta.EMA(dataframe, timeperiod=5)
    dataframe['ema10'] = ta.EMA(dataframe, timeperiod=10)
    dataframe['ema50'] = ta.EMA(dataframe, timeperiod=50)
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
    dataframe['ao'] = awesome_oscillator(dataframe)
    macd = ta.MACD(dataframe)
    dataframe['macd'] = macd['macd']
    dataframe['macdsignal'] = macd['macdsignal']
    dataframe['macdhist'] = macd['macdhist']
    hilbert = ta.HT_SINE(dataframe)
    dataframe['htsine'] = hilbert['sine']
    dataframe['htleadsine'] = hilbert['leadsine']
    dataframe['plus_dm'] = ta.PLUS_DM(dataframe)
    dataframe['plus_di'] = ta.PLUS_DI(dataframe)
    dataframe['minus_dm'] = ta.MINUS_DM(dataframe)
    dataframe['minus_di'] = ta.MINUS_DI(dataframe)
    return dataframe
```

###### 指标库

Out of the box, freqtrade installs the following technical libraries:

- [ta-lib](https://ta-lib.github.io/ta-lib-python/)
- [pandas-ta](https://twopirllc.github.io/pandas-ta/)
- [technical](https://github.com/freqtrade/technical/)

Additional technical libraries can be installed as necessary, or custom indicators may be written / invented by the strategy author.

##### 策略启动周期

Most indicators have an instable startup period, in which they are either not available (NaN), or the calculation is incorrect. This can lead to inconsistencies, since Freqtrade does not know how long this instable period should be. To account for this, the strategy can be assigned the `startup_candle_count` attribute. This should be set to the maximum number of candles that the strategy requires to calculate stable indicators. In the case where a user includes higher timeframes with informative pairs, the `startup_candle_count` does not necessarily change. The value is the maximum period (in candles) that any of the informatives timeframes need to compute stable indicators.

Most indicators have an instable startup period, in which they are either not available (NaN), or the calculation is incorrect. This can lead to inconsistencies, since Freqtrade does not know how long this instable period should be. To account for this, the strategy can be assigned the `startup_candle_count` attribute. This should be set to the maximum number of candles that the strategy requires to calculate stable indicators. In the case where a user includes higher timeframes with informative pairs, the `startup_candle_count` does not necessarily change. The value is the maximum period (in candles) that any of the informatives timeframes need to compute stable indicators.

You can use [recursive-analysis](https://www.freqtrade.io/en/stable/recursive-analysis/) to check and find the correct `startup_candle_count` to be used.

In this example strategy, this should be set to 400 (`startup_candle_count = 400`), since the minimum needed history for ema100 calculation to make sure the value is correct is 400 candles.

```python
    dataframe['ema100'] = ta.EMA(dataframe, timeperiod=100)
```

By letting the bot know how much history is needed, backtest trades can start at the specified timerange during backtesting and hyperopt.

> If you receive a warning like `WARNING - Using 3 calls to get OHLCV. This can result in slower operations for the bot. Please check if you really need 1500 candles for your strategy` - you should consider if you really need this much historic data for your signals. Having this will cause Freqtrade to make multiple calls for the same pair, which will obviously be slower than one network request. As a consequence, Freqtrade will take longer to refresh candles - and should therefore be avoided if possible. This is capped to 5 total calls to avoid overloading the exchange, or make freqtrade too slow.

> `startup_candle_count` should be below `ohlcv_candle_limit * 5` (which is 500 * 5 for most exchanges) - since only this amount of candles will be available during Dry-Run/Live Trade operations.

###### 例子

Let's try to backtest 1 month (January 2019) of 5m candles using an example strategy with EMA100, as above.

```bash
freqtrade backtesting --timerange 20190101-20190201 --timeframe 5m
```

Assuming `startup_candle_count` is set to 400, backtesting knows it needs 400 candles to generate valid buy signals. It will load data from `20190101 - (400 * 5m)` - which is ~2018-12-30 11:40:00. If this data is available, indicators will be calculated with this extended timerange. The instable startup period (up to 2019-01-01 00:00:00) will then be removed before starting backtesting.

> If data for the startup period is not available, then the timerange will be adjusted to account for this startup period - so Backtesting would start at 2019-01-02 09:20:00.


##### 入场信号

Edit the method `populate_entry_trend()` in your strategy file to update your entry strategy.

It's important to always return the dataframe without removing/modifying the columns `"open", "high", "low", "close", "volume"`, otherwise these fields would contain something unexpected.

This method will also define a new column, `"enter_long"` (`"enter_short"` for shorts), which needs to contain 1 for entries, and 0 for "no action". `enter_long` is a mandatory column that must be set even if the strategy is shorting only.

Sample from `user_data/strategies/sample_strategy.py`:

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Based on TA indicators, populates the buy signal for the given dataframe
    :param dataframe: DataFrame populated with indicators
    :param metadata: Additional information, like the currently traded pair
    :return: DataFrame with buy column
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

Short-entries can be created by setting `enter_short` (corresponds to `enter_long` for long trades). The `enter_tag` column remains identical. Short-trades need to be supported by your exchange and market configuration! Please make sure to set [`can_short`](https://www.freqtrade.io/en/stable/strategy-customization/) appropriately on your strategy if you intend to short.

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
            (dataframe['tema'] <= dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['enter_long', 'enter_tag']] = (1, 'rsi_cross')

    dataframe.loc[
        (
            (qtpylib.crossed_below(dataframe['rsi'], 70)) &  # Signal: RSI crosses below 70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['enter_short', 'enter_tag']] = (1, 'rsi_cross')

    return dataframe
```

> Buying requires sellers to buy from - therefore volume needs to be > 0 (`dataframe['volume'] > 0`) to make sure that the bot does not buy/sell in no-activity periods.

##### 退出信号

Edit the method `populate_exit_trend()` into your strategy file to update your exit strategy. The exit-signal can be suppressed by setting `use_exit_signal` to false in the configuration or strategy. `use_exit_signal` will not influence [signal collision rules](https://www.freqtrade.io/en/stable/strategy-customization/#colliding-signals) - which will still apply and can prevent entries.

It's important to always return the dataframe without removing/modifying the columns `"open", "high", "low", "close", "volume"`, otherwise these fields would contain something unexpected.

This method will also define a new column, `"exit_long"` (`"exit_short"` for shorts), which needs to contain 1 for exits, and 0 for "no action".

Sample from `user_data/strategies/sample_strategy.py`:

```python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    """
    Based on TA indicators, populates the exit signal for the given dataframe
    :param dataframe: DataFrame populated with indicators
    :param metadata: Additional information, like the currently traded pair
    :return: DataFrame with buy column
    """
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # Signal: RSI crosses above 70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
    return dataframe
```

Short-exits can be created by setting `exit_short` (corresponds to `exit_long`). The `exit_tag` column remains identical. Short-trades need to be supported by your exchange and market configuration!

```python
def populate_exit_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (qtpylib.crossed_above(dataframe['rsi'], 70)) &  # Signal: RSI crosses above 70
            (dataframe['tema'] > dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] < dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['exit_long', 'exit_tag']] = (1, 'rsi_too_high')
    dataframe.loc[
        (
            (qtpylib.crossed_below(dataframe['rsi'], 30)) &  # Signal: RSI crosses below 30
            (dataframe['tema'] < dataframe['bb_middleband']) &  # Guard
            (dataframe['tema'] > dataframe['tema'].shift(1)) &  # Guard
            (dataframe['volume'] > 0)  # Make sure Volume is not 0
        ),
        ['exit_short', 'exit_tag']] = (1, 'rsi_too_low')
    return dataframe
```

##### Minimal ROI

This dict defines the minimal Return On Investment (ROI) a trade should reach before exiting, independent from the exit signal.

It is of the following format, with the dict key (left side of the colon) being the minutes passed since the trade opened, and the value (right side of the colon) being the percentage.

```python
minimal_roi = {
    "40": 0.0,
    "30": 0.01,
    "20": 0.02,
    "0": 0.04
}
```

The above configuration would therefore mean:

- Exit whenever 4% profit was reached
- Exit when 2% profit was reached (in effect after 20 minutes)
- Exit when 1% profit was reached (in effect after 30 minutes)
- Exit when trade is non-loosing (in effect after 40 minutes)

The calculation does include fees.

To disable ROI completely, set it to an empty dictionary:

```python
minimal_roi = {}
```

To use times based on candle duration (timeframe), the following snippet can be handy. This will allow you to change the timeframe for the strategy, and ROI times will still be set as candles (e.g. after 3 candles ...)

```python
from freqtrade.exchange import timeframe_to_minutes

class AwesomeStrategy(IStrategy):

    timeframe = "1d"
    timeframe_mins = timeframe_to_minutes(timeframe)
    minimal_roi = {
        "0": 0.05,                      # 5% for the first 3 candles
        str(timeframe_mins * 3): 0.02,  # 2% after 3 candles
        str(timeframe_mins * 6): 0.01,  # 1% After 6 candles
    }
```

> `minimal_roi` will take the `trade.open_date` as reference, which is the time the trade was initialized / the first order for this trade was placed.  
This will also hold true for limit orders that don't fill immediately (usually in combination with "off-spot" prices through `custom_entry_price()`), as well as for cases where the initial order is replaced through `adjust_entry_price()`. The time used will still be from the initial `trade.open_date` (when the initial order was first placed), not from the newly placed order date.

##### Stoploss

Setting a stoploss is highly recommended to protect your capital from strong moves against you.

Sample of setting a 10% stoploss:

```python
stoploss = -0.10
```

For the full documentation on stoploss features, look at the dedicated [stoploss page](https://www.freqtrade.io/en/stable/stoploss/).

##### Timeframe

This is the set of candles the bot should download and use for the analysis. Common values are `"1m"`, `"5m"`, `"15m"`, `"1h"`, however all values supported by your exchange should work.

Please note that the same entry/exit signals may work well with one timeframe, but not with the others.

This setting is accessible within the strategy methods as the `self.timeframe` attribute.

##### Can short

To use short signals in futures markets, you will have to let us know to do so by setting `can_short=True`. Strategies which enable this will fail to load on spot markets. Disabling of this will have short signals ignored (also in futures markets).

##### Metadata dict

The metadata-dict (available for `populate_entry_trend`, `populate_exit_trend`, `populate_indicators`) contains additional information. Currently this is `pair`, which can be accessed using `metadata['pair']` - and will return a pair in the format `XRP/BTC`.

The Metadata-dict should not be modified and does not persist information across multiple calls. Instead, have a look at the [Storing information](https://www.freqtrade.io/en/stable/strategy-advanced/#storing-information-persistent) section.

#### 策略文件的加载

By default, freqtrade will attempt to load strategies from all `.py` files within `user_data/strategies`.

Assuming your strategy is called `AwesomeStrategy`, stored in the file `user_data/strategies/AwesomeStrategy.py`, then you can start freqtrade with `freqtrade trade --strategy AwesomeStrategy`. Note that we're using the class-name, not the file name.

You can use `freqtrade list-strategies` to see a list of all strategies Freqtrade is able to load (all strategies in the correct folder). It will also include a "status" field, highlighting potential problems.

> You can use a different directory by using `--strategy-path user_data/otherPath`. This parameter is available to all commands that require a strategy.

#### 提供有用信息的交易对

##### 从不可交易对中获取数据

Data for additional, informative pairs (reference pairs) can be beneficial for some strategies. OHLCV data for these pairs will be downloaded as part of the regular whitelist refresh process and is available via `DataProvider` just as other pairs (see below). These parts will **not** be traded unless they are also specified in the pair whitelist, or have been selected by Dynamic Whitelisting.

The pairs need to be specified as tuples in the format `("pair", "timeframe")`, with pair as the first and timeframe as the second argument.

```python
def informative_pairs(self):
    return [("ETH/USDT", "5m"),
            ("BTC/TUSD", "15m"),
            ]
```

> As these pairs will be refreshed as part of the regular whitelist refresh, it's best to keep this list short. All timeframes and all pairs can be specified as long as they are available (and active) on the used exchange. It is however better to use resampling to longer timeframes whenever possible to avoid hammering the exchange with too many requests and risk being blocked.

> Informative_pairs can also provide a 3rd tuple element defining the candle type explicitly. Availability of alternative candle-types will depend on the trading-mode and the exchange. In general, spot pairs cannot be used in futures markets, and futures candles can't be used as informative pairs for spot bots. Details about this may vary, if they do, this can be found in the exchange documentation.

```python
def informative_pairs(self):
    return [
        ("ETH/USDT", "5m", ""),   # Uses default candletype, depends on trading_mode (recommended)
        ("ETH/USDT", "5m", "spot"),   # Forces usage of spot candles (only valid for bots running on spot markets).
        ("BTC/TUSD", "15m", "futures"),  # Uses futures candles (only bots with `trading_mode=futures`)
        ("BTC/TUSD", "15m", "mark"),  # Uses mark candles (only bots with `trading_mode=futures`)
    ]
```

##### 有用信息交易对的装饰器(@informative())

In most common case it is possible to easily define informative pairs by using a decorator. All decorated `populate_indicators_*` methods run in isolation, not having access to data from other informative pairs, in the end all informative dataframes are merged and passed to main `populate_indicators()` method. When hyperopting, use of hyperoptable parameter `.value` attribute is not supported. Please use `.range` attribute. See [optimizing an indicator parameter](https://www.freqtrade.io/en/stable/hyperopt/#optimizing-an-indicator-parameter) for more information.

```python
def informative(timeframe: str, asset: str = '',
                fmt: Optional[Union[str, Callable[[KwArg(str)], str]]] = None,
                *,
                candle_type: Optional[CandleType] = None,
                ffill: bool = True) -> Callable[[PopulateIndicators], PopulateIndicators]:
    """
    A decorator for populate_indicators_Nn(self, dataframe, metadata), allowing these functions to
    define informative indicators.

    Example usage:

        @informative('1h')
        def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
            dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
            return dataframe

    :param timeframe: Informative timeframe. Must always be equal or higher than strategy timeframe.
    :param asset: Informative asset, for example BTC, BTC/USDT, ETH/BTC. Do not specify to use
                current pair. Also supports limited pair format strings (see below)
    :param fmt: Column format (str) or column formatter (callable(name, asset, timeframe)). When not
    specified, defaults to:
    * {base}_{quote}_{column}_{timeframe} if asset is specified.
    * {column}_{timeframe} if asset is not specified.
    Pair format supports these format variables:
    * {base} - base currency in lower case, for example 'eth'.
    * {BASE} - same as {base}, except in upper case.
    * {quote} - quote currency in lower case, for example 'usdt'.
    * {QUOTE} - same as {quote}, except in upper case.
    Format string additionally supports this variables.
    * {asset} - full name of the asset, for example 'BTC/USDT'.
    * {column} - name of dataframe column.
    * {timeframe} - timeframe of informative dataframe.
    :param ffill: ffill dataframe after merging informative pair.
    :param candle_type: '', mark, index, premiumIndex, or funding_rate
    """
```

> Most of the time we do not need power and flexibility offered by `merge_informative_pair()`, therefore we can use a decorator to quickly define informative pairs.


```python
from datetime import datetime
from freqtrade.persistence import Trade
from freqtrade.strategy import IStrategy, informative

class AwesomeStrategy(IStrategy):

    # This method is not required. 
    # def informative_pairs(self): ...

    # Define informative upper timeframe for each pair. Decorators can be stacked on same 
    # method. Available in populate_indicators as 'rsi_30m' and 'rsi_1h'.
    @informative('30m')
    @informative('1h')
    def populate_indicators_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    # Define BTC/STAKE informative pair. Available in populate_indicators and other methods as
    # 'btc_rsi_1h'. Current stake currency should be specified as {stake} format variable 
    # instead of hard-coding actual stake currency. Available in populate_indicators and other 
    # methods as 'btc_usdt_rsi_1h' (when stake currency is USDT).
    @informative('1h', 'BTC/{stake}')
    def populate_indicators_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    # Define BTC/ETH informative pair. You must specify quote currency if it is different from
    # stake currency. Available in populate_indicators and other methods as 'eth_btc_rsi_1h'.
    @informative('1h', 'ETH/BTC')
    def populate_indicators_eth_btc_1h(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    # Define BTC/STAKE informative pair. A custom formatter may be specified for formatting
    # column names. A callable `fmt(**kwargs) -> str` may be specified, to implement custom
    # formatting. Available in populate_indicators and other methods as 'rsi_upper_1h'.
    @informative('1h', 'BTC/{stake}', '{column}_{timeframe}')
    def populate_indicators_btc_1h_2(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        dataframe['rsi_upper'] = ta.RSI(dataframe, timeperiod=14)
        return dataframe

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # Strategy timeframe indicators for current pair.
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)
        # Informative pairs are available in this method.
        dataframe['rsi_less'] = dataframe['rsi'] < dataframe['rsi_1h']
        return dataframe
```

> Do not use `@informative` decorator if you need to use data of one informative pair when generating another informative pair. Instead, define informative pairs manually as described [in the DataProvider section](https://www.freqtrade.io/en/stable/strategy-customization/#complete-data-provider-sample).

> Use string formatting when accessing informative dataframes of other pairs. This will allow easily changing stake currency in config without having to adjust strategy code.

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    stake = self.config['stake_currency']
    dataframe.loc[
        (
            (dataframe[f'btc_{stake}_rsi_1h'] < 35)
            &
            (dataframe['volume'] > 0)
        ),
        ['enter_long', 'enter_tag']] = (1, 'buy_signal_rsi')

    return dataframe
```

> Alternatively column renaming may be used to remove stake currency from column names: `@informative('1h', 'BTC/{stake}', fmt='{base}_{column}_{timeframe}')`.

> Methods tagged with `@informative()` decorator must always have unique names! Re-using same name (for example when copy-pasting already defined informative method) will overwrite previously defined method and not produce any errors due to limitations of Python programming language. In such cases you will find that indicators created in earlier-defined methods are not available in the dataframe. Carefully review method names and make sure they are unique!

#### 额外的数据(DataProvider)

The strategy provides access to the `DataProvider`. This allows you to get additional data to use in your strategy.

All methods return `None` in case of failure (do not raise an exception).

Please always check the mode of operation to select the correct method to get data (samples see below).

> Dataprovider is available during hyperopt, however it can only be used in `populate_indicators()` within a strategy. It is not available in `populate_buy()` and `populate_sell()` methods, nor in `populate_indicators()`, if this method located in the hyperopt file.

##### DataProvider可能的选项

- [`available_pairs`](https://www.freqtrade.io/en/stable/strategy-customization/#available_pairs) - Property with tuples listing cached pairs with their timeframe (pair, timeframe).
- [`current_whitelist()`](https://www.freqtrade.io/en/stable/strategy-customization/#current_whitelist) - Returns a current list of whitelisted pairs. Useful for accessing dynamic whitelists (i.e. VolumePairlist)
- [`get_pair_dataframe(pair, timeframe)`](https://www.freqtrade.io/en/stable/strategy-customization/#get_pair_dataframepair-timeframe) - This is a universal method, which returns either historical data (for backtesting) or cached live data (for the Dry-Run and Live-Run modes).
- [`get_analyzed_dataframe(pair, timeframe)`](https://www.freqtrade.io/en/stable/strategy-customization/#get_analyzed_dataframepair-timeframe) - Returns the analyzed dataframe (after calling `populate_indicators()`, `populate_buy()`, `populate_sell()`) and the time of the latest analysis.
- `historic_ohlcv(pair, timeframe)` - Returns historical data stored on disk.
- `market(pair)` - Returns market data for the pair: fees, limits, precisions, activity flag, etc. See [ccxt documentation](https://github.com/ccxt/ccxt/wiki/Manual#markets) for more details on the Market data structure.
- `ohlcv(pair, timeframe)` - Currently cached candle (OHLCV) data for the pair, returns DataFrame or empty DataFrame.
- [`orderbook(pair, maximum)`](https://www.freqtrade.io/en/stable/strategy-customization/#orderbookpair-maximum) - Returns latest orderbook data for the pair, a dict with bids/asks with a total of `maximum` entries.
- [`ticker(pair)`](https://www.freqtrade.io/en/stable/strategy-customization/#tickerpair) - Returns current ticker data for the pair. See [ccxt documentation](https://github.com/ccxt/ccxt/wiki/Manual#price-tickers) for more details on the Ticker data structure.
- `runmode` - Property containing the current runmode.

##### 使用样例

###### available_pairs

```python
for pair, timeframe in self.dp.available_pairs:
    print(f"available {pair}, {timeframe}")
```

###### current_whitelist()

Imagine you've developed a strategy that trades the `5m` timeframe using signals generated from a `1d` timeframe on the top 10 volume pairs by volume.

The strategy might look something like this:

_Scan through the top 10 pairs by volume using the `VolumePairList` every 5 minutes and use a 14 day RSI to buy and sell._

Due to the limited available data, it's very difficult to resample `5m` candles into daily candles for use in a 14 day RSI. Most exchanges limit us to just 500-1000 candles which effectively gives us around 1.74 daily candles. We need 14 days at least!

Since we can't resample the data we will have to use an informative pair; and since the whitelist will be dynamic we don't know which pair(s) to use.

This is where calling `self.dp.current_whitelist()` comes in handy.

```python
    def informative_pairs(self):

        # get access to all pairs available in whitelist.
        pairs = self.dp.current_whitelist()
        # Assign tf to each pair so they can be downloaded and cached for strategy.
        informative_pairs = [(pair, '1d') for pair in pairs]
        return informative_pairs
```

> Current whitelist is not supported for `plot-dataframe`, as this command is usually used by providing an explicit pairlist - and would therefore make the return values of this method misleading.

###### get_pair_dataframe(pair, timeframe)

```python
# fetch live / historical candle (OHLCV) data for the first informative pair
inf_pair, inf_timeframe = self.informative_pairs()[0]
informative = self.dp.get_pair_dataframe(pair=inf_pair,
                                         timeframe=inf_timeframe)
```

> In backtesting, `dp.get_pair_dataframe()` behavior differs depending on where it's called. Within `populate_*()` methods, `dp.get_pair_dataframe()` returns the full timerange. Please make sure to not "look into the future" to avoid surprises when running in dry/live mode. Within [callbacks](https://www.freqtrade.io/en/stable/strategy-callbacks/), you'll get the full timerange up to the current (simulated) candle.

###### get_analyzed_dataframe(pair, timeframe)

This method is used by freqtrade internally to determine the last signal. It can also be used in specific callbacks to get the signal that caused the action (see [Advanced Strategy Documentation](https://www.freqtrade.io/en/stable/strategy-advanced/) for more details on available callbacks).

```python
# fetch current dataframe
dataframe, last_updated = self.dp.get_analyzed_dataframe(pair=metadata['pair'],
                                                         timeframe=self.timeframe)
```

> Returns an empty dataframe if the requested pair was not cached. You can check for this with `if dataframe.empty:` and handle this case accordingly. This should not happen when using whitelisted pairs.

###### orderbook(pair, maximum)

```python
if self.dp.runmode.value in ('live', 'dry_run'):
    ob = self.dp.orderbook(metadata['pair'], 1)
    dataframe['best_bid'] = ob['bids'][0][0]
    dataframe['best_ask'] = ob['asks'][0][0]
```

The orderbook structure is aligned with the order structure from [ccxt](https://github.com/ccxt/ccxt/wiki/Manual#order-book-structure), so the result will look as follows:

```json
{
    'bids': [
        [ price, amount ], // [ float, float ]
        [ price, amount ],
        ...
    ],
    'asks': [
        [ price, amount ],
        [ price, amount ],
        //...
    ],
    //...
}
```

Therefore, using `ob['bids'][0][0]` as demonstrated above will result in using the best bid price. `ob['bids'][0][1]` would look at the amount at this orderbook position.

> The order book is not part of the historic data which means backtesting and hyperopt will not work correctly if this method is used, as the method will return up-to-date values.

###### ticker(pair)

```python
if self.dp.runmode.value in ('live', 'dry_run'):
    ticker = self.dp.ticker(metadata['pair'])
    dataframe['last_price'] = ticker['last']
    dataframe['volume24h'] = ticker['quoteVolume']
    dataframe['vwap'] = ticker['vwap']
```

> Although the ticker data structure is a part of the ccxt Unified Interface, the values returned by this method can vary for different exchanges. For instance, many exchanges do not return `vwap` values, some exchanges does not always fills in the `last` field (so it can be None), etc. So you need to carefully verify the ticker data returned from the exchange and add appropriate error handling / defaults.

> This method will always return up-to-date values - so usage during backtesting / hyperopt without runmode checks will lead to wrong results.


###### 发送通知

The dataprovider `.send_msg()` function allows you to send custom notifications from your strategy. Identical notifications will only be sent once per candle, unless the 2nd argument (`always_send`) is set to True.

```python
    self.dp.send_msg(f"{metadata['pair']} just got hot!")

    # Force send this notification, avoid caching (Please read warning below!)
    self.dp.send_msg(f"{metadata['pair']} just got hot!", always_send=True)
```

Notifications will only be sent in trading modes (Live/Dry-run) - so this method can be called without conditions for backtesting.

> You can spam yourself pretty good by setting `always_send=True` in this method. Use this with great care and only in conditions you know will not happen throughout a candle to avoid a message every 5 seconds.

###### 完整的DataProvider样例

```python
from freqtrade.strategy import IStrategy, merge_informative_pair
from pandas import DataFrame

class SampleStrategy(IStrategy):
    # strategy init stuff...

    timeframe = '5m'

    # more strategy init stuff..

    def informative_pairs(self):

        # get access to all pairs available in whitelist.
        pairs = self.dp.current_whitelist()
        # Assign tf to each pair so they can be downloaded and cached for strategy.
        informative_pairs = [(pair, '1d') for pair in pairs]
        # Optionally Add additional "static" pairs
        informative_pairs += [("ETH/USDT", "5m"),
                              ("BTC/TUSD", "15m"),
                            ]
        return informative_pairs

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        if not self.dp:
            # Don't do anything if DataProvider is not available.
            return dataframe

        inf_tf = '1d'
        # Get the informative pair
        informative = self.dp.get_pair_dataframe(pair=metadata['pair'], timeframe=inf_tf)
        # Get the 14 day rsi
        informative['rsi'] = ta.RSI(informative, timeperiod=14)

        # Use the helper function merge_informative_pair to safely merge the pair
        # Automatically renames the columns and merges a shorter timeframe dataframe and a longer timeframe informative pair
        # use ffill to have the 1d value available in every row throughout the day.
        # Without this, comparisons between columns of the original and the informative pair would only work once per day.
        # Full documentation of this method, see below
        dataframe = merge_informative_pair(dataframe, informative, self.timeframe, inf_tf, ffill=True)

        # Calculate rsi of the original dataframe (5m timeframe)
        dataframe['rsi'] = ta.RSI(dataframe, timeperiod=14)

        # Do other stuff
        # ...

        return dataframe

    def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:

        dataframe.loc[
            (
                (qtpylib.crossed_above(dataframe['rsi'], 30)) &  # Signal: RSI crosses above 30
                (dataframe['rsi_1d'] < 30) &                     # Ensure daily RSI is < 30
                (dataframe['volume'] > 0)                        # Ensure this candle had volume (important for backtesting)
            ),
            ['enter_long', 'enter_tag']] = (1, 'rsi_cross')
```

#### 额外的数据(Wallets)

The strategy provides access to the `wallets` object. This contains the current balances on the exchange.

> Wallets behaves differently depending on the function it's called. Within `populate_*()` methods, it'll return the full wallet as configured. Within [callbacks](https://www.freqtrade.io/en/stable/strategy-callbacks/), you'll get the wallet state corresponding to the actual simulated wallet at that point in the simulation process.

Please always check if `wallets` is available to avoid failures during backtesting.

```python
if self.wallets:
    free_eth = self.wallets.get_free('ETH')
    used_eth = self.wallets.get_used('ETH')
    total_eth = self.wallets.get_total('ETH')
```

##### 对于wallets的可能选项

- `get_free(asset)` - currently available balance to trade
- `get_used(asset)` - currently tied up balance (open orders)
- `get_total(asset)` - total available balance - sum of the 2 above

#### 额外的数据(Trades)

A history of Trades can be retrieved in the strategy by querying the database.

At the top of the file, import Trade.

```python
from freqtrade.persistence import Trade
```

The following example queries for the current pair and trades from today, however other filters can easily be added.

```python
trades = Trade.get_trades_proxy(pair=metadata['pair'],
                                open_date=datetime.now(timezone.utc) - timedelta(days=1),
                                is_open=False,
            ]).order_by(Trade.close_date).all()
# Summarize profit for this pair.
curdayprofit = sum(trade.close_profit for trade in trades)
```

For a full list of available methods, please consult the [Trade object](https://www.freqtrade.io/en/stable/trade-object/) documentation.

> Trade history is not available in `populate_*` methods during backtesting or hyperopt, and will result in empty results.

#### 阻止一个特定交易对正在发生的交易

Freqtrade locks pairs automatically for the current candle (until that candle is over) when a pair is sold, preventing an immediate re-buy of that pair.

Locked pairs will show the message `Pair <pair> is currently locked.`.

##### Locking pairs from within the strategy

Sometimes it may be desired to lock a pair after certain events happen (e.g. multiple losing trades in a row).

Freqtrade has an easy method to do this from within the strategy, by calling `self.lock_pair(pair, until, [reason])`. `until` must be a datetime object in the future, after which trading will be re-enabled for that pair, while `reason` is an optional string detailing why the pair was locked.

Locks can also be lifted manually, by calling `self.unlock_pair(pair)` or `self.unlock_reason(<reason>)` - providing reason the pair was locked with. `self.unlock_reason(<reason>)` will unlock all pairs currently locked with the provided reason.

To verify if a pair is currently locked, use `self.is_pair_locked(pair)`.

> Locked pairs will always be rounded up to the next candle. So assuming a `5m` timeframe, a lock with `until` set to 10:18 will lock the pair until the candle from 10:15-10:20 will be finished.

> Manually locking pairs is not available during backtesting, only locks via Protections are allowed.

##### Pair locking example

```python
from freqtrade.persistence import Trade
from datetime import timedelta, datetime, timezone
# Put the above lines a the top of the strategy file, next to all the other imports
# --------

# Within populate indicators (or populate_buy):
if self.config['runmode'].value in ('live', 'dry_run'):
    # fetch closed trades for the last 2 days
    trades = Trade.get_trades_proxy(
        pair=metadata['pair'], is_open=False, 
        open_date=datetime.now(timezone.utc) - timedelta(days=2))
    # Analyze the conditions you'd like to lock the pair .... will probably be different for every strategy
    sumprofit = sum(trade.close_profit for trade in trades)
    if sumprofit < 0:
        # Lock pair for 12 hours
        self.lock_pair(metadata['pair'], until=datetime.now(timezone.utc) + timedelta(hours=12))
```

#### 打印已建立的数据帧

To inspect the created dataframe, you can issue a print-statement in either `populate_entry_trend()` or `populate_exit_trend()`. You may also want to print the pair so it's clear what data is currently shown.

```python
def populate_entry_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            #>> whatever condition<<<
        ),
        ['enter_long', 'enter_tag']] = (1, 'somestring')

    # Print the Analyzed pair
    print(f"result for {metadata['pair']}")

    # Inspect the last 5 rows
    print(dataframe.tail())

    return dataframe
```

Printing more than a few rows is also possible (simply use `print(dataframe)` instead of `print(dataframe.tail())`), however not recommended, as that will be very verbose (~500 lines per pair every 5 seconds).

#### 当开发策略时可能会犯的共同错误

##### 在回测中使用未来函数

Backtesting analyzes the whole time-range at once for performance reasons. Because of this, strategy authors need to make sure that strategies do not look-ahead into the future. This is a common pain-point, which can cause huge differences between backtesting and dry/live run methods, since they all use data which is not available during dry/live runs, so these strategies will perform well during backtesting, but will fail / perform badly in real conditions.

The following lists some common patterns which should be avoided to prevent frustration:

- don't use `shift(-1)` or other negative values. This uses data from the future in backtesting, which is not available in dry or live modes.
- don't use `.iloc[-1]` or any other absolute position in the dataframe within `populate_` functions, as this will be different between dry-run and backtesting. Absolute `iloc` indexing is safe to use in callbacks however - see [Strategy Callbacks](https://www.freqtrade.io/en/stable/strategy-callbacks/).
- don't use `dataframe['volume'].mean()`. This uses the full DataFrame for backtesting, including data from the future. Use `dataframe['volume'].rolling(<window>).mean()` instead
- don't use `.resample('1h')`. This uses the left border of the interval, so moves data from an hour to the start of the hour. Use `.resample('1h', label='right')` instead.

> You may also want to check the 2 helper commands [lookahead-analysis](https://www.freqtrade.io/en/stable/lookahead-analysis/) and [recursive-analysis](https://www.freqtrade.io/en/stable/recursive-analysis/), which can each help you figure out problems with your strategy in different ways. Please treat them as what they are - helpers to identify most common problems. A negative result of each does not guarantee that there's none of the above errors included.

##### 冲突的信号

When conflicting signals collide (e.g. both `'enter_long'` and `'exit_long'` are 1), freqtrade will do nothing and ignore the entry signal. This will avoid trades that enter, and exit immediately. Obviously, this can potentially lead to missed entries.

The following rules apply, and entry signals will be ignored if more than one of the 3 signals is set:

- `enter_long` -> `exit_long`, `enter_short`
- `enter_short` -> `exit_short`, `enter_long`

#### 有关于更远点的策略思考

To get additional Ideas for strategies, head over to the [strategy repository](https://github.com/freqtrade/freqtrade-strategies). Feel free to use them as they are - but results will depend on the current market situation, pairs used etc. - therefore please backtest the strategy for your exchange/desired pairs first, evaluate carefully, use at your own risk. Feel free to use any of them as inspiration for your own strategies. We're happy to accept Pull Requests containing new Strategies to that repo.

下一小节，看[[#回测]]
## 策略回调

## 止损

## 插件

## 启动机器人

本页介绍机器人的不同参数以及如何运行它。

> If you've used `setup.sh`, don't forget to activate your virtual environment (`source .venv/bin/activate`) before running freqtrade commands.

> The clock on the system running the bot must be accurate, synchronized to a NTP server frequently enough to avoid problems with communication to the exchanges.

#### 机器人的命令

```bash
usage: freqtrade [-h] [-V]
                 {trade,create-userdir,new-config,new-strategy,download-data,convert-data,convert-trade-data,list-data,backtesting,edge,hyperopt,hyperopt-list,hyperopt-show,list-exchanges,list-hyperopts,list-markets,list-pairs,list-strategies,list-timeframes,show-trades,test-pairlist,install-ui,plot-dataframe,plot-profit,webserver}
                 ...

Free, open source crypto trading bot

positional arguments:
  {trade,create-userdir,new-config,new-strategy,download-data,convert-data,convert-trade-data,list-data,backtesting,edge,hyperopt,hyperopt-list,hyperopt-show,list-exchanges,list-hyperopts,list-markets,list-pairs,list-strategies,list-timeframes,show-trades,test-pairlist,install-ui,plot-dataframe,plot-profit,webserver}
    trade               Trade module.
    create-userdir      Create user-data directory.
    new-config          Create new config
    new-strategy        Create new strategy
    download-data       Download backtesting data.
    convert-data        Convert candle (OHLCV) data from one format to
                        another.
    convert-trade-data  Convert trade data from one format to another.
    list-data           List downloaded data.
    backtesting         Backtesting module.
    edge                Edge module.
    hyperopt            Hyperopt module.
    hyperopt-list       List Hyperopt results
    hyperopt-show       Show details of Hyperopt results
    list-exchanges      Print available exchanges.
    list-hyperopts      Print available hyperopt classes.
    list-markets        Print markets on exchange.
    list-pairs          Print pairs on exchange.
    list-strategies     Print available strategies.
    list-timeframes     Print available timeframes for the exchange.
    show-trades         Show trades.
    test-pairlist       Test your pairlist configuration.
    install-ui          Install FreqUI
    plot-dataframe      Plot candles with indicators.
    plot-profit         Generate plot showing profits.
    webserver           Webserver module.

optional arguments:
  -h, --help            show this help message and exit
  -V, --version         show program's version number and exit
```

##### 机器人的交易命令

```bash
usage: freqtrade trade [-h] [-v] [--logfile FILE] [-V] [-c PATH] [-d PATH]
                       [--userdir PATH] [-s NAME] [--strategy-path PATH]
                       [--db-url PATH] [--sd-notify] [--dry-run]
                       [--dry-run-wallet DRY_RUN_WALLET]

optional arguments:
  -h, --help            show this help message and exit
  --db-url PATH         Override trades database URL, this is useful in custom
                        deployments (default: `sqlite:///tradesv3.sqlite` for
                        Live Run mode, `sqlite:///tradesv3.dryrun.sqlite` for
                        Dry Run).
  --sd-notify           Notify systemd service manager.
  --dry-run             Enforce dry-run for trading (removes Exchange secrets
                        and simulates trades).
  --dry-run-wallet DRY_RUN_WALLET, --starting-balance DRY_RUN_WALLET
                        Starting balance, used for backtesting / hyperopt and
                        dry-runs.

Common arguments:
  -v, --verbose         Verbose mode (-vv for more, -vvv to get all messages).
  --logfile FILE        Log to the file specified. Special values are:
                        'syslog', 'journald'. See the documentation for more
                        details.
  -V, --version         show program's version number and exit
  -c PATH, --config PATH
                        Specify configuration file (default:
                        `userdir/config.json` or `config.json` whichever
                        exists). Multiple --config options may be used. Can be
                        set to `-` to read config from stdin.
  -d PATH, --datadir PATH
                        Path to directory with historical backtesting data.
  --userdir PATH, --user-data-dir PATH
                        Path to userdata directory.

Strategy arguments:
  -s NAME, --strategy NAME
                        Specify strategy class name which will be used by the
                        bot.
  --strategy-path PATH  Specify additional strategy lookup path.
```

##### 如何指定使用哪个配置文件

```bash
freqtrade trade -c path/far/far/away/config.json
```

默认情况下，机器人从当前工作目录加载配置文件 `config.json`。

##### 如何指定使用多个配置文件

```bash
freqtrade trade -c ./config.json -c path/to/secrets/keys.config.json
```

##### 自定义数据存储在哪

freqtrade允许创建用户数据目录，`freqtrade create-userdir --userdir <dir>`

```txt
user_data/
├── backtest_results
├── data
├── hyperopts
├── hyperopt_results
├── plot
└── strategies
```

 You can add the entry "user_data_dir" setting to your configuration, to always point your bot to this directory. Alternatively, pass in `--userdir` to every command. The bot will fail to start if the directory does not exist, but will create necessary subdirectories.

This directory should contain your custom strategies, custom hyperopts and hyperopt loss functions, backtesting historical data (downloaded using either backtesting command or the download script) and plot outputs.

It is recommended to use version control to keep track of changes to your strategies.

##### 如何使用--strategy

This parameter will allow you to load your custom strategy class. To test the bot installation, you can use the `SampleStrategy` installed by the `create-userdir` subcommand (usually `user_data/strategy/sample_strategy.py`).

The bot will search your strategy file within `user_data/strategies`. To use other directories, please read the next section about `--strategy-path`.

To load a strategy, simply pass the class name (e.g.: `CustomStrategy`) in this parameter.

**Example:** In `user_data/strategies` you have a file `my_awesome_strategy.py` which has a strategy class called `AwesomeStrategy` to load it:

```bash
freqtrade trade --strategy AwesomeStrategy
```

If the bot does not find your strategy file, it will display in an error message the reason (File not found, or errors in your code).

更多的关于策略的请看 [[#自定义策略]]

##### 如何使用--strategy-path

This parameter allows you to add an additional strategy lookup path, which gets checked before the default locations (The passed path must be a directory!):

```bash
freqtrade trade --strategy AwesomeStrategy --strategy-path /some/directory
```

###### 如何安装一个策略

非常简单，就是复制粘贴策略文件到`user_data/strategies`目录。或者使用--strategy-path来指定额外的策略目录，然后就可以通过--strategy使用了

##### 如何使用--db-url

When you run the bot in Dry-run mode, per default no transactions are stored in a database. If you want to store your bot actions in a DB using `--db-url`. This can also be used to specify a custom database in production mode. Example command:

```bash
freqtrade trade -c config.json --db-url sqlite:///tradesv3.dry_run.sqlite
```

下一步请看 [[#自定义策略]]
## 控制机器人

## 数据下载

[Data Downloading - Freqtrade](https://www.freqtrade.io/en/stable/data-download/)

## 回测

Backtesting requires historic data to be available. To learn how to get data for the pairs and exchange you're interested in, head over to the [[#数据下载]] section of the documentation.

#### 回测命令参考

```bash
usage: freqtrade backtesting [-h] [-v] [--logfile FILE] [-V] [-c PATH]
                             [-d PATH] [--userdir PATH] [-s NAME]
                             [--strategy-path PATH] [-i TIMEFRAME]
                             [--timerange TIMERANGE]
                             [--data-format-ohlcv {json,jsongz,hdf5}]
                             [--max-open-trades INT]
                             [--stake-amount STAKE_AMOUNT] [--fee FLOAT]
                             [-p PAIRS [PAIRS ...]] [--eps] [--dmmp]
                             [--enable-protections]
                             [--dry-run-wallet DRY_RUN_WALLET]
                             [--timeframe-detail TIMEFRAME_DETAIL]
                             [--strategy-list STRATEGY_LIST [STRATEGY_LIST ...]]
                             [--export {none,trades,signals}]
                             [--export-filename PATH]
                             [--breakdown {day,week,month} [{day,week,month} ...]]
                             [--cache {none,day,week,month}]

optional arguments:
  -h, --help            show this help message and exit
  -i TIMEFRAME, --timeframe TIMEFRAME
                        Specify timeframe (`1m`, `5m`, `30m`, `1h`, `1d`).
  --timerange TIMERANGE
                        Specify what timerange of data to use.
  --data-format-ohlcv {json,jsongz,hdf5,feather,parquet}
                        Storage format for downloaded candle (OHLCV) data.
                        (default: `feather`).
  --max-open-trades INT
                        Override the value of the `max_open_trades`
                        configuration setting.
  --stake-amount STAKE_AMOUNT
                        Override the value of the `stake_amount` configuration
                        setting.
  --fee FLOAT           Specify fee ratio. Will be applied twice (on trade
                        entry and exit).
  -p PAIRS [PAIRS ...], --pairs PAIRS [PAIRS ...]
                        Limit command to these pairs. Pairs are space-
                        separated.
  --eps, --enable-position-stacking
                        Allow buying the same pair multiple times (position
                        stacking).
  --dmmp, --disable-max-market-positions
                        Disable applying `max_open_trades` during backtest
                        (same as setting `max_open_trades` to a very high
                        number).
  --enable-protections, --enableprotections
                        Enable protections for backtesting.Will slow
                        backtesting down by a considerable amount, but will
                        include configured protections
  --dry-run-wallet DRY_RUN_WALLET, --starting-balance DRY_RUN_WALLET
                        Starting balance, used for backtesting / hyperopt and
                        dry-runs.
  --timeframe-detail TIMEFRAME_DETAIL
                        Specify detail timeframe for backtesting (`1m`, `5m`,
                        `30m`, `1h`, `1d`).
  --strategy-list STRATEGY_LIST [STRATEGY_LIST ...]
                        Provide a space-separated list of strategies to
                        backtest. Please note that timeframe needs to be set
                        either in config or via command line. When using this
                        together with `--export trades`, the strategy-name is
                        injected into the filename (so `backtest-data.json`
                        becomes `backtest-data-SampleStrategy.json`
  --export {none,trades,signals}
                        Export backtest results (default: trades).
  --export-filename PATH, --backtest-filename PATH
                        Use this filename for backtest results.Requires
                        `--export` to be set as well. Example: `--export-filen
                        ame=user_data/backtest_results/backtest_today.json`
  --breakdown {day,week,month} [{day,week,month} ...]
                        Show backtesting breakdown per [day, week, month].
  --cache {none,day,week,month}
                        Load a cached backtest result no older than specified
                        age (default: day).

Common arguments:
  -v, --verbose         Verbose mode (-vv for more, -vvv to get all messages).
  --logfile FILE        Log to the file specified. Special values are:
                        'syslog', 'journald'. See the documentation for more
                        details.
  -V, --version         show program's version number and exit
  -c PATH, --config PATH
                        Specify configuration file (default:
                        `userdir/config.json` or `config.json` whichever
                        exists). Multiple --config options may be used. Can be
                        set to `-` to read config from stdin.
  -d PATH, --datadir PATH
                        Path to directory with historical backtesting data.
  --userdir PATH, --user-data-dir PATH
                        Path to userdata directory.

Strategy arguments:
  -s NAME, --strategy NAME
                        Specify strategy class name which will be used by the
                        bot.
  --strategy-path PATH  Specify additional strategy lookup path.
```

#### 回测你的策略

Now you have good Entry and exit strategies and some historic data, you want to test it against real data. This is what we call [backtesting](https://en.wikipedia.org/wiki/Backtesting).

Backtesting will use the crypto-currencies (pairs) from your config file and load historical candle (OHLCV) data from `user_data/data/<exchange>` by default. If no data is available for the exchange / pair / timeframe combination, backtesting will ask you to download them first using `freqtrade download-data`. For details on downloading, please refer to the [Data Downloading](https://www.freqtrade.io/en/stable/data-download/) section in the documentation.

The result of backtesting will confirm if your bot has better odds of making a profit than a loss.

All profit calculations include fees, and freqtrade will use the exchange's default fees for the calculation.

> Using dynamic pairlists is possible (not all of the handlers are allowed to be used in backtest mode), however it relies on the current market conditions - which will not reflect the historic status of the pairlist. Also, when using pairlists other than StaticPairlist, reproducibility of backtesting-results cannot be guaranteed. Please read the [pairlists documentation](https://www.freqtrade.io/en/stable/plugins/#pairlists) for more information.

> To achieve reproducible results, best generate a pairlist via the [`test-pairlist`](https://www.freqtrade.io/en/stable/utils/#test-pairlist) command and use that as static pairlist.

> By default, Freqtrade will export backtesting results to `user_data/backtest_results`. The exported trades can be used for [further analysis](https://www.freqtrade.io/en/stable/backtesting/#further-backtest-result-analysis) or can be used by the [plotting sub-command](https://www.freqtrade.io/en/stable/plotting/#plot-price-and-indicators) (`freqtrade plot-dataframe`) in the scripts directory.

##### 开始回测的余额

Backtesting will require a starting balance, which can be provided as `--dry-run-wallet <balance>` or `--starting-balance <balance>` command line argument, or via `dry_run_wallet` configuration setting. This amount must be higher than `stake_amount`, otherwise the bot will not be able to simulate any trade.

##### 动态质押投资金额

Backtesting supports [dynamic stake amount](https://www.freqtrade.io/en/stable/configuration/#dynamic-stake-amount) by configuring `stake_amount` as `"unlimited"`, which will split the starting balance into `max_open_trades` pieces. Profits from early trades will result in subsequent higher stake amounts, resulting in compounding of profits over the backtesting period.

##### 回测的一些命令例子

With 5 min candle (OHLCV) data (per default)

```bash
freqtrade backtesting --strategy AwesomeStrategy
```

Where `--strategy AwesomeStrategy` / `-s AwesomeStrategy` refers to the class name of the strategy, which is within a python file in the `user_data/strategies` directory.

With 1 min candle (OHLCV) data

```bash
freqtrade backtesting --strategy AwesomeStrategy --timeframe 1m
```

Providing a custom starting balance of 1000 (in stake currency)

```bash
freqtrade backtesting --strategy AwesomeStrategy --dry-run-wallet 1000
```

Using a different on-disk historical candle (OHLCV) data source

Assume you downloaded the history data from the Binance exchange and kept it in the `user_data/data/binance-20180101` directory. You can then use this data for backtesting as follows:

```bash
freqtrade backtesting --strategy AwesomeStrategy --datadir user_data/data/binance-20180101
```

Comparing multiple Strategies

```bash
freqtrade backtesting --strategy-list SampleStrategy1 AwesomeStrategy --timeframe 5m
```

Where `SampleStrategy1` and `AwesomeStrategy` refer to class names of strategies.

Prevent exporting trades to file

```bash
freqtrade backtesting --strategy backtesting --export none --config config.json
```

Only use this if you're sure you'll not want to plot or analyze your results further.

Exporting trades to file specifying a custom filename

```bash
freqtrade backtesting --strategy backtesting --export trades --export-filename=backtest_samplestrategy.json
```

Please also read about the [strategy startup period](https://www.freqtrade.io/en/stable/strategy-customization/#strategy-startup-period).

Supplying custom fee value

Sometimes your account has certain fee rebates (fee reductions starting with a certain account size or monthly volume), which are not visible to ccxt. To account for this in backtesting, you can use the `--fee` command line option to supply this value to backtesting. This fee must be a ratio, and will be applied twice (once for trade entry, and once for trade exit).

For example, if the commission fee per order is 0.1% (i.e., 0.001 written as ratio), then you would run backtesting as the following:

```bash
freqtrade backtesting --fee 0.001
```

> Only supply this option (or the corresponding configuration parameter) if you want to experiment with different fee values. By default, Backtesting fetches the default fee from the exchange pair/market info.

Running backtest with smaller test-set by using timerange

Use the `--timerange` argument to change how much of the test-set you want to use.

For example, running backtesting with the `--timerange=20190501-` option will use all available data starting with May 1st, 2019 from your input data.

```bash
freqtrade backtesting --timerange=20190501-
```

You can also specify particular date ranges.

The full timerange specification:

- Use data until 2018/01/31: `--timerange=-20180131`
- Use data since 2018/01/31: `--timerange=20180131-`
- Use data since 2018/01/31 till 2018/03/01 : `--timerange=20180131-20180301`
- Use data between POSIX / epoch timestamps 1527595200 1527618600: `--timerange=1527595200-1527618600`


####  理解回测的结果

The most important in the backtesting is to understand the result.

A backtesting result will look like that:

```txt
================================================ BACKTESTING REPORT =================================================
| Pair     | Trades |   Avg Profit % |   Tot Profit BTC |   Tot Profit % | Avg Duration |  Wins Draws Loss   Win%  |
|----------+--------+----------------+------------------+----------------+--------------+--------------------------|
| ADA/BTC  |     35 |          -0.11 |      -0.00019428 |          -1.94 | 4:35:00      |    14     0    21   40.0 |
| ARK/BTC  |     11 |          -0.41 |      -0.00022647 |          -2.26 | 2:03:00      |     3     0     8   27.3 |
| BTS/BTC  |     32 |           0.31 |       0.00048938 |           4.89 | 5:05:00      |    18     0    14   56.2 |
| DASH/BTC |     13 |          -0.08 |      -0.00005343 |          -0.53 | 4:39:00      |     6     0     7   46.2 |
| ENG/BTC  |     18 |           1.36 |       0.00122807 |          12.27 | 2:50:00      |     8     0    10   44.4 |
| EOS/BTC  |     36 |           0.08 |       0.00015304 |           1.53 | 3:34:00      |    16     0    20   44.4 |
| ETC/BTC  |     26 |           0.37 |       0.00047576 |           4.75 | 6:14:00      |    11     0    15   42.3 |
| ETH/BTC  |     33 |           0.30 |       0.00049856 |           4.98 | 7:31:00      |    16     0    17   48.5 |
| IOTA/BTC |     32 |           0.03 |       0.00005444 |           0.54 | 3:12:00      |    14     0    18   43.8 |
| LSK/BTC  |     15 |           1.75 |       0.00131413 |          13.13 | 2:58:00      |     6     0     9   40.0 |
| LTC/BTC  |     32 |          -0.04 |      -0.00006886 |          -0.69 | 4:49:00      |    11     0    21   34.4 |
| NANO/BTC |     17 |           1.26 |       0.00107058 |          10.70 | 1:55:00      |    10     0     7   58.5 |
| NEO/BTC  |     23 |           0.82 |       0.00094936 |           9.48 | 2:59:00      |    10     0    13   43.5 |
| REQ/BTC  |      9 |           1.17 |       0.00052734 |           5.27 | 3:47:00      |     4     0     5   44.4 |
| XLM/BTC  |     16 |           1.22 |       0.00097800 |           9.77 | 3:15:00      |     7     0     9   43.8 |
| XMR/BTC  |     23 |          -0.18 |      -0.00020696 |          -2.07 | 5:30:00      |    12     0    11   52.2 |
| XRP/BTC  |     35 |           0.66 |       0.00114897 |          11.48 | 3:49:00      |    12     0    23   34.3 |
| ZEC/BTC  |     22 |          -0.46 |      -0.00050971 |          -5.09 | 2:22:00      |     7     0    15   31.8 |
| TOTAL    |    429 |           0.36 |       0.00762792 |          76.20 | 4:12:00      |   186     0   243   43.4 |
============================================= LEFT OPEN TRADES REPORT =============================================
| Pair     |  Trades |   Avg Profit % |   Tot Profit BTC |   Tot Profit % | Avg Duration   |  Win Draw Loss Win% |
|----------+---------+----------------+------------------+----------------+----------------+---------------------|
| ADA/BTC  |       1 |           0.89 |       0.00004434 |           0.44 | 6:00:00        |    1    0    0  100 |
| LTC/BTC  |       1 |           0.68 |       0.00003421 |           0.34 | 2:00:00        |    1    0    0  100 |
| TOTAL    |       2 |           0.78 |       0.00007855 |           0.78 | 4:00:00        |    2    0    0  100 |
==================== EXIT REASON STATS ====================
| Exit Reason        |   Exits |  Wins |  Draws |  Losses |
|--------------------+---------+-------+--------+---------|
| trailing_stop_loss |     205 |   150 |      0 |      55 |
| stop_loss          |     166 |     0 |      0 |     166 |
| exit_signal        |      56 |    36 |      0 |      20 |
| force_exit         |       2 |     0 |      0 |       2 |

================== SUMMARY METRICS ==================
| Metric                      | Value               |
|-----------------------------+---------------------|
| Backtesting from            | 2019-01-01 00:00:00 |
| Backtesting to              | 2019-05-01 00:00:00 |
| Max open trades             | 3                   |
|                             |                     |
| Total/Daily Avg Trades      | 429 / 3.575         |
| Starting balance            | 0.01000000 BTC      |
| Final balance               | 0.01762792 BTC      |
| Absolute profit             | 0.00762792 BTC      |
| Total profit %              | 76.2%               |
| CAGR %                      | 460.87%             |
| Sortino                     | 1.88                |
| Sharpe                      | 2.97                |
| Calmar                      | 6.29                |
| Profit factor               | 1.11                |
| Expectancy (Ratio)          | -0.15 (-0.05)       |
| Avg. stake amount           | 0.001      BTC      |
| Total trade volume          | 0.429      BTC      |
|                             |                     |
| Long / Short                | 352 / 77            |
| Total profit Long %         | 1250.58%            |
| Total profit Short %        | -15.02%             |
| Absolute profit Long        | 0.00838792 BTC      |
| Absolute profit Short       | -0.00076 BTC        |
|                             |                     |
| Best Pair                   | LSK/BTC 26.26%      |
| Worst Pair                  | ZEC/BTC -10.18%     |
| Best Trade                  | LSK/BTC 4.25%       |
| Worst Trade                 | ZEC/BTC -10.25%     |
| Best day                    | 0.00076 BTC         |
| Worst day                   | -0.00036 BTC        |
| Days win/draw/lose          | 12 / 82 / 25        |
| Avg. Duration Winners       | 4:23:00             |
| Avg. Duration Loser         | 6:55:00             |
| Max Consecutive Wins / Loss | 3 / 4               |
| Rejected Entry signals      | 3089                |
| Entry/Exit Timeouts         | 0 / 0               |
| Canceled Trade Entries      | 34                  |
| Canceled Entry Orders       | 123                 |
| Replaced Entry Orders       | 89                  |
|                             |                     |
| Min balance                 | 0.00945123 BTC      |
| Max balance                 | 0.01846651 BTC      |
| Max % of account underwater | 25.19%              |
| Absolute Drawdown (Account) | 13.33%              |
| Drawdown                    | 0.0015 BTC          |
| Drawdown high               | 0.0013 BTC          |
| Drawdown low                | -0.0002 BTC         |
| Drawdown Start              | 2019-02-15 14:10:00 |
| Drawdown End                | 2019-04-11 18:15:00 |
| Market change               | -5.88%              |
=====================================================
```

##### 回测报表

The 1st table contains all trades the bot made, including "left open trades".

The last line will give you the overall performance of your strategy, here:

```txt
| TOTAL    |    429 |           0.36 |         152.41 |       0.00762792 |          76.20 | 4:12:00      |   186     0   243   43.4 |
```

The bot has made `429` trades for an average duration of `4:12:00`, with a performance of `76.20%` (profit), that means it has earned a total of `0.00762792 BTC` starting with a capital of 0.01 BTC.

The column `Avg Profit %` shows the average profit for all trades made. The column `Tot Profit %` shows instead the total profit % in relation to the starting balance. In the above results, we have a starting balance of 0.01 BTC and an absolute profit of 0.00762792 BTC - so the `Tot Profit %` will be `(0.00762792 / 0.01) * 100 ~= 76.2%`.

Your strategy performance is influenced by your entry strategy, your exit strategy, and also by the `minimal_roi` and `stop_loss` you have set.

For example, if your `minimal_roi` is only `"0": 0.01` you cannot expect the bot to make more profit than 1% (because it will exit every time a trade reaches 1%).

Your strategy performance is influenced by your entry strategy, your exit strategy, and also by the `minimal_roi` and `stop_loss` you have set.

For example, if your `minimal_roi` is only `"0": 0.01` you cannot expect the bot to make more profit than 1% (because it will exit every time a trade reaches 1%).

```python
"minimal_roi": {
    "0":  0.01
},
```

On the other hand, if you set a too high `minimal_roi` like `"0": 0.55` (55%), there is almost no chance that the bot will ever reach this profit. Hence, keep in mind that your performance is an integral mix of all different elements of the strategy, your configuration, and the crypto-currency pairs you have set up.

##### Exit Reason table

The 2nd table contains a recap of exit reasons. This table can tell you which area needs some additional work (e.g. all or many of the `exit_signal` trades are losses, so you should work on improving the exit signal, or consider disabling it).

##### Left open trades table

The 3rd table contains all trades the bot had to `force_exit` at the end of the backtesting period to present you the full picture. This is necessary to simulate realistic behavior, since the backtest period has to end at some point, while realistically, you could leave the bot running forever. These trades are also included in the first table, but are also shown separately in this table for clarity.

##### summary metrics

The last element of the backtest report is the summary metrics table. It contains some useful key metrics about performance of your strategy on backtesting data.

```txt
================== SUMMARY METRICS ==================
| Metric                      | Value               |
|-----------------------------+---------------------|
| Backtesting from            | 2019-01-01 00:00:00 |
| Backtesting to              | 2019-05-01 00:00:00 |
| Max open trades             | 3                   |
|                             |                     |
| Total/Daily Avg Trades      | 429 / 3.575         |
| Starting balance            | 0.01000000 BTC      |
| Final balance               | 0.01762792 BTC      |
| Absolute profit             | 0.00762792 BTC      |
| Total profit %              | 76.2%               |
| CAGR %                      | 460.87%             |
| Sortino                     | 1.88                |
| Sharpe                      | 2.97                |
| Calmar                      | 6.29                |
| Profit factor               | 1.11                |
| Expectancy (Ratio)          | -0.15 (-0.05)       |
| Avg. stake amount           | 0.001      BTC      |
| Total trade volume          | 0.429      BTC      |
|                             |                     |
| Long / Short                | 352 / 77            |
| Total profit Long %         | 1250.58%            |
| Total profit Short %        | -15.02%             |
| Absolute profit Long        | 0.00838792 BTC      |
| Absolute profit Short       | -0.00076 BTC        |
|                             |                     |
| Best Pair                   | LSK/BTC 26.26%      |
| Worst Pair                  | ZEC/BTC -10.18%     |
| Best Trade                  | LSK/BTC 4.25%       |
| Worst Trade                 | ZEC/BTC -10.25%     |
| Best day                    | 0.00076 BTC         |
| Worst day                   | -0.00036 BTC        |
| Days win/draw/lose          | 12 / 82 / 25        |
| Avg. Duration Winners       | 4:23:00             |
| Avg. Duration Loser         | 6:55:00             |
| Max Consecutive Wins / Loss | 3 / 4               |
| Rejected Entry signals      | 3089                |
| Entry/Exit Timeouts         | 0 / 0               |
| Canceled Trade Entries      | 34                  |
| Canceled Entry Orders       | 123                 |
| Replaced Entry Orders       | 89                  |
|                             |                     |
| Min balance                 | 0.00945123 BTC      |
| Max balance                 | 0.01846651 BTC      |
| Max % of account underwater | 25.19%              |
| Absolute Drawdown (Account) | 13.33%              |
| Drawdown                    | 0.0015 BTC          |
| Drawdown high               | 0.0013 BTC          |
| Drawdown low                | -0.0002 BTC         |
| Drawdown Start              | 2019-02-15 14:10:00 |
| Drawdown End                | 2019-04-11 18:15:00 |
| Market change               | -5.88%              |
=====================================================
```

- `Backtesting from` / `Backtesting to`: Backtesting range (usually defined with the `--timerange` option).
- `Max open trades`: Setting of `max_open_trades` (or `--max-open-trades`) - or number of pairs in the pairlist (whatever is lower).
- `Total/Daily Avg Trades`: Identical to the total trades of the backtest output table / Total trades divided by the backtesting duration in days (this will give you information about how many trades to expect from the strategy).
- `Starting balance`: Start balance - as given by dry-run-wallet (config or command line).
- `Final balance`: Final balance - starting balance + absolute profit.
- `Absolute profit`: Profit made in stake currency.
- `Total profit %`: Total profit. Aligned to the `TOTAL` row's `Tot Profit %` from the first table. Calculated as `(End capital − Starting capital) / Starting capital`.
- `CAGR %`: Compound annual growth rate.
- `Sortino`: Annualized Sortino ratio.
- `Sharpe`: Annualized Sharpe ratio.
- `Calmar`: Annualized Calmar ratio.
- `Profit factor`: profit / loss.
- `Avg. stake amount`: Average stake amount, either `stake_amount` or the average when using dynamic stake amount.
- `Total trade volume`: Volume generated on the exchange to reach the above profit.
- `Best Pair` / `Worst Pair`: Best and worst performing pair, and it's corresponding `Tot Profit %`.
- `Best Trade` / `Worst Trade`: Biggest single winning trade and biggest single losing trade.
- `Best day` / `Worst day`: Best and worst day based on daily profit.
- `Days win/draw/lose`: Winning / Losing days (draws are usually days without closed trade).
- `Avg. Duration Winners` / `Avg. Duration Loser`: Average durations for winning and losing trades.
- `Max Consecutive Wins / Loss`: Maximum consecutive wins/losses in a row.
- `Rejected Entry signals`: Trade entry signals that could not be acted upon due to `max_open_trades` being reached.
- `Entry/Exit Timeouts`: Entry/exit orders which did not fill (only applicable if custom pricing is used).
- `Canceled Trade Entries`: Number of trades that have been canceled by user request via `adjust_entry_price`.
- `Canceled Entry Orders`: Number of entry orders that have been canceled by user request via `adjust_entry_price`.
- `Replaced Entry Orders`: Number of entry orders that have been replaced by user request via `adjust_entry_price`.
- `Min balance` / `Max balance`: Lowest and Highest Wallet balance during the backtest period.
- `Max % of account underwater`: Maximum percentage your account has decreased from the top since the simulation started. Calculated as the maximum of `(Max Balance - Current Balance) / (Max Balance)`.
- `Absolute Drawdown (Account)`: Maximum Account Drawdown experienced. Calculated as `(Absolute Drawdown) / (DrawdownHigh + startingBalance)`.
- `Drawdown`: Maximum, absolute drawdown experienced. Difference between Drawdown High and Subsequent Low point.
- `Drawdown high` / `Drawdown low`: Profit at the beginning and end of the largest drawdown period. A negative low value means initial capital lost.
- `Drawdown Start` / `Drawdown End`: Start and end datetime for this largest drawdown (can also be visualized via the `plot-dataframe` sub-command).
- `Market change`: Change of the market during the backtest period. Calculated as average of all pairs changes from the first to the last candle using the "close" column.
- `Long / Short`: Split long/short values (Only shown when short trades were made).
- `Total profit Long %` / `Absolute profit Long`: Profit long trades only (Only shown when short trades were made).
- `Total profit Short %` / `Absolute profit Short`: Profit short trades only (Only shown when short trades were made).

##### Daily / Weekly / Monthly breakdown

You can get an overview over daily / weekly or monthly results by using the `--breakdown <>` switch.

To visualize daily and weekly breakdowns, you can use the following:

```bash
freqtrade backtesting --strategy MyAwesomeStrategy --breakdown day week
```

```txt
======================== DAY BREAKDOWN =========================
|        Day |   Tot Profit USDT |   Wins |   Draws |   Losses |
|------------+-------------------+--------+---------+----------|
| 03/07/2021 |           200.0   |      2 |       0 |        0 |
| 04/07/2021 |           -50.31  |      0 |       0 |        2 |
| 05/07/2021 |           220.611 |      3 |       2 |        0 |
| 06/07/2021 |           150.974 |      3 |       0 |        2 |
| 07/07/2021 |           -70.193 |      1 |       0 |        2 |
| 08/07/2021 |           212.413 |      2 |       0 |        3 |
```

The output will show a table containing the realized absolute Profit (in stake currency) for the given timeperiod, as well as wins, draws and losses that materialized (closed) on this day. Below that there will be a second table for the summarized values of weeks indicated by the date of the closing Sunday. The same would apply to a monthly breakdown indicated by the last day of the month.

##### 回测结果缓存

To save time, by default backtest will reuse a cached result from within the last day when the backtested strategy and config match that of a previous backtest. To force a new backtest despite existing result for an identical run specify `--cache none` parameter.

> Caching is automatically disabled for open-ended timeranges (`--timerange 20210101-`), as freqtrade cannot ensure reliably that the underlying data didn't change. It can also use cached results where it shouldn't if the original backtest had missing data at the end, which was fixed by downloading more data. In this instance, please use `--cache none` once to force a fresh backtest.

##### 更进一步的回测结果分析

To further analyze your backtest results, freqtrade will export the trades to file by default. You can then load the trades to perform further analysis as shown in the [data analysis](https://www.freqtrade.io/en/stable/strategy_analysis_example/#load-backtest-results-to-pandas-dataframe) backtesting section.

#### 回测的前提假设

Since backtesting lacks some detailed information about what happens within a candle, it needs to take a few assumptions:

- Exchange [trading limits](https://www.freqtrade.io/en/stable/backtesting/#trading-limits-in-backtesting) are respected
- Entries happen at open-price
- All orders are filled at the requested price (no slippage) as long as the price is within the candle's high/low range
- Exit-signal exits happen at open-price of the consecutive candle
- Exits don't free their trade slot for a new trade until the next candle
- Exit-signal is favored over Stoploss, because exit-signals are assumed to trigger on candle's open
- ROI
    - Exits are compared to high - but the ROI value is used (e.g. ROI = 2%, high=5% - so the exit will be at 2%)
    - Exits are never "below the candle", so a ROI of 2% may result in a exit at 2.4% if low was at 2.4% profit
    - ROI entries which came into effect on the triggering candle (e.g. `120: 0.02` for 1h candles, from `60: 0.05`) will use the candle's open as exit rate
    - Force-exits caused by `<N>=-1` ROI entries use low as exit value, unless N falls on the candle open (e.g. `120: -1` for 1h candles)
- Stoploss exits happen exactly at stoploss price, even if low was lower, but the loss will be `2 * fees` higher than the stoploss price
- Stoploss is evaluated before ROI within one candle. So you can often see more trades with the `stoploss` exit reason comparing to the results obtained with the same strategy in the Dry Run/Live Trade modes
- Low happens before high for stoploss, protecting capital first
- Trailing stoploss
    - Trailing Stoploss is only adjusted if it's below the candle's low (otherwise it would be triggered)
    - On trade entry candles that trigger trailing stoploss, the "minimum offset" (`stop_positive_offset`) is assumed (instead of high) - and the stop is calculated from this point. This rule is NOT applicable to custom-stoploss scenarios, since there's no information about the stoploss logic available.
    - High happens first - adjusting stoploss
    - Low uses the adjusted stoploss (so exits with large high-low difference are backtested correctly)
    - ROI applies before trailing-stop, ensuring profits are "top-capped" at ROI if both ROI and trailing stop applies
- Exit-reason does not explain if a trade was positive or negative, just what triggered the exit (this can look odd if negative ROI values are used)
- Evaluation sequence (if multiple signals happen on the same candle)
    - Exit-signal
    - Stoploss
    - ROI
    - Trailing stoploss

Taking these assumptions, backtesting tries to mirror real trading as closely as possible. However, backtesting will **never** replace running a strategy in dry-run mode. Also, keep in mind that past results don't guarantee future success.

In addition to the above assumptions, strategy authors should carefully read the [Common Mistakes](https://www.freqtrade.io/en/stable/strategy-customization/#common-mistakes-when-developing-strategies) section, to avoid using data in backtesting which is not available in real market conditions.

##### 回测中的交易限制

Exchanges have certain trading limits, like minimum (and maximum) base currency, or minimum/maximum stake (quote) currency. These limits are usually listed in the exchange documentation as "trading rules" or similar and can be quite different between different pairs.

Backtesting (as well as live and dry-run) does honor these limits, and will ensure that a stoploss can be placed below this value - so the value will be slightly higher than what the exchange specifies. Freqtrade has however no information about historic limits.

This can lead to situations where trading-limits are inflated by using a historic price, resulting in minimum amounts > 50$.

For example:

BTC minimum tradable amount is 0.001. BTC trades at 22.000\$ today (0.001 BTC is related to this) - but the backtesting period includes prices as high as 50.000\$. Today's minimum would be `0.001 * 22_000` - or 22\$.  
However the limit could also be 50\$ - based on `0.001 * 50_000` in some historic setting.

###### 交易精度限制

Most exchanges pose precision limits on both price and amounts, so you cannot buy 1.0020401 of a pair, or at a price of 1.24567123123.  
Instead, these prices and amounts will be rounded or truncated (based on the exchange definition) to the defined trading precision. The above values may for example be rounded to an amount of 1.002, and a price of 1.24567.

These precision values are based on current exchange limits (as described in the [above section](https://www.freqtrade.io/en/stable/backtesting/#trading-limits-in-backtesting)), as historic precision limits are not available.

#### 提高回测准确度

One big limitation of backtesting is it's inability to know how prices moved intra-candle (was high before close, or vice-versa?). So assuming you run backtesting with a 1h timeframe, there will be 4 prices for that candle (Open, High, Low, Close).

While backtesting does take some assumptions (read above) about this - this can never be perfect, and will always be biased in one way or the other. To mitigate this, freqtrade can use a lower (faster) timeframe to simulate intra-candle movements.

To utilize this, you can append `--timeframe-detail 5m` to your regular backtesting command.

```bash
freqtrade backtesting --strategy AwesomeStrategy --timeframe 1h --timeframe-detail 5m
```

This will load 1h data as well as 5m data for the timeframe. The strategy will be analyzed with the 1h timeframe, and Entry orders will only be placed at the main timeframe, however Order fills and exit signals will be evaluated at the 5m candle, simulating intra-candle movements.

All callback functions (`custom_exit()`, `custom_stoploss()`, ... ) will be running for each 5m candle once the trade is opened (so 12 times in the above example of 1h timeframe, and 5m detailed timeframe).

`--timeframe-detail` must be smaller than the original timeframe, otherwise backtesting will fail to start.

Obviously this will require more memory (5m data is bigger than 1h data), and will also impact runtime (depending on the amount of trades and trade durations). Also, data must be available / downloaded already.

> You can use this function as the last part of strategy development, to ensure your strategy is not exploiting one of the [backtesting assumptions](https://www.freqtrade.io/en/stable/backtesting/#assumptions-made-by-backtesting). Strategies that perform similarly well with this mode have a good chance to perform well in dry/live modes too (although only forward-testing (dry-mode) can really confirm a strategy).


#### 回测多种策略

To compare multiple strategies, a list of Strategies can be provided to backtesting.

This is limited to 1 timeframe value per run. However, data is only loaded once from disk so if you have multiple strategies you'd like to compare, this will give a nice runtime boost.

All listed Strategies need to be in the same directory, unless also `--recursive-strategy-search` is specified, where sub-directories within the strategy directory are also considered.

```bash
freqtrade backtesting --timerange 20180401-20180410 --timeframe 5m --strategy-list Strategy001 Strategy002 --export trades
```

This will save the results to `user_data/backtest_results/backtest-result-<datetime>.json`, including results for both `Strategy001` and `Strategy002`. There will be an additional table comparing win/losses of the different strategies (identical to the "Total" row in the first table). Detailed output for all strategies one after the other will be available, so make sure to scroll up to see the details per strategy.

```txt
================================================== STRATEGY SUMMARY ===================================================================
| Strategy    |  Trades |   Avg Profit % |   Tot Profit BTC |   Tot Profit % | Avg Duration   |  Wins |  Draws | Losses | Drawdown % |
|-------------+---------+----------------+------------------+----------------+----------------+-------+--------+--------+------------|
| Strategy1   |     429 |           0.36 |       0.00762792 |          76.20 | 4:12:00        |   186 |      0 |    243 |       45.2 |
| Strategy2   |    1487 |          -0.13 |      -0.00988917 |         -98.79 | 4:43:00        |   662 |      0 |    825 |     241.68 |
```

如果你的策略能盈利，下一步可以研究如何找到可以提高你策略性能的[[#超参数(hyperopt)]]

## 超参数(hyperopt)

[Hyperopt - Freqtrade](https://www.freqtrade.io/en/stable/hyperopt/)
## FreqAI

## 空头/杠杆


## 工具的子命令

## 绘图(plotting)


## 交易相关的注意事项

## 数据分析

## 高级话题

## FAQ

## 策略迁移

## 更新Freqtrade

