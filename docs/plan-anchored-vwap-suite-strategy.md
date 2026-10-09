# Plan - Anchored VWAP Suite Strategy

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). The
`15m/1h` profile below is the baseline. General intraday support is pending.

Review: trigger/trend literals and the static dependency array fix the pair.
The Daily Open anchor is a semantic limit, not just a schema restriction:
daily bars reset the anchor on every bar and leave no same-anchor crossover.

Goal: Support valid intraday triggers `T` with confirmation `C > T`.
Out of scope: Silently turning Daily Open reclaim into weekly/monthly reclaim.
Steps:

1. In `src/strategies/anchored-vwap-daily-reclaim/parameters.ts`, accept shared
   enum values with `T < 1d`, UTC-day-compatible boundaries and `C > T`.
   Resolve `[T, C]`, retaining existing default values.
2. In `signal.ts` and prepared replay, reconstruct the current UTC-day anchor
   from its start, including pre-roll needed for RMA/EMA. Keep spacing in `T`
   bars and reset/crossover eligibility tied to the anchor identity.
3. Test `15m/1h`, `1h/4h` and `4h/1d` across midnight, missing volume,
   same-anchor equality and next-open execution. Reject daily/weekly triggers
   explicitly rather than returning a misleading zero-trade success.

Verification: Static trace the first hourly bar of a new UTC day: it has no
previous same-anchor value and cannot reclaim. A later bar can cross only its
own day's AVWAP using context already closed at its decision timestamp.

Full daily/weekly applicability requires a separately specified longer-anchor
variant, including anchor choice, reset boundaries and sufficient observations.
It is not equivalent to changing only `triggerTimeframe` on this strategy.

## Original baseline specification

Source: https://www.tradingview.com/script/UYUaORRN-Anchored-VWAP-Suite-Flux-Charts/

Goal: Add a backtest-capable `anchored-vwap-daily-reclaim` strategy using one
causal Daily Open anchored VWAP from the suite.

Out of scope: Do not trade swing, lookback, session, weekly, monthly, high/low,
or merged-label anchors in version 1; do not modify the Pine or TypeScript
indicator and do not use an intra-bar anchor value.

Steps:

1. Require a Pine-parity `src/indicators/anchored-vwap-suite.ts` and focused
   tests from the indicator-port task before strategy implementation starts.
   The strategy itself operates only in
   `src/strategies/anchored-vwap-daily-reclaim/` plus registry integration.
2. Add a strict schema with trigger `15m`, trend `1h`, Daily Open anchor, the
   source indicator's default `RMA` method and period `20`, fixed notional
   `200 USDT`, leverage `2`, ATR14 stop `1.5`, target `2R`, maximum hold `20`,
   maximum close-to-AVWAP distance `1 ATR`, minimum signal spacing `8` trigger
   bars, break-even enabled, and trailing disabled. A signal is eligible only
   when at least eight fully closed `15m` bars have elapsed since the previous
   accepted signal for that symbol.
3. LONG requires the latest closed `15m` candle to cross from at/below to above
   the current Daily Open AVWAP while the latest fully closed `1h` candle is
   above a rising EMA50 and `close - AVWAP <= 1 ATR`. SHORT is symmetric with
   `AVWAP - close <= 1 ATR`. Skip the first daily candle because no prior value
   exists for a same-anchor crossover; fill at the next `15m` open.
4. Reconstruct the current UTC-day anchor causally, use only closed candles,
   bound recursive RMA/EMA history, and prove that appending future candles does
   not alter earlier AVWAP or signals.
5. Reuse the shared fixed-notional, margin, fee, funding, spread, slippage,
   liquidation, break-even, stop/target, and time-exit paths; register the
   strategy without changing the engine or runner.
6. Add tests for daily reset, RMA parity, crossover equality, closed-`1h`
   selection, long/short symmetry, no-volume handling, next-open fill, costs,
   registry resolution, and backtest replay parity.
7. Research sequentially: frozen baseline; stop/R:R/hold sensitivity; Standard,
   EMA and RMA method diagnostics; close-to-AVWAP distance
   `0.5/1.0/1.5/2.0 ATR`; spacing `0/4/8/16`; one-entry-per-UTC-day and
   no-EMA-confirmation
   ablations; then one chronological 70/30 out-of-sample validation. Never tune
   method, distance, entry spacing and exits in the same stage.
8. Accept only with profit factor above `1.5`, Sharpe above `1.5`, drawdown below
   `12%`, at least `250` completed trades, positive expectancy after fees,
   funding, spread and slippage, zero liquidations, positive doubled-cost
   results, performance above the 90th percentile of matched random-entry runs,
   out-of-sample decay below `20%`, at least three of four symbols positive out
   of sample, and no symbol above 60% of aggregate profit.

Verification: Run indicator and strategy tests, statically trace one bullish
reclaim and one bearish loss of AVWAP across a UTC-day boundary, then run a
short `15m` backtest with `1h` confirmation data.
