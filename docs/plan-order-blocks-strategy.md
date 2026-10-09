# Plan - Order Blocks Strategy

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). The
fixed `15m/4h` profile below remains a baseline; portability is pending.

Review: `parameters.ts` uses literal trigger/trend values and `strategy.ts`
loads a fixed pair. Block lifecycle calculations already take trigger candles.

Goal: Support `order-block-retest` on valid `T < C` profiles.
Out of scope: Changing block detection, first-retest rules or volume semantics.
Steps:

1. In `src/strategies/order-block-retest/parameters.ts`, validate enum-based
   `triggerTimeframe` and `trendTimeframe`, enforce `C > T` and resolve `[T, C]`.
2. In `signal.ts`, derive warm-up using the selected pair. Swing length,
   maximum zone age and hold count remain `T` bars; retain stabilization and
   confirmed-pivot latency before accepting a first retest.
3. Test default `15m/4h`, `1h/4h` and `4h/1d` profiles for first/repeated retests,
   invalidation, source volume, next-open execution and bounded/full parity.

Verification: Static trace a block confirmed at an hourly close, followed by
a later first retest. The historical pivot cannot be traded before confirmation;
an invalidated zone cannot revive after changing timeframe or rebuilding state.

## Original baseline specification

Source: https://www.tradingview.com/script/bLdpFVuq-Order-Blocks-Flux-Charts/

Goal: Add a backtest-capable `order-block-retest` strategy that trades the first
confirmed rejection of a fresh active order block.

Out of scope: Do not enter when a block forms, combine overlapping blocks,
trade repeated retests, convert failed blocks into breakers, or modify display
and volumetric rendering behavior.

Steps:

1. Require a Pine-parity `src/indicators/order-blocks.ts` with causal formation,
   direction, volume split, retest, and invalidation state before strategy work.
   Operate only in `src/strategies/order-block-retest/` plus registry files.
2. Add a strict schema with trigger `15m`, trend `4h`, Pine-default swing length
   and Wick invalidation, fixed notional `200 USDT`, leverage `2`, maximum zone
   age `192` bars, maximum entry distance `1 ATR`, and first-retest-only enabled.
3. LONG requires exactly one bullish active block receiving its first confirmed
   retest, the rejection close above the block, and a latest fully closed `4h`
   close above a rising EMA50. SHORT is symmetric below a bearish block. Fill at
   the next `15m` open.
4. Put the stop beyond the far block boundary plus `0.1 ATR`, never closer than
   `1.5 ATR`; target `2R`, maximum hold `20`, break-even enabled, trailing
   disabled. Shared execution owns fees, funding, spread, slippage, margin,
   liquidation, and time exits.
5. Reject invalidated, more-than-192-bar-old, ambiguous, repeated, or
   more-than-`1 ATR` distant blocks. Measure bullish distance as
   `close - zoneTop` and bearish distance as `zoneBottom - close`; age and
   distance boundaries are inclusive. Use raw active zones rather than
   display-combined or nearest-zone collections and never read a forming `4h`
   candle.
6. Register and test formation confirmation, first/repeated retests,
   invalidation, direction and volume audit values, `4h` causality, bounded
   lifecycle, ambiguity, barriers, next-open fills, costs, and replay parity.
7. Research sequentially: frozen baseline; stop/buffer/R:R/hold sensitivity;
   swing length, zone age and entry distance; Wick versus Close invalidation;
   volume and `4h` EMA ablations; then one chronological 70/30 out-of-sample
   validation. Keep breaker conversion outside this optimization surface.
8. Accept only with profit factor above `1.5`, Sharpe above `1.5`, drawdown below
   `12%`, at least `250` completed trades, positive expectancy after all costs,
   zero liquidations, positive doubled-cost results, performance above the 90th
   percentile of matched random-entry runs, out-of-sample decay below `20%`, at
   least three of four symbols positive out of sample, no symbol above 60% of
   aggregate profit, and separate profit/expectancy disclosure by block-size
   and zone-age bucket.

Verification: Run indicator and strategy tests; statically trace bullish and
bearish first-retest scenarios plus an invalidation case; run a short `15m`
backtest with complete `4h` history.
