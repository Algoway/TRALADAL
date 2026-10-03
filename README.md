# TRALADAL

**TRALADAL** stands for **TRadingView ALert ADapter for ALgoway**.

It is an open-source Pine Script v6 strategy adapter for TradingView. The script takes numeric outputs from indicators, evaluates them as entry, confirmation, filter, or exit conditions, turns those conditions into TradingView strategy orders, and can send the same events as structured AlgoWay JSON alerts.

TRALADAL does not generate trading signals. It provides a bridge between indicator outputs, TradingView Strategy Tester, and alert-based execution.

> TradingView script: https://www.tradingview.com/script/oc2K8Hkq-TradingView-Alert-Adapter-for-AlgoWay/

---

## How TRALADAL works

A typical workflow is:

```text
Indicator output
    |
    v
TRALADAL condition engine
    |
    +-- TradingView strategy entry / exit
    |
    +-- AlgoWay JSON alert
```

The important part is that the original indicator does not need to be rewritten as a strategy. TRALADAL reads indicator outputs through TradingView source inputs and applies its own strategy and alert logic on top of them.

The script is configured with:

- up to three LONG conditions;
- up to three SHORT conditions;
- optional LONG and SHORT exit conditions;
- AND / OR logic between enabled conditions;
- event, crossover, comparison, and level-based condition modes;
- percent-of-equity, cash, or contract sizing;
- optional stop loss, take profit, and trailing stop logic;
- optional reversal on an opposite entry signal;
- AlgoWay JSON generation through `alert()`.

All entry and exit alerts use `alert.freq_once_per_bar_close`, and strategy orders are processed with `process_orders_on_close=true`.

---

## Using outputs from another indicator

TRALADAL conditions use `input.source()` fields. In TradingView, a source can be selected from price series or from compatible plotted outputs exposed by indicators on the same chart.

This makes it possible to work with several common indicator designs:

- a numeric pulse such as `1` on a signal bar and `0` otherwise;
- an oscillator such as RSI;
- a moving average, VWAP, MACD line, or another continuous series;
- two indicator lines that should cross each other;
- a line that should cross or remain above / below a fixed level.

For closed-source or invite-only indicators, the source code is not required. What matters is whether TradingView exposes a usable numeric output from that script in the source selector. A visual shape by itself is not enough unless it is backed by a selectable series.

---

## Condition modes

Each LONG and SHORT slot can use one of the following modes.

| Mode | Meaning |
| --- | --- |
| `Event (non-zero)` | TRUE when source A is not `na` and is not zero |
| `A CrossOver B` | TRUE when A crosses above B |
| `A CrossUnder B` | TRUE when A crosses below B |
| `A > B` | TRUE while A is greater than B |
| `A < B` | TRUE while A is less than B |
| `A >= B` | TRUE while A is greater than or equal to B |
| `A <= B` | TRUE while A is less than or equal to B |
| `A CrossOver Level` | TRUE when A crosses above a fixed level |
| `A CrossUnder Level` | TRUE when A crosses below a fixed level |
| `A > Level` | TRUE while A is above a fixed level |
| `A < Level` | TRUE while A is below a fixed level |
| `A >= Level` | TRUE while A is at or above a fixed level |
| `A <= Level` | TRUE while A is at or below a fixed level |

TRALADAL evaluates the raw condition every bar, but entries fire only on the transition from FALSE to TRUE:

```text
FALSE -> TRUE  = fire
TRUE  -> TRUE  = no new fire
TRUE  -> FALSE = reset
FALSE -> TRUE  = fire again
```

This edge-trigger behavior is important when using persistent conditions such as `RSI < 30` or `close > EMA`. The strategy does not repeatedly enter on every bar while the condition remains true.

---

## Signal, confirmation, and filter

LONG and SHORT entries each provide three independent condition slots:

1. **Signal** — primary trigger;
2. **Confirm** — optional confirmation;
3. **Filter** — optional additional condition.

Only enabled slots participate in the result.

With `AND`, every enabled condition must be true at the same time.

With `OR`, any enabled condition can activate the entry.

For example:

```text
Signal:  fast EMA crosses above slow EMA
Confirm: RSI > 50
Filter:  close > VWAP
Logic:   AND
```

A LONG event is produced only when the combined condition changes from false to true.

---

## Example 1: RSI level strategy

Add an RSI indicator to the chart and select its RSI line as the TRALADAL source.

LONG configuration:

```text
Long Enable #1 = ON
Mode #1        = A CrossOver Level
A #1           = RSI output
Level #1       = 30
```

SHORT configuration:

```text
Short Enable #1 = ON
Mode #1         = A CrossUnder Level
A #1            = RSI output
Level #1        = 70
```

