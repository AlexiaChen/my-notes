
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
- **Trade**: Open position. 也就是未平仓头寸。头寸的意思就是你买或卖的位置，对于买，就是多头头寸。对于卖，就是空投头寸。平仓的意思就是你这个买或卖的位置的反向操作。
- **Open Order**: Order which is currently placed on the exchange, and is not yet complete.
- **Pair**: Tradable pair, usually in the format of Base/Quote (e.g. `XRP/USDT` for spot, `XRP/USDT:USDT` for futures).
- **Timeframe**: Candle length to use (e.g. `"5m"`, `"1h"`, ...).
- **Indicators**: Technical indicators (SMA, EMA, RSI, ...).
- **Limit order**: Limit orders which execute at the defined limit price or better.
- **Market order**: Guaranteed to fill, may move price depending on the order size.
- **Current Profit**: Currently pending (unrealized) profit for this trade. This is mainly used throughout the bot and UI.
- **Realized Profit**: Already realized profit. Only relevant in combination with [partial exits](https://www.freqtrade.io/en/stable/strategy-callbacks/#adjust-trade-position) - which also explains the calculation logic for this.
- **Total Profit**: Combined realized and unrealized profit. The relative number (%) is calculated against the total investment in this trade.