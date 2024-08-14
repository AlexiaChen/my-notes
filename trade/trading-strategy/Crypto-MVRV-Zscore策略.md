
The "Crypto Valuation Extremes: MVRV ZScore - Strategy " represents a cutting-edge approach to cryptocurrency trading, leveraging the Market Value to Realized Value (MVRV) Z-Score. This metric is pivotal for identifying overvalued or undervalued conditions in the crypto market, particularly Bitcoin. It assesses the current market valuation against the realized capitalization, providing insights that are not apparent through conventional analysis.

Strategy, How It Works: Detailed Explanation  
  
The strategy leverages the Market Value to Realized Value (MVRV) Z-Score, specifically designed for cryptocurrencies, with a focus on Bitcoin. This metric is crucial for determining whether Bitcoin is currently undervalued or overvalued compared to its historical 'realized' price. Below is an in-depth explanation of the strategy's components and calculations.  
  
🔶Conceptual Foundation  
  
- Market Capitalization (MC): This represents the total dollar market value of Bitcoin's circulating supply. It is calculated as the current price of Bitcoin multiplied by the number of coins in circulation.  
- Realized Capitalization (RC): Unlike MC, which values all coins at the current market price, RC is computed by valuing each coin at the price it was last moved or traded. Essentially, it is a summation of the value of all bitcoins, priced at the time they were last transacted.  
- MVRV Ratio: This ratio is derived by dividing the Market Capitalization by the Realized Capitalization (The ratio of MC to RC (MVRV Ratio = MC / RC)). A ratio greater than 1 indicates that the current price is higher than the average price at which all bitcoins were purchased, suggesting potential overvaluation. Conversely, a ratio below 1 suggests undervaluation.  
  
🔶 MVRV Z-Score Calculation  
  
The Z-Score is a statistical measure that indicates the number of standard deviations an element is from the mean. For this strategy, the MVRV Z-Score is calculated as follows:  
  
MVRV Z-Score = (MC - RC) / Standard Deviation of (MC - RC)  
  
This formula quantifies Bitcoin's deviation from its 'normal' valuation range, offering insights into market sentiment and potential price reversals.  
  
🔶 Spread Z-Score for Trading Signals  
  
The strategy refines this approach by calculating a 'spread Z-Score', which adjusts the MVRV Z-Score over a specific period (default: 252 days). This is done to smooth out short-term market volatility and focus on longer-term valuation trends. The spread Z-Score is calculated as follows:  
  
Spread Z-Score = (Market Z-Score - MVVR Ratio - SMA of Spread) / Standard Deviation of Spread  
  
Where:  
- SMA of Spread is the simple moving average of the spread over the specified period.  
- Spread refers to the difference between the Market Z-Score and the MVRV Ratio.  
  
🔶 Trading Signals  
  
- Long Entry Condition: A long (buy) signal is generated when the spread Z-Score crosses above the long entry threshold, indicating that Bitcoin is potentially undervalued.  
- Short Entry Condition: A short (sell) signal is triggered when the spread Z-Score falls below the short entry threshold, suggesting overvaluation.  
  
These conditions are based on the premise that extreme deviations from the mean (as indicated by the Z-Score) are likely to revert to the mean over time, presenting opportunities for strategic entry and exit points.  
  
█ Practical Application  
  
Traders use these signals to make informed decisions about opening or closing positions in the Bitcoin market. By quantifying market valuation extremes, the strategy aims to capitalize on the cyclical nature of price movements, identifying high-probability entry and exit points based on historical valuation norms.  
  
█ Trade Direction  
  
A unique feature of this strategy is its configurable trade direction. Users can specify their preference for engaging in long positions, short positions, or both. This flexibility allows traders to tailor the strategy according to their risk tolerance, market outlook, or trading style, making it adaptable to various market conditions and trader objectives.  
  
█ Usage  
  
To implement this strategy, traders should first adjust the input parameters to align with their trading preferences and risk management practices. These parameters include the trade direction, Z-Score calculation period, and the thresholds for long and short entries. Once configured, the strategy automatically generates trading signals based on the calculated spread Z-Score, providing clear indications for potential entry and exit points.  
  
