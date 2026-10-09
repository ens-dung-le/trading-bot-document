# Strategy timeframe support

Status: reviewed specification, 2026-09-16. This document and the timeframe
sections linked below describe required implementation work, not completed
runtime support. Existing default profiles and historical results stay valid
only for their original timeframe combinations.

Goal: Make a selected timeframe flow consistently through strategy validation,
data loading, signal evaluation, execution and reporting.

Out of scope: Changing entry models, risk budgets, leverage defaults, historical
backtest results, or paper/live promotion rules.

## Review findings

1. **P1 - Changing a CLI flag does not change the strategy.** Most parameter
   schemas use `z.literal(...)`; `Strategy.requiredTimeframes` is a static
   array. `scripts/run-backtest.ts` separately checks the CLI value against
   parameters. Changing only one of these can reject a run or load the wrong
   history. See `src/strategies/strategy.interface.ts` and the individual plans.
2. **P1 - Some runtime paths still assume a particular timeframe.** DCA checks
   its parameters against a fixed `1m` array inside `evaluateDcaStopStraddle`.
   Multi-MA reads `15m`, `1h` and `4h` directly and gates entry branches on
   `15m`. Schema changes alone leave these paths incorrect or inactive.
3. **P1 - Calendar and intrabar strategies need additional data contracts.**
   Daily anchors, previous-day boxes and session opening ranges cannot be
   reconstructed by treating one daily or weekly candle as intraday history.
   Grid and DCA outcomes also depend on the unobserved order of OHLC touches.
4. **P1 - Weekly alignment is not established by the enum.**
   `alignTimestampToTimeframe` in `src/common/utilities/time.utility.ts` floors
   against the Unix epoch. For seven-day buckets that anchors on Thursday,
   whereas a Monday-aligned venue needs a different origin. Validate alignment
   against the selected exchange before enabling weekly aggregation/sync.
5. **P2 - Selection and loading differ by entry point.** Telegram and the worker
   special-case Box through `boxStrategyRequiredTimeframes`; the single-run
   CLI reads the static array. Batch, worker, CLI and runtime subscriptions
   need one parameter-aware dependency resolution path.
6. **P2 - A run can execute without validating its economic meaning.** Holding
   counts, forecast horizons and learned coefficients change meaning with
   timeframe. Missing funding, mixed sizing or a coarse fill model must be
   visible in reports; successful execution does not establish a profitable
   or validated model.

## Candidate timeframes and validity

Use the existing `Timeframe` enum as the single vocabulary:

`1m`, `3m`, `5m`, `15m`, `30m`, `1h`, `2h`, `4h`, `6h`, `8h`, `12h`, `1d`, `1w`.

These are candidates, not a claim that every exchange or every strategy already
supports every combination. A runnable profile needs all of the following:

- Strategy semantics permit the selected trigger and its dependencies.
- The exchange supplies each interval, or a verified aggregation path can
  build it from complete, aligned, closed lower-timeframe candles.
- History covers the evaluation interval and each dependency's warm-up.
- Backtest, paper and live support are reported separately; portability does
  not enable a currently disabled mode.

An invalid profile must return a specific reason before loading or queuing a
run. Do not silently substitute another timeframe or treat no-data as no-trades.

## Resolve the complete profile once

Let `T` be the signal/trigger timeframe, `C` a trend/context timeframe, `L` a
lower-timeframe input, and `E` the execution replay timeframe when distinct.

1. Merge the selected trigger into a copy of the stored parameters, then run
   the strategy's strict parser. Reject unknown fields. Explicit conflicting
   CLI and parameter values must fail with an actionable message.
2. Resolve strategy data dependencies from those validated parameters using
   the existing `Strategy`/registry boundary. Generalize the existing Box
   resolution pattern; do not create a second registry with duplicated rules.
   A legacy static array may describe the default profile, but must not drive
   a run using different parameters.
3. Pass the same resolved parameters and dependencies to coverage/download,
   CLI, batch, worker, backtest preparation and paper/live subscriptions.
   Stateful strategies keep their dedicated runner and runtime-owned replay
   timeframe instead of being converted into signal strategies.
