# Backtrader -- Development Guide

## Setup

### Installation

```bash
# From PyPI
pip install backtrader

# Development installation
git clone https://github.com/mementum/backtrader.git
cd backtrader
pip install -e .

# With plotting support
pip install backtrader[plotting]
# or
pip install matplotlib
```

### Requirements

- **Python**: 3.x
- **Core**: No mandatory external dependencies (pure Python)
- **Optional**: matplotlib (plotting), pytz (timezone), TA-Lib (indicators)

## Project Structure

```
backtrader/
  src/backtrader/
    __init__.py          # Package imports and re-exports
    cerebro.py           # Central engine (Cerebro)
    strategy.py          # Strategy base class
    order.py             # Order, OrderData, OrderExecutionBit
    trade.py             # Trade tracking and history
    position.py          # Position management
    broker.py            # Broker base class
    feed.py              # Data feed base classes
    indicator.py         # Indicator base class
    observer.py          # Observer base class
    analyzer.py          # Analyzer base class
    sizer.py             # Position sizer base class
    signal.py            # Signal-based strategy support
    timer.py             # Timer infrastructure
    writer.py            # CSV/file output writer
    comminfo.py          # Commission info schemes
    fillers.py           # Order fill simulation
    linebuffer.py        # Core line buffer (ring buffer)
    lineroot.py          # Line hierarchy root
    lineiterator.py      # Line iteration protocol
    lineseries.py        # Multi-line containers
    dataseries.py        # OHLCV data series
    resamplerfilter.py   # Resampling and replaying
    tradingcal.py        # Trading calendar
    metabase.py          # MetaParams metaclass system
    mathsupport.py       # Math utilities
    errors.py            # Error definitions
    functions.py         # Utility functions
    flt.py               # Filter base
    store.py             # Store base for live trading
    talib.py             # TA-Lib integration bridge
    version.py           # Version info

    brokers/             # Broker implementations
      __init__.py
      bbroker.py         # BackBroker (backtesting)
      ibbroker.py        # Interactive Brokers
      oandabroker.py     # Oanda

    feeds/               # Data feed implementations
      __init__.py
      csvgeneric.py      # Generic CSV
      yahoo.py           # Yahoo Finance
      pandafeed.py       # pandas DataFrame
      ibdata.py          # Interactive Brokers
      oandadata.py       # Oanda

    indicators/          # 100+ built-in indicators
      __init__.py
      sma.py, ema.py, rsi.py, macd.py, ...
      contrib/           # Community contributed indicators

    analyzers/           # Built-in analyzers
      sharpe.py, drawdown.py, tradeanalyzer.py, ...

    observers/           # Built-in observers
      broker.py, buysell.py, trades.py, drawdown.py, ...

    sizers/              # Position sizer implementations
    stores/              # Store implementations (IB, Oanda)
    filters/             # Data filters
    signals/             # Signal definitions
    strategies/          # Pre-built strategies
    studies/             # Study modules
    utils/               # Utilities and Python 3 compat
    plot/                # Matplotlib plotting
    btrun/               # Command-line runner
```

## Strategy Creation Guide

### Basic Strategy

```python
import backtrader as bt

class SmaCross(bt.Strategy):
    params = (
        ('fast', 10),
        ('slow', 30),
    )

    def __init__(self):
        # Indicators are auto-registered
        sma_fast = bt.ind.SMA(self.data.close, period=self.p.fast)
        sma_slow = bt.ind.SMA(self.data.close, period=self.p.slow)
        self.crossover = bt.ind.CrossOver(sma_fast, sma_slow)

    def next(self):
        if self.crossover > 0:  # Fast crosses above slow
            self.buy()
        elif self.crossover < 0:  # Fast crosses below slow
            self.close()

# Run
cerebro = bt.Cerebro()
data = bt.feeds.YahooFinanceCSVData(dataname='AAPL.csv')
cerebro.adddata(data)
cerebro.addstrategy(SmaCross)
cerebro.broker.setcash(100000)
cerebro.broker.setcommission(commission=0.001)
cerebro.run()
cerebro.plot()
```

### Strategy with Order Management

