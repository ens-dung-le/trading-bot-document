# Market regime analysis specification

## Version 3 addendum: Gaussian 1h/4h direction model

Newly queued backtests and live analytics use
`market-regime-v3-gaussian-mtf-1h-4h` for every strategy. Regime detection is
market-owned analytics and is no longer selected from the strategy name.

- The exact recursive Gaussian filter, adaptive ATR width and Efficiency Ratio
  from `Gaussian Filter Trend [QuantAlgo]` are evaluated on closed candles.
- `4h` owns structural direction and volatility; `1h` confirms it.
- Matching `1h` and `4h` directions produce `uptrend` or `downtrend`.
- Opposing directions produce `sideways`; a sideways `1h` pullback may retain
  the directional `4h` structure with reduced confidence.
- On each source timeframe, low Efficiency Ratio after the stepped Gaussian
  line remains flat classifies chop as `sideways`. The default is ER `<= 0.25`
  after two flat source candles.
- Historical replay and live processing share the same incremental classifier.
  Appending future candles must not change an earlier label.
- Backtests continue to persist the resulting contiguous periods in
  `backtest_regime_periods`, keyed to the run, so every report retains the exact
  periods and model snapshot it used.
- Live mode restores Gaussian state from stored `1h` and `4h` history, then
  advances it only when a new trigger candle closes. Regime analytics do not
  change strategy execution in this version.

Versions 1 and 2 remain supported for queued jobs and stored report replay.

## Version 2 addendum: 1h/4h direction model

For `multi-ma-quantum` with its `15m` trigger, newly queued runs use
`market-regime-v2-mtf-1h-4h`. Strategy evaluation remains on `15m`; only the
analytics classifier changes.

- `4h` is the structural direction and volatility anchor.
- `1h` confirms the anchor direction.
- Matching directions use that direction.
- A `1h` sideways reading inside a directional `4h` structure retains the
  `4h` direction with reduced confidence.
- Opposing directional readings are classified as `sideways` until `4h`
  confirms the reversal.
- Both higher-timeframe candles must have closed by the `15m` decision time.
- The resulting label is carried forward across `15m` candles for trade and
  mark-to-market attribution.

Version 1 remains replayable for historical snapshots and other strategy profiles.

> **Status: approved for implementation (2026-09-09).**
> This document defines version 1 of market-regime classification and backtest
> reporting. Every numeric value described as a default is configurable and is
> snapshotted with each backtest. The defaults are starting hypotheses, not
> universal trading thresholds.

## 1. Objective

The bot must not be switched off merely because the market is trending or
volatile.
Instead of treating a large candle range as a reason to switch a strategy off,
the system classifies each active chart into:

- one direction: `uptrend`, `sideways`, or `downtrend`; and
- one volatility level: `normal`, `high`, or `extreme`.

The resulting nine regimes are analytics in version 1. They explain where a
strategy made or lost money, but they do not change its signals, size, leverage,
orders, or enabled state.

Operational and account-risk protection remains independent. Missing or stale
data, an unknown exchange/order state, a missing protective stop, a failed
cancel, a circuit-breaker limit, or another hard-safety condition may still
block exposure or de-risk a position. A regime label, including
`extreme`, is not by itself a hard-safety event.

## 2. Scope and non-goals

### 2.1 In scope

1. One causal classifier for every active `{symbol, triggerTimeframe}` chart.
2. The same classifier, state transitions, configuration, and candle visibility
   rules in backtest, paper, and live execution.
3. Contiguous regime periods and confidence values for diagnostics.
4. Mark-to-market profit-and-loss attribution to the regime active through each
   evaluated interval.
5. Direction, direction-by-volatility, period, and transition statistics in the
   primary backtest report.
6. Entry-regime attribution in a diagnostic appendix and the trade CSV only.
7. Additive persistence and backward-compatible rendering of historical runs.

### 2.2 Out of scope for version 1

- Selecting, enabling, or disabling a strategy from a regime label.
- Changing position size, leverage, stops, or scale-in behavior from a regime.
- Predicting the next regime.
- Treating the classifier as a crash detector or replacement for the hard
  safety guard.
