# Advanced Strategies

This page explains some advanced concepts available for strategies.
If you're just getting started, please be familiar with the methods described in the [Strategy Customization](strategy-customization.md) documentation and with the [Freqtrade basics](bot-basics.md) first.

[Freqtrade basics](bot-basics.md) describes in which sequence each method described below is called, which can be helpful to understand which method to use for your custom needs.

!!! Note
    All callback methods described below should only be implemented in a strategy if they are actually used.

!!! Tip
    You can get a strategy template containing all below methods by running `freqtrade new-strategy --strategy MyAwesomeStrategy --template advanced`

## Storing information

Storing information can be accomplished by creating a new dictionary within the strategy class.

The name of the variable can be chosen at will, but should be prefixed with `cust_` to avoid naming collisions with predefined strategy variables.

```python
class AwesomeStrategy(IStrategy):
    # Create custom dictionary
    custom_info = {}

    def populate_indicators(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
        # Check if the entry already exists
        if not metadata["pair"] in self.custom_info:
            # Create empty entry for this pair
            self.custom_info[metadata["pair"]] = {}

        if "crosstime" in self.custom_info[metadata["pair"]]:
            self.custom_info[metadata["pair"]]["crosstime"] += 1
        else:
            self.custom_info[metadata["pair"]]["crosstime"] = 1
```

!!! Warning
    The data is not persisted after a bot-restart (or config-reload). Also, the amount of data should be kept smallish (no DataFrames and such), otherwise the bot will start to consume a lot of memory and eventually run out of memory and crash.

!!! Note
    If the data is pair-specific, make sure to use pair as one of the keys in the dictionary.

## Dataframe access

You may access dataframe in various strategy functions by querying it from dataprovider.

``` python
from freqtrade.exchange import timeframe_to_prev_date

class AwesomeStrategy(IStrategy):
    def confirm_trade_exit(self, pair: str, trade: 'Trade', order_type: str, amount: float,
                           rate: float, time_in_force: str, sell_reason: str,
                           current_time: 'datetime', **kwargs) -> bool:
        # Obtain pair dataframe.
        dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)

        # Obtain last available candle. Do not use current_time to look up latest candle, because 
        # current_time points to current incomplete candle whose data is not available.
        last_candle = dataframe.iloc[-1].squeeze()
        # <...>

        # In dry/live runs trade open date will not match candle open date therefore it must be 
        # rounded.
        trade_date = timeframe_to_prev_date(self.timeframe, trade.open_date_utc)
        # Look up trade candle.
        trade_candle = dataframe.loc[dataframe['date'] == trade_date]
        # trade_candle may be empty for trades that just opened as it is still incomplete.
        if not trade_candle.empty:
            trade_candle = trade_candle.squeeze()
            # <...>
```

!!! Warning "Using .iloc[-1]"
    You can use `.iloc[-1]` here because `get_analyzed_dataframe()` only returns candles that backtesting is allowed to see.
    This will not work in `populate_*` methods, so make sure to not use `.iloc[]` in that area.
    Also, this will only work starting with version 2021.5.

***

## Enter Tag

When your strategy has multiple buy signals, you can name the signal that triggered.
Then you can access you buy signal on `custom_sell`

```python
def populate_buy_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['rsi'] < 35) &
            (dataframe['volume'] > 0)
        ),
        ['buy', 'enter_tag']] = (1, 'buy_signal_rsi')

    return dataframe

def custom_sell(self, pair: str, trade: Trade, current_time: datetime, current_rate: float,
                    current_profit: float, **kwargs):
    dataframe, _ = self.dp.get_analyzed_dataframe(pair, self.timeframe)
    last_candle = dataframe.iloc[-1].squeeze()
    if trade.enter_tag == 'buy_signal_rsi' and last_candle['rsi'] > 80:
        return 'sell_signal_rsi'
    return None

```

!!! Note
    `enter_tag` is limited to 100 characters, remaining data will be truncated.

## Exit tag

Similar to [Buy Tagging](#buy-tag), you can also specify a sell tag.

``` python
def populate_sell_trend(self, dataframe: DataFrame, metadata: dict) -> DataFrame:
    dataframe.loc[
        (
            (dataframe['rsi'] > 70) &
            (dataframe['volume'] > 0)
        ),
        ['sell', 'exit_tag']] = (1, 'exit_rsi')

    return dataframe
```

The provided exit-tag is then used as sell-reason - and shown as such in backtest results.

!!! Note
    `sell_reason` is limited to 100 characters, remaining data will be truncated.

## Derived strategies

The strategies can be derived from other strategies. This avoids duplication of your custom strategy code. You can use this technique to override small parts of your main strategy, leaving the rest untouched:

``` python
class MyAwesomeStrategy(IStrategy):
    ...
    stoploss = 0.13
    trailing_stop = False
    # All other attributes and methods are here as they
    # should be in any custom strategy...
    ...

class MyAwesomeStrategy2(MyAwesomeStrategy):
    # Override something
    stoploss = 0.08
    trailing_stop = True
```

Both attributes and methods may be overridden, altering behavior of the original strategy in a way you need.

!!! Note "Parent-strategy in different files"
    If you have the parent-strategy in a different file, you'll need to add the following to the top of your "child"-file to ensure proper loading, otherwise freqtrade may not be able to load the parent strategy correctly.

    ``` python
    import sys
    from pathlib import Path
    sys.path.append(str(Path(__file__).parent))

    from myawesomestrategy import MyAwesomeStrategy
    ```

## Embedding Strategies

Freqtrade provides you with an easy way to embed the strategy into your configuration file.
This is done by utilizing BASE64 encoding and providing this string at the strategy configuration field,
in your chosen config file.

### Encoding a string as BASE64

This is a quick example, how to generate the BASE64 string in python

```python
from base64 import urlsafe_b64encode

with open(file, 'r') as f:
    content = f.read()
content = urlsafe_b64encode(content.encode('utf-8'))
```

The variable 'content', will contain the strategy file in a BASE64 encoded form. Which can now be set in your configurations file as following

```json
"strategy": "NameOfStrategy:BASE64String"
```

Please ensure that 'NameOfStrategy' is identical to the strategy name!

## Performance warning

When executing a strategy, one can sometimes be greeted by the following in the logs

> PerformanceWarning: DataFrame is highly fragmented.

This is a warning from [`pandas`](https://github.com/pandas-dev/pandas) and as the warning continues to say:
use `pd.concat(axis=1)`.
This can have slight performance implications, which are usually only visible during hyperopt (when optimizing an indicator).

For example:

```python
for val in self.buy_ema_short.range:
    dataframe[f'ema_short_{val}'] = ta.EMA(dataframe, timeperiod=val)
```

should be rewritten to

```python
frames = [dataframe]
for val in self.buy_ema_short.range:
    frames.append({
        f'ema_short_{val}': ta.EMA(dataframe, timeperiod=val)
    })

# Append columns to existing dataframe
merged_frame = pd.concat(frames, axis=1)
```
