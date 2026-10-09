# Plan - Support and Resistance MTF Strategy

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). Existing
`15m / 1h / 4h` values below describe the baseline; portability is pending.

Review: parameters lock the trigger, level tuple and trend to fixed values;
`strategy.ts` also loads a static triple. Each level's source interval is part
of its identity and confirmation latency.

Goal: Support `support-resistance-break-retest` with `T < C1 < C2`.
Out of scope: Adding a third level source or changing zone conflict handling.
Steps:

1. In `src/strategies/support-resistance-break-retest/parameters.ts`, accept an
   enum trigger and two distinct ordered higher level intervals. Keep the
   baseline convention `trendTimeframe = C2` and validate consistency.
2. Resolve `[T, C1, C2]`; in `signal.ts`, derive warm-up per source, preserve
   each pivot's source close/confirmation time and retain source timeframe in
   zone identities. Level age and break-to-retest delay remain trigger bars.
3. Test `15m/1h/4h`, `1h/4h/1d` and `4h/1d/1w`, including same-timestamp
   confirmations, role reversal, conflicting zones and prepared/full parity.

Verification: Static trace a newly confirmed daily resistance at 00:00: earlier
hourly triggers must not see it. Then trace a later break and first retest with
age and delay measured on the hourly trigger series, not daily source indices.

Profiles without two supported higher intervals are invalid; repeated source
values or a lower interval cannot silently satisfy the MTF requirement.

## Original baseline specification

Source: https://www.tradingview.com/script/3WgfGErN-Support-and-Resistance-MTF-Flux-Charts/

Goal: Add a backtest-capable `support-resistance-break-retest` strategy that
trades the first retest after a confirmed MTF level break and role reversal.

Out of scope: Do not trade unbroken support/resistance touches, use a third
level timeframe, select among simultaneous conflicting zones, change display
logic, or enable live trading.

Steps:

1. Require a Pine-parity `src/indicators/support-resistance-mtf.ts` with causal
   level, break, inverse-level, and retest events before strategy work. Operate
   only in `src/strategies/support-resistance-break-retest/` plus registry files.
2. Add a strict schema with `15m` execution, `1h` and `4h` level inputs, pivot
   range and strength mapped from the indicator defaults, dynamic ATR zone
   width, false-break volume filtering enabled, fixed notional `200 USDT`, and
   leverage `2`, maximum level age `192` trigger bars, maximum break-to-retest
   delay `32` bars, and maximum entry distance `1 ATR`.
3. Consume the indicator's confirmed break, inverse-level, and retest events as
   the only structural source; do not redetect a break in the strategy. LONG
   requires exactly one first retest of resistance converted into support,
   closing above the zone while the latest closed `4h` close is above a rising
   EMA50. SHORT applies the symmetric support-to-resistance sequence.
4. Fill at the next `15m` open. Put the stop beyond the far side of the retested
   zone plus `0.1 ATR`, never closer than `1.5 ATR`; target `2R`, maximum hold
   `20`, break-even enabled, trailing disabled. The shared engine owns fees,
   funding, spread, slippage, liquidation, and time-exit execution.
5. Reject a level older than 192 bars, a retest more than 32 bars after its
   break, a broken-again or ambiguous zone, and entry distance above `1 ATR`.
   Measure LONG distance as `close - zoneTop` and SHORT distance as
   `zoneBottom - close`; boundaries are inclusive. Use closed HTF candles and
   preserve the indicator's event timing.
6. Register the strategy and test MTF visibility, volume-confirmed breaks,
   false-break rejection, inversion, first retest, ambiguity, bounded state,
   barrier math, next-open execution, costs, and replay parity.
7. Research sequentially: frozen baseline; exit sensitivity; pivot/strength and
   zone-width sensitivity; level-age, retest-delay and entry-distance sensitivity;
   `1h`-only, `4h`-only and combined-level diagnostics; EMA and volume-filter
   ablations; then chronological 70/30 out-of-sample validation.
8. Accept only with profit factor above `1.5`, Sharpe above `1.5`, drawdown below
   `12%`, at least `250` completed trades, positive net expectancy, zero
   liquidations, positive doubled-cost results, performance above the 90th
   percentile of matched random-entry runs, out-of-sample decay below `20%`, at
   least three of four symbols positive out of sample, no symbol above 60% of
   aggregate profit, and separate profit/expectancy disclosure by `1h`/`4h`
   level source and market regime.

Verification: Run indicator and strategy tests; statically trace one
resistance-break-to-support retest and its bearish inverse; run a short `15m`
backtest with complete `1h` and `4h` data.
