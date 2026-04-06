# Backtrader -- Workflow

## Backtesting Pipeline

```mermaid
sequenceDiagram
    participant U as User
    participant CE as Cerebro
    participant D as Data Feed
    participant S as Strategy
    participant IND as Indicators
    participant BR as Broker
    participant OBS as Observers
    participant ANA as Analyzers

    U->>CE: cerebro = bt.Cerebro()
    U->>CE: cerebro.adddata(data)
    U->>CE: cerebro.addstrategy(MyStrategy)
    U->>CE: cerebro.broker.setcash(100000)

    U->>CE: results = cerebro.run()

    Note over CE: Phase 1: Data Preloading
    CE->>D: preload() if preload=True
    D->>D: Load all bars into memory

    Note over CE: Phase 2: Strategy Instantiation
    CE->>S: Create strategy instance
    S->>IND: __init__: Declare indicators
    IND->>IND: Auto-calculate minimum periods

    Note over CE: Phase 3: Vectorized Indicators
    alt runonce=True
        CE->>IND: Run all indicators in vectorized mode
    end

    Note over CE: Phase 4: Event Loop
    loop For each bar in data
        CE->>D: Advance data lines
        CE->>BR: Check pending orders

        alt minperiod not reached
            CE->>S: prenext()
        else minperiod reached (first time)
            CE->>S: nextstart()
        else normal operation
            CE->>S: next()
        end

        S->>BR: buy() / sell() / close()
        BR-->>S: notify_order(order)
        BR-->>S: notify_trade(trade)
        CE->>OBS: Update observers
        CE->>ANA: Update analyzers
    end

    Note over CE: Phase 5: Results
    CE->>ANA: Collect analyzer results
    CE-->>U: Return list of strategy instances
```

## Strategy Execution Flow

```mermaid
flowchart TD
    INIT["Strategy.__init__()"] --> DECLARE["Declare indicators<br/>self.sma = bt.ind.SMA(self.data.close, period=20)"]
    DECLARE --> AUTOPERIOD["Minimum period auto-calculated<br/>from indicator dependency tree"]
    AUTOPERIOD --> WARMUP["Warmup phase: prenext() called<br/>until min period reached"]

    WARMUP --> FIRSTBAR["nextstart() called once<br/>on first valid bar"]
    FIRSTBAR --> LOOP["next() called for each<br/>subsequent bar"]

    LOOP --> LOGIC{Trading Logic}
    LOGIC -->|Buy| BUY["self.buy(data, size, price,<br/>exectype, valid, oco, ...)"]
    LOGIC -->|Sell| SELL["self.sell(data, size, price,<br/>exectype, valid, oco, ...)"]
    LOGIC -->|Close| CLOSE["self.close(data)"]
    LOGIC -->|Bracket| BRACKET["self.buy_bracket(<br/>limitprice, stopprice, size)"]
    LOGIC -->|Cancel| CANCEL["self.cancel(order)"]
    LOGIC -->|No action| NEXT[Continue]

    BUY --> NOTIFY
    SELL --> NOTIFY
    CLOSE --> NOTIFY
    BRACKET --> NOTIFY
    CANCEL --> NOTIFY
    NEXT --> LOOP

    NOTIFY["notify_order(order)<br/>notify_trade(trade)"]
    NOTIFY --> LOOP
```

## Order Processing Flow

```mermaid
flowchart TD
    subgraph Order Submission
        CREATE[Strategy: buy/sell/close] --> SIZER[Sizer calculates size]
        SIZER --> SUBMIT[Broker.submit order]
        SUBMIT --> QUEUE[Order queue<br/>Status: Submitted]
    end

    subgraph Order Matching (BackBroker)
        QUEUE --> ACCEPT[Status: Accepted]
        ACCEPT --> MATCH{Match against bar}

        MATCH --> MKT{Market Order}
        MKT --> FILL_OPEN[Fill at Open price]

        MATCH --> CLOSE_ORD{Close Order}
        CLOSE_ORD --> FILL_CLOSE[Fill at Close price]

        MATCH --> LMT{Limit Order}
        LMT --> LMT_CHK{Price within<br/>High/Low range?}
        LMT_CHK -->|Yes| FILL_LMT[Fill at limit price]
        LMT_CHK -->|No| PENDING[Remain pending]

        MATCH --> STP{Stop Order}
        STP --> STP_CHK{Stop triggered?}
        STP_CHK -->|Yes| FILL_STP[Fill at stop price]
        STP_CHK -->|No| PENDING

        MATCH --> STPLMT{StopLimit}
        STPLMT --> STPLMT_CHK{Stop triggered?}
        STPLMT_CHK -->|Yes| CONVERT[Convert to Limit]
        STPLMT_CHK -->|No| PENDING
        CONVERT --> LMT_CHK

        MATCH --> TRAIL{StopTrail}
        TRAIL --> TRAIL_UPD[Update trail price<br/>based on High/Low]
        TRAIL_UPD --> TRAIL_CHK{Trail hit?}
        TRAIL_CHK -->|Yes| FILL_TRAIL[Fill at trail price]
        TRAIL_CHK -->|No| PENDING
    end

    subgraph Fill Processing
        FILL_OPEN --> EXEC[Create OrderExecutionBit]
        FILL_CLOSE --> EXEC
        FILL_LMT --> EXEC
        FILL_STP --> EXEC
        FILL_TRAIL --> EXEC

        EXEC --> PARTIAL{Fully filled?}
        PARTIAL -->|Partial| PART_STATUS[Status: Partial]
        PARTIAL -->|Full| COMP_STATUS[Status: Completed]

        PART_STATUS --> POSITION[Update Position]
        COMP_STATUS --> POSITION
        POSITION --> TRADE[Update Trade<br/>P&L, Commission]
        TRADE --> STRATEGY_NOTIFY[notify_order + notify_trade]
    end

    PENDING --> VALID{Order valid?<br/>Expiry check}
    VALID -->|Expired| EXPIRED[Status: Expired]
    VALID -->|Valid| QUEUE
```

