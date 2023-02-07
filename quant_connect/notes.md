# Quant Connect Notes

> [Main Table of Contents](../README.md)

## References

[Complete API Reference](https://www.lean.io/docs/v2/lean-engine/class-reference/index.html)  
[Documentation](https://www.quantconnect.com/docs/v2/)

## On This Page
- CLI Installation in WSL & conda environment
- Pitfalls / Warnings / Solutions
- Define Algorithm Parameters
- Important Event Handlers
- Schedule Event Handlers / Methods
- Security vs ActiveSecurities
- Ticks and Bars - Available Resolution
- Warm Up RollingWindows, Indicators
- Rolling Windows
  - Rolling Window DataTypes
  - Rolling Window Methods
  - Rolling Window Other Operations
  - Rolling Window COMMON USE CASE
- Consolidators
- Indicators
  - Shortcut Helpers
  - Non-Shortcut Helpers
  - Non-Shortcut Helpers Use Cases
  - Custom Indicators
  - Indicator Extensions
- Benchmarks
- Custom Data
- Symbols vs Tickers
  - Create symbols
  - Deserialize a Symbol hash
  - Create symbol from industry standard IDs
  - Ticker changes are tracked by security identifiers
- Write code for different environments
- Algorithm development concepts
- Algorithm Performance and Backtest Results
## CLI Installation
  1. Install Docker Desktop on Windows + [all VS Code Docker extensions + installation instructions](https://learn.microsoft.com/en-us/windows/wsl/tutorials/wsl-containers) and Docker Desktop is open
    - GOTCHA: After running `docker run hello-world` and it looks like it hangs with the message "Unable to find image 'hello-world:latest' locally" just give it a few minutes for it to pull from remote source and actually run it.
  3. In WSL terminal: `sudo apt-get install python3-tk` [Ref](https://www.quantconnect.com/docs/v2/lean-cli/installation/installing-lean-cli)
  4. `conda create -n qc python=3.8` then `conda activate qc`
  5. `pip install lean`  
    - If getting `evdev` error, `conda deactivate` (maybe have to run 2+ times to fully get out of conda environments)  
    - Remove the environment: `conda remove -n qc --all`  
    - In WSL terminal: `sudo apt-get install python3-dev build-essential` and go back to step 3
  6. Create empty dir `mkdir quant_connect`
  7. Get API creds by going to QC website, under account settings will send via email userId and token.
  8. `lean login` use the creds here
  9. `lean init`   
    - GOTCHA: This might look like it hangs at "Pulling quantconnect/lean:latest..." message, but it just takes a long while, wait at least a couple minutes before doing quitting

## Pitfalls / Warnings / Solutions

- Universe Selection Data (Coarse and Fine Fundamental Data)
  - [The live coarse and fine fundamental data used in dynamic universe selection arrives at 7 AM Eastern Time (ET)](https://www.quantconnect.com/docs/v2/writing-algorithms/universes/equity)
  - In backtests coarse fundamental gets triggered at midnight UTC (~7pm eastern the day before)
  - TODO: For premarket trading this timing will not do. Will need to rely on external data (e.g. Alpaca snapshots) and use a [pre-built](https://www.quantconnect.com/docs/v2/writing-algorithms/algorithm-framework/universe-selection/key-concepts) schedule based Universe Selection class to subclass. BottomBreak Strategy is one implementation of this solution.
- Gotcha when removing securities from `self.ActiveSecurities` with `RemoveSecurity()` or is it if after `self.Liquidate` a Symbol in the `OnSecuritieschanged` event handler [Tutorial that introduces the gotcha and the solution](https://www.quantconnect.com/learning/task/224/Dynamic-Universes)
  - The function removes the security with the specified symbol from ActiveSecurities dictionary BUT doesn't do so immediately. The symbol gets moved to pending for removal list for at least one iteration, which means I would have to track if an iteration has passed or not before opening a new position to avoid confusion and the bigger probelm which is that if an order is created/submitted before the removal, the stock doesn't get removed from the universe at all. Much less effort in tracking activeStocks set myself as proposed in the tutorial.
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

## Define Algorithm Parameters ([YT ref 21:00](https://www.youtube.com/watch?v=oFvYbfDOJ5c&list=PLtqRgJ_TIq8Y6YG8G-ETIFW_36mvxMLad&index=10))

- Define parameters and use the `self.GetParameter` method to use it in an algo.
- This is useful b/c after deploying an algorithm don't want to touch the source code. But allows access to parameters for future modifications.
- Opens up the option to optimize a strategy based on select parameters

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

## Schedule Event Handlers / Methods

- Schedule Manager Methods

```python
def cb(self):
	pass

# Schedule a method to run every Mon, Fri at noon
mysched = self.Schedule.On(self.DateRules.Every(DayOfWeek.Monday, DayOfWeek.Friday), self.TimeRules.At(12,0), self.cb)

# Remove event from schedule
self.Schedule.Remove(mysched)
```

## Security vs ActiveSecurities

- Security holds all securities for statistics purposes
- ActiveSecurities only holds securities in current universe"

## Ticks and Bars - Available Resolution

| Type                     | Resolution                  |
| ------------------------ | --------------------------- |
| Trade Tick<br>Quote Tick | Tick                        |
| TradeBar                 | Second, Minute, Hour, Daily |
| QuoteBar                 | Second, Minute              |

## Warm Up RollingWindows, Indicators

- WarmUp in `Initialize` with a call to `History` has two advantages
  - RollingWindow, Indicator is immediately avail to use in `OnData` and other methods
  - Don't have to constantly check if RollingWindow, Indicator is full/ready in `OnData`

## Rolling Windows ([Ref](https://www.quantconnect.com/docs/v2/writing-algorithms/historical-data/rolling-window))

- Can store a wide variety of data types
- Data moves from left to right
  - Most recent data avail at window[0]
  - Oldest data avail at window[-1]
- When compelled to call `self.History` in `OnData` instead rework code for the more efficient `RollingWindow`
- Reminder: WarmUp in `Initialize` otherwise constantly check is window is ready in `OnData`

### RollingWindow DataTypes

can be any native or C# type

| Type               |
| ------------------ |
| IndicatorDataPoint |
| float              |
| TradeBar           |
| QuoteBar           |

### RollingWindow Methods

| Method               | Description                                   |
| -------------------- | --------------------------------------------- |
| .Add                 | Add data to RollingWindow                     |
| .IsReady             | bool if RollingWindow is full                 |
| .MostRecentlyRemoved | Get the item that was most recently removed   |
| .Reset               | Delete all of the elements from RollingWindow |

### RollingWindow Other Operations

- Create RollingWindow
  ```python
  self.<window_name> = RollingWindow[<type>](<number_of_windows>)
  ```
- Access/Index element

  ```python
  current_close = self.close_window[0]
  previous_close = self.close_window[1]
  oldest_close = self.close_window[self.close_window.Count-1]
  # TODO: DOES
  ### Create RollingWindow THIS NOT WORK?
  oldest_close = self.close_window[-1]
  ```

- Get RollingWindow as a list or DataGrame

  ```python
  # Cast RollingWindow to list
  rw_list = list(self.rolling_window)

  # Cast RollingWindow to DataFrame
  rw_df = self.PandasConverter.GetDataFrame[<rollingwindowtype>](self.rolling_window)
  ```

### Rolling Window - COMMON USE CASE

- Store historical indicator values
- Indicators emit `Updated` event every time `<indicator_name>.Update` is called. Use an event handler to add to a RollingWindow.

  ```python
  def Initialize(self):
  		# Creates an indicator and adds to a RollingWindow when it is updated
  		self.sma_window = RollingWindow[IndicatorDataPoint](5)
  		# Updated event handler
  		self.SMA("SPY", 5).Updated += (lambda sender, updated: self.sma_window.Add(updated))
  ```

## Consolidators ([YT Ref](https://www.youtube.com/watch?v=rQOn9iTchIg&list=PLtqRgJ_TIq8Y6YG8G-ETIFW_36mvxMLad&index=7))

- Create custom resolution bars
- Create higher resolution bars than subbed data
  - e.g. Add Minute resolution equity data but algo uses a mixture of minute, hour, daily bars
- Hang on to references to consolidators to remove them from algo if/when a security leaves universe and is no longer needed
- The event handler receives the fully formed bar

  ```python
  # Commonly used signature
  # period can be: timedelta/time span, Calendar object period, Resolution object period
  self.mycons = Consolidate(Symbol, period, eventHandler)
  ```

  ```python
  # Example: Track Daily Close and Open Trade Bars
  def Initialize():
  	# Sub to minute rez data
  	spy = self.AddEquity('SPY', Resolution.Minute).Symbol
  	self.rw = RollingWindow[TradeBar](2)
  	# Eventhandler receives fully formed daily bar and adds that to rolling window
  	# Automatically added to self.SubscriptionManager
  	self.mycons = Consolidate(spy, Resolution.Daily, lambda tradebar: self.rw.Add(tradebar))
  ```

- Consolidators are tracked in the `self.SubscriptionManager`

  - When Symbol leaves the universe be sure to remove any associated consolidators from algo

    ```python
    # Remove consolidator from algo
    self.SubscriptionManager.RemoveConsolidator(Symbol, self.mycons)
    ```

## Indicators ([Ref](https://www.quantconnect.com/docs/v2/writing-algorithms/indicators/key-concepts))

- Standard deviation indicators can be used as a measure of volatility (TODO: come back to this)
- Reminder: WarmUp in `Initialize` otherwise constantly check is window is ready in `OnData`

  ```python
  def Initialize(self):
  	self.sma = self.SMA(self.spy, 30, Resolution.Daily)

  def OnData(self, data):
  	# Check if an indicator is warmed up
  	if not self.<indicator>.IsReady:
  		return
  ```

- Warm up indicators by creating an indicator in `Initialize` then pump in data for period requested

  ```python
  def Initialize(self):
  	self.sma = self.SMA(self.spy, 30, Resolution.Daily)
  	# Indicator immediately ready to be used in `OnData`
  	closing_prices = self.History(self.spy, 30, Resolution.Daily)['close']  # Hx pd Series
  	for time, price in closing_prices.loc[self.spy].items():
  			self.sma.Update(time, price)
  ```

### Shortcut Helpers

- Abbreviated Names
  - e.g. self.SMA()
- Automatically set to receive "close" bars at resoluion requested
  - Notes: the data sub resolution <=indicator resolution requested
  ```python
  symbol = self.AddEquity('SPY', Resolution.Hour).Symbol
  # Can be Resolution higher than Hour
  self.SMA(symbol, 10, Resolution.Hour | Resolution.Daily)
  ```

### Non-shortcut Helpers

- Unabreviated Names
  - e.g. self.SimpleMovingAverage()
- Not automatically set to receive bars
- Use `self.RegisterIndicator()` for indicator to automatically receives bars

  ```python
  # The two forms below will automatically receive a bar at end of every day
  # By default the indicators update using close price
  self.SMA(symbol, 10, Resolution.Daily)
  # equivalent to:
  indicator = self.SimpleMovingAverage(10, MovingAverageType.Simple)
  self.RegisterIndicator(symbol, indicator, Resolution.Daily)
  ```

### Non-shortcut Helpers Use Cases

- Want to use custom period

  ```python
  # Calculate the SMA with ten 7-minute bars
  self.symbol = self.AddEquity("SPY", Resolution.Minute).Symbol
  self.indicator = SimpleMovingAverage(10)
  self.RegisterIndicator(self.symbol, self.indicator, timedelta(minutes=7))
  ```

### Custom Indicators

- Always check built-in indicators first
- Want to update indicator with value other than close price
  - TradeBar uses TradeBar close price by default
  - QuoteBar uses QuoteBarr mid close price by default
- Must inherit `PythonIndicator` class

  ```python
  	def Initialize(self):
  		self.spy = self.AddEquity("SPY", Resolution.Daily).Symbol
  		self.sma = CustomSMA('CustomSMA', 30) # 30 day SMA
  		# RegisterIndicator allows CustomSMA to receive trade bar daily
  		# This means self.sma is not ready for 30 days
  		# If StartDate 1,1,2020 then ready around 2,1,2020
  		self.RegisterIndicator(self.spy, self.sma, Resolution.Daily)

  class CustomSMA(PythonIndicator):
  	def __init__(self, name, period):
  		self.Name = name
  		self.Value = 0
  		self.Time = datetime.min
  		self.queue = deque(maxlen=period)

  	def Update(self, input): # input is daily bar
  		self.queue.appendleft(input.Open)
  		self.Time = input.EndTime
  		count = len(self.queue)
  		self.Value = sum(self.queue) / count
  		return (count == self.queue.maxlen)
  ```

### Indicator Extensions

- Indicator Extensions are indicator methods to tracks more than one indicator simultaneously

  ```python
  pep = Identity('PEP')
  coke = Identity('KO')
  delta = IndicatorExtensions.Minus(pep, coke)
  ```

- Indicator Extension methods

  | Method        | Returns                  |
  | ------------- | ------------------------ |
  | .Plus()       | CompositeIndicator       |
  | .Minus()      | CompositeIndicator       |
  | .Times()      | CompositeIndicator       |
  | .Over()       | CompositeIndicator       |
  | .MIN()        | MinimumIndicator         |
  | .MAX()        | MaximumIndicator         |
  | .EMA()        | ExponentialMovingAverage |
  | .SMA()        | SimpleMovingAverage      |
  | .WeightedBy() | CompositeIndicator       |
  | Of()          | object                   |

## Benchmarks

- Bemchmark can be set to custom data type (class)

## CustomData ([Awesome YT Ref](https://www.youtube.com/watch?v=X7XwkHsE-4Y&list=PLtqRgJ_TIq8Y6YG8G-ETIFW_36mvxMLad&index=9)) ([YT Ref](https://www.youtube.com/watch?v=EbUEMW5135M&list=PLtqRgJ_TIq8Y6YG8G-ETIFW_36mvxMLad&index=19))

- Custom data can be LocalFile, RemoteFile, Rest, Streaming
- General Steps:

  1.  Add custom data to algo with `self.AddData(Type:class, Ticker:str, Resolution)`
  2.  Create custom class that inherits `PythonData`. Must override `GetSource`, `Reader` Methods
  3.  Use custom data in `OnData` or other event handlers

  ```python
  # EXAMPLE: Custom data creation and usage flow
  from nltk.sentiment import SentimentIntensityAnalyzer

  def Initialize(self):
  	"""Add custom data to algo.
  	Custom data must contain the resolution requested here
  	"""
  	self.musk = self.AddData(MustTweet, 'MUSKTWTS', Resolution.Minute).Symbol


  def OnData(self, data):
  	"""Use custom data"""
  	if self.musk in data:
  		score = data[self.musk].Value
  		content = data[self.musk].Tweet

  		if score > 0.5:
  			self.SetHoldings(self.tsla, 1)
  		if score <= -0.5:
  			self.SetHoldings(self.tsla, -1)
  		self.Log(f'Score: {str(score)}, Tweet: {content}')

  # create custom class
  class MuskTweet(PythonData):
  	sia = SentimentIntensityAnalyzer()

  	def GetSource(self, config, date, isLive):
  		"""Define source and connection method.
  		Return SubscriptionDataSource
  		"""
  		if isLive == False:
  			source='https://www.dropbox.com/s/ovnsrgg1fou1y0r/MuskTweetsPreProcessed.csv?dl=0'
  			return SubscriptionDataSource(source, SubscriptionTransportMedium.RemoteFile)
  		else:
  			source='wss://.../ws...'
  			return SubscriptionDataSource(source, SubscriptionTransportMedium.Streaming)

  	def Reader(self, config, line, date, isLive):
  		"""Define/Parse the structure of the custom data.
  		line parameter is the custom data received line by line.
  		Return None or an instantiated version of this class
  		"""
  		if isLive == False:
  			if not (line.strip() and line[0].isdigit()):
  				return None
  			data = line.split(',')
  			tweet = MustTweet()

  			try:
  				# required overrides
  				tweet.Symbol = config.Symbol
  				tweet.EndTime = datetime.strptime(data[0], '%Y-%m-%d %H:%M:%S')
  												+ timedelta(minutes=20)
  				content = data[1].lower()
  				if 'tsla' in content or 'tesla' in content:
  					tweet.Value = self.sia.polarity_scores(content)['compound']
  				else:
  					tweet.Value = 0

  				# add custom attribute
  				tweet['Tweet'] = str(content)

  			except ValueError:
  				return None

  			return tweet

  		else: # handle Streming API data structure
  			pass
  ```

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

## Algorithm Performance and Backtest Results

- [Great YT Ref](https://www.youtube.com/watch?v=oFvYbfDOJ5c&list=PLtqRgJ_TIq8Y6YG8G-ETIFW_36mvxMLad&index=10)

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
h4 {
	color: lightGreen;
}
</style>
