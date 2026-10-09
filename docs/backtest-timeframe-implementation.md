# Telegram Backtest Timeframe Implementation

Status: implemented 2026-09-16.

The Telegram Backtest menu offers `1m`, `5m`, `15m`, `1h`, and `4h`. A
strategy is shown only when its code declares and tests all five choices. The
queued snapshot owns the selected trigger, so Paper profiles are unchanged.

## Enabled strategies

- `box-strategy`: selected trigger plus its closed `15m` box source.
- `five-gate-directional` and `triple-sma-directional`: all indicators use the
  selected trigger series.
- `nadaraya-band-mean-reversion`, `vwap-divergence-directional`,
  `anchored-vwap-daily-reclaim`, `wick-body-ratio-trend-forecast`,
  `fair-value-gap-retest`, `order-block-retest`, `breaker-block-retest`, and
  `ict-suspension-block-retest`: a selected trigger uses closed context candles
  at `1m -> 5m -> 15m -> 1h -> 4h -> 1d`. The existing stored profile is kept
  for Paper and other non-Telegram runs. The 4h VWAP profile uses a monthly
  anchor because a daily session has too few bars for its history windows.

## Intentionally hidden strategies

- `dca-stop-straddle`: OCO, DCA rung, and take-profit ordering needs an
  intrabar replay contract before large candles can be evaluated safely.
- `fvg-quality-retest`: its child-volume dependency has no lower source below
  `1m`.
- `support-resistance-break-retest` and `multi-ma-quantum`: their 4h profiles
  require unverified weekly context.
- `multi-indicator-directional`: each forecast horizon needs a separately
  trained and validated model snapshot.
- `trailing-geometric-grid` and `grid-amount-scaler`: stateful execution
  strategies need a separate replay-timeframe contract.

No unsupported strategy may be enabled by changing only a schema literal.