- Using tick, order-book, mark-price, funding, or open-interest data. Version 1
  intentionally uses only closed OHLCV candles on the chart's trigger
  timeframe, so existing candle history can be replayed consistently.
- Claiming that profitable historical cells will remain profitable live.

Any future regime-dependent execution policy requires a separate proposal,
out-of-sample evidence, risk limits, and an independently deployable feature
flag.

## 3. Safety and causality invariants

The following rules are mandatory:

1. A classification at candle `t` may use candle `t` and earlier closed candles,
   but never a later candle.
2. The current NATR sample is not part of its own percentile baseline.
3. Candles are observed once, in increasing open-timestamp order, after the
   market-data layer accepts them as closed.
4. State is isolated by the exact `{symbol, triggerTimeframe}` key. No symbol or
   timeframe contributes to another chart's indicators or history.
5. Replaying the same ordered candles and configuration must produce identical
   observations and periods after a process restart.
6. Backtest, paper, and live use one shared classifier implementation rather
   than separately reimplementing the formulas.
7. Enabling regime analytics must not alter trades, rejected signals, fills,
   fees, funding, equity, or risk decisions.
8. The classifier must not call entry `block`/`unblock`, cancel orders, or alter
   a position. Hard-safety controls retain those responsibilities.
9. Warm-up is reported as unclassified. It is not a fourth market regime and
   must not be silently counted as `sideways` or `normal`.
10. All configuration and the classifier method version are frozen when a
    backtest job is queued and persisted with the finished run.
11. The adapter must enforce the expected timeframe cadence. A missing candle
    is repaired or surfaced by data-quality/hard-safety handling; it must not be
    silently treated as one unusually long normal interval by the classifier.

## 4. Classification model

### 4.1 Public states

```text
MarketDirection = uptrend | sideways | downtrend
VolatilityLevel = normal | high | extreme

MarketRegime = { direction, volatility }
```

An observation that is not ready has `isReady=false`, `regime=null`, and null
confidence values. Once ready, each observation belongs to exactly one of the
nine direction-by-volatility combinations.

### 4.2 Configurable version 1 defaults

| Configuration | Default | Meaning |
|---|---:|---|
| `adxPeriod` | `14` | Wilder ADX lookback. |
| `emaPeriod` | `50` | Close-price EMA used for direction. |
| `atrPeriod` | `14` | Wilder ATR lookback used by direction and NATR. |
| `emaSlopeLookbackCandleCount` | `5` | Distance between the current and prior EMA in the slope calculation. |
| `trendEntryAdx` | `25` | Minimum ADX for entry into a directional state. |
| `trendExitAdx` | `20` | ADX at or below which a latched trend may exit. |
| `trendEntryNormalizedSlope` | `0.25` | Absolute normalized slope required to enter a trend. |
| `trendExitNormalizedSlope` | `0.05` | Signed threshold used to remain in a latched trend. |
| `directionConfirmationCandleCount` | `2` | Consecutive closed-candle candidates required for a direction change. |
| `volatilityLookbackCandleCount` | `252` | Prior valid NATR samples in the rolling distribution. |
| `highVolatilityPercentileRank` | `80` | Start of the `high` category. |
| `extremeVolatilityPercentileRank` | `95` | Start of the `extreme` category. |

Configuration validation must require positive integer indicator/lookback
periods and enforce:

```text
0 <= trendExitAdx < trendEntryAdx
0 <= trendExitNormalizedSlope < trendEntryNormalizedSlope
0 < highVolatilityPercentileRank < extremeVolatilityPercentileRank < 100
directionConfirmationCandleCount >= 1
```

The immutable version identifier for these formulas and semantics is
`market-regime-v1`. Changing any default requires a new configuration snapshot.
A material formula or transition change requires a new classifier method
version as well.

### 4.3 Indicator definitions

All indicators use only closed candles in the chart's trigger timeframe.

True range and Wilder ATR are:

