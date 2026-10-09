# Plan - FVG Quality Scorer Strategy

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). The
`1m / 15m / 1h` triple below is a baseline, not universal runtime support.

Review: all three parameters are literals and `strategy.ts` has a fixed
dependency array. The scorer needs actual lower-timeframe volume and higher
context; relabeling a trigger without both inputs changes the score.

Goal: Support `fvg-quality-retest` with validated triples `L < T < C`.
Out of scope: Disabling a score axis, fabricating child volume or changing grades.
Steps:

1. In `src/strategies/fvg-quality-retest/parameters.ts`, validate all three
   enum values, strict interval order and compatible child-bar boundaries.
   Resolve the exact `[T, L, C]` dependencies for loading and subscriptions.
2. In `signal.ts`, retain the existing stabilization/window calculation but
   compute it using the resolved intervals. Require all closed child candles
   within each scored trigger interval; missing volume history cannot be zero.
3. Test baseline `1m/15m/1h` and hourly `5m/1h/4h` triples for identical scoring
   rules, first `Tested` lifecycle, grade boundaries and bounded/full parity.

Verification: Static trace the 10:00-11:00 hourly bar using twelve aligned
closed five-minute children. Its 10:55 child becomes eligible at 11:00; a
missing child or a child closing after 11:00 invalidates the complete input.

Current enum limits: `T=1m` lacks a smaller interval and `T=1w` lacks a larger
one. Reject those triples explicitly until the required data capabilities
exist. Using the trigger as its own LTF/HTF is not a supported substitute.

## Original baseline specification

Source: https://www.tradingview.com/script/olDO4cWI-FVG-Quality-Scorer-Flux-Charts/

Goal: Add a backtest-capable `fvg-quality-retest` strategy that trades the first
retest of a high-quality FVG produced by the existing TypeScript indicator.

Out of scope: Do not modify the scorer, trade new/partial/mitigated events, mix
several gaps into one signal, add scale-in, or enable live trading.

Steps:

1. Operate only in `src/strategies/fvg-quality-retest/`, plus the existing
   strategy registry files; keep `src/indicators/fvg-quality-scorer.ts`
   unchanged and preserve its four-axis score and lifecycle state machine.
2. Add a strict parameter schema with trigger `15m`, scorer LTF `1m`, scorer
   HTF `1h`, minimum trade grade `A`, fixed notional `200 USDT`, leverage `2`,
   maximum gap age `192` trigger bars, maximum entry distance `1 ATR`, maximum
   hold `20` trigger bars,
   `breakEvenStopEnabled: true`, and `trailingStopEnabled: false`.
3. On each closed `15m` candle, accept exactly one latest `Tested` event whose
   gap is still active, is at most 192 bars old, has grade `A`, and rejects
   toward its originating side. A bullish gap must close above its upper
   boundary with `close - upperBoundary <= 1 ATR` and maps to LONG; a bearish
   gap must close below its lower boundary with
   `lowerBoundary - close <= 1 ATR` and maps to SHORT. First test means the zone
   has no earlier `Tested` or `PartiallyFilled` event; a transition from
   `PartiallyFilled` back to `Tested` is not eligible. Fill at the next `15m`
   open.
4. Build a hybrid stop beyond the far FVG boundary plus `0.1 ATR`, with a
   minimum distance of `1.5 ATR`; place take profit at `2R`. Let the shared
   engine own sizing conversion, fees, funding, spread, slippage, break-even,
   liquidation, and time exit.
5. Make the lifecycle window finite with `maximumGapAgeCandleCount`, include the
   scorer's recursive-indicator stabilization prefix, and cover scorer range,
   swing/BOS state, `1m` volume aggregation, and the latest closed `1h` nesting
   candle. Prove identical latest decisions at grade thresholds before using a
   bounded window; otherwise use prepared full-series replay.
6. Register the strategy and add parameter, signal, strategy, registry, causal
   replay, cost-accounting, first-test, ambiguity, and bounded/full parity tests.
7. Research sequentially: frozen baseline; stop/buffer/R:R/hold sensitivity;
   grade `A` versus `A+B`, gap-age and entry-distance sensitivity; HTF/context
   ablation; then one untouched chronological 70/30 out-of-sample check.
8. Accept only with profit factor above `1.5`, Sharpe above `1.5`, drawdown below
   `12%`, at least `250` completed trades, positive expectancy after all costs,
   zero liquidations, positive doubled-cost results, performance above the 90th
   percentile of matched random-entry runs, out-of-sample decay below `20%`, at
   least three of four symbols positive out of sample, and no symbol above 60%
   of aggregate profit.

Verification: Run the existing FVG scorer tests plus the new strategy tests;
statically trace bullish and bearish first-test cases and run a short `15m`
database-backed backtest proving next-open fills and net costs.
