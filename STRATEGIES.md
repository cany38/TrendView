# Writing indicators and strategies

TrendView separates visual indicators from trading strategies. An indicator draws lines or bands on the chart. A strategy emits long, short, and close orders that run inside the Rust broker emulator with real historical bars, take profit, stop loss, trailing stops, commission, slippage, and leverage.

## Where files live

Put your Python file inside the `strategies/` folder. The app runs every `.py` file there using Python 3 standard library modules and expects a single JSON document on stdout.

## Quick start with the SDK

Import `Strategy`, `load_bars`, and `emit` from `trendview_sdk.py`. The helper handles CSV loading, date parsing, and common calculations (`sma`, `ema`, `rsi`, `atr`).

```python
#!/usr/bin/env python3
import sys
from trendview_sdk import Strategy, ema, emit, load_bars

def run_symbol(symbol, period, bars):
    closes = [b["close"] for b in bars]
    fast = ema(closes, 12)
    slow = ema(closes, 26)

    strat = Strategy(
        name="My EMA Strategy",
        symbol=symbol,
        period=period,
        kind="strategy",
        take_profit_pct=5.0,
        stop_loss_pct=2.5,
        trailing_stop_pct=0.0,
        direction="both",
    )
    strat.plot("EMA 12", bars, fast, "#2962ff")
    strat.plot("EMA 26", bars, slow, "#ff9800")

    for i in range(1, len(bars)):
        f, s, pf, ps = fast[i], slow[i], fast[i - 1], slow[i - 1]
        if None in (f, s, pf, ps):
            continue
        if pf <= ps and f > s:
            strat.entry_long(bars[i], "Long")
        elif pf >= ps and f < s:
            strat.entry_short(bars[i], "Short")

    return strat.to_dict()

def main():
    out = [
        run_symbol(sym, per, bars)
        for (sym, per), bars in load_bars(sys.argv[1]).items()
        if len(bars) >= 30
    ]
    emit(out)

if __name__ == "__main__":
    main()
```

## Indicator vs strategy fields

Set `kind="indicator"` if you only want lines on the chart. Omit order calls (`entry_long`, `entry_short`, `close`) and call `plot(...)` for each line.

Set `kind="strategy"` to place the script under `Indicators -> Technicals -> Strategies` with the strategy badge and open the bottom Strategy Tester when attached.

Supported order calls:

- `strat.entry_long(bar, signal="Long")` opens a long position (and closes an open short if direction allows both).
- `strat.entry_short(bar, signal="Short")` opens a short position (and closes an open long).
- `strat.close(bar, signal="Close")` closes the active position at that bar.

Default risk properties passed to `Strategy(...)` can be changed live later inside the Strategy Settings dialog (`Properties` tab):

- `take_profit_pct`: percentage gain target per position (`0` disables it).
- `stop_loss_pct`: percentage stop loss from average entry price (`0` disables it).
- `trailing_stop_pct`: percentage trailing stop tracked from the best price since entry (`0` disables it).
- `direction`: `"both"`, `"long"`, or `"short"`.

## Raw JSON format (without the SDK)

If you prefer writing raw JSON without importing `trendview_sdk.py`, print an array of objects in this shape:

```json
[
  {
    "name": "Custom Cross",
    "symbol": "BINANCE:BTCUSDT",
    "period": "1d",
    "kind": "strategy",
    "overlays": [
      { "label": "Fast", "color": "#2962ff", "points": [["2026-01-02", 94200.5]] }
    ],
    "orders": [
      { "date": "2026-01-10", "ts": 1768003200000, "action": "long", "signal": "Breakout" },
      { "date": "2026-01-24", "ts": 1769212800000, "action": "close", "signal": "Exit" }
    ],
    "properties": {
      "takeProfitPct": 6.0,
      "stopLossPct": 3.0,
      "trailingStopPct": 0.0,
      "direction": "both"
    }
  }
]
```

## Deep backtesting and intrabar rules

Inside the bottom Strategy Tester, open the date range menu to switch between the visible chart range, preset windows (7, 30, 90, or 365 days), entire history, or a custom date range. On desktop, TrendView reads stored SQLite history and pulls missing historical candles from the exchange before running the simulation in Rust with live progress and cancel support.

Default bar detalization checks four intrabar price points per candle (open, low, high, close for bullish bars, open, high, low, close for bearish bars). Switching to high detalization interpolates 28 steps across each bar for tighter take profit, stop loss, and trailing stop fills.