4. Union strategy dependencies with required execution and reporting series.
   The existing Gaussian regime report uses `1h`/`4h`; those are reporting
   inputs, not additional strategy entry timeframes. Deduplicate by timeframe.
5. Persist the resolved trigger, contexts, lower inputs, execution interval,
   source/aggregation boundaries, parameters and source revision in the run.
   Include timeframe in prepared-state/cache identities and experiment keys.
   Reject in-place timeframe changes while positions or pending orders exist;
   a new flat run must rebuild warm-up and state.

Single-timeframe strategies depend on `[T]`. One-context strategies depend on
`[T, C]`, with `duration(C) > duration(T)`. Ordered two-context strategies need
`T < C1 < C2`. Scorers with lower input need `L < T < C`, complete child-bar
coverage, and compatible boundary alignment. Explicit context choices preserve
intent; do not silently choose a new trend model when the trigger changes.

Examples of valid intended profiles (subject to history):

| Family | Existing baseline example | Requested 1h example |
| --- | --- | --- |
| Single-frame | Triple SMA `5m` | `T=1h` |
| One context | Wick/Body `15m + 1h` | `T=1h, C=4h` |
| Two contexts | S/R `15m + 1h + 4h` | `T=1h, C1=4h, C2=1d` |
| Lower + higher inputs | FVG quality `1m / 15m / 1h` | `L=5m, T=1h, C=4h` |
| Previous-day box | `T=15m`, day source `15m` | `T=1h`, day source `15m` |
| Stateful replay | Grid `E=1m` | `E=1h`, explicitly coarse OHLC replay |

The enum has no timeframe below `1m` or above `1w`. Therefore a strict
lower-input scorer cannot currently accept `T=1m`, and strict HTF-confirmed
strategies cannot currently accept `T=1w`. Two-HTF profiles also need two
available intervals above `T`. True support at those edges requires adding
data/timeframe capabilities end to end, with boundary and causality tests;
disabling a gate or duplicating `T` is not an equivalent implementation.

## Periods, clocks and model meaning

- Indicator periods, swing confirmation, pattern lengths, zone age, cooldown,
  forecast horizon and holding counts remain counts of their owning series.
  Preserve existing numeric defaults. State that owner's timeframe explicitly.
- Record effective elapsed duration with the resolved count. For example,
  20 trigger bars means five hours at `15m` and twenty hours at `1h`. An
  existing zero-as-disabled sentinel must remain zero-as-disabled.
- A research profile that deliberately preserves elapsed duration must store
  the chosen new count and rounding policy. It is a separate explicit profile,
  not an automatic rescaling of all parameters. Do not round a sub-bar holding
  target up and claim the original holding horizon was preserved exactly.
- UTC-day anchors, session windows, funding timestamps and order timeouts
  specified in seconds remain calendar/clock based. Do not multiply them by
  the new candle duration. Session boundaries that split a candle require a
  complete finer series rather than prorating its OHLC/volume.
- Multi-Indicator's coefficients forecast the next bar at their training
  interval. A new `T` needs training/validation for that horizon with a frozen
  chronological split. Hourly coefficients at a different `T` are an explicitly
  unvalidated research transfer, not a supported trained profile.

## Closed candles, warm-up and fills

At decision time `d = trigger.openTimestamp + duration(T)`, all dependency
candles must satisfy `openTimestamp + duration(inputTimeframe) <= d`.
Appending future candles must not change earlier signals. Confirmed pivots
become usable at their confirmation close, never their historical pivot time.

Calculate warm-up per dependency, including strategy lookback, recursive
stabilization, zone lifecycle and reporting inputs. Reuse
`src/strategies/warm-up.ts` for duration conversion where appropriate, but also
verify actual closed-candle counts and calendar coverage. A single scalar
count cannot prove each MTF series has sufficient history. Load pre-roll
before the evaluation start; exclude it from trades and performance metrics.

