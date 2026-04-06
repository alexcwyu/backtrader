# Backtrader -- State Management

## Order State Machine

```mermaid
stateDiagram-v2
    [*] --> Created: buy() / sell() / close()

    Created --> Submitted: Broker receives order
    Submitted --> Accepted: Broker accepts order

    Accepted --> Partial: Partially filled<br/>(volume/filler constraints)
    Partial --> Partial: Additional partial fills
    Partial --> Completed: Fully filled

    Accepted --> Completed: Fully filled in one step

    Accepted --> Canceled: broker.cancel(order)
    Accepted --> Expired: Validity period ended
    Accepted --> Margin: Insufficient margin
    Accepted --> Rejected: Broker rejects

    Partial --> Canceled: Canceled while partial
    Partial --> Expired: Expired while partial

    Completed --> [*]: Order done
    Canceled --> [*]: Order done
    Expired --> [*]: Order done
    Margin --> [*]: Order done
    Rejected --> [*]: Order done
```

### Order Status Codes

| Status | Value | Description |
|--------|-------|-------------|
| `Created` | 0 | Order created by strategy |
| `Submitted` | 1 | Order submitted to broker |
| `Accepted` | 2 | Broker accepted the order |
| `Partial` | 3 | Order partially filled |
| `Completed` | 4 | Order fully filled |
| `Canceled` | 5 | Order canceled (also `Cancelled`) |
| `Expired` | 6 | Order expired (validity ended) |
| `Margin` | 7 | Insufficient margin |
| `Rejected` | 8 | Broker rejected the order |

### Order Execution Types

| Type | Value | Description |
|------|-------|-------------|
| `Market` | 0 | Fill at next bar's open |
| `Close` | 1 | Fill at current bar's close |
| `Limit` | 2 | Fill at limit price or better |
| `Stop` | 3 | Trigger at stop price, then market fill |
| `StopLimit` | 4 | Trigger at stop, then limit order |
| `StopTrail` | 5 | Trailing stop with absolute amount |
| `StopTrailLimit` | 6 | Trailing stop with limit |
| `Historical` | 7 | Historical order replay |

### Order Notification Flow

```mermaid
sequenceDiagram
    participant S as Strategy
    participant BR as Broker
    participant O as Order

    S->>BR: buy(size=100)
    BR->>O: Create order
    BR-->>S: notify_order(Created)

    BR->>BR: Submit to execution
    BR-->>S: notify_order(Submitted)

    BR->>BR: Accept order
    BR-->>S: notify_order(Accepted)

    Note over BR: Next bar arrives

    BR->>BR: Match order against bar
    BR->>O: Add OrderExecutionBit
    BR-->>S: notify_order(Completed)
    BR-->>S: notify_trade(trade)
```

## Trade Lifecycle

```mermaid
stateDiagram-v2
    [*] --> Created: Trade object instantiated

    Created --> Open: First order fill<br/>size != 0

    Open --> Open: Position increased<br/>avg price recalculated
    Open --> Open: Position reduced<br/>partial P&L realized

    Open --> Closed: Position reaches 0<br/>all P&L realized

    note right of Open
        Attributes tracked:
        - size (current position)
        - price (average entry)
        - value (market value)
        - pnl (unrealized)
        - pnlcomm (unrealized - commission)
        - commission (accumulated)
        - baropen / dtopen
    end note

    note right of Closed
        Final attributes:
        - barclose / dtclose
        - barlen (duration)
        - pnl (realized gross)
        - pnlcomm (realized net)
        - commission (total)
        - isclosed = True
    end note

    Closed --> [*]
```

### Trade States

| State | Value | Description |
|-------|-------|-------------|
| `Created` | 0 | Trade object created |
| `Open` | 1 | Trade has non-zero position |
| `Closed` | 2 | Trade position returned to zero |

### Trade History

Each trade maintains a history list of `TradeHistory` entries. Each entry records:
- **status**: dt, barlen, size, price, value, pnl, pnlcomm
- **event**: order reference, size delta, fill price, commission

```python
# Accessing trade history
def notify_trade(self, trade):
    if trade.isclosed:
        for h in trade.history:
            print(f"  {h.status.dt}: size={h.status.size} pnl={h.status.pnl}")
```

## Position Tracking

```mermaid
flowchart TD
    subgraph Position Object
        SIZE[size: int<br/>Current position]
        PRICE[price: float<br/>Average entry price]
        ORIG[price_orig: float<br/>Previous price before update]
        OPENED[upopened: int<br/>Units opened in last update]
        CLOSED[upclosed: int<br/>Units closed in last update]
        ADJBASE[adjbase: float<br/>Adjusted base for P&L]
    end

    subgraph Update Scenarios
        U1[Buy 100 @ 50<br/>Position: 0 to 100]
        U1 --> R1["size=100, price=50<br/>opened=100, closed=0"]

        U2[Buy 50 @ 55<br/>Position: 100 to 150]
        U2 --> R2["size=150, price=51.67<br/>opened=50, closed=0<br/>(weighted average)"]

        U3[Sell 80 @ 60<br/>Position: 150 to 70]
        U3 --> R3["size=70, price=51.67<br/>opened=0, closed=-80<br/>(price unchanged on reduce)"]

        U4[Sell 100 @ 65<br/>Position: 70 to -30]
        U4 --> R4["size=-30, price=65<br/>opened=-30, closed=-70<br/>(reversal: new price)"]

        U5[Buy 30 @ 60<br/>Position: -30 to 0]
        U5 --> R5["size=0, price=0<br/>opened=0, closed=30"]
    end
```