```python
class ManagedStrategy(bt.Strategy):
    params = (('period', 20),)

    def __init__(self):
        self.sma = bt.ind.SMA(period=self.p.period)
        self.order = None  # Track pending order

    def notify_order(self, order):
        if order.status in [order.Completed]:
            if order.isbuy():
                self.log(f'BUY @ {order.executed.price:.2f}')
            else:
                self.log(f'SELL @ {order.executed.price:.2f}')
        elif order.status in [order.Canceled, order.Margin, order.Rejected]:
            self.log('Order failed')
        self.order = None  # Reset

    def notify_trade(self, trade):
        if trade.isclosed:
            self.log(f'TRADE P&L: {trade.pnl:.2f} (net: {trade.pnlcomm:.2f})')

    def next(self):
        if self.order:  # Skip if order pending
            return

        if not self.position:
            if self.data.close[0] > self.sma[0]:
                self.order = self.buy()
        else:
            if self.data.close[0] < self.sma[0]:
                self.order = self.sell()

    def log(self, txt):
        dt = self.data.datetime.date(0)
        print(f'{dt}: {txt}')
```

### Bracket Orders (SL/TP)

```python
class BracketStrategy(bt.Strategy):
    def next(self):
        if not self.position and self.buy_signal():
            price = self.data.close[0]
            self.buy_bracket(
                limitprice=price * 1.05,  # Take profit at +5%
                stopprice=price * 0.97,   # Stop loss at -3%
                size=100
            )

    def buy_signal(self):
        return self.data.close[0] > self.data.close[-1]
```

### Multi-Data Strategy

```python
class PairsStrategy(bt.Strategy):
    def __init__(self):
        # self.data0 and self.data1 are the two data feeds
        self.spread = self.data0.close - self.data1.close
        self.zscore = bt.ind.ZScore(self.spread, period=20)

    def next(self):
        if self.zscore[0] > 2.0:
            self.sell(data=self.data0)
            self.buy(data=self.data1)
        elif self.zscore[0] < -2.0:
            self.buy(data=self.data0)
            self.sell(data=self.data1)
        elif abs(self.zscore[0]) < 0.5:
            self.close(data=self.data0)
            self.close(data=self.data1)
```

## Custom Indicator Development

### Basic Custom Indicator

```python
class MyIndicator(bt.Indicator):
    lines = ('myline',)  # Declare output lines
    params = (('period', 20),)

    def __init__(self):
        # Declarative (vectorized) approach - preferred
        self.lines.myline = bt.ind.SMA(self.data.close, period=self.p.period)
        # or compute from scratch:
        # self.addminperiod(self.p.period)

    # Alternative: event-driven approach
    # def next(self):
    #     self.lines.myline[0] = sum(self.data.close.get(size=self.p.period)) / self.p.period
```

### Multi-Line Indicator

```python
class BollingerBands(bt.Indicator):
    lines = ('mid', 'top', 'bot',)
    params = (('period', 20), ('devfactor', 2.0),)

    plotinfo = dict(subplot=False)  # Overlay on price

    def __init__(self):
        self.lines.mid = bt.ind.SMA(self.data.close, period=self.p.period)
        stddev = bt.ind.StdDev(self.data.close, period=self.p.period)
        self.lines.top = self.lines.mid + self.p.devfactor * stddev
        self.lines.bot = self.lines.mid - self.p.devfactor * stddev
```

### Using TA-Lib Indicators

```python
import backtrader.talib as btta

class TALibStrategy(bt.Strategy):
    def __init__(self):
        self.rsi = btta.RSI(self.data.close, timeperiod=14)
        self.macd, self.signal, self.hist = btta.MACD(self.data.close)
```

## Data Source Integration

### Generic CSV

```python
data = bt.feeds.GenericCSVData(
    dataname='mydata.csv',
    dtformat='%Y-%m-%d',
    datetime=0, open=1, high=2, low=3, close=4, volume=5,
    openinterest=-1,
    fromdate=datetime.datetime(2020, 1, 1),
    todate=datetime.datetime(2024, 1, 1),
)
```

### pandas DataFrame

```python
import pandas as pd

df = pd.read_csv('data.csv', index_col='Date', parse_dates=True)
data = bt.feeds.PandasData(dataname=df)
cerebro.adddata(data)
```