## Data Feed Handling

```mermaid
flowchart TD
    subgraph Data Sources
        CSV[CSV Files] --> LOADER
        YAHOO[Yahoo Finance] --> LOADER
        PANDAS[pandas DataFrame] --> LOADER
        IB[Interactive Brokers] --> LOADER
        OANDA[Oanda API] --> LOADER
        LOADER[Feed Parser]
    end

    subgraph Data Processing
        LOADER --> LINES[Lines<br/>Open, High, Low, Close,<br/>Volume, OpenInterest, DateTime]
        LINES --> FILTERS{Filters}
        FILTERS --> SESSION[SessionFilter<br/>Trading hours only]
        FILTERS --> CALENDAR[CalendarFilter<br/>Trading days only]
        FILTERS --> CUSTOM[Custom Filters]
    end

    subgraph Timeframe Management
        LINES --> RESAMPLE[Resampler<br/>e.g., 1min to 1h]
        LINES --> REPLAY[Replayer<br/>Build bars incrementally]
        RESAMPLE --> MULTI[Multiple timeframes<br/>available to strategy]
        REPLAY --> MULTI
    end

    subgraph Multi-Data Sync
        MULTI --> SYNC[Date synchronization]
        SYNC --> D0[data0: Primary]
        SYNC --> D1[data1: Secondary]
        SYNC --> DN[dataN: Additional]
        D0 --> STRATEGY[Strategy receives<br/>synchronized bars]
        D1 --> STRATEGY
        DN --> STRATEGY
    end
```

### Multi-Timeframe Example

```python
cerebro = bt.Cerebro()

# Add primary 1-minute data
data0 = bt.feeds.GenericCSVData(dataname='data_1min.csv')
cerebro.adddata(data0)

# Resample to 15-minute
data1 = cerebro.resampledata(data0, timeframe=bt.TimeFrame.Minutes, compression=15)

# In strategy: self.data0 (1min), self.data1 (15min)
```

## Optimization Flow

```mermaid
flowchart TD
    A[cerebro.optstrategy<br/>MyStrategy,<br/>period=range 10 50,<br/>mult=1.0 2.0 3.0] --> B[Generate parameter grid]

    B --> C{optdatas=True?}
    C -->|Yes| D[Preload data once<br/>Share across workers]
    C -->|No| E[Each worker loads data]

    D --> F[Multiprocessing Pool<br/>maxcpus workers]
    E --> F

    F --> W1[Worker 1: params_1]
    F --> W2[Worker 2: params_2]
    F --> WN[Worker N: params_N]

    W1 --> R1[Run backtest]
    W2 --> R2[Run backtest]
    WN --> RN[Run backtest]

    R1 --> COLLECT{optreturn=True?}
    R2 --> COLLECT
    RN --> COLLECT

    COLLECT -->|Yes| LIGHT[Return OptReturn<br/>params + analyzers only]
    COLLECT -->|No| FULL[Return full Strategy<br/>with all data]

    LIGHT --> RESULTS[List of results]
    FULL --> RESULTS
```

### Optimization Example

```python
cerebro = bt.Cerebro()
cerebro.adddata(data)

cerebro.optstrategy(
    MyStrategy,
    period=range(10, 50, 5),
    multiplier=[1.0, 1.5, 2.0, 2.5]
)

cerebro.addanalyzer(bt.analyzers.SharpeRatio)
results = cerebro.run(maxcpus=4)

# Results is a list of lists (one per parameter combination)
for result in results:
    strat = result[0]
    sharpe = strat.analyzers.sharperatio.get_analysis()
    print(f'Params: {strat.params.__dict__}, Sharpe: {sharpe}')
```

## Live Trading Flow

```mermaid
sequenceDiagram
    participant U as User
    participant CE as Cerebro
    participant ST as Store (IB/Oanda)
    participant BR as Broker
    participant D as Data Feed
    participant S as Strategy

    U->>ST: Create store connection
    U->>BR: Get broker from store
    U->>CE: cerebro.setbroker(broker)
    U->>D: Create live data feed
    U->>CE: cerebro.adddata(data)
    U->>CE: cerebro.addstrategy(MyStrategy)

    U->>CE: cerebro.run()

    Note over CE: preload=False, runonce=False (auto)

    loop Continuous
        D->>D: Receive tick/bar from exchange
        D-->>S: notify_data(data, status)
        CE->>S: next()
        S->>BR: buy() / sell()
        BR->>ST: Submit to exchange
        ST-->>BR: Execution report
        BR-->>S: notify_order(order)
        BR-->>S: notify_trade(trade)
    end
```

## Signal-Based Strategy Flow

```mermaid
flowchart TD
    A[cerebro.add_signal<br/>SIGNAL_LONG, indicator] --> B[Cerebro wraps in SignalStrategy]
    B --> C[Signal indicators computed]
    C --> D{Signal value?}
    D -->|> 0| E[Long signal]
    D -->|< 0| F[Short signal]
    D -->|0| G[No signal / Close]

    E --> H{Current position?}
    H -->|Short| I[Close short + Open long]
    H -->|None| J[Open long]
    H -->|Long| K[Hold]

    F --> L{Current position?}
    L -->|Long| M[Close long + Open short]
    L -->|None| N[Open short]
    L -->|Short| O[Hold]
```

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [State Management](state-management.md) — State lifecycle and data models
- [Development](development.md) — Development guide and best practices