This creates entries when RSI crosses back above 30 for LONG and back below 70 for SHORT.

Using `A < Level` instead of `A CrossUnder Level` produces different behavior. The first mode represents a persistent state; the second represents the crossing event itself. TRALADAL's edge detection prevents repeated entries while a persistent state remains true.

---

## Example 2: moving-average crossover

Use two selectable moving-average outputs.

LONG:

```text
Long Enable #1 = ON
Mode #1        = A CrossOver B
A #1           = fast moving average
B #1           = slow moving average
```

SHORT:

```text
Short Enable #1 = ON
Mode #1         = A CrossUnder B
A #1            = fast moving average
B #1            = slow moving average
```

No modification of the moving-average indicator code is required as long as its plotted series are available in TradingView's source selector.

---

## Example 3: signal plus confirmation

Assume one indicator outputs entry pulses while another provides a trend line.

LONG:

```text
Logic           = AND

Long Enable #1  = ON
Mode #1         = Event (non-zero)
A #1            = indicator LONG signal output

Long Enable #2  = ON
Mode #2         = A > B
A #2            = close
B #2            = trend line output
```

The entry signal is accepted only when the event occurs while price is above the selected trend line.

The same structure can be extended with condition #3 as a filter.

---

## Example 4: event-style indicator output

Some indicators expose a numeric series that is zero most of the time and becomes non-zero only when a signal occurs.

For that type of output:

```text
Enable #1 = ON
Mode #1   = Event (non-zero)
A #1      = signal series
```

The internal test is equivalent to:

```text
not na(source) and source != 0
```

TRALADAL then applies its own FALSE-to-TRUE edge detection before creating a strategy entry or alert.

---

## Optional exits

Entry logic and exit logic are separate.

Each side has one optional exit condition:

- `EXIT LONG` closes an open LONG position;
- `EXIT SHORT` closes an open SHORT position.

Exit conditions support the same event, crossover, comparison, and level modes as entry conditions.

When an enabled exit condition fires, TRALADAL calls `strategy.close()` and sends an AlgoWay `flat` alert for the current position quantity.

Example:

```text
Entry LONG:
RSI crosses above 30

Exit LONG:
RSI crosses above 70
```

This can be used instead of fixed take-profit logic when the indicator itself defines the exit.

---

## Reversal behavior

`Flip on opposite entry signal` controls whether an opposite entry can reverse an existing position.

When enabled:

```text
LONG position + SHORT fire = reverse to SHORT
SHORT position + LONG fire = reverse to LONG
```

When disabled, an opposite entry signal is ignored while the opposite position is still open.

The strategy uses `pyramiding=0`, so it does not build multiple same-direction strategy entries.

---

## Position sizing

TRALADAL supports three sizing modes.

### Percent of equity

The script calculates:

```text
contracts = (strategy.equity * percentage / 100) / close
```

### Cash

The script calculates:

```text
contracts = cash_value / close
```

### Contracts

The configured value is used directly as the contract quantity.

The calculated quantity is used for both the TradingView strategy order and the AlgoWay alert. It is formatted to three decimal places in the JSON payload.

Broker, exchange, CFD, forex, and futures products can use different native quantity conventions. The TradingView strategy quantity is therefore a model quantity; the final execution behavior also depends on the selected AlgoWay platform and account configuration.

---

## Risk-management modes

The `SL/TP/TRAIL Mode` setting has three states.

### Off

No stop loss, take profit, or trailing stop is applied by the TradingView strategy.

Entry alerts use the basic AlgoWay payload and include the current price.

### Strategy

Stop loss, take profit, and trailing logic are managed inside the TradingView strategy.

Entry alerts remain basic and do not include risk fields.

When the TradingView strategy closes a position through its risk-management logic, TRALADAL detects the completed strategy trade and sends a `flat` alert so the external execution state can be closed as well.

### Alert

TradingView does not apply the configured stop loss, take profit, or trailing stop to the strategy position.

Instead, those values are added to the AlgoWay entry JSON.

In the current source, risk fields are disabled for the `ctrader` platform selector and the script falls back to the basic entry payload.

---

## Stop loss, take profit, and trailing calculations

Risk distances use `syminfo.mintick`.

For a LONG position:

```text
stop loss   = entry price - stop_loss_pips * syminfo.mintick
take profit = entry price + take_profit_pips * syminfo.mintick
```

For a SHORT position:

```text
stop loss   = entry price + stop_loss_pips * syminfo.mintick
take profit = entry price - take_profit_pips * syminfo.mintick
```