```text
TR[t] = max(
  high[t] - low[t],
  abs(high[t] - close[t-1]),
  abs(low[t] - close[t-1])
)

ATR14[t] = WilderAverage(TR, 14)[t]
NATR[t]  = ATR14[t] / close[t]
```

EMA and the ATR-normalized EMA displacement are:

```text
EMA50[t] = EMA(close, 50)[t]

normalizedSlope[t] =
  (EMA50[t] - EMA50[t-5]) / ATR14[t]
```

The slope is the total EMA displacement over the configured five-candle
lookback expressed in current ATR units. It is deliberately **not** divided by
five. A positive value points upward and a negative value points downward.

ADX uses the existing Wilder implementation with period 14. Version 1 uses ADX
for trend strength and the signed EMA slope for direction. `+DI/-DI` agreement
is not an additional version 1 condition; it may be evaluated later as a
versioned refinement.

If a required indicator is null, non-finite, or has an invalid denominator,
the observation is not ready and must not invent a regime.

### 4.4 Direction state machine and hysteresis

For a ready candle, define:

```text
strongUp   = ADX >= 25 and normalizedSlope >=  0.25
strongDown = ADX >= 25 and normalizedSlope <= -0.25

upStillValid   = ADX > 20 and normalizedSlope >  0.05
downStillValid = ADX > 20 and normalizedSlope < -0.05
```

The numeric constants above are the configurable defaults in section 4.2.
Boundary behavior is intentional: entry thresholds are inclusive; reaching an
exit threshold ends the continuation condition.

The desired direction is determined from the currently latched direction:

| Current direction | Condition | Desired direction |
|---|---|---|
| Uninitialized | `strongUp` | `uptrend` |
| Uninitialized | `strongDown` | `downtrend` |
| Uninitialized | otherwise | `sideways` |
| `sideways` | `strongUp` | `uptrend` |
| `sideways` | `strongDown` | `downtrend` |
| `sideways` | otherwise | `sideways` |
| `uptrend` | `strongDown` | `downtrend` |
| `uptrend` | not `upStillValid` | `sideways` |
| `uptrend` | otherwise | `uptrend` |
| `downtrend` | `strongUp` | `uptrend` |
| `downtrend` | not `downStillValid` | `sideways` |
| `downtrend` | otherwise | `downtrend` |

The first ready direction is assigned immediately. Every later change requires
the same desired direction for two consecutive closed candles by default:

- A desired direction equal to the current state clears the pending candidate.
- A desired direction different from the pending candidate replaces it and
  starts its count at one.
- A matching pending candidate increments its count.
- The state changes only when the count reaches
  `directionConfirmationCandleCount`.

This permits a confirmed direct `uptrend -> downtrend` or
`downtrend -> uptrend` transition when the full opposite entry conditions hold.

### 4.5 Volatility percentile

The current NATR is ranked against exactly the previous 252 valid NATR samples
by default:

```text
rank[t] =
  100 * count(previousNatr < NATR[t]) / volatilityLookbackCandleCount
```

The comparison is strictly lower. Equal historical values do not count below
the current value. No interpolation is used. After classification, the current
NATR is appended and the oldest value is evicted so the history remains fixed
length.

```text
rank < 80        -> normal
80 <= rank < 95  -> high
rank >= 95       -> extreme
```

Volatility changes immediately on a closed candle; it does not use direction's
two-candle confirmation. This is a relative-volatility label for the same
symbol and timeframe, not a claim that two different charts with the same rank
have equal absolute risk.

### 4.6 Warm-up

The default classifier needs:

```text
max(
  2 * adxPeriod,
  emaPeriod + emaSlopeLookbackCandleCount,
  atrPeriod + volatilityLookbackCandleCount
) = 266 closed candles
```

These candles are pre-roll history. A backtest's requested evaluation period
must not be shortened to obtain them. The data loader should fetch them before
the evaluation start. If sufficient valid pre-roll cannot be obtained, the run
must fail with a clear classifier warm-up/data-coverage error rather than use
future evaluation candles as its baseline.

Live and paper runtimes seed the classifier with the same amount of ordered
closed history before publishing ready observations. Until then, the chart is
reported as warming up, while strategy behavior remains governed by its normal
indicator and safety rules.

