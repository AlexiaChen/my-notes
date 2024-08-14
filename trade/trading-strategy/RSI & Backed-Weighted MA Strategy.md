**INTRODUCTION :**  
  
This strategy is based on two well-known indicators that work best together: the Relative Strength Index (RSI) and the Moving Average (MA). We're going to use the RSI as a trend-follower indicator, rather than a reversal indicator as most are used to. To the signals sent by the RSI, we'll add a condition on the chart's MA, filtering out irrelevant signals and considerably increasing our winning rate. This is a medium/long-term strategy. There's also a money management method enabling us to reinvest part of the profits or reduce the size of orders in the event of substantial losses.  
  
  
**RSI :**  
  
The RSI is one of the best-known and most widely used indicators in trading. Its purpose is to warn traders when an asset is overbought or oversold. It was designed to send reversal signals, but we're going to use it as a trend indicator by increasing its length to 20. The RSI formula is as follows :  
  
_RSI (n) = 100 - (100 / (1 + (H (n)/L (n))))_  
  
With n the length of the RSI, H(n) the average of days closing above the open and L(n) the average of days closing below the open.  
  
  
**MA :**  
  
The Moving Average is also widely used in technical analysis, to smooth out variations in an asset. The SMA formula is as follows :  
  
_SMA (n) = (P1 + P2 + ... + Pn) / n_  
  
where n is the length of the MA.  
  
However, an SMA does not weight any of its terms, which means that the price 10 days ago has the same importance as the price 2 days ago or today's price... That's why in this strategy we use a RWMA, i.e. a back-weighted moving average. It weights old prices more heavily than new ones. This will enable us to limit the impact of short-term variations and focus on the trend that was dominating. The RWMA used weights :  

- The 4 most recent terms by : 100 / (4+(n-4)*1.30)  
    
- The other oldest terms by : weight_4_first_term*1.30  
    

So the older terms are weighted 1.30 more than the more recent ones. The moving average thus traces a trend that accentuates past values and limits the noise of short-term variations.  
  
  
**PARAMETERS :**  
  

- RSI Length : Lenght of RSI. Default is 20.  
    
- MA Type : Choice between a SMA or a RWMA which permits to minimize the impact of short term reversal. Default is RWMA.  
    
- MA Length : Length of the selected MA. Default is 19.  
    
- RSI Long Signal : Minimum value of RSI to send a LONG signal. Default is 60.  
    
- RSI Short signal : Maximum value of RSI to send a SHORT signal. Default is 40.  
    
- ROC MA Long Signal : Maximum value of Rate of Change MA to send a LONG signal. Default is 0.  
    
- ROC MA Short signal : Minimum value of Rate of Change MA to send a SHORT signal. Default is 0.  
    
- TP activation in multiple of ATR : Threshold value to trigger trailing stop Take Profit. This threshold is calculated as multiple of the ATR (Average True Range). Default value is 5 meaning that to trigger the trailing TP the price need to move 5*ATR in the right direction.  
    
- Trailing TP in percentage : Percentage value of trailing Take Profit. This Trailing TP follows the profit if it increases, remaining selected percentage below it, but stops if the profit decreases. Default is 3%.  
    
- Fixed Ratio : This is the amount of gain or loss at which the order quantity is changed. Default is 400, which means that for each $400 gain or loss, the order size is increased or decreased by a user-selected amount.  
    
- Increasing Order Amount : This is the amount to be added to or subtracted from orders when the fixed ratio is reached. The default is $200, which means that for every $400 gain, $200 is reinvested in the strategy. On the other hand, for every $400 loss, the order size is reduced by $200.  
    
- Initial capital : $1000  
    
- Fees : Interactive Broker fees apply to this strategy. They are set at 0.18% of the trade value.  
    
- Slippage : 3 ticks or $0.03 per trade. Corresponds to the latency time between the moment the signal is received and the moment the order is executed by the broker.  
    
