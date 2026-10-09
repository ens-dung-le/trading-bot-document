# Plan - Breaker Blocks Strategy

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). The
fixed `15m/4h` rules below describe the baseline; other pairs are pending.

Review: parameter literals and a static required-timeframe list restrict the
strategy to `15m + 4h`, despite a candle-based breaker lifecycle.

Goal: Support `breaker-block-retest` on valid `T < C` profiles.
Out of scope: Replacing breaker conversion or allowing repeated retests.
Steps:

1. In `src/strategies/breaker-block-retest/parameters.ts`, validate enum-based
   trigger/trend values, require `C > T` and resolve `[T, C]` for all run paths.
2. In `signal.ts`, recompute warm-up for this pair and preserve the original
   order-block, break, breaker creation and first-retest sequence on `T`.
   Zone age remains trigger bars; EMA confirmation uses completed `C` bars.
3. Test `15m/4h`, `1h/4h` and `4h/1d`, including a break and retest that appear
   inside the same coarse candle. Preserve the indicator's event order and
   reject any entry that would require an invented intrabar sequence.

Verification: Static trace a bearish block that breaks bullish, then receives
its first later retest; reverse all directions for SHORT. No context bar with
a close after the retest decision time may confirm the trade.

## Original baseline specification

Source: https://www.tradingview.com/script/zO0geJ5O-Breaker-Blocks-Flux-Charts/

Goal: Add a backtest-capable `breaker-block-retest` strategy that trades the
first return to an active breaker created by a failed order block.

Out of scope: Do not trade the original order block, enter on the breaking
candle, trade repeated retests, combine zones, or add discretionary SMC filters.

Steps:

1. Require a Pine-parity `src/indicators/breaker-blocks.ts` with original order
   block, break, breaker creation, retest, invalidation, direction, and volume
   fields before strategy work. Operate only in
   `src/strategies/breaker-block-retest/` plus registry files.
2. Add a strict schema with trigger `15m`, trend `4h`, Pine-default swing length
   and Wick invalidation, fixed notional `200 USDT`, leverage `2`, maximum zone
   age `192` bars, maximum entry distance `1 ATR`, and first-retest-only enabled.
3. LONG requires exactly one latest bullish breaker first-retest event, an
   active zone, a close above its upper boundary, and a latest fully closed `4h`
   close above a rising EMA50. SHORT requires the symmetric bearish conditions.
   Fill at the next `15m` open.
4. Place the stop beyond the far breaker boundary plus `0.1 ATR`, with a minimum
   distance of `1.5 ATR`; target `2R`, maximum hold `20`, break-even enabled,
   trailing disabled. The shared engine owns fees, funding, spread, slippage,
   liquidation, and time-exit execution. Do not recalculate breaker direction
   in the strategy.
5. Bound the order-block-to-breaker lifecycle without removing a zone until it
   is more than 192 trigger bars old. Reject older, invalidated, ambiguous,
   repeated, or more-than-`1 ATR` distant retests using only closed candles.
   Measure bullish distance as `close - zoneTop` and bearish distance as
   `zoneBottom - close`; age and distance boundaries are inclusive.
6. Register and test failed-order-block conversion, break timing, first retest,
   Wick/Close parity, volume audit data, `4h` causality, bounded/full state,
   barrier math, next-open fill, costs, and replay parity.
7. Research sequentially: frozen baseline; stop/buffer/R:R/hold sensitivity;
   swing length, zone age and entry distance; Wick versus Close invalidation;
   volume and `4h` EMA ablations; then one chronological 70/30 out-of-sample
   validation. Compare first retests with repeated retests only as a diagnostic.
8. Accept only with profit factor above `1.5`, Sharpe above `1.5`, drawdown below
   `12%`, at least `250` completed trades, positive expectancy after all costs,
   zero liquidations, positive doubled-cost results, performance above the 90th
   percentile of matched random-entry runs, out-of-sample decay below `20%`, at
   least three of four symbols positive out of sample, no symbol above 60% of
   aggregate profit, and separate profit/expectancy disclosure by swing-length
   and zone-age bucket.

Verification: Run indicator and strategy tests; statically trace one bearish
order block breaking into a bullish breaker and the inverse case; run a short
`15m` backtest with `4h` confirmation data.
