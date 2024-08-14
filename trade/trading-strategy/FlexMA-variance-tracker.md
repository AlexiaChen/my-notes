The FlexiMA Variance Tracker by PresentTrading introduces a novel approach to technical trading strategies. Unlike traditional methods, it calculates deviations between a chosen indicator source (such as price or average) and a moving average with a variable length. This flexibility is achieved through a unique combination of a starting factor and an increment factor, allowing the moving average to adapt dynamically within a specified range. This strategy provides a more responsive and nuanced view of market trends, setting it apart from standard trading methodologies.

 Strategy, How It Works: Detailed Explanation  
  
The FlexiMA Variance Tracker, developed by PresentTrading, stands at the forefront of trading strategies, distinguished by its adaptive and multifaceted approach to market analysis. This strategy intricately weaves various technical elements to construct a comprehensive trading logic. Here's an in-depth professional breakdown:  
  
🔶Foundation on Variable-Length Moving Averages:  
  
Central to this strategy is the concept of variable-length Moving Averages (MAs). Unlike traditional MAs with a fixed period, this strategy dynamically adjusts the length of the MA based on a starting factor and an incremental factor. This approach allows the strategy to adapt to market volatility and trend strength more effectively.

Each MA iteration offers a distinct temporal perspective, capturing short-term price movements to long-term trends. This aggregation of various time frames provides a richer and more nuanced market analysis, essential for making informed trading decisions.  
  
  
🔶Deviation Analysis and Normalization:  
  
The strategy calculates deviations of the price (or the chosen indicator source) from each of these MAs. These deviations are pivotal in identifying the immediate market direction relative to the average trend captured by each MA.  
  
To standardize these deviations for comparability, they undergo a normalization process. The choice of normalization method (Max-Min or Absolute Sum) can significantly influence the interpretation of market conditions, offering distinct insights into price movements and trend strength.  
  
🔹Normalization: Absolute Sum

Composite Oscillator Construction:  
  
A composite oscillator is derived from the median of these normalized deviations. The median serves as a balanced and robust central trend indicator, minimizing the impact of outliers and market noise.  
Additionally, the standard deviation of these deviations is computed, providing a measure of market volatility. This volatility indicator is crucial for assessing market risk and can guide traders in setting appropriate stop-loss and take-profit levels.  
  
🔶Integration with SuperTrend Indicator:  
  