### 4.7 Confidence

Confidence is diagnostic and must never be interpreted as a calibrated
probability.

Define `clamp01(x) = min(1, max(0, x))` and:

```text
adxEvidence = clamp01((ADX - trendExitAdx) /
                      (trendEntryAdx - trendExitAdx))

slopeEvidence = clamp01((abs(normalizedSlope) - trendExitNormalizedSlope) /
                        (trendEntryNormalizedSlope - trendExitNormalizedSlope))

trendEvidence = min(adxEvidence, slopeEvidence)
```

Direction confidence is:

- `1 - trendEvidence` for `sideways`;
- `trendEvidence` for `uptrend` when the slope is positive, otherwise zero; and
- `trendEvidence` for `downtrend` when the slope is negative, otherwise zero.

Let `H = highVolatilityPercentileRank` and
`E = extremeVolatilityPercentileRank`. Volatility confidence measures distance
from the closest category boundary:

```text
normal:  clamp01((H - rank) / H)
high:    clamp01(min(
           (rank - H) / ((E - H) / 2),
           (E - rank) / ((E - H) / 2)
         ))
extreme: clamp01((rank - E) / (100 - E))
```

With the defaults, the three denominators are 80, 7.5, and 5. The reported
combined `confidence` is:

```text
confidence = min(directionConfidence, volatilityConfidence)
```

A value near zero is expected at a boundary and should be displayed as low
confidence, not discarded or reassigned.

## 5. Observation and period contract

Each processed candle produces an observation conceptually equivalent to:

```ts
interface MarketRegimeObservation {
  candleTimestamp: number; // candle open timestamp
  isReady: boolean;
  regime: {
    direction: "uptrend" | "sideways" | "downtrend";
    volatility: "normal" | "high" | "extreme";
  } | null;
  confidence: number | null;
  directionConfidence: number | null;
  volatilityConfidence: number | null;
  transition: MarketRegimeTransition | null;
  metrics: {
    averageDirectionalIndex: number | null;
    positiveDirectionalIndicator: number | null;
    negativeDirectionalIndicator: number | null;
    exponentialMovingAverage: number | null;
    averageTrueRange: number | null;
    normalizedEmaSlope: number | null;
    normalizedAverageTrueRange: number | null;
    volatilityPercentileRank: number | null;
    pendingDirection: MarketDirection | null;
    pendingDirectionCandleCount: number;
  };
}
```

Exact TypeScript naming may follow repository conventions, but these values and
semantics are required.

`candleTimestamp` identifies the candle in stored period/report data. The
adapter derives `effectiveTimestamp = candleTimestamp + timeframeDuration`; it
is the earliest instant at which execution code may use the observation.

A regime period is a maximal contiguous sequence of ready observations with
the same direction and volatility:

```ts
interface MarketRegimePeriod {
  startedAtTimestamp: number; // first candle open timestamp
  endedAtTimestamp: number | null; // final candle open timestamp
  direction: MarketDirection;
  volatility: VolatilityLevel;
  candleCount: number;
  averageConfidence: number;
  minimumConfidence: number;
}
```

If either axis changes, the old period ends at the previous classified candle's
open timestamp and the new period starts at the current candle's open
timestamp. The current live period has a null end; a finished backtest closes
its aggregate/display interval at the final eligible evaluation candle while
the raw final period may retain a null end to mean "still current at end of
data." Because `endedAtTimestamp` is inclusive, period duration is
`candleCount * timeframeDuration`, not `endedAtTimestamp - startedAtTimestamp`.
Warm-up observations never form periods.

## 6. Runtime integration

### 6.1 Instance lifecycle

- Create one classifier per active chart, keyed by
  `{symbol, triggerTimeframe}`.
- Seed it from ordered closed trigger-timeframe candles when the chart starts or
  restarts.
- Observe only accepted closed trigger-timeframe candles inside the existing
  serialized market-event flow.
- Ignore forming-candle/ticker updates for version 1 classification.
- Reset and reseed when the symbol, trigger timeframe, configuration, or runtime
  generation changes.