### Interactive Brokers (Live)

```python
store = bt.stores.IBStore(host='127.0.0.1', port=7497, clientId=1)
cerebro.broker = store.getbroker()

data = store.getdata(
    dataname='AAPL-STK-SMART-USD',
    timeframe=bt.TimeFrame.Minutes,
    compression=1,
    historical=True,
    fromdate=datetime.datetime(2024, 1, 1),
)
cerebro.adddata(data)
```

### Resampling and Replaying

```python
# Resample intraday to daily
data0 = bt.feeds.GenericCSVData(dataname='1min_data.csv')
cerebro.adddata(data0)

# Add resampled version
cerebro.resampledata(data0, timeframe=bt.TimeFrame.Days)
# or replay (builds bars incrementally)
cerebro.replaydata(data0, timeframe=bt.TimeFrame.Days)
```

## Testing

### Manual Testing

```python
# Quick test with built-in sample
cerebro = bt.Cerebro()
data = bt.feeds.YahooFinanceCSVData(dataname='sample_data.csv')
cerebro.adddata(data)
cerebro.addstrategy(MyStrategy)

# Add analyzers for validation
cerebro.addanalyzer(bt.analyzers.SharpeRatio, _name='sharpe')
cerebro.addanalyzer(bt.analyzers.DrawDown, _name='drawdown')
cerebro.addanalyzer(bt.analyzers.TradeAnalyzer, _name='trades')

results = cerebro.run()
strat = results[0]

# Inspect results
print('Sharpe:', strat.analyzers.sharpe.get_analysis())
print('DrawDown:', strat.analyzers.drawdown.get_analysis())
print('Trades:', strat.analyzers.trades.get_analysis())
```

### Debugging Strategies

```python
class DebugStrategy(bt.Strategy):
    def next(self):
        # Log current state
        self.log(f'Close: {self.data.close[0]:.2f}, '
                 f'Cash: {self.broker.getcash():.2f}, '
                 f'Value: {self.broker.getvalue():.2f}, '
                 f'Position: {self.position.size}')

    def log(self, txt):
        dt = self.data.datetime.date(0)
        print(f'{dt}: {txt}')
```

### Command-Line Runner

Backtrader includes `btrun` for running strategies without writing a script:

```bash
btrun --data mydata.csv --strategy MyModule:MyStrategy --cash 100000
```

## Common Patterns

### Custom Commission Scheme

```python
class FixedCommission(bt.CommInfoBase):
    params = (('commission', 10), ('stocklike', True),)

    def _getcommission(self, size, price, pseudoexec):
        return self.p.commission  # Fixed $10 per trade

cerebro.broker.addcommissioninfo(FixedCommission())
```

### Custom Sizer

```python
class PercentSizer(bt.Sizer):
    params = (('percents', 10),)

    def _getsizing(self, comminfo, cash, data, isbuy):
        position = self.broker.getposition(data)
        if not position:
            size = int((cash * self.p.percents / 100) / data.close[0])
            return size
        return position.size

cerebro.addsizer(PercentSizer, percents=25)
```

### Timer-Based Execution

```python
class TimerStrategy(bt.Strategy):
    def __init__(self):
        self.add_timer(
            when=bt.timer.SESSION_START,
            offset=datetime.timedelta(minutes=30),
        )

    def notify_timer(self, timer, when, *args, **kwargs):
        # Called 30 minutes after session start
        self.rebalance()
```

## Configuration Reference

