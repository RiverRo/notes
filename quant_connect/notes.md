# Quant Connect Notes

> [Main Table of Contents](../README.md)

## References

[Complete API Reference](https://www.lean.io/docs/v2/lean-engine/class-reference/index.html)  
[Documentation](https://www.quantconnect.com/docs/v2/)

## On This Page

- Pitfalls / Warnings / Solutions
- Important Top-Level Methods
- Security vs ActiveSecurities
- Ticks and Bars - Available Resolution
- Consolidators
- Indicators
- Benchmarks
- Symbols vs Tickers
  - Create symbols
  - Deserialize a Symbol hash
  - Create symbol from industry standard IDs
  - Ticker changes are tracked by security identifiers
- Examples
- Write code for different environments
- Algorithm development concepts

## Pitfalls / Warnings / Solutions

- Universe Selection Data (Coarse and Fine Fundamental Data)
  - [The live coarse and fine fundamental data used in dynamic universe selection arrives at 7 AM Eastern Time (ET)](https://www.quantconnect.com/docs/v2/writing-algorithms/universes/equity)
  - In backtests coarse fundamental gets triggered at midnight UTC (~7pm eastern the day before)
  - TODO: For premarket trading this timing will not do. Will need to rely on external data (e.g. Alpaca snapshots) and use a [pre-built](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/universe-selection/key-concepts) schedule based Universe Selection class to subclass. BottomBreak Strategy is one implementation of this solution.
- Gotcha when removing securities from `self.ActiveSecurities` with `RemoveSecurity()`
  - The function removes the security with the specified symbol from ActiveSecurities dictionary BUT doesn't do so immediately. The symbol gets moved to pending for removal list for at least one iteration, which means I would have to track if an iteration has passed or not before opening a new position to avoid confusion. Much less effort in tracking activeStocks set myself as proposed in the tutorial.
  - [Tutorial that introduces the gotcha and the solution](https://www.quantconnect.com/learning/task/224/Dynamic-Universes)
- Efficiency in code
  - Instead of continuously accessing dictionaries, access the dictionary object once, save to a ref then access the properties off the ref
    ```python
    d['key']['prop1']
    d['key']['prop2']
    # Do this instead
    ref = d['key']
    ref['prop1']
    ref['prop2']
    ```
- Consolidators
  - Remove manually created consolidator updates from SubscriptionManager once a security leaves the universe so it doesn't slow down algo
- Reality Modeling
  - Reality modeling is at portfoliio or security level
  - Reality Modeling illiquid assets
    - Existing reality models assumed highly liquid assets
    - Create custom models for high frequency and/or illiquid assets
- Checking Time / Scheduling
  - Using a Schedule Event is preferred over manually checking time
  - If I must manually check times, use ranges, NEVER exact times (e.g. 9:30:10 < time < 9:30:20)
- Trading / Order Errors
  - [Comprehensive List of errors](https://www.quantconnect.com/docs/v2/writing-algorithms/trading-and-orders/order-errors)
- Look-ahead bias manifestation
  - Data Aggregation / Sync with other data. Other data providers commonly timestamp their bars to the start time of the bar.
    - This can cause one-off errors: 1) when you import the data into QC 2) compare indicator values across different platform
  - Bars which are periods are not emitted until `Endtime` in backtest to avoid look-ahead bias
    - Daily bars are emitted at 00:00 midnight of next day to avoid look-ahead bias
  - Use raw (big price jumps in split and dividends) vs adjusted data normalization mode TODO: run strat both ways
  - Choose assets that performed well in backtests instead of dynamic universe selection
- Manage stale fills on backtests for illiquid assets
  - Use custom fill model
- Overfitting manifestations
  - Data dredging: only paying attention to statistically significant results
  - Hypertuning parameters
  - Too many parameters
  - Use stale testing data
  - [Remove outliers](https://www.quantconnect.com/docs/v2/writing-algorithms/key-concepts/research-guide#09-Outliers) with Winsorizaiton method, IQR method, Factor ranking method
- [Non-deterministic State From Algorithm Restarts (LIVE vs BT)](https://www.quantconnect.com/docs/v2/writing-algorithms/live-trading/reconciliation)
  - If you stop and redeploy your live trading algorithm, it needs to restart in a stateful way or else deviations can occur between backtesting and live trading. To avoid issues, redeploy your algorithm in a stateful way using the SetWarmUp and History methods. Furthermore, use the ObjectStore to save state information between your live trading deployments
- [Bug in Risk Management model and stop loss logic](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/alpha/key-concepts)
  - TLDR: Add stop loss logic in Alpha Model instead of Risk Management model
  - In some cases, if you add a Risk Management model that uses stop loss logic, the Risk Management model generates PortfolioTarget objects with a 0 quantity to make the Execution model liquidate your positions, but then the Portfolio Construction model generates PortfolioTarget objects with a non-zero quantity to make the Execution model re-enter the same position. This issue can occur if your Portfolio Construction model rebalances and your Alpha model still has an active insight for the liquidated securities. To avoid this issue, add the stop loss order logic to the Alpha model. When the stop loss is hit, emit a flat insight for the security. Note that this is a workaround, but it violates the separation of concerns principle since the Alpha Model shouldn't react to open positions
- [Split Adjustment of Indicators and Corporate Events (LIVE vs BT)](https://www.quantconnect.com/docs/v2/writing-algorithms/live-trading/reconciliation)
  - Backtests use adjusted price data by default. Therefore, if you don't change the data normalization mode, the indicators in your backtests are updated with adjusted price data. In contrast, if a split or dividend occurs in live trading, your indicators will temporarily contain price data from before the corporate event and price data from after the corporate event. If this occurs, your indicators will produce different signals in your backtests compared to your live trading deployment. To avoid issues, reset and warm up your indicators when your algorithm receives a corporate event.
- [Live Data Delays (LIVE vs BT)](https://www.quantconnect.com/docs/v2/writing-algorithms/live-trading/reconciliation)
  - In backtests, your algorithm receives data at perfect timing. If you request minute resolution data, your algorithm receives the bars at the top of each minute. In live trading, bars have a slight delay, so you may receive them milliseconds after the top of each minute. Take the following scenario:
    1. You subscribe to minute resolution data and and scheduled event for 10:00 AM
    2. The Scheduled Event checks the current asset price  
       In live trading, the Scheduled Event executes at exactly 10:00 AM but your algorithm may receive the 9:59-10:00 AM bar at 10:00:00.01 AM. Therefore, when you check the price in the Scheduled Event, the price from the 9:58-9:59 AM bar is the latest price. In backtesting, the Scheduled Event gets the price from the 9:59-10:00 AM bar since your algorithm receives the bar at perfect timing.

## Important Event Handlers

- Initialize()
- PostInitialize
- OnWarmUpFinished(self)
- OnData(self, Slice) _Non-Framework_
- Update(self, QCAlgorithm, Slice) _Framework_
- OnSecuritiesChanged(self, changes)
- Updated()
- Schedule.On()
- Schedule.Remove()
- OnOrderEvent(self, OrderEvent)
- OnMarginCall(self, List[SubmitOrderRequest])
- OnMarginCallWarnings(self)
- OnEndOfDays
- OnEndOfAlgorithm(self)

### Initialize()

- Called once
- Define settings here
- Set benchmarks
- Create indicators
- Warmup indicators
- Plot values with PlotIndicator

  | List of common methods used here      | Description                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                      |
  | ------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
  | UniverseSettings.[property] = [value] | \*Set before AddUniverse<br>Usually first line<br>GETTER/SETTER                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
  | Settings.[property] = [value]         | Set algorithm settings<br>\*NOTE: for FreePortfolioValue: To successfully update the FreePortfolioValue, you must update it after the Initialize method. In PostInitialize                                                                                                                                                                                                                                                                                                                                                       |
  | SetStartDate()                        |
  | SetEndDate()                          |
  | SetTimeZone()                         |
  | SetCash()                             |
  | SetBrokerageModel()                   |
  | SetBenchmark()                        |
  | SetSecurityInitializer(Symbol)        | -Security level reality model setting and data requests<br> _\*WARNING:_ This call Overwrite reality model set by brokerage instead look into extending the default security initalizer model [Reference](https://www.quantconnect.com/docs/v2/writing-algorithms/initialization#07-Set-Security-Initializer)                                                                                                                                                                                                                    |
  | AddUniverse()                         | -Add a universe (basket) of securities to algo through a filtering process (filter for stocks via coarse and fine fundamental built-indata OR custom data) and sub to the data<br>-Can add multiple universes                                                                                                                                                                                                                                                                                                                    |
  | AddEquity()                           | -Add equity to algo and subscribe to built-in data<br>-Tracked in Securities dictionary keyed by Symbol                                                                                                                                                                                                                                                                                                                                                                                                                          |
  | AddData()                             | Add custom data                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                  |
  | PlotIndicator()                       | Plot all values of given indicators (up to four) as the indicator updates                                                                                                                                                                                                                                                                                                                                                                                                                                                        |
  | SetWarmUp(period)                     | -If you create indicators at the beginning of your algorithm, you can set an algorithm warm-up period to warm up the indicators. When you set an algorithm warm-up period, the engine pumps data in and automatically updates all the indicators from before the start date of the algorithm. To ensure that all the indicators are ready after the algorithm warm-up period, choose a lookback period that contains sufficient data.<br>-Warmup indicators<br>-Populate hx data arrays<br>-Pump data into algo before StartDate |

## Security vs ActiveSecurities

- Security holds all securities for statistics purposes
- ActiveSecurities only holds securities in current universe"

## Ticks and Bars - Available Resolution

| Type                     | Resolution                  |
| ------------------------ | --------------------------- |
| Trade Tick<br>Quote Tick | Tick                        |
| TradeBar                 | Second, Minute, Hour, Daily |
| QuoteBar                 | Second, Minute              |

## Consolidators

- consolidators == custom bars

## Indicators

- Standard deviation indicators can be used as a measure of volatility (TODO: come back to this)

## Benchmarks

- Bemchmark can be set to custom data type (class)

## Symbols vs Tickers

- Use Symbol objects whenever possible instead of ticker strings
  - The Symbol object is specific whereas with a ticker string, the algo is guessing on certain properties like Market, SecurityType, etc.

### Create symbols

```python
Symbol.Create()  # Create is a convenience method to Symbol for most security types
```

### Deserialize a Symbol hash

```python
Symbol("GOOCV VP83T1ZUHROL")
```

### Create symbol from industry standard IDs

- From e.g. cupsi, sedol, isin, figi

### Ticker changes are tracked by security identifiers [Reference](https://www.quantconnect.com/docs/v2/writing-algorithms/key-concepts/security-identifiers)

## Examples

| Example                                                                                                                                                     | Description                      |
| ----------------------------------------------------------------------------------------------------------------------------------------------------------- | -------------------------------- |
| [CustomDataUsingMapFileRegressionAlgorithm](https://github.com/QuantConnect/Lean/blob/master/Algorithm.Python/CustomDataUsingMapFileRegressionAlgorithm.py) | How to handle ticker name change |

## Write code for different environments

- Check for the environment

  ```python
  platform.node()
  ```

## Algorithm development concepts

- Hypothesis driven: Develop strategies around central idea or hypothesis
- Data Mining driven: Follow statistical anomalies and eliminate when edge is gone
- Each Security subscription uses 5MB of RAM
- Tick feed doesn't include OTC

## Document Styles

<style>
h1 {
  color: DeepSkyBlue;
}
h2 {
color: yellow;
}
h3 {
  color: LightCoral;
}
</style>