Market signals fill no earlier than the next executable open after the signal
close, with configured costs. Default signal replay uses `E=T`. If a documented
strategy needs `E<T`, load it separately and activate orders at `d`; do not let
a finer candle earlier than `d` execute the new signal. Prepared replay and
incremental evaluation must return identical decisions.

Preserve each existing runner's fill policy. For grids, a close-created order
cannot fill retroactively from that same candle's high/low. For DCA or bracket
orders, ambiguous same-bar touches need the existing conservative precedence
or verified finer replay. Report the chosen execution interval and model.

## Per-strategy implementation map

All target entries below are pending implementation and verification. The
linked strategy section specifies its particular changes and tests.

| Strategy | Reviewed current restriction | Target / remaining semantic limit |
| --- | --- | --- |
| [Triple SMA](plan-triple-sma-directional-strategy.md#timeframe-portability-review-2026-09-16) | `5m` schema and array | Any valid `T`; SMA periods stay bars |
| [Five Gate](plan-five-gate-directional-strategy.md#timeframe-portability-review-2026-09-16) | `1m` schema and array | Any valid `T`; time exit stays bars |
| [Multi-Indicator](plan-multi-indicator-directional-strategy.md#timeframe-portability-review-2026-09-16) | `1h` schema and array | Any valid `T` with horizon-specific model validation |
| [Multi-MA Quantum](plan-multi-ma-quantum-timeframes.md) | `15m` trigger, fixed MTF branches | Configurable ordered contexts; remove hidden `15m` gates |
| [DCA](plan-dca-negative-paper-strategy.md#timeframe-portability-review-2026-09-16) | Parameter enum but runtime `1m` check | Any valid `T`; execution resolution disclosed |
| [Nadaraya](plan-nadaraya-smoothed-bollinger-bands-strategy.md#timeframe-portability-review-2026-09-16) | `5m + 15m` | `T < C`; one-sided smoothing retained |
| [VWAP Divergence](plan-vwap-divergence-strategy.md#timeframe-portability-review-2026-09-16) | `5m + 15m`, session anchor | `T < C`; compatible explicit calendar anchor |
| [Anchored VWAP](plan-anchored-vwap-suite-strategy.md#timeframe-portability-review-2026-09-16) | `15m + 1h`, Daily Open | Intraday `T < C`; daily/weekly needs a different anchor model |
| [Wick/Body](plan-wick-body-ratio-trend-forecast-strategy.md#timeframe-portability-review-2026-09-16) | `15m + 1h` | `T < C`; forecast length remains trigger bars |
| [Order Block](plan-order-blocks-strategy.md#timeframe-portability-review-2026-09-16) | `15m + 4h` | `T < C`; confirmed first retest |
| [Breaker Block](plan-breaker-blocks-strategy.md#timeframe-portability-review-2026-09-16) | `15m + 4h` | `T < C`; preserve formation/break/retest sequence |
| [Fair Value Gap](plan-fair-value-gaps-strategy.md#timeframe-portability-review-2026-09-16) | `15m + 1h` | `T < C`; gap age stays trigger bars |
| [ICT Suspension](plan-ict-suspension-blocks-strategy.md#timeframe-portability-review-2026-09-16) | `15m + 4h` | `T < C`; preserve lifecycle and bounded history |
| [FVG Quality](plan-fvg-quality-scorer-strategy.md#timeframe-portability-review-2026-09-16) | `1m / 15m / 1h` | `L < T < C`; complete volume inputs |
| [Support/Resistance](plan-support-resistance-mtf-strategy.md#timeframe-portability-review-2026-09-16) | `15m / 1h / 4h` | `T < C1 < C2`; source-specific confirmed levels |
| [Box](plan-box-strategy.md#timeframe-portability-review-2026-09-16) | Five trigger choices, `15m` day source | Intraday `T`; previous UTC-day levels expire with the day |
| [Trailing Grid](plan-trailing-geometric-grid-strategy.md#timeframe-portability-review-2026-09-16) | Default `1m`, dedicated replay | Any valid `E`; retain current 1x parity accounting |
| [Amount Scaler](plan-grid-amount-scaler-strategy.md#timeframe-portability-review-2026-09-16) | Default `1m`, dedicated replay | Any valid `E`; retain current spot/1x accounting |
| [Opening Range](plan-opening-range-breakout-strategy.md#timeframe-portability-review-2026-09-16) | Planned only; source chart <=15m | Session-valid `T`, finer range source; no cross-session candle |
| [Gaussian Trend](plan-gaussian-filter-trend-strategy.md#timeframe-portability-review-2026-09-16) | Planned only; no registered strategy | Any valid `T`; recursive state and closed-bar turns |

## Implementation order and verification gates

Steps:
1. Extend parameter-aware dependency resolution at `src/strategies/strategy.interface.ts`
   and `src/strategies/strategy.registry.ts`; connect the existing CLI, batch,
   worker, coverage and runtime consumers. Keep default-profile compatibility.
2. Apply each linked strategy's changes within its own module, reusing shared
   warm-up, closed-HTF selection, sizing and execution helpers.
3. Validate exchange interval availability/aggregation, especially weekly
   boundaries, and snapshot the resolved profile in the existing run records.
4. Execute the verification matrix below before marking a profile supported.

Verification: static trace and diff review for this documentation update.
Implementation acceptance requires the following executable checks later:

| Check | Expected evidence |
| --- | --- |
| Enum matrix | Every candidate `T` is accepted with a valid complete profile, or rejected with the precise semantic/data reason |
| Baseline regression | Original default parameters produce the same signals, fills, costs and PnL |
| Entry-point parity | CLI, batch, worker and applicable runtime paths resolve identical dependencies and decisions |
| Boundary causality | Before/at/after HTF close; no future candle leakage or early pivot visibility |
| Warm-up | One candle short in each dependency prevents evaluation; complete pre-roll enables it |
| State and restart | Trigger/profile in identities; no cache bleed; same restored expiry/cooldown behavior |
| Execution | Same-bar ambiguous touches, gaps, next-open fills, liquidation and funding timing remain consistent |
| Coarse frames | Calendar strategy rejection and missing LTF/HTF reported explicitly; weekly alignment fixture matches venue |
| Reporting | Signal/input/execution intervals, period endpoints, costs, funding coverage, sizing and model scope preserved |

Static traces reviewed for this specification:

- With `T=1h, C=4h`, the trigger `[10:00,11:00)` closes at 11:00.
  Context `[08:00,12:00)` is unavailable; the last eligible context is
  `[04:00,08:00)`. At 12:00, `[08:00,12:00)` becomes eligible.
- A grid order created at the 11:00 close cannot use the high/low of
  `[10:00,11:00)` to fill. With hourly replay it first becomes eligible in
  `[11:00,12:00)` under the existing resting-order policy.
- A Daily Open AVWAP on daily candles resets on every bar. There is no previous
  value within the same anchor for its reclaim rule, so `T=1d` is invalid for
  the current Daily Open model, not a successful zero-trade backtest.

## Comparable backtest reports

Keep each symbol, strategy profile and timeframe as a separate experiment.
Three-month and six-month windows overlap; do not sum their profits or label
their average profit factor as a portfolio statistic. Compare to frozen
out-of-sample periods and report insufficient sample size rather than relaxing
acceptance thresholds to make coarse timeframes pass.

For the requested $5,000 / $100 / 10x experiment, state whether $100 means
margin or notional. If it is margin, one entry requests $1,000 notional before
contract rounding. Do not multiply PnL by leverage after sizing. DCA adds,
Multi-Indicator position levels and grid contract ladders can change total
exposure; record them explicitly. Timeframe portability does not add futures
margin/liquidation support to the current spot-style grid simulators.

Report missing funding history separately from zero funding charged. Freeze
actual symbol specification as well as display labels. A report's screening
rule, such as PF >1.5 and drawdown <15%, is separate from the repository's full
acceptance gate; neither result alone certifies a strategy for live trading.