### Position Update Semantics

The Position class follows specific rules for price averaging:

| Scenario | Size Change | Price Behavior |
|----------|-------------|----------------|
| **Open from zero** | 0 to N | Price = fill price |
| **Increase** | Same direction | Price = weighted average |
| **Reduce** | Opposite, partial | Price unchanged (FIFO) |
| **Close** | To zero | Price = 0 |
| **Reverse** | Through zero | Price = fill price (new direction) |

## Broker State Management

```mermaid
flowchart TD
    subgraph BackBroker State
        CASH[Cash<br/>Available funds]
        VALUE[Portfolio Value<br/>Cash + Position values]
        ORDERS[Pending Orders<br/>List]
        POSITIONS[Positions<br/>Per-data dict]
        TRADES[Open Trades<br/>Per-data dict]
    end

    subgraph Commission Info
        COMM[CommInfoBase]
        COMM --> STOCK[Stock-like<br/>commission = % of value]
        COMM --> FUTURE[Futures-like<br/>commission = fixed per contract]
        COMM --> MARGIN[Margin requirement]
        COMM --> LEVERAGE[Leverage]
        COMM --> INTEREST[Short interest]
    end

    subgraph Fill Simulation
        FILLER[Fillers]
        FILLER --> VOL[FixedSize<br/>Max units per bar]
        FILLER --> BAR[FixedBarPerc<br/>% of bar volume]
    end

    CASH --> VALUE
    POSITIONS --> VALUE
    COMM --> ORDERS
    FILLER --> ORDERS
```

### Broker Cash Flow

```
Starting Cash
  + Closed position P&L
  - Commission on trades
  - Margin requirements (locked)
  - Interest on short positions
  = Current Cash

Portfolio Value = Cash + sum(position.size * current_price) for all positions
```

## Strategy Lifecycle States

```mermaid
stateDiagram-v2
    [*] --> Instantiated: Cerebro creates strategy

    state Instantiated {
        [*] --> ParamsSet: MetaParams applies defaults
        ParamsSet --> BrokerLinked: broker reference set
        BrokerLinked --> SizerSet: Default sizer configured
        SizerSet --> InitCalled: __init__() runs
        InitCalled --> IndicatorsCreated: Indicators auto-registered
    }

    Instantiated --> PeriodCalc: Minimum periods calculated

    state Running {
        [*] --> PreNext: Bar arrived, min period not met
        PreNext --> PreNext: Additional prenext() calls
        PreNext --> NextStart: Min period first reached
        NextStart --> Next: Subsequent bars
        Next --> Next: Repeating next() calls
    }

    PeriodCalc --> Running

    Running --> Stopped: Data exhausted / cerebro.stop()
    Stopped --> [*]
```

## Analyzer State

Analyzers maintain state across the entire backtest:

```mermaid
flowchart LR
    subgraph Analyzer Lifecycle
        CREATE[create_analysis] --> START[start<br/>Initialize state]
        START --> PRENEXT[prenext<br/>Warmup phase]
        PRENEXT --> NEXT[next<br/>Update per bar]
        NEXT --> STOP[stop<br/>Finalize calculations]
        STOP --> GET[get_analysis<br/>Return results dict]
    end

    subgraph Built-in Analyzers
        SR[SharpeRatio]
        DD[DrawDown]
        TA[TradeAnalyzer]
        RET[Returns]
        VWR[VariabilityWeightedReturn]
        SQN[SQN]
        PYFOLIO[PyFolio]
    end
```

## Observer State

Observers are passive monitors that record data into lines for plotting:

| Observer | Lines | Description |
|----------|-------|-------------|
| `Broker` | cash, value | Portfolio cash and total value |
| `BuySell` | buy, sell | Buy/sell signal markers |
| `Trades` | pnlplus, pnlminus | Profit/loss per trade |
| `DrawDown` | drawdown, maxdrawdown | Current and max drawdown |
| `Benchmark` | benchmark | Benchmark comparison line |

## Data Feed States

```mermaid
stateDiagram-v2
    [*] --> Disconnected: Feed created

    Disconnected --> Delayed: Connection delayed
    Disconnected --> Connected: Connection established
    Disconnected --> [*]: Connection failed

    Delayed --> Connected: Connection ready

    Connected --> Live: Live data streaming
    Connected --> Backfilling: Historical data loading

    Backfilling --> Live: Backfill complete

    Live --> Disconnected: Connection lost
    Live --> [*]: Feed stopped

    note right of Live
        Data status notifications:
        CONNECTED, DISCONNECTED,
        DELAYED, LIVE, BACKFILLING
    end note
```

---
## See Also
- [README](README.md) — Project overview and quick start
- [Architecture](architecture.md) — System design and components
- [Workflow](workflow.md) — Event flows and processing pipelines
- [Development](development.md) — Development guide and best practices
