# TRALADAL (TRadingView ALert ADapter for ALgoway)

TRALADAL is a Pine Script v6 strategy-adapter for TradingView that lets you automate execution via AlgoWay WITHOUT writing JSON manually.

You define entry/exit conditions (up to 3 conditions for Long and Short), and TRALADAL automatically:
- places/closures trades in the TradingView strategy (optional)
- sends ready-to-use AlgoWay alerts via `alert()` (payload is generated automatically)

---

## What it does

### Provider
- Off  = no alerts
- AlgoWay = send alerts via `alert()`

### Platform selector (script settings)
metatrader5, tradelocker, matchtrader, dxtrade, ctrader, capitalcom,
alpaca, tradovate, bybit, binance, okx, bitmex, bitmart, bitget, bingx

### Position sizing (used for both strategy entries and alerts)
- Percent of equity
- Cash
- Contracts

### Entries
- LONG: up to 3 conditions (AND/OR)
- SHORT: up to 3 conditions (AND/OR)
- Fires on the edge: condition becomes TRUE on bar close

### Optional exits
- EXIT LONG: 1 condition (edge trigger)
- EXIT SHORT: 1 condition (edge trigger)

### Reversal
- Flip on opposite entry signal (Long <-> Short)

---

## SL / TP / Trailing modes

SL/TP/TRAIL Mode:
- Off
  = no SL/TP/TSL in strategy and no SL/TP/TSL in alerts
- Strategy
  = SL/TP/TSL applied inside TradingView strategy
  = alerts are sent without risk parameters
- Alert
  = strategy does not apply SL/TP/TSL
  = risk parameters are sent via alerts (generated automatically)

Note: in this script, “pips” are calculated as `(pips * syminfo.mintick)`.
For some symbols this equals ticks, not broker “forex pips”.

---

## How to use (no JSON)

1) Add TRALADAL to your chart.
2) In script settings:
   - Automation Provider = AlgoWay
   - Trading Platform = select your platform
   - Configure Order size type/value
3) Configure LONG/SHORT conditions:
   - enable needed conditions (Enable #1/#2/#3)
   - choose Mode (Event/Cross/Compare/Level)
   - set A/B/Level (can be series/lines from your indicator)
4) Create a TradingView alert:
   - Condition: “Any alert() function call”
   - Webhook URL: your AlgoWay webhook endpoint

---

## License

MIT