The FlexiMA strategy integrates the SuperTrend indicator, renowned for its effectiveness in identifying trend direction and reversals. The SuperTrend's incorporation enhances the strategy's ability to filter out false signals and confirm genuine market trends.  
* The SuperTrend Toolkit is made by [@EliCobra](https://www.tradingview.com/u/EliCobra/)  
This combination of the variable-length MA oscillator with the SuperTrend indicator forms a potent duo, offering traders a dual-confirmation mechanism for trade signals.  
  
🔹Supertrend's incorporation

Strategic Trade Signal Generation:  
  
Trade signals are generated when there is a confluence between the composite oscillator and the SuperTrend indicator. For example, a long position signal might be considered when the oscillator suggests an uptrend, and the SuperTrend flips to bullish.  
The strategy's parameters are fully customizable, enabling traders to tailor the signal generation process to their specific trading style, risk tolerance, and market conditions.  
  
█ Usage  
  
To effectively employ the FlexiMA Variance Tracker strategy:  
Traders should set their desired trade direction and fine-tune the starting and increment factors according to their market analysis and risk tolerance.  
Indicator Length: 5

The strategy is suitable for a wide range of markets and can be adapted to different time frames, making it a versatile tool for various trading scenarios.  
  
█ Default Settings Impact on Performance: FlexiMA Variance Tracker  
  
1. Trade Direction (Configurable: Long, Short, Both): Determines trade types. 'Long' for buying, 'Short' for selling, 'Both' adapts to market trends.  
  
2. Indicator Source: HLC3: Balances market sentiment by considering high, low, and close, providing comprehensive period analysis.  
  
4. Indicator Length (Default: 10): Baseline for moving averages. Shorter lengths increase responsiveness but add noise, while longer lengths favor trends.  
  
5. Starting and Increment Factor (Default: 1.0): Adjusts MA lengths range. Higher values capture broad market dynamics, lower values focus analysis.  
  
6. Normalization Method (Options: None, Max-Min, Absolute Sum): Standardizes deviations. 'None' for raw deviations, 'Max-Min' for relative scaling, 'Absolute Sum' emphasizes relative strength.  
  
7. SuperTrend Settings (ATR Length: 10, Multiplier: 15.0): Influences indicator sensitivity. Short ATR or high multiplier for short-term, long ATR or low multiplier for long-term trends.  
  
8. Additional Settings (Mesh Style, Color Customization): Enhances visual clarity. Mesh style for detailed deviation view, colors for quick market condition identification.

```cpp
// This source code is subject to the terms of the Mozilla Public License 2.0 at https://mozilla.org/MPL/2.0/
// © PresentTrading

// The core idea of the FlexiMA Variance Tracker is to calculate a series of deviations between the indicator source 
// (like a price or average) and a moving average, where the length of the moving average is not constant but varies within 
// a defined range. This variation is controlled by the starting factor and an increment factor, which together determine 
// the length of the moving average in each iteration.

//@version=5
strategy("FlexiMA Variance Tracker - Strategy [presentTrading]",shorttitle = "FlexiMA-VT - Strategy [presentTrading]", overlay = false, precision=3, commission_value= 0.1, commission_type=strategy.commission.percent, slippage= 1, currency=currency.USD, default_qty_type = strategy.percent_of_equity, default_qty_value = 10, initial_capital= 10000)

// Trading settings
tradeDirection = input.string("Both", title="Trade Direction", options=["Long", "Short", "Both"])

// Input parameters
indicatorSource = input.source(hlc3, title="Indicator Source")
indicatorLength = input.int(10, minval = 2, title="Indicator Length")
startingFactor = input.float(1.0, title="Starting Factor", minval = 0)
incrementFactor_1 = input.float(1.0, minval = 0, step = .10, title="Increment Factor")
normalizeMethod = input.string('None', options = ['None', 'Max-Min', 'Absolute Sum'], title="Normalization Method")


// This function normalizes an array of deviations according to the specified normalization method.
normalize(diffs, den, normMethod) =>
    // Initialize an array to hold the normalized deviations.
    normalizedDiffs = array.new_float(0)
    // Loop through each deviation value to apply normalization.
    for val in diffs
        // Determine the normalization value based on the chosen method.
        normValue = switch normMethod
            // "Max-Min" normalization scales the deviation based on the range of the array values.
            "Max-Min" => (val - array.min(diffs)) / array.range(diffs)
            // "Absolute Sum" normalization scales the deviation based on the sum of absolute values.
            "Absolute Sum" => val / den
            // Default case, no normalization is applied, so the value remains as is.
            => val
        // Add the normalized value to the array.
        array.push(normalizedDiffs, normValue)
    // Return the array of normalized values.
    normalizedDiffs

// This function calculates the standard deviation of an array of values.
customStdev(values) =>
    // Calculate the mean (average) of the values.
    mean = array.sum(values) / array.size(values)
    // Initialize an array to hold the squared differences from the mean.
    squareDiffs = array.new_float(0)
    // Loop through each value to calculate the squared difference from the mean.
    for val in values
        // Push the squared difference to the array.
        array.push(squareDiffs, math.pow(val - mean, 2))
    // Calculate the variance as the average of the squared differences.
    variance = array.sum(squareDiffs) / (array.size(values) - 1)
    // Return the square root of the variance, which is the standard deviation.
    math.sqrt(variance)


// This function calculates an oscillator based on a moving average (MA) where the length of the MA varies.
// Each iteration applies a different length to the MA calculation by modifying the length factor.
variableLengthMAOscillator(source, length, startFactor, incFactor, normMethod) =>
    diffs = array.new_float(0)  // Array to store the deviations of the source from the MA.
    den = 0.0  // A variable to accumulate the sum of absolute deviations, used for normalization.
    factor = startFactor  // The starting factor for the MA length variation.

    // Loop to calculate the MA and deviations for different lengths.
    for i = 0 to 19
        maLength = math.round(length * factor)  // Adjust the MA length by the current factor.
        maValue = ta.sma(source, maLength)  // Calculate the Simple Moving Average (SMA) for the given length.
        deviation = source - maValue  // Compute the deviation from the MA.
        array.push(diffs, deviation)  // Store the deviation in the array.
        den += math.abs(deviation)  // Add the absolute deviation to the denominator for normalization.
        factor += incFactor  // Increment the factor for the next MA length adjustment.

    // Normalize the deviations array using the chosen method and calculate the median and standard deviation.
    normalizedDiffs = normalize(diffs, den, normMethod)
    medianValue = array.median(normalizedDiffs)  // Find the median of the normalized deviations.
    stdevValue = customStdev(normalizedDiffs)  // Calculate the standard deviation of the normalized deviations.
    [medianValue, stdevValue, diffs, den]  // Return the median, standard deviation, and other variables for further use.

// Calculate the Variable Length MA Oscillator
// Access the SuperTrend Polyfactor Oscillator values
[medianValue, stdevValue,diffs,den] = variableLengthMAOscillator(indicatorSource, indicatorLength, startingFactor, incrementFactor_1, normalizeMethod)

//Style
mesh = input(true, inline = 'mesh', group = 'Style')
upCss = input.color(color.new(#089981, 90), '', inline = 'mesh', group = 'Style')
dnCss = input.color(color.new(#f23645, 90), '', inline = 'mesh', group = 'Style')

//-----------------------------------------------------------------------------}
//Plots
//-----------------------------------------------------------------------------{
// This function normalizes an array of deviations according to the specified normalization method.

norm(value, diffs, den)=>
    normalizeMethod = switch normalizeMethod
        'Max-Min' => (value - diffs.min()) / diffs.range()
        'Absolute Sum' => value / den
        => value

norm(array.get(diffs, 0),diffs,den)

// Plot each individual deviation using the array.get function
plot(mesh ? norm(array.get(diffs, 0),diffs,den) : na, "Deviation 1", color=array.get(diffs, 0) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 1),diffs,den) : na, "Deviation 2", color=array.get(diffs, 1) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 2),diffs,den) : na, "Deviation 3", color=array.get(diffs, 2) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 3),diffs,den) : na, "Deviation 4", color=array.get(diffs, 3) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 4),diffs,den) : na, "Deviation 5", color=array.get(diffs, 4) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 5),diffs,den) : na, "Deviation 6", color=array.get(diffs, 5) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 6),diffs,den) : na, "Deviation 7", color=array.get(diffs, 6) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 7),diffs,den) : na, "Deviation 8", color=array.get(diffs, 7) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 8),diffs,den) : na, "Deviation 9", color=array.get(diffs, 8) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 9),diffs,den) : na, "Deviation 10", color=array.get(diffs, 9) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 10),diffs,den) : na, "Deviation 11", color=array.get(diffs, 10) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 11),diffs,den) : na, "Deviation 12", color=array.get(diffs, 11) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 12),diffs,den) : na, "Deviation 13", color=array.get(diffs, 12) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 13),diffs,den) : na, "Deviation 14", color=array.get(diffs, 13) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 14),diffs,den) : na, "Deviation 15", color=array.get(diffs, 14) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 15),diffs,den) : na, "Deviation 16", color=array.get(diffs, 15) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 16),diffs,den) : na, "Deviation 17", color=array.get(diffs, 16) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 17),diffs,den) : na, "Deviation 18", color=array.get(diffs, 17) > 0 ? upCss : dnCss, style=plot.style_histogram)
plot(mesh ? norm(array.get(diffs, 18),diffs,den) : na, "Deviation 19", color=array.get(diffs, 18) > 0 ? upCss : dnCss, style=plot.style_histogram)

//Median
plot(medianValue, 'Median', medianValue > (normalizeMethod == 'Max-Min' ? .5 : 0) ? #90bff9 : #ffcc80)

//Stdev Area
up = plot(stdevValue, color = na, editable = false)
dn = plot(-stdevValue, color = na, editable = false)
fill(up, dn, color.new(#90bff9, 80), title = 'Stdev Area')

//-----------------------------------------------------------------------------}


type bar
    float o = na
    float h = na
    float l = na
    float c = na

type supertrend
    float s = na
    int   d = na

method src(bar b, simple string src) =>
    float x = switch src
        'oc2'   => math.avg(b.o, b.c          )
        'hl2'   => math.avg(b.h, b.l          )
        'hlc3'  => math.avg(b.h, b.l, b.c     )
        'ohlc4' => math.avg(b.o, b.h, b.l, b.c)
        'hlcc4' => math.avg(b.h, b.l, b.c, b.c)

    x

method atr(bar b, simple int len) =>
    float tr = 
         na(b.h[1]) ? 
         b.h - b.l  : 
         math.max(
             math.max(
                 b.h - b.l, 
                 math.abs(b.h - b.c[1])), 
             math.abs    (b.l - b.c[1]))

    len == 1 ? tr : ta.rma(tr, len)

method st(bar b, simple float factor, simple int len) =>
	float atr = b.atr( len )
	float up  = b.src('hl2') + factor * atr
    up       := up < nz(up[1]) or b.c[1] > nz(up[1]) ? up : nz(up[1])
    float dn  = b.src('hl2') - factor * atr
    dn       := dn > nz(dn[1]) or b.c[1] < nz(dn[1]) ? dn : nz(dn[1])
	
    float st  = na
    int   dir = na
    dir := switch
        na(atr[1])         => 1
        st[1] == nz(up[1]) => dir := b.c > up ? -1 : +1
        =>                    dir := b.c < dn ? +1 : -1
	st  :=                    dir == -1       ? dn : up
	
    supertrend.new(st, dir)


var string tp = 'Choose an indicator source of which to base the SuperTrend on.'
var string g1 = "SuperTrend Settings", var string gu = "UI Options"
src  = medianValue
len  = input.int   (10       , "Length"            ,             inline = '1', group = g1)
mlt  = input.float (15.       , "Factor"            , 1, 20, 0.5, inline = '1', group = g1)
clbl = input.bool  (true    , "Contrarian Signals",                           group = gu)
colu = input.color (#008cff, "Bull Color"        ,                           group = gu)
cold = input.color (#ff4800, "Bear Color"        ,                           group = gu)


bar        b  = bar.new(
       nz(src[1])               ,
       math.max(nz(src[1]), src),
       math.min(nz(src[1]), src),
       src                      )
float      tr = b  .atr(     len)
supertrend st = b  .st (mlt, len)


color cst = switch st.d
    +1 => cold
    -1 => colu
    
t = plot(st.d > 0 ? st.s : na, 'Bear ST'  , cst           , 1, plot.style_linebr)
d = plot(st.d < 0 ? st.s : na, 'Bull ST'  , cst           , 1, plot.style_linebr)
i = plot(st.d                , 'Direction', display =          display.none     )
c = plot(b.src('oc2')        , 'Filler'   , display =          display.none     )

fill(t, c, color.new(cold, 90))
fill(d, c, color.new(colu, 90))

bool scon = clbl and math.sign(ta.change(st.d)) ==  1 ? true : false
bool bcon = clbl and math.sign(ta.change(st.d)) == -1 ? true : false

plotshape(scon ? st.s + tr / 3 : na, 'Sell Signal', shape.labeldown, location.absolute, color.new(cold, 60), 0, '𝓢', chart.fg_color)
plotshape(bcon ? st.s - tr / 3 : na, 'Buy Signal' , shape.labelup  , location.absolute, color.new(colu, 60), 0, '𝓑', chart.fg_color)



// Entry conditions
shouldEnterLong = st.d < 0
shouldEnterShort = st.d > 0

// Strategy logic
if  (tradeDirection == "Long" or tradeDirection == "Both")
    if shouldEnterLong
        strategy.entry("Long Entry", strategy.long)

if (tradeDirection == "Short" or tradeDirection == "Both")
    if shouldEnterShort
        strategy.entry("Short Entry", strategy.short)

if  (tradeDirection == "Long" or tradeDirection == "Both")
    if shouldEnterShort
        strategy.close("Long Entry")

if (tradeDirection == "Short" or tradeDirection == "Both")
    if shouldEnterLong
        strategy.close("Short Entry")

```