- Log regime transitions with symbol, timeframe, old/new labels, confidence,
  and underlying metrics. Notifications may be added later without changing
  trading behavior.

### 6.2 Separation from hard safety

The legacy market-anomaly entry block is retired for new version 1 runs. If its
range-spike detector is temporarily retained during migration, its result is
diagnostic-only and must not contribute the `market anomaly` operational block
reason.

The independent hard-safety path remains able to act on, at minimum:

- stale, missing, invalid, or out-of-order market data;
- exchange connectivity or health failure;
- an unknown order/position state or failed reconciliation;
- a protective stop that is absent or cannot be verified;
- cancel/reduce-only actions that cannot be confirmed; and
- per-position, account exposure, margin, drawdown, or liquidation-buffer
  limits.

No `uptrend`, `sideways`, `downtrend`, `normal`, `high`, or `extreme` label can
clear a hard-safety block. Clearing and recovery remain owned by the safety
condition that installed the block.

## 7. Backtest integration and attribution

### 7.1 Walk-forward processing

For each chart, the backtest must:

1. Load pre-roll candles before the requested evaluation start.
2. Replay them chronologically to warm indicators and the prior-NATR window,
   without including them in report P&L.
3. Process each evaluation candle once through the same classifier used live.
4. Attach observations to equity intervals and execution events using the two
   explicit timestamp rules in sections 7.2 and 7.3.
5. Persist the method version and complete configuration snapshot.

Appending future candles to a data set must never change any earlier regime,
confidence, transition, or period boundary.

### 7.2 Primary mark-to-market attribution

Whole-trade P&L is not the primary regime measure because a position can span
several regimes. For two consecutive equity observations, calculate:

```text
equityDelta[t] = equity[t] - equity[t-1]
```

Attribute the delta from the previous candle close through the current candle
close to the regime produced by the **current closed candle**. This is
descriptive attribution: the current candle's completed OHLC is allowed to
describe the interval that just ended, but that label was not available for a
decision at the start of the interval. Fees, funding, modeled slippage, and
realized and unrealized movement reflected by the current equity point stay in
that same delta. No interval may be attributed twice or omitted. If the current
observation is not ready, its delta is reported as unclassified rather than
assigned to one of the nine regimes.

The reconciliation invariant is:

```text
sum(all classified MTM P&L) + unclassified MTM P&L
  = ending equity - starting equity
```

within the repository's stated floating-point tolerance.

For reporting:

- Market-time share counts eligible trigger-timeframe intervals.
- Asset return uses close-to-close log returns assigned with the same interval
  rule, then compounds the assigned returns for each group.
- Entry count means an entry occurred while that regime was active; it does not
  assign the trade's later P&L to that regime.
- Worst drawdown for a group is the largest peak-to-trough decline inside any
  one of its contiguous periods, resetting the local peak at each period start.

### 7.3 Entry attribution is diagnostic only

For execution-event annotations, use the latest ready regime whose effective
close timestamp is less than or equal to the event timestamp. An order filled
at a candle open therefore receives the previous closed candle's regime, not a
label calculated later from the candle in which it fills.

For diagnostics, the full realized trade result is grouped by that entry
regime. This answers whether entries opened under a label later performed well;
it does not describe where the position actually made or lost money.

The diagnostic table may contain trade count, win rate, profit factor, net P&L,
and expectancy per trade. It must be titled:

```text
REGIME ENTRY ATTRIBUTION (DIAGNOSTIC)
```

and include a note that positions may cross regimes.

### 7.4 Progressive walk-forward runs

The classifier has two different kinds of adaptation and they must not be
confused:

- Indicator state and the rolling 252-sample NATR distribution update online
  after every closed candle.
- Formula choices and configuration thresholds remain frozen for the entire
  deployed/backtested version.

When thresholds are researched, choose them using only a chronological training
window, freeze them, and evaluate the immediately following window. Then roll
the train/evaluation boundary forward and repeat. The report must distinguish
calibration data from aggregated out-of-sample evaluation data. It must never
select thresholds from the full period and report that same period as an
out-of-sample result.

