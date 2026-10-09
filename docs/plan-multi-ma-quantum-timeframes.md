# Plan - Multi-MA Quantum timeframe support

Status: reviewed specification, 2026-09-16; runtime changes pending. The
[v1 strategy report](multi-ma-quantum-v1-strategy-report.md) remains the record
of its existing profile. Apply the [shared timeframe contract](strategy-timeframe-support.md).

## Review findings

- `src/strategies/multi-ma-quantum/parameters.ts` fixes the trigger to `15m`
  and restricts `trendTimeframes` to specific tuples.
- `strategy.ts` declares `['15m', '1h', '4h']`, gates entry branches on
  `Timeframe.FifteenMinutes`, reads context arrays with `Timeframe.OneHour`
  and `Timeframe.FourHours`, and indexes a `15m` array in an entry helper.
- Both prepared and incremental evaluation consume these assumptions.
  Broadening validation without changing them could silently disable primary
  entries or match an index to a candle from a different series.
- `scripts/support/backtest-setup.ts` restricts the legacy MTF regime-report
  model to Multi-MA on `15m`. Keep old snapshot interpretation intact and use
  an explicitly compatible reporting model for other profiles, rather than
  bypassing that legacy guard or assuming a CLI flag updates the snapshot.

Goal: Apply the same Multi-MA entry/exit families to a selected trigger and
explicit context roles, including a `1h / 4h / 1d` research profile.

Out of scope: Changing MA/SuperTrend formulas, entry enablement defaults,
fixed-contract sizing, stop/target rules or historical results.

Steps:
1. In `src/strategies/multi-ma-quantum/parameters.ts`, validate the trigger from
   the shared enum and validate ordered, distinct context intervals from the
   existing `trendTimeframes` field. Retain its supported role/count semantics;
   require the actual contexts needed by every enabled entry family. Preserve
   the default `15m / 1h / 4h` profile and explicitly inventory any legacy
   tuples that need a compatibility migration rather than silently rewriting
   stored configurations.
2. In `strategy.ts`, replace `15m` branch gates and fixed `1h`/`4h` lookups
   with trigger/fast-context/slow-context roles from the validated profile.
   Reuse those roles in prepared-series building, pullback/retest detection,
   primary entries and incremental evaluation. Keep feature flags controlling
   entry families; selecting another trigger must not silently turn one off.
3. Derive `[T, ...trendTimeframes]` through shared resolution. Recalculate
   warm-up per input with the existing helper, and audit the legacy reporting
   snapshot constraint in `scripts/support/backtest-setup.ts`. Durations,
   array indices and decision timestamps must belong to the actual series.
4. Extend parameter, entry-family, preparation and replay tests with baseline
   `15m/1h/4h`, `1h/4h/1d`, and valid alternative role sets. Reject insufficient
   context roles and profiles with no available higher intervals. Verify
   original baseline signals, costs, sizing and exits remain unchanged.

Verification: Static trace an hourly trigger closing at 11:00. Its four-hour
fast context is the 04:00-08:00 bar; the 08:00-12:00 bar cannot be used. Its
daily context must also already be closed. Trace a primary entry with the same
feature flag at `15m` and `1h`: neither path may disappear due to a literal
timeframe comparison. Prepared and incremental decisions must match for each
entry family, including at context close boundaries and after restart.

The current enum cannot supply contexts above `1w`. Full weekly-trigger MTF
support needs longer interval/data capabilities and their alignment tests;
using the trigger itself as a higher-timeframe context is not equivalent.