It is advisable for traders to backtest the strategy under different market conditions to validate its effectiveness and adjust the settings as necessary. Continuous monitoring and adjustment are crucial, as market dynamics evolve over time.  
  
█ Default Settings  
  
- Trade Direction: Both (Allows for both long and short positions)  
- Z-Score Calculation Period: 252 days (Approximately one trading year, capturing a comprehensive market cycle)  
- Long Entry Threshold: 0.382 (Indicative of moderate undervaluation)  
- Short Entry Threshold: -0.382 (Signifies moderate overvaluation)  
  
These default settings are designed to balance sensitivity to market valuation extremes with a pragmatic approach to trade execution. They aim to filter out noise and focus on significant market movements, providing a solid foundation for both new and experienced traders looking to exploit the unique insights offered by the MVRV Z-Score in the cryptocurrency market.


```cpp
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © PresentTrading

// The strategy hinges on the Market Value to Realized Value (MVRV) Z-Score, a financial metric tailored for cryptocurrencies, specifically Bitcoin in this context. 
// This score helps identify whether Bitcoin is undervalued or overvalued relative to its 'realized' price. 

//@version=5
strategy("Crypto MVRV ZScore - Strategy [PresentTrading]",shorttitle = "Crypto MVRV ZScore - Strategy [PresentTrading]" , overlay = false, precision=3, 
 commission_value= 0.1, commission_type=strategy.commission.percent, slippage= 1, 
  currency=currency.USD, default_qty_type = strategy.percent_of_equity, default_qty_value = 10, initial_capital= 10000)

// The setting of input
tradeDirection = input.string(title="Trade Direction", defval="Both", options=["Both", "Long", "Short"])
zscoreCalculationPeriod = input.int(252, "Z-Score Calculation Period")
longEntryThreshold = input.float(0.382, "Long Entry Threshold")
shortEntryThreshold = input.float(-0.382, "Short Entry Threshold")

// Market Capitalization Data Queries
bitcoinMarketCap = request.security("GLASSNODE:BTC_MARKETCAP", "D", close)
realMarketCap = request.security("COINMETRICS:BTC_MARKETCAPREAL", "D", close)

// Initialize variables for calculation
var marketCapArray = array.new<float>()
var mvrvRatio = 0.0
var marketZscore = 0.0

// Updating market cap data and calculating MVRV and Z-Score
array.push(marketCapArray, bitcoinMarketCap)
marketCapStdDev = array.stdev(marketCapArray)
mvrvRatio := bitcoinMarketCap / realMarketCap
marketZscore := (bitcoinMarketCap - realMarketCap) / marketCapStdDev


// Calculating the spread Z-Score for trading signals
spreadZScoreRaw = (marketZscore - mvrvRatio - ta.sma(marketZscore - mvrvRatio, zscoreCalculationPeriod)) / ta.stdev(marketZscore - mvrvRatio, zscoreCalculationPeriod)
spreadZScore = ta.sma(spreadZScoreRaw,2)

// Horizontal lines for visual reference points
hline(3, color=color.blue)
hline(longEntryThreshold, color=color.green)
hline(0, color=color.white)
hline(shortEntryThreshold, color=color.red)
hline(-3, color=color.blue)
plot(spreadZScore, "Market Z-Score", color=color.yellow, linewidth=1)

// Defining trade entry and exit conditions based on Z-Score spread
longEntryCondition = ta.crossover(spreadZScore, longEntryThreshold)
longExitCondition = ta.crossunder(spreadZScore, shortEntryThreshold)
shortEntryCondition = ta.crossunder(spreadZScore, shortEntryThreshold)
shortExitCondition = ta.crossover(spreadZScore, longEntryThreshold)


// Executing trades based on defined conditions and user input
if longEntryCondition and (tradeDirection == "Long" or tradeDirection == "Both")
    strategy.entry("Long", strategy.long)
if longExitCondition and (tradeDirection == "Long" or tradeDirection == "Both")
    strategy.close("Long")

if shortEntryCondition and (tradeDirection == "Short" or tradeDirection == "Both")
    strategy.entry("Short", strategy.short)
if shortExitCondition and (tradeDirection == "Short" or tradeDirection == "Both")
    strategy.close("Short")
```