The setting is named `pips` in the script, but technically the calculation uses minimum tick size. Depending on the symbol, one configured unit may therefore represent a tick rather than the broker convention commonly called one forex pip.

The trailing stop also advances in multiples of `syminfo.mintick`.

---

## AlgoWay JSON payloads

TRALADAL builds JSON internally, so no manual alert message is required.

### Basic LONG entry

```json
{
  "platform_name": "metatrader5",
  "ticker": "EURUSD",
  "order_id": "Long",
  "order_action": "buy",
  "order_contracts": "0.100",
  "price": "1.12345"
}
```

### Basic SHORT entry

```json
{
  "platform_name": "metatrader5",
  "ticker": "EURUSD",
  "order_id": "Short",
  "order_action": "sell",
  "order_contracts": "0.100",
  "price": "1.12345"
}
```

### Entry with risk fields

When `SL/TP/TRAIL Mode = Alert` and the selected platform supports the risk payload in the current script:

```json
{
  "platform_name": "metatrader5",
  "ticker": "EURUSD",
  "order_id": "Long",
  "order_action": "buy",
  "order_contracts": "0.100",
  "stop_loss": "1.12000",
  "take_profit": "1.13000",
  "trailing_pips": "100"
}
```

### Flat / exit

```json
{
  "platform_name": "metatrader5",
  "ticker": "EURUSD",
  "order_id": "Long",
  "order_action": "flat",
  "order_contracts": "0.100"
}
```

The actual values are generated dynamically from the selected platform, chart ticker, strategy state, configured sizing, and active risk mode.

---

## Creating the TradingView alert

After the indicator sources and TRALADAL conditions are configured:

1. Create a new TradingView alert.
2. Select the TRALADAL strategy.
3. Use **Any alert() function call** as the alert condition.
4. Add the AlgoWay webhook endpoint in TradingView's webhook field.
5. Do not manually duplicate the JSON in the alert message. TRALADAL generates it inside the script.

Because the script uses `alert.freq_once_per_bar_close`, execution alerts are emitted at bar close rather than on every intrabar recalculation.

---

## Backtesting indicator logic

TRALADAL is a `strategy()`, not an `indicator()`. Once external indicator outputs are connected to its conditions, the resulting entries and exits appear in TradingView Strategy Tester.

This allows the same condition configuration to be evaluated with metrics such as:

- net profit;
- number of closed trades;
- win rate;
- profit factor;
- pessimistic profit factor;
- maximum drawdown.

The script also displays a compact statistics table on the chart.

Backtest results should not be interpreted as a guarantee of live execution results. TradingView strategy fills, bar data, slippage assumptions, broker rules, latency, spread, commissions, and external execution behavior can all produce differences.

---

## Repainting and signal confirmation

TRALADAL cannot remove repainting from an upstream indicator.

The adapter evaluates the values it receives. When an indicator changes historical or realtime output after additional market data becomes available, TRALADAL receives the changed value as well.

The script reduces one common source of alert inconsistency by sending alerts only once per bar close, but that does not make a repainting source non-repainting.

When testing an external indicator, verify:

- whether its historical signals move or disappear;
- whether it uses higher-timeframe data;
- whether it uses future-looking calculations;
- whether the plotted output is confirmed only at bar close;
- whether the historical series behaves the same way as the realtime series.

A backtest is only as reliable as the source series being tested.

---

## Strategy assumptions in the current source

The current script defines the following TradingView strategy settings:

```text
Pine Script version       = 6
pyramiding                = 0
process_orders_on_close   = true
initial capital           = 100000 USD
default quantity type     = percent of equity
default quantity value    = 10
commission value          = 0.08
slippage                  = 15
```

The explicit TRALADAL order-size controls override the practical entry quantity calculation used by `strategy.entry()`.

When comparing results with another strategy, make sure these assumptions are understood before comparing performance statistics.

---

## Current platform selector

The current source exposes these `platform_name` values:

```text
metatrader5
tradelocker
matchtrader
dxtrade
ctrader
capitalcom
alpaca
tradovate
bybit
binance
okx
bitmex
bitmart
bitget
bingx
```

This list documents the selector implemented in this repository version. Availability or execution behavior of a platform can change independently of the Pine Script source.

---

## What TRALADAL does not do

TRALADAL is deliberately an adapter, not a signal generator.

It does not:

- decide which indicator is profitable;
- repair repainting logic in another script;
- read values that TradingView does not expose as selectable sources;
- guarantee that a TradingView backtest matches broker fills;
- bypass TradingView alert or account limitations;
- provide trading recommendations.

Its job is narrower: take available indicator outputs, evaluate them consistently, create strategy events, and optionally serialize those events into AlgoWay alerts.

---

## License

MIT