## 8. Report contract

### 8.1 Telegram headline and primary summary

The normal bot headline remains compact. The primary human-readable backtest
summary adds these sections when regime analysis is available:

1. **Classifier configuration**: method version, timeframe, formula periods,
   thresholds, percentile window, and warm-up count.
2. **Classifier quality**: classified coverage, period count, median period
   duration, median candidate-to-confirmation lag, low-confidence market time,
   and short-period count.
3. **Performance by direction**: one row each for uptrend, sideways, and
   downtrend with period count, market-time share, asset return, entries, MTM
   net P&L, and worst intra-period drawdown.
4. **Direction by volatility**: nine rows/cells with market-time share, MTM P&L
   contribution, and worst segment drawdown. Segment drawdown is the largest
   local peak-to-trough decline inside a contiguous period of that composite
   regime.
5. **Notable periods**: the best and worst contiguous periods with UTC start,
   end, direction, volatility, confidence, candle count, entry count, and MTM
   P&L.
6. **Direction transitions**: transition count, median confirmation lag, and
   MTM P&L over the first three candles after confirmation. The three-candle
   horizon is a configurable reporting default.
7. **Automatic observations**: factual concentration warnings such as a
   negative regime expectancy or losses concentrated around transitions. These
   are diagnostics, not instructions to disable a strategy.

Metrics without enough samples must display `insufficient sample` instead of a
misleading zero or infinity. The report must disclose unclassified time/P&L and
must not imply 100% coverage when pre-roll or data gaps prevented it.

The `REGIME ENTRY ATTRIBUTION (DIAGNOSTIC)` table is deliberately excluded from
the Telegram headline and the primary summary sections.

The initial reporting defaults are also configurable and snapshotted:

| Reporting setting | Default | Use |
|---|---:|---|
| `lowConfidenceThreshold` | `0.25` | Count ready intervals below this combined confidence. |
| `shortPeriodMaximumCandleCount` | `2` | Identify likely label churn. |
| `transitionPnlHorizonCandleCount` | `3` | Measure P&L immediately after a confirmed direction transition. |
| `notablePeriodCountPerTail` | `3` | Show the three best and three worst periods. |

These values control presentation and diagnostics only. They cannot affect the
classifier or trading behavior.

For this table, a direction transition requires `from.direction !=
to.direction`; a volatility-only period change is excluded. Confirmation lag is
the inclusive number of consecutive candidate candles from the first candidate
through the confirming candle. Post-transition P&L uses the next three complete
equity intervals after the transition's effective close, excluding the candle
whose close confirmed the transition. Truncated end-of-data windows disclose
their observed candle count and are not presented as full three-candle samples.

### 8.2 Detailed attachment

The downloaded `backtest-result.txt` appends
`REGIME ENTRY ATTRIBUTION (DIAGNOSTIC)` after the primary sections. A separate
`regime-periods.csv` should expose every contiguous period for audit, including:

```text
start_timestamp,end_timestamp,direction,volatility,candle_count,
minimum_confidence,average_confidence,entry_count,asset_return,
mark_to_market_profit_and_loss,worst_drawdown
```

Exact line wrapping may differ, but columns and units must be documented and
stable within a report-format version.

### 8.3 Trade CSV

`trades.csv` retains all current columns and adds nullable regime context after
the existing columns:

```text
entry_regime,entry_volatility,entry_regime_confidence,
entry_regime_candle_timestamp,exit_regime,exit_volatility,
exit_regime_confidence,exit_regime_candle_timestamp,
regime_transition_count,regime_model_version
```

`entry_regime` and `exit_regime` contain the direction value. The paired
volatility column completes the composite regime. The regime candle timestamp
identifies the closed candle that supplied the causally available annotation;
`regime_transition_count` counts confirmed composite-regime changes while the
position was open. Fields are empty if the classifier was not ready at the
event timestamp. These columns support the appendix and later auditing; they do
not change stored or reported trade P&L.

## 9. Persistence and compatibility

### 9.1 Persisted data

At minimum, a finished run must retain:

- classifier method/report-format versions and its complete configuration;
- regime summary data needed to reproduce the primary report;
- all regime periods, including period timestamps, labels, confidence, and MTM
  aggregates; and
- nullable entry/exit regime labels and confidence on each trade.

Configuration cannot be resolved from current environment variables when an
old report is downloaded. The stored snapshot is authoritative.

Per-candle observations remain ephemeral in version 1; persisting one row for
every observation would duplicate candle history. The result-level contract is
additive and optional for compatibility:

```ts
interface BacktestResult {
  marketRegimePeriods?: readonly MarketRegimePeriod[];
  marketRegimePerformance?: MarketRegimePerformance;
}

interface BacktestTrade {
  entryRegime?: RegimeEventContext | null;
  exitRegime?: RegimeEventContext | null;
  regimeTransitionCount?: number;
  regimeModelVersion?: string;
}

interface RegimeEventContext {
  direction: MarketDirection;
  volatility: VolatilityLevel;
  confidence: number;
  candleTimestamp: number;
}
```

### 9.2 Database migration

Migration is additive and recoverable. The intended relational shape is:

1. Add nullable `market_regime_model_version`,
   `market_regime_configuration JSONB`, and `market_regime_summary JSONB` to
   `backtest_runs`.
2. Add nullable entry/exit direction, volatility, confidence, and source-candle
   timestamp columns to `backtest_trades`, plus nullable
   `regime_transition_count` and `regime_model_version`.
3. Add `backtest_regime_periods`, keyed to `backtest_runs`, with direction,
   volatility, start/end timestamps, candle count, confidence aggregates, asset
   return, MTM P&L, and segment drawdown.
4. Do not rewrite historical trades, reports, or equity points in place.

Old rows therefore remain valid with null regime data. New code must not assume
that every queued job or stored run has the new snapshot.

### 9.3 Job and report compatibility

- New jobs freeze the regime configuration and method version in the existing
  execution-configuration JSON when queued.
- Jobs queued before the migration either complete under an explicitly
  versioned legacy execution policy or produce no regime section; they must not
  silently inherit whichever thresholds are current when a worker starts.
- Existing finished runs render their original report without regime sections,
  and their current `trades.csv` layout remains downloadable.
- A report generated from new persisted rows includes the new columns and
  sections without requiring access to live candle history.
- Historical `marketAnomalyGuard` configuration and anomaly-period fields stay
  readable for legacy reports. They are not relabeled as market regimes.
- Reclassifying an old run is a separate, explicitly versioned derived analysis;
  it must not mutate or pretend to reproduce the original run.

### 9.4 Deployment sequence

1. Apply additive database migration and deploy readers tolerant of nulls.
2. Deploy classifier and persistence behind the new execution/report version.
3. Run backtests with analytics enabled and disabled; verify identical trading
   results and complete attribution reconciliation.
4. Enable paper/shadow observation and compare live incremental observations
   with an offline replay of the same candles.
5. Remove the legacy anomaly-based entry block for new runs only after the hard
   safety regression suite passes.
6. Enable regime reporting for live charts. This rollout still does not allow
   regimes to alter execution.

## 10. Verification and acceptance criteria

### 10.1 Classifier unit tests

- Indicator fixtures verify EMA50, Wilder ATR14, NATR, and ADX14 values against
  an independent reference within documented tolerance.
- The first ready observation occurs only after the required default 266 closed
  candles, and earlier observations have a null regime.
- Prefix invariance proves adding future candles does not modify prior output.
- The percentile test proves the current sample is excluded, history is capped
  at 252, and ties use the strict-lower rule.
- Boundary tests cover ranks exactly at 80 and 95 and every inclusive/exclusive
  direction threshold.
- Hysteresis tests cover one-candle noise, candidate replacement/reset,
  sideways entry, trend exit, and confirmed direct reversal.
- Seed/replay/reset produces the same current state, observations, and periods
  as uninterrupted incremental observation.
- Invalid/non-finite candle or indicator inputs never produce a ready regime.
- Confidence remains finite and inside `[0, 1]` at every boundary.

### 10.2 Integration and parity tests