- Important : A bot has been used to test the different parameters and determine which ones maximize return while limiting drawdown. This strategy is the most optimal on [![](https://s3-symbol-logo.tradingview.com/crypto/XTVCETH.svg)ETHUSD](https://www.tradingview.com/symbols/ETHUSD/) with a timeframe set to 6h. Parameters are set as follows :  
    

MA type: RWMA  
MA Length: 19  
RSI Long Signal: >60  
RSI Short Signal : <40  
ROC MA Long Signal : <0  
ROC MA Short Signal : >0  
TP Activation in multiple ATR : 5  
Trailing TP in percentage : 3  
  
  
**ENTER RULES :**  
  
The principle is very simple:  

- If the asset is overbought after a bear market, we are LONG.  
    
- If the asset is oversold after a bull market, we are SHORT.  
    

We have defined a bear market as follows : Rate of Change (20) RWMA < 0  
We have defined a bull market as follows : Rate of Change (20) RWMA > 0  
The Rate of Change is calculated using this formula : (RWMA/RWMA(20) - 1)*100  
Overbought is defined as follows : RSI > 60  
Oversold is defined as follows : RSI < 40  
  
**LONG CONDITION :**  
RSI > 60 and (RWMA/RWMA(20) - 1)*100 < -1  
  
**SHORT CONDITION :**  
RSI < 40 and (RWMA/RWMA(20) - 1)*100 > 1  
  
  
**EXIT RULES FOR WINNING TRADE :**  
  
We have a trailing TP allowing us to exit once the price has reached the "TP Activation in multiple ATR" parameter, i.e. 5*ATR by default in the profit direction. TP trailing is triggered at this point, not limiting our gains, and securing our profits at 3% below this trigger threshold.  
  
Remember that the True Range is : maximum(H-L, H-C(1), C-L(1))  
with C : Close, H : High, L : Low  
The Average True Range is therefore the average of these TRs over a length defined by default in the strategy, i.e. 20.  
  
  
**RISK MANAGEMENT :**  
  
This strategy may incur losses. The method for limiting losses is to set a Stop Loss equal to 3*ATR. This means that if the price moves against our position and reaches three times the ATR, we exit with a loss.  
Sometimes the ATR can result in a SL set below 10% of the trade value, which is not acceptable. In this case, we set the SL at 10%, limiting losses to a maximum of 10%.  
  
  
**MONEY MANAGEMENT :**  
  
The fixed ratio method was used to manage our gains and losses. For each gain of an amount equal to the value of the fixed ratio, we increase the order size by a value defined by the user in the "Increasing order amount" parameter. Similarly, each time we lose an amount equal to the value of the fixed ratio, we decrease the order size by the same user-defined value. This strategy increases both performance and drawdown.  
  
  
Enjoy the strategy and don't forget to take the trade :)

```cpp
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © gsanson66


//This code is based on RSI and a backed weighted MA
//@version=5
strategy("RSI + MA STRATEGY", overlay=true, initial_capital=1000, default_qty_type=strategy.fixed, commission_type=strategy.commission.percent, commission_value=0.18, slippage=3)


//------------------------TOOL TIPS---------------------------//

t1 = "Choice between a Standard MA (SMA) or a backed-weighted MA (RWMA) which permits to minimize the impact of short term reversal. Default is RWMA."
t2 = "Value of RSI to send a LONG or a SHORT signal. RSI above 60 is a LONG signal and RSI below 40 is a SHORT signal."
t3 = "Rate of Change Value of selected MA to send a LONG or a SHORT signal. By default : ROC MA below 0 is a LONG signal and ROC MA above 0 is a SHORT signal"
t4 = "Threshold value to trigger trailing Take Profit. This threshold is calculated as a multiple of the ATR (Average True Range)."
t5 = "Percentage value of trailing Take Profit. This Trailing TP follows the profit if it increases, remaining selected percentage below it, but stops if the profit decreases."
t6 = "The maximum percentage of the trade value we can lose. Default is 10%"
t7 = "Each gain or losse (relative to the previous reference) in an amount equal to this fixed ratio will change quantity of orders."
t8 = "The amount of money to be added to or subtracted from orders once the fixed ratio has been reached."


//------------------------FUNCTIONS---------------------------//

//@function which calculate a retro weighted moving average to minimize the impact of short term reversal
rwma(source, length) =>
    sum = 0.0
    denominator = 0.0
    weight = 0.0
    weight_x = 100/(4+(length-4)*1.30)
    weight_y = 1.30*weight_x
    for i=0 to length - 1
        if i <= 3
            weight := weight_x
        else
            weight := weight_y
        sum := sum + source[i] * weight
        denominator := denominator + weight
    rwma = sum/denominator

//@function which permits the user to choose a moving average type
ma(source, length, type) =>
    switch type
        "SMA" => ta.sma(source, length)
        "RWMA" => rwma(source, length)

//@function Displays text passed to `txt` when called.
debugLabel(txt, color) =>
    label.new(bar_index, high, text = txt, color=color, style = label.style_label_lower_right, textcolor = color.black, size = size.small)

//@function which looks if the close date of the current bar falls inside the date range
inBacktestPeriod(start, end) => (time >= start) and (time <= end)


//--------------------------------USER INPUTS-------------------------------//

//Technical parameters
rsiLengthInput = input.int(20, minval=1, title="RSI Length", group="RSI Settings")
maTypeInput = input.string("RWMA", title="MA Type", options=["SMA", "RWMA"], group="MA Settings", inline="1", tooltip=t1)
maLenghtInput = input.int(19, minval=1, title="MA Length", group="MA Settings", inline="1")
rsiLongSignalValue = input.int(60, minval=1, maxval=99, title="RSI Long Signal", group="Strategy parameters", inline="3")
rsiShortSignalValue = input.int(40, minval=1, maxval=99, title="RSI Short Signal", group="Strategy parameters", inline="3", tooltip=t2)
rocMovAverLongSignalValue = input.float(0, maxval=0, title="ROC MA Long Signal", group="Strategy parameters", inline="4")
rocMovAverShortSignalValue = input.float(0, minval=0, title="ROC MA Short Signal", group="Strategy parameters", inline="4", tooltip=t3)
//TP Activation and Trailing TP
takeProfitActivationInput = input.float(5, minval=1.0, title="TP activation in multiple of ATR", group="Strategy parameters", tooltip=t4)
trailingStopInput = input.float(3, minval=0, title="Trailing TP in percentage", group="Strategy parameters", tooltip=t5)
maxSl = input.float(10, "Max loss per trade (in %)", minval=0, tooltip=t6)
//Money Management
fixedRatio = input.int(defval=400, minval=1, title="Fixed Ratio Value ($)", group="Money Management", tooltip=t7)
increasingOrderAmount = input.int(defval=200, minval=1, title="Increasing Order Amount ($)", group="Money Management", tooltip=t8)
//Backtesting period
startDate = input.time(title="Start Date", defval=timestamp("1 Jan 2017 00:00:00"), group="Backtesting Period")
endDate = input.time(title="End Date", defval=timestamp("1 July 2024 00:00:00"), group="Backtesting Period")


//------------------------------VARIABLES INITIALISATION-----------------------------//

float rsi = ta.rsi(close, rsiLengthInput)
float ma = ma(close, maLenghtInput, maTypeInput)
float roc_ma = ((ma/ma[maLenghtInput]) - 1)*100
float atr = ta.atr(20)
var float trailingStopOffset = na
var float trailingStopActivation = na
var float trailingStop = na
var float stopLoss = na
var bool long = na
var bool short = na
var bool bufferTrailingStopDrawing = na
float theoreticalStopPrice = na
bool inRange = na
equity = math.abs(strategy.equity - strategy.openprofit)
var float capital_ref = strategy.initial_capital
var float cashOrder = strategy.initial_capital * 0.95


//------------------------------CHECKING SOME CONDITIONS ON EACH SCRIPT EXECUTION-------------------------------//

//Checking if the date belong to the range
inRange := inBacktestPeriod(startDate, endDate)

//Checking performances of the strategy
if equity > capital_ref + fixedRatio
    spread = (equity - capital_ref)/fixedRatio
    nb_level = int(spread)
    increasingOrder = nb_level * increasingOrderAmount
    cashOrder := cashOrder + increasingOrder
    capital_ref := capital_ref + nb_level*fixedRatio
if equity < capital_ref - fixedRatio
    spread = (capital_ref - equity)/fixedRatio
    nb_level = int(spread)
    decreasingOrder = nb_level * increasingOrderAmount
    cashOrder := cashOrder - decreasingOrder
    capital_ref := capital_ref - nb_level*fixedRatio

//Checking if we close all trades in case where we exit the backtesting period
if strategy.position_size!=0 and not inRange
    debugLabel("END OF BACKTESTING PERIOD : we close the trade", color=color.rgb(116, 116, 116))
    strategy.close_all()
    bufferTrailingStopDrawing := false
    stopLoss := na
    trailingStopActivation := na
    trailingStop := na
    short := false
    long := false


//------------------------------STOP LOSS AND TRAILING STOP ACTIVATION----------------------------//

// We handle the stop loss and trailing stop activation 
if (low <= stopLoss or high >= trailingStopActivation) and long
    if high >= trailingStopActivation
        bufferTrailingStopDrawing := true
    else if low <= stopLoss
        long := false
    stopLoss := na
    trailingStopActivation := na
if (low <= trailingStopActivation or high >= stopLoss) and short
    if low <= trailingStopActivation
        bufferTrailingStopDrawing := true
    else if high >= stopLoss
        short := false
    stopLoss := na
    trailingStopActivation := na


//-------------------------------------TRAILING STOP--------------------------------------//

// If the traling stop is activated, we manage its plotting with the bufferTrailingStopDrawing
if bufferTrailingStopDrawing and long
    theoreticalStopPrice := high - trailingStopOffset * syminfo.mintick
    if na(trailingStop)
        trailingStop := theoreticalStopPrice
    else if theoreticalStopPrice > trailingStop
        trailingStop := theoreticalStopPrice
    else if low <= trailingStop
        trailingStop := na
        bufferTrailingStopDrawing := false
        long := false
if bufferTrailingStopDrawing and short
    theoreticalStopPrice := low + trailingStopOffset * syminfo.mintick
    if na(trailingStop)
        trailingStop := theoreticalStopPrice
    else if theoreticalStopPrice < trailingStop
        trailingStop := theoreticalStopPrice
    else if high >= trailingStop
        trailingStop := na
        bufferTrailingStopDrawing := false
        short := false


//---------------------------------LONG CONDITION--------------------------//

if rsi >= 60 and roc_ma <= rocMovAverLongSignalValue and inRange and not long
    if short
        bufferTrailingStopDrawing := false
        stopLoss := na
        trailingStopActivation := na
        trailingStop := na
        short := false
    trailingStopActivation := close + takeProfitActivationInput*atr
    trailingStopOffset := (trailingStopActivation * trailingStopInput/100) / syminfo.mintick
    stopLossAtr = close - 3*atr
    if (stopLossAtr/close) - 1 < -maxSl/100 
        stopLoss := close*(1-maxSl/100)
    else
        stopLoss := stopLossAtr 
    long := true
    qty = cashOrder/close
    strategy.entry("Long", strategy.long, qty)
    strategy.exit("Exit Long", "Long", stop = stopLoss, trail_price = trailingStopActivation,
                 trail_offset = trailingStopOffset)


//--------------------------------SHORT CONDITION-------------------------------//

if rsi <= 40 and roc_ma >= rocMovAverShortSignalValue and inRange and not short
    if long
        bufferTrailingStopDrawing := false
        stopLoss := na
        trailingStopActivation := na
        trailingStop := na
        long := false
    trailingStopActivation := close - takeProfitActivationInput*atr
    trailingStopOffset := (trailingStopActivation * trailingStopInput/100) / syminfo.mintick
    stopLossAtr = close + 3*atr
    if (stopLossAtr/close) - 1 > maxSl/100
        stopLoss := close*(1+maxSl/100)
    else
        stopLoss := stopLossAtr
    short := true
    qty = cashOrder/close
    strategy.entry("Short", strategy.short, qty)
    strategy.exit("Exit Short", "Short", stop = stopLoss, trail_price = trailingStopActivation,
                 trail_offset = trailingStopOffset)


//--------------------------------PLOTTING ELEMENT---------------------------------//

// Plotting of element in the graph
plotchar(rsi, "RSI", "", location.top, color.rgb(0, 214, 243))
plot(ma, "MA", color.rgb(219, 219, 18))
plotchar(roc_ma, "ROC MA", "", location.top, color=color.orange)
// Visualizer trailing stop and stop loss movement
plot(stopLoss, "SL", color.red, 3, plot.style_linebr)
plot(trailingStopActivation, "Trigger Trail", color.green, 3, plot.style_linebr)
plot(trailingStop, "Trailing Stop",  color.blue, 3, plot.style_linebr)

```