
[Trading Strategies & Indicators Built by TradingView Community](https://www.tradingview.com/script/M4y5HT3h-Triple-EMA-QQE-Trend-Following-Strategy-TradeDots/)

![[Pasted image 20240621173141.png]]


“三重 EMA + QQE 趋势跟踪策略”利用两个复杂的技术指标的力量，即三重指数移动平均线 （TEMA） 和定性定量估计 （QQE），以生成精确的买入和卖出信号。该策略擅长通过识别短期价格动量和动态超买或超卖条件来捕捉趋势变化。


## How it works

该策略集成了两个关键指标：  
  
**三重指数移动平均线 （TEMA）：**TEMA 通过减少滞后和更有效地平滑数据来增强传统移动平均线。它通过将 EMA 公式应用于价格三次来实现这一点，如下所示：

```pascal
// pine script
tema(src, length) =>  
    ema1 = ta.ema(src, length)  
    ema2 = ta.ema(ema1, length)  
    ema3 = ta.ema(ema2, length)  
    tema = 3*ema1 - 3*ema2 + ema3
```

这种计算有助于提高对价格变动的敏感性。  
  
**定性定量估计 （QQE）：**QQE 指标通过纳入平滑机制来改进标准 RSI。它从标准 RSI 开始，在此 RSI 上叠加 5 周期 EMA，然后使用 27 周期 EMA 的双重应用来增强结果。然后通过将结果乘以因子数来得出慢速尾随线。这种方法建立了一个更精细、更不紧张的趋势跟踪信号，补充了 TEMA，以增强波动条件下的整体市场时机。


## Application

参考对“交易策略”的见解，策略实施如下：  
  
首先，我们使用两条TEMA线：一条设置为20周期，另一条设置为40周期。然后按照以下规则操作：  
  

1. 40周期的TEMA正在上升  
    
2. 20 周期 TEMA 高于 40 周期 TEMA  
    
3. 价格收于20周期TEMA上方  
    
4. 今天不是星期一  
    
5. RSI MA 越过慢速追踪线  
    

  
该策略不采用主动止盈机制;相反，它利用追踪止损让价格自然达到止损，从而最大限度地提高潜在利润率。

## Default setup

佣金：0.01%  
初始资本：$10,000  
每笔交易净值：80%  
  
建议用户调整和个性化此交易策略，以更好地匹配他们的个人交易偏好和风格。

## Script

![[Pasted image 20240621172645.png]]

```pascal
// This Pine Script™ code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © tradedots

//@version=5
strategy("Triple EMA + QQE Trend Following Strategy [TradeDots]", overlay = true, max_lines_count = 500, max_boxes_count = 100, initial_capital = 10000, currency = currency.USD, commission_type = strategy.commission.percent, commission_value = 0.01, default_qty_type = strategy.percent_of_equity, default_qty_value = 80)

//pipsize
GetPipSize() =>
    syminfo.mintick * (syminfo.type == "forex" ? 10 : 1)

pipSize = GetPipSize()

var ts = 0.
var float stopLoss = na

//qqe values
rsi_period = input(14, title='RSI Length', group = "QQE")
rsi_smoothing_period = input(5, title='RSI Smoothing Period', group = "QQE")
factor = input(4.238, title='QQE Factor', group = "QQE")

//tema
ma_source = input(close, "TEMA Source", inline="MA1", group="TEMA")
tema1_length = input.int(20, "TEMA #1 Length", minval=1,group="TEMA")
tema2_length = input.int(40, "TEMA #2 Length", minval=1,group="TEMA")

input_stop_loss = input.int(120, "Stop Loss (pip)", group = "Strategy")

tema(src, length) =>
    ema1 = ta.ema(src, length)
    ema2 = ta.ema(ema1, length)
    ema3 = ta.ema(ema2, length)

    tema = 3*ema1 - 3*ema2 + ema3

tema1 = tema(ma_source, tema1_length)
tema2 = tema(ma_source, tema2_length)

tema1Plot = plot(tema1, title = " TEMA #1", color = #c7f9cc, linewidth = 2)
tema2Plot = plot(tema2, title = " TEMA #2", color = #57cc99, linewidth = 2)

// calculating qqe
rsi = ta.rsi(close, rsi_period)
rsiMa = ta.ema(rsi, rsi_smoothing_period)
atrRsi = math.abs(rsiMa[1] - rsiMa)

final_smooth = rsi_period * 2 - 1
MaAtrRsi = ta.ema(atrRsi, final_smooth)
dar = ta.ema(MaAtrRsi, final_smooth) * factor

crossover = ta.crossover(rsiMa, ts)
crossunder = ta.crossunder(rsiMa, ts)

ts := nz(crossover ? rsiMa - MaAtrRsi * factor
  : crossunder ? rsiMa + MaAtrRsi * factor
  : rsiMa > ts ? math.max(rsiMa - MaAtrRsi * factor, ts)
  : math.min(rsiMa + MaAtrRsi * factor, ts), rsiMa)

// entry and exit conditions
long_condition = close > tema1 and tema1 > tema2 and tema2 > tema2[1] and ta.crossover(rsiMa, ts) and dayofweek != dayofweek.monday
short_condition = close < tema1 and tema1 < tema2 and tema2 < tema2[1] and ta.crossunder(rsiMa, ts) and dayofweek != dayofweek.monday

if long_condition and strategy.opentrades == 0
    strategy.entry("Long", strategy.long)
    label.new(bar_index, low, "🟢 Long", color = #9cff87, style = label.style_label_up)
    stopLoss := close - input_stop_loss * pipSize

else if short_condition and strategy.opentrades == 0
    strategy.entry("Short", strategy.short)
    label.new(bar_index, high, "🔴 Short", color = #f9396a, style = label.style_label_down, textcolor = color.white)
    stopLoss := close + input_stop_loss * pipSize

else if strategy.opentrades == 0
    stopLoss := na

if barstate.isconfirmed and strategy.opentrades > 0
    if strategy.opentrades.size(strategy.opentrades- 1) > 0
        if close < stopLoss
            strategy.close("Long")
        stopLoss := (close - input_stop_loss * pipSize) > stopLoss ? close - input_stop_loss * pipSize : stopLoss
    else if strategy.opentrades.size(strategy.opentrades - 1) < 0
        if close > stopLoss
            strategy.close("Short")
        stopLoss := (close + input_stop_loss * pipSize) < stopLoss ? close + input_stop_loss * pipSize : stopLoss

// Check if position is closed (Print label if true)
is_pos_closed = (strategy.position_size[1] != 0 and strategy.position_size == 0)
is_short = strategy.position_size < 0
is_long = strategy.position_size > 0

if is_pos_closed and is_short[1]
    label.new(bar_index, low, "🟢 Close Short", color = #9cff87, style = label.style_label_up)

else if is_pos_closed and is_long[1]
    label.new(bar_index, high, "🔴 Close Long", color = #f9396a, style = label.style_label_down, textcolor = color.white)

fill(tema2Plot, tema1Plot, math.max(tema2,tema1), tema2,color.new(#9cff87, 85), color.new(#9cff87, 70))
fill(tema2Plot, tema1Plot, tema2, math.min(tema2,tema1),color.new(#f9396a, 70), color.new(#f9396a, 85))

plot(stopLoss, color=#f9396a, style = plot.style_cross, linewidth = 2, title = "Trail Stop")
```