### Cerebro Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `preload` | `bool` | `True` | Preload data feeds into memory before running strategies. |
| `runonce` | `bool` | `True` | Run indicators in vectorized mode for faster computation. Strategies and observers still run event-based. |
| `live` | `bool` | `False` | Force live-trading mode. Deactivates `preload` and `runonce`. |
| `maxcpus` | `int \| None` | `None` | Maximum CPU cores for optimization. `None` uses all available cores. |
| `stdstats` | `bool` | `True` | Automatically add default observers: Broker (cash/value), Trades, and BuySell. |
| `oldbuysell` | `bool` | `False` | Use deprecated buy/sell observer plotting behavior (markers at execution price instead of high/low). |
| `oldtrades` | `bool` | `False` | Use old Trades observer (same markers for all datas). |
| `exactbars` | `bool \| int` | `False` | Memory optimization. `True`/`1` reduces line buffers to minimum period (disables plotting). `-1` keeps strategy-level data. `-2` keeps only named attributes. |
| `objcache` | `bool` | `False` | Experimental: reuse identical line objects to reduce memory. |
| `writer` | `bool` | `False` | Add a default WriterFile that prints to stdout. |
| `tradehistory` | `bool` | `False` | Enable update event logging for all trades in all strategies. |
| `optdatas` | `bool` | `True` | Preload data once in the main process during optimization (approx. 20% speedup). |
| `optreturn` | `bool` | `True` | Return lightweight objects (params + analyzers only) from optimization instead of full Strategy objects (13-15% speedup). |
| `oldsync` | `bool` | `False` | Use pre-1.9.0.99 data synchronization behavior (data0 as master). |
| `tz` | `str \| pytz \| int \| None` | `None` | Global timezone for strategies. `None` = UTC. An int references the timezone of `self.datas[n]`. |
| `cheat_on_open` | `bool` | `False` | Call `next_open()` before `next()`, allowing orders based on previous indicators but using the open price. |
| `broker_coo` | `bool` | `True` | Auto-enable cheat-on-open in the broker when `cheat_on_open` is `True`. |
| `quicknotify` | `bool` | `False` | Deliver broker notifications as soon as possible (relevant for live trading). |

### BackBroker (Default Broker) Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `cash` | `float` | `10000` | Initial cash balance. Set via `cerebro.broker.setcash()`. |
| `commission` | `float` | `0.0` | Commission rate. Set via `cerebro.broker.setcommission()`. |
| `checksubmit` | `bool` | `True` | Check available cash/margin before accepting orders. |
| `eosbar` | `bool` | `False` | Use end-of-session bar for order matching. |
| `filler` | `callable \| None` | `None` | Volume filler for partial fills (e.g., `bt.fillers.FixedSize(100)`). |
| `slip_perc` | `float` | `0.0` | Slippage as a percentage of price. |
| `slip_fixed` | `float` | `0.0` | Slippage as a fixed price amount. |
| `slip_open` | `bool` | `False` | Apply slippage to open-price orders (cheat-on-open). |
| `slip_match` | `bool` | `True` | Match slipped price against high/low range. If `False`, the order always fills at the slipped price. |
| `slip_limit` | `bool` | `True` | Apply slippage to limit orders (cap at limit price). |
| `slip_out` | `bool` | `False` | Allow slippage to move the price outside the high/low range. |
| `coc` | `bool` | `False` | Cheat-on-close: fill market orders at the close price of the current bar. |
| `coo` | `bool` | `False` | Cheat-on-open: fill orders at the open price of the current bar. |
| `int2pnl` | `bool` | `True` | Include interest in P&L calculations. |
| `shortcash` | `bool` | `True` | Increase cash when entering short positions (realistic margin accounting). |
| `fundstartval` | `float` | `100.0` | Starting fund-like NAV value for fund mode. |
| `fundmode` | `bool` | `False` | Track value as a fund (NAV-based) rather than raw cash + positions. |

### PandasData Feed Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `dataname` | `pd.DataFrame` | *(required)* | The pandas DataFrame with OHLCV data. |
| `datetime` | `int \| None` | `None` | Column index for datetime. `None` uses the DataFrame index. |
| `open` | `int \| str` | `0` | Column index or name for Open. Set to `-1` to disable. |
| `high` | `int \| str` | `1` | Column index or name for High. |
| `low` | `int \| str` | `2` | Column index or name for Low. |
| `close` | `int \| str` | `3` | Column index or name for Close. |
| `volume` | `int \| str` | `4` | Column index or name for Volume. Set to `-1` if not available. |
| `openinterest` | `int \| str` | `5` | Column index or name for Open Interest. Set to `-1` if not available. |
| `fromdate` | `datetime \| None` | `None` | Start date filter. |
| `todate` | `datetime \| None` | `None` | End date filter. |

## Troubleshooting

### 1. `ImportError: No module named 'backtrader'`
**Cause**: Package not installed or installed in a different Python environment.
**Fix**: Run `pip install backtrader` in the correct virtual environment. For development, use `pip install -e .` from the repo root.

### 2. Strategy `next()` is never called
**Cause**: Indicators have minimum period requirements. Until all indicators have enough data, only `prenext()` is called.
**Fix**: Check your indicator periods. Override `prenext()` to log warmup progress: `def prenext(self): print(f'Warming up: {len(self)}')`. Ensure your data has more bars than your longest indicator period.

### 3. Orders rejected with `Margin` status
**Cause**: Insufficient cash or margin to place the order.
**Fix**: Check `self.broker.getcash()` before ordering. Reduce position size, increase initial capital with `cerebro.broker.setcash()`, or use a custom sizer that accounts for available margin.

### 4. `AttributeError` when accessing `self.data.close` in `__init__`
**Cause**: In `__init__`, data lines are not yet populated with values. You must use them declaratively (for indicator computation), not access their scalar values.
**Fix**: Access scalar values only in `next()`. In `__init__`, use lines for indicator declarations: `self.sma = bt.ind.SMA(self.data.close, period=20)`.

### 5. Plotting fails with `ModuleNotFoundError: matplotlib`
**Cause**: matplotlib is an optional dependency not installed by default.
**Fix**: Install it: `pip install matplotlib` or `pip install backtrader[plotting]`.

### 6. Live trading connection errors (Interactive Brokers)
**Cause**: TWS or IB Gateway is not running, or the port/clientId is wrong.
**Fix**: Ensure TWS/Gateway is running with API access enabled. Default port is 7497 (paper) or 7496 (live). Each connection needs a unique `clientId`.

### 7. Optimization is very slow
**Cause**: Large parameter space with the default `maxcpus=None` may still be slow; `optreturn=False` or `optdatas=False` add overhead.
**Fix**: Keep `optreturn=True` and `optdatas=True` (defaults). Reduce the parameter space. Set `maxcpus` to a specific number to control resource usage.

### 8. Data timezone issues / bars appear at wrong times
**Cause**: Data timestamps and the strategy timezone are misaligned.
**Fix**: Set `tz` on Cerebro or on individual data feeds. Use `pytz` instances for explicit control: `cerebro = bt.Cerebro(tz=pytz.timezone('US/Eastern'))`.

### 9. `next()` called with stale indicator values
**Cause**: Using `runonce=False` with indicators that expect vectorized mode, or mixing incompatible indicator types.
**Fix**: Keep `runonce=True` (default) unless you have a specific reason to disable it. If you need event-driven indicator computation, implement the `next()` method on your custom indicator.

## Security Considerations

### API Keys and Broker Credentials
- Never hardcode IB, Oanda, or other broker credentials in strategy files. Use environment variables or a dedicated secrets manager.
- The `IBStore` and `OandaStore` accept credentials as constructor arguments. Pass them via `os.environ.get()` rather than string literals.

### Live Trading Safety
- Always test strategies in paper-trading mode before going live. For IB, use port 7497 (paper) instead of 7496 (live).
- Implement position limits and maximum order size checks in `notify_order()` to prevent runaway orders.
- Use `cerebro.broker.set_checksubmit(True)` (default) to verify margin/cash before order submission.

### Data Integrity
- Validate external data feeds for NaN values, duplicate timestamps, and gaps before feeding them to Cerebro.
- When using live data feeds, implement reconnection logic and handle partial bars gracefully.

### Look-Ahead Bias
- Indicators computed in `__init__` use declarative mode and do not leak future data. However, custom `next()` logic that accesses `self.data.close[1]` (future index) will introduce look-ahead bias.
- Resampled data must be properly aligned. Use `cerebro.resampledata()` rather than manual resampling to ensure correct bar boundaries.

### Pickle and Serialization
- Optimization results may be pickled when using multiprocessing. Avoid storing credentials or sensitive data as strategy parameters or attributes.
- The `optreturn=True` setting (default) reduces the amount of data serialized between processes.

### Network Exposure
- Live trading stores (IB, Oanda) open network connections. Ensure they run on trusted networks. Use firewalls to restrict access to broker API ports.
- The `btrun` command-line tool can load arbitrary Python modules. Do not run it with untrusted strategy files.

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [State Management](state-management.md) — State lifecycle and data models