- The same candle fixture and configuration produce identical observations and
  period boundaries in backtest, paper, and live adapters.
- Restarting after any candle and reseeding from history produces the same next
  transition as an uninterrupted runtime.
- Multiple symbols and timeframes do not contaminate one another.
- Duplicate/recovered candles do not create duplicate observations or periods.
- Analytics enabled versus disabled produces identical trades, rejections,
  costs, funding, liquidation count, and equity curve.
- `extreme` volatility alone never installs or clears an operational entry
  block.
- Existing hard-safety conditions still block/de-risk exactly as before.

### 10.3 Attribution and report tests

- Regime MTM P&L plus unclassified P&L reconciles exactly to total net equity
  change within the declared numeric tolerance.
- A position spanning multiple regimes contributes MTM deltas to each period,
  while its full trade result appears only in diagnostic entry attribution.
- Period boundaries, market-time shares, transition windows, best/worst period,
  and drawdown resets have deterministic fixtures.
- Empty-trade runs, an end-of-data open position, funding-only changes, duplicate
  equity timestamps, and partial warm-up render without `NaN` or misleading
  totals.
- The primary summary and Telegram headline do not contain the entry-attribution
  table; the detailed text and trade CSV do.
- Reports reconstructed from persisted rows match the originals without
  rereading candles.
- Legacy jobs/runs with null regime fields keep their previous report and CSV
  behavior.

### 10.4 Walk-forward review before using results operationally

The classifier can ship as analytics after the deterministic tests above pass.
Before regime results influence capital or strategy behavior, validation must
also include:

1. Chronological calibration, validation, and untouched out-of-sample windows;
   no random candle split.
2. Multiple symbols, trigger timeframes, bull, bear, sideways, and crisis
   periods.
3. Sensitivity analysis around every direction threshold, percentile boundary,
   lookback, and confirmation count.
4. Period stability and churn metrics, including time near a confidence
   boundary and very short periods.
5. Minimum sample disclosure for each of the nine cells and for each transition.
6. Comparison against simple challenger definitions, such as EMA-only or
   return/volatility quantile regimes.
7. A fresh paper/shadow window that was not used to select defaults.

No cell's historical profitability is sufficient on its own to authorize live
strategy switching. That decision requires a later spec with explicit loss
limits, false-discovery controls, and staged canary deployment.

## 11. Interpretation notes

- ADX measures strength, not direction. Direction comes from the signed EMA
  displacement.
- NATR percentile is relative to recent history. `Extreme` means extreme versus
  the previous rolling window, not necessarily a market-wide crisis.
- Hysteresis and confirmation intentionally delay transitions to reduce label
  churn. Transition-lag P&L must therefore be visible in the report.
- The 252-sample window represents very different wall-clock durations on
  different timeframes. Results must always name the timeframe, and calibration
  may choose different snapshotted values later.
- Confidence expresses distance from configured thresholds. It is not the
  probability that a regime is objectively correct.
- Regimes describe historical conditions; they do not make the strategy robust
  to liquidity gaps, exchange outages, liquidation mechanics, or execution
  failure. Those remain hard-safety and execution-model concerns.

## 12. Method basis

- J. Welles Wilder Jr., *New Concepts in Technical Trading Systems* (1978), is
  the basis for True Range, ATR, directional movement, and ADX smoothing.
- EMA direction, ATR normalization, the 252-sample empirical percentile,
  hysteresis, confidence mapping, and all version 1 thresholds are this
  project's testable model design. No cited source establishes those defaults
  as optimal for KuCoin Futures, a particular symbol, or a particular
  timeframe.
- [EU MiFID II RTS 6](https://eur-lex.europa.eu/eli/reg_del/2017/589/oj/eng)
  and the
  [FIA automated-trading risk-control practices](https://www.fia.org/sites/default/files/2024-07/FIA_WP_AUTOMATED%20TRADING%20RISK%20CONTROLS_FINAL_0.pdf)
  motivate keeping analytics separate from tested pre-trade, real-time,
  post-trade, and kill controls. They do not prescribe this classifier.
