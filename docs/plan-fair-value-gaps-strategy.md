# Plan - Fair Value Gaps Volumetric Strategy

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). The
`15m/1h` profile below remains the original baseline; generalization is pending.

Review: trigger/trend literals and the static dependency array fix `15m + 1h`.
FVGs must be detected anew on the selected trigger, not copied from old runs.

Goal: Support `fair-value-gap-retest` on valid `T < C` profiles.
Out of scope: Changing the three-candle formation, volume filter or retest rule.
Steps:

1. In `src/strategies/fair-value-gap-retest/parameters.ts`, accept supported
   trigger/trend values, validate `C > T` and resolve `[T, C]`.
2. In `signal.ts`, derive warm-up using formation/lifecycle history on `T`
   and closed EMA history on `C`. Keep gap age, hold and swing periods as bars;
   retain volume units and the original indicator's causal detection rules.
3. Test `15m/1h`, `1h/4h` and `4h/1d` for formation versus first rejection,
   gap age boundaries, invalidation, costs and bounded/full history parity.

Verification: Static trace three completed hourly formation candles followed
by a distinct eligible first rejection. A formation event alone cannot enter;
the signal fills no earlier than the next executable open.

## Original baseline specification

Source: https://www.tradingview.com/script/W4oyz3qC-Fair-Value-Gaps-Volumetric-Flux-Charts/

Goal: Add a backtest-capable `fair-value-gap-retest` strategy that trades the
first rejection of an active volumetric FVG.

Out of scope: Do not combine overlapping gaps for trading, enter on gap
formation, trade repeated retests, change visual settings, or reuse the FVG
Quality Scorer as a substitute for this indicator.

Steps:

1. Require a Pine-parity `src/indicators/fair-value-gaps-volumetric.ts` and its
   tests before strategy implementation. Operate only in
   `src/strategies/fair-value-gap-retest/` plus registry files.
2. Add a strict schema with trigger `15m`, trend `1h`, Wick invalidation,
   Average Range filtering, Same Type detection, Normal sensitivity, minimum
   directional volume ratio `0.55`, maximum gap age `192` trigger bars, fixed
   notional `200 USDT`, and leverage `2`. The volume threshold is explicitly a
   strategy-owned research default, not Pine behavior.
3. LONG requires exactly one bullish active FVG receiving its first wick test,
   the test candle closing above the upper boundary, directional volume ratio
   at or above the strategy threshold, and a latest closed `1h` close above a
   rising EMA50. SHORT is symmetric below a bearish FVG.
4. Confirm the retest only from closed `15m` candles and fill at the next `15m`
   open. Reject mitigated, older-than-192-bar, ambiguous, discontinuous-session,
   or more-than-`1 ATR` distant gaps. Measure bullish distance as
   `close - upperBoundary` and bearish distance as `lowerBoundary - close`; age
   and distance boundaries are inclusive.
5. Place a stop beyond the far FVG boundary plus `0.1 ATR`, with at least
   `1.5 ATR` distance; target `2R`, hold at most `20` bars, enable break-even,
   and disable trailing. Shared layers own all execution costs and liquidation.
6. Register and test Pine parity, formation versus retest timing, volume ratio,
   invalidation, closed-HTF causality, bounded lifecycle, ambiguity, barriers,
   next-open fills, cost accounting, and replay parity.
7. Research sequentially: frozen baseline; stop/buffer/R:R/hold sensitivity;
   volume ratio `0.50/0.55/0.60/0.65`; gap age and entry distance; detection
   sensitivity, Same Type and invalidation diagnostics; EMA ablation; then one
   chronological 70/30 out-of-sample validation.
8. Accept only with profit factor above `1.5`, Sharpe above `1.5`, drawdown below
   `12%`, at least `250` completed trades, positive expectancy after all costs,
   zero liquidations, positive doubled-cost results, performance above the 90th
   percentile of matched random-entry runs, out-of-sample decay below `20%`, at
   least three of four symbols positive out of sample, no symbol above 60% of
   aggregate profit, and separate profit/expectancy disclosure by volume and
   gap-age bucket.

Verification: Run the ported indicator and strategy tests; statically trace a
bullish and bearish first-rejection lifecycle; run a short `15m` backtest using
matched `1h` history and all configured costs.
