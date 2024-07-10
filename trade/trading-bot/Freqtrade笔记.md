
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
- **Open Order**: 就是这个订单还没有被平仓，
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

- 从持久化存储中获取未平仓的交易
- 计算当前的可交易的交易对列表
- Download OHLCV data for the pairlist including all [informative pairs](https://www.freqtrade.io/en/stable/strategy-customization/#get-data-for-non-tradeable-pairs)  
    This step is only executed once per Candle to avoid unnecessary network traffic.
- Call `bot_loop_start()` strategy callback.
- Analyze strategy per pair.
    - Call `populate_indicators()`
    - Call `populate_entry_trend()`
    - Call `populate_exit_trend()`
- Update trades open order state from exchange.
    - Call `order_filled()` strategy callback for filled orders.
    - Check timeouts for open orders.
        - Calls `check_entry_timeout()` strategy callback for open entry orders.
        - Calls `check_exit_timeout()` strategy callback for open exit orders.
        - Calls `adjust_entry_price()` strategy callback for open entry orders.
- Verifies existing positions and eventually places exit orders.
    - Considers stoploss, ROI and exit-signal, `custom_exit()` and `custom_stoploss()`.
    - Determine exit-price based on `exit_pricing` configuration setting or by using the `custom_exit_price()` callback.
    - Before a exit order is placed, `confirm_trade_exit()` strategy callback is called.
- Check position adjustments for open trades if enabled by calling `adjust_trade_position()` and place additional order if required.
- Check if trade-slots are still available (if `max_open_trades` is reached).
- Verifies entry signal trying to enter new positions.
    - Determine entry-price based on `entry_pricing` configuration setting, or by using the `custom_entry_price()` callback.
    - In Margin and Futures mode, `leverage()` strategy callback is called to determine the desired leverage.
    - Determine stake size by calling the `custom_stake_amount()` callback.
    - Before an entry order is placed, `confirm_trade_entry()` strategy callback is called.

以上循环会一直重复

#### 回测和超参数的执行逻辑

其实也差不多

[backtesting](https://www.freqtrade.io/en/stable/backtesting/) or [hyperopt](https://www.freqtrade.io/en/stable/hyperopt/) do only part of the above logic, since most of the trading operations are fully simulated.

- Load historic data for configured pairlist.
- Calls `bot_start()` once.
- Calculate indicators (calls `populate_indicators()` once per pair).
- Calculate entry / exit signals (calls `populate_entry_trend()` and `populate_exit_trend()` once per pair).
- Loops per candle simulating entry and exit points.
    - Calls `bot_loop_start()` strategy callback.
    - Check for Order timeouts, either via the `unfilledtimeout` configuration, or via `check_entry_timeout()` / `check_exit_timeout()` strategy callbacks.
    - Calls `adjust_entry_price()` strategy callback for open entry orders.
    - Check for trade entry signals (`enter_long` / `enter_short` columns).
    - Confirm trade entry / exits (calls `confirm_trade_entry()` and `confirm_trade_exit()` if implemented in the strategy).
    - Call `custom_entry_price()` (if implemented in the strategy) to determine entry price (Prices are moved to be within the opening candle).
    - In Margin and Futures mode, `leverage()` strategy callback is called to determine the desired leverage.
    - Determine stake size by calling the `custom_stake_amount()` callback.
    - Check position adjustments for open trades if enabled and call `adjust_trade_position()` to determine if an additional order is requested.
    - Call `order_filled()` strategy callback for filled entry orders.
    - Call `custom_stoploss()` and `custom_exit()` to find custom exit points.
    - For exits based on exit-signal, custom-exit and partial exits: Call `custom_exit_price()` to determine exit price (Prices are moved to be within the closing candle).
    - Call `order_filled()` strategy callback for filled exit orders.
- Generate backtest report output

> Both Backtesting and Hyperopt include exchange default Fees in the calculation. Custom fees can be passed to backtesting / hyperopt by specifying the `--fee` argument.


> Backtesting will call each callback at max. once per candle (`--timeframe-detail` modifies this behavior to once per detailed candle). Most callbacks will be called once per iteration in live (usually every ~5s) - which can cause backtesting mismatches.