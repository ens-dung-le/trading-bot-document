# Plan - Gaussian Filter Trend Strategy

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). This
strategy remains a design, not a registered trading implementation. The
existing Gaussian market-regime classifier is not this entry strategy.

Review: the proposed Gaussian/ER/ATR trend model is naturally candle-based;
preset descriptions recommend horizons but do not require a fixed trigger.

Goal: Define `gaussian-filter-trend` on any data-valid trigger `T`.
Out of scope: Changing the indicator formulas, presets, license requirements or
the separate reporting classifier's `1h/4h` configuration.
Steps:

1. In planned `src/strategies/gaussian-filter-trend/parameters.ts`, accept `T`
   from the shared enum and resolve `[T]`. Snapshot the effective preset
   parameters without auto-switching presets when the timeframe changes.
2. In planned recursive evaluation, keep Gaussian, ER, smoothing and ATR
   periods as `T` bars. Calculate sufficient stabilization history and retain
   prior direction across restarts without creating a new entry mid-trend.
3. Test `1m`, `1h`, `1d` and venue-aligned `1w` for bullish/bearish turns,
   unchanged flat direction, next-open fills and prepared/incremental parity.

Verification: Static trace a bullish turn on a closed hourly bar: emit one
signal; another bullish hour emits none. A forming-bar reversal cannot change
that completed signal, and a restart in the same trend cannot create a turn.

## Original source and baseline specification

> **Status: draft for customer review; no strategy implementation is authorized yet.**
>
> Source: `Gaussian Filter Trend [QuantAlgo]`, Pine Script supplied by the
> customer.
>
> The supplied excerpt contains no explicit license notice. Behavioral analysis
> can be used for this specification, but permission must be confirmed before
> source code is copied or directly ported.

## 1. Objective

Add a new candle-driven KuCoin Futures USDT-M strategy named
`gaussian-filter-trend` that converts the supplied trend indicator into causal,
reproducible backtest, paper and live behavior.

The strategy must:

- calculate the Gaussian filter, Efficiency Ratio, adaptive ATR width and
  stepped trend line from closed candles only;
- enter Long when the confirmed trend changes to bullish;
- enter Short when the confirmed trend changes to bearish;
- close or reverse an opposite open position on a confirmed trend change;
- avoid repeated entries while the trend direction remains unchanged;
- protect every live position with an exchange-side stop;
- derive sizing from the shared risk and symbol specification layers;
- produce the same signal decisions in backtest, paper and live modes;
- retain source-default and live-candidate profiles as separate versions;
- remain disabled for live trading until walk-forward and paper gates pass.

The Pine source is an `indicator()`, not a TradingView `strategy()`. It defines
trend states and alerts but does not define position size, leverage, stop loss,
take profit, fees, funding, liquidation behavior or order fills. Those are
explicit strategy-design decisions, not hidden properties of the indicator.

## 2. Verified source behavior

### 2.1 Price source

The Gaussian filter reads one configured price series. The source tooltip
recommends:

- `close` for the standard response;
- `hl2 = (high + low) / 2` to reduce wick noise;
- `hlc3 = (high + low + close) / 3` for additional smoothing.

TradingView's `input.source` can accept arbitrary chart series. The bot should
restrict this to a versioned enum such as `close`, `hl2` and `hlc3` so a stored
configuration can always be replayed without another indicator dependency.

### 2.2 Presets

The source defines these effective numeric presets:

| Setting | Default | Fast Response | Smooth Trend |
|---|---:|---:|---:|
| Gaussian length | `14` | `8` | `21` |
| Gaussian poles | `4` | `2` | `4` |
| ATR length | `14` | `10` | `21` |
| Fixed width multiplier | `1.5` | `1.2` | `2.0` |
| Efficiency length | `10` | `7` | `18` |
| Adaptive trend multiplier | `0.8` | `0.6` | `1.0` |
| Adaptive chop multiplier | `2.5` | `1.8` | `3.2` |
| Efficiency EMA smoothing | `5` | `3` | `8` |

The source describes the intended use as:

- Default: swing trading on approximately `1h` to daily charts;
- Fast Response: intraday trading on approximately `5m` to `1h` charts;
- Smooth Trend: position trading on daily and weekly charts.

Selecting a preset overrides the numeric fields in the table. It does not
override `priceSource` or `adaptiveWidth` in the supplied code. When adaptive
width remains enabled, `fixedWidthMultiplier` is stored but not used.

The bot must snapshot the resolved effective values in every backtest and live
configuration. A report must not require the reader to know which hidden preset
overrides were active.

### 2.3 Gaussian filter

For configured length `L` and pole count `P`, the source calculates:

```text
beta  = (1 - cos(2 * pi / L)) / (1.414^(2 / P) - 1)
alpha = -beta + sqrt(beta^2 + 2 * beta)
```

Each stage is a recursive one-pole filter:

```text
stage1[t] = alpha * source[t] + (1 - alpha) * stage1[t-1]
stage2[t] = alpha * stage1[t] + (1 - alpha) * stage2[t-1]
stage3[t] = alpha * stage2[t] + (1 - alpha) * stage3[t-1]
stage4[t] = alpha * stage3[t] + (1 - alpha) * stage4[t-1]
```

An uninitialized previous stage is seeded from its current input. The selected
filtered price is stage 1, 2, 3 or 4 according to `P`.

For strict TradingView parity, calculations must use the source constant
`1.414`. Replacing it with a differently rounded square root can create small
boundary differences in long replays and should be treated as a model-version
change.

### 2.4 Efficiency Ratio and adaptive width

For efficiency lookback `E`:

```text
netMove[t]    = abs(source[t] - source[t-E])
pathLength[t] = sum(abs(source[i] - source[i-1]), E changes ending at t)

rawEfficiency[t] = 0                         when pathLength[t] = 0
rawEfficiency[t] = netMove[t] / pathLength[t] otherwise

smoothedEfficiency = EMA(rawEfficiency, efficiencySmoothing)
```

The ratio is near `1` when price travels efficiently in one direction and near
`0` when the same total movement produces little net displacement.

Width multiplier:

```text
adaptive width enabled:
  widthMultiplier = chopMultiplier
                    + (trendMultiplier - chopMultiplier)
                    * smoothedEfficiency

adaptive width disabled:
  widthMultiplier = fixedMultiplier
```

With the source defaults, a clean trend tightens the width toward `0.8 ATR`,
while chop widens it toward `2.5 ATR`.

```text
trendWidth = ATR(atrLength) * widthMultiplier
upperBand  = filteredPrice + trendWidth
lowerBand  = filteredPrice - trendWidth
```

### 2.5 Stepped trend line

Let `previousTrendLine` be the last completed value:

```text
trendLine = previousTrendLine

if upperBand < trendLine:
  trendLine = upperBand

if lowerBand > trendLine:
  trendLine = lowerBand
```

This creates a volatility deadband. The line stays unchanged while the
Gaussian output remains inside that band and moves only when a band walks
beyond the previous line.

Direction is determined by the line's step, not directly by price crossing the
line:

```text
trendLine[t] > trendLine[t-1] -> bullish direction
trendLine[t] < trendLine[t-1] -> bearish direction
trendLine[t] = trendLine[t-1] -> keep the previous direction
```

Direction begins neutral and persists through flat trend-line candles after
the first directional step.

### 2.6 Alert events

```text
turnedBull = current direction is bullish and previous direction was not bullish
turnedBear = current direction is bearish and previous direction was not bearish
```

The first change from neutral to bullish or bearish is an alert. A direction
that remains unchanged does not create another alert.

The star field, line colors, candle colors and background colors are visual
only. They must not affect strategy decisions or warm-up.

## 3. Causality and repaint policy

The formulas do not request future or higher-timeframe data, but the Pine
indicator does not guard alerts with `barstate.isconfirmed`. On a realtime
TradingView candle, source, ATR, Efficiency Ratio, bands and trend line can
change before the candle closes. An intrabar bullish signal can disappear or
become bearish by the final close.

The bot must use closed-candle confirmation:

```text
signal candle timestamp = opening timestamp of the completed candle
decision timestamp      = signal candle open + trigger timeframe
signal direction        = final direction calculated at that close
```

No forming candle, local wall clock, network response or later candle may be
read by the strategy. TradingView alerts must be configured as once-per-bar
close when they are used for parity comparison.

Expected consequence: the bot may alert later than an intrabar TradingView
notification, but it will not trade a state that disappeared before close.

## 4. Proposed trading contract

### 4.1 Entry

When flat:

```text
turnedBull -> market Long signal
turnedBear -> market Short signal
no state change -> no entry
```

The entry reference price is the signal candle close. Backtests fill a market
entry at the next trigger candle open plus the configured spread and slippage.
Live and paper use their confirmed executor fill.

The strategy must not enter simply because the current state is bullish or
bearish. Entry is edge-triggered. Therefore:

- restarting mid-trend does not create a new trade;
- a protective stop that closes a trade does not cause immediate re-entry into
  the same unchanged trend;
- the strategy waits for the next genuine direction change.

### 4.2 Opposite signal

When a position is open and the trend turns against it:

```text
Long position  + turnedBear -> close Long, then enter Short
Short position + turnedBull -> close Short, then enter Long
```

The requested execution policy is `close_and_reverse`. Closing must be
confirmed before the opposite entry is submitted so temporary hedging cannot
occur in isolated one-way mode.

The current backtest path can schedule an opposite close and entry together,
but the live `CloseAndReverse` branch currently closes and returns without
submitting the reverse entry. Implementation must fix and parity-test this
shared behavior or introduce an ordered close-then-enter intent before this
strategy can be considered complete.

### 4.3 Same-direction signal

No same-direction scale-in is allowed. The indicator emits only direction
changes, and adding exposure on unchanged direction would be a separate rule
with no source basis.

## 5. Protective stop and exit model

### 5.1 Source boundary

The supplied indicator has no stop loss, take profit or time exit. A strategy
without an exchange-side stop is not eligible for live trading in this bot.

Version 1 therefore adds a clearly labeled live-safety overlay. Its performance
must be reported separately from raw signal-following analysis.

### 5.2 Proposed Gaussian trend-line stop

The recommended source-related stop is the stepped trend line itself, with an
optional outward buffer:

```text
Long candidate stop  = trendLine * (1 - stopBufferRate)
Short candidate stop = trendLine * (1 + stopBufferRate)
```

Entry is rejected if this stop is not strictly below a Long reference price or
strictly above a Short reference price after tick normalization.

On each newly closed candle in the same trend, the candidate stop may replace
the active stop only when it reduces risk:

```text
Long:  newStop = max(currentStop, normalizedCandidateStop)
Short: newStop = min(currentStop, normalizedCandidateStop)
```

The stop may never move backward. Every accepted change must modify the
exchange-side protective order and persist the updated position state.

Using the trend line as an intrabar resting stop is intentionally safer and not
identical to waiting for a closed-candle direction flip. Price may touch the
line and recover before close, causing the bot to exit while the indicator
keeps its previous direction. Reports must identify these as protective-stop
exits rather than Gaussian reversal exits.

### 5.3 Other exits

Version 1 proposes:

- opposite confirmed trend change: close and reverse;
- protective trend-line stop: close the full remaining position;
- liquidation: shared highest-priority emergency outcome;
- manual stop and circuit breaker: existing shared behavior;
- take profit: disabled because the source defines none;
- generic break-even stop: disabled for source clarity;
- generic ATR trailing stop: disabled because the Gaussian line owns trailing;
- maximum holding exit: disabled by default.

Existing exit precedence remains:

```text
liquidation -> protective stop -> indicator reversal -> time exit
```

If the protective stop and reversal are both observable in the same OHLC bar,
the stop wins because the bar does not prove that the closing reversal existed
before the stop price traded.

## 6. Sizing and account risk

The proposed version uses the shared risk-to-stop sizing model:

```text
risk amount = configured equity basis * riskPerTradeRate
risk distance = abs(entry reference price - initial stop price)
raw quantity = risk amount / risk distance
```

The engine then applies contract multiplier, quantity step, minimum quantity,
leverage, margin ceiling, liquidation safety buffer, open-position count and
correlation-adjusted exposure checks.

An entry must be rejected when:

- the Gaussian stop is unavailable or on the wrong side;
- the stop is too close to cover expected round-trip cost safely;
- the stop exceeds the configured maximum distance;
- quantity rounds below the exchange minimum;
- required margin or total exposure exceeds its account limit;
- protection cannot be confirmed at the exchange after entry fill.

The source supplies no risk percentage or leverage. Those values must be
approved independently and stored in every execution snapshot.

## 7. Proposed profiles

### 7.1 Source-default calculation profile

The source-default calculation values are:

```text
preset                    = default
priceSource               = close
gaussianLength            = 14
gaussianPoles             = 4
atrLength                 = 14
adaptiveWidth             = true
fixedWidthMultiplier      = 1.5
efficiencyLength          = 10
adaptiveTrendMultiplier   = 0.8
adaptiveChopMultiplier    = 2.5
efficiencySmoothing       = 5
```

The source does not select a chart timeframe. Its tooltip suggests this profile
for approximately `1h` to daily charts. A trigger timeframe must be chosen
before a runnable bot configuration can be seeded.

### 7.2 Intraday live-candidate calculation profile

If this strategy is intended to join the bot's current intraday portfolio, the
Fast Response preset is the appropriate starting hypothesis:

```text
preset                    = fast_response
gaussianLength            = 8
gaussianPoles             = 2
atrLength                 = 10
adaptiveWidth             = true
fixedWidthMultiplier      = 1.2
efficiencyLength          = 7
adaptiveTrendMultiplier   = 0.6
adaptiveChopMultiplier    = 1.8
efficiencySmoothing       = 3
```

The customer must still choose whether the live trigger is `5m`, `15m`, `30m`
or `1h`. The same numeric parameters must not be assumed to have equal meaning
or equal risk across all four timeframes.

## 8. Parameter contract

The strategy parameter snapshot should contain:

| Group | Fields | Validation |
|---|---|---|
| Runtime | `triggerTimeframe`, optional `defaultSymbol`, `defaultLeverage` | supported closed-candle timeframe |
| Preset | `preset`, resolved effective values | strict enum; effective snapshot required |
| Source | `priceSource` | `close`, `hl2` or `hlc3` |
| Gaussian filter | `gaussianLength`, `gaussianPoles` | length `>= 2`; poles integer `1..4` |
| ATR width | `atrLength`, `adaptiveWidth`, `fixedWidthMultiplier` | positive values |
| Efficiency | `efficiencyLength`, `efficiencySmoothing` | positive integers; efficiency length `>= 2` |
| Adaptive width | `adaptiveTrendMultiplier`, `adaptiveChopMultiplier` | positive; trend multiplier `<=` chop multiplier |
| Stop overlay | `trendLineStopEnabled`, `trendLineStopBufferRate` | enabled for live; buffer non-negative and capped |
| Execution | `oppositeSignalPolicy`, `maximumHoldingCandleCount` | v1 requires `close_and_reverse`; holding `0` disables |
| Shared switches | `breakEvenStopEnabled`, `trailingStopEnabled`, `volatilityFilterEnabled` | explicit booleans |

Preset handling must follow one of these reproducible approaches:

1. store only fully resolved numeric parameters and use the preset as a label;
2. store both requested inputs and the fully resolved effective snapshot.

The implementation must not recalculate historical reports from whatever a
preset means in a later software version.

## 9. Warm-up and initialization

The Gaussian filter is recursive. Although the Pine code produces a value from
the first candle by seeding each stage, its later value remains influenced by
where the loaded history began. Starting the same backtest from two different
pre-roll dates can otherwise change a trend flip near the evaluation boundary.

Warm-up must include:

- enough candles for ATR;
- `efficiencyLength` prior changes;
- enough non-null Efficiency Ratio samples for its EMA smoothing;
- enough recursive Gaussian samples for initialization error to decay below a
  documented tolerance;
- one previous trend-line and direction value for edge detection.

The Gaussian settling count should be calculated using the exact configured
`alpha`, pole cascade and a fixed tolerance such as `1e-6`, rather than an
unexplained multiple of length.

Conceptually:

```text
minimumIndicatorReady = max(
  atrLength + 1,
  efficiencyLength + efficiencySmoothing,
  gaussianSettlingCandleCount
)

requiredWarmUp = minimumIndicatorReady + 1
```

The report must print the actual warm-up count and calculation model version.
No entry is allowed while any required value is unavailable.

## 10. Repository placement and capability gaps

No Gaussian Filter Trend implementation currently exists in the repository.
The Nadaraya-Watson module uses Gaussian-shaped weights for a different
indicator and must not be reused as if it were the recursive pole filter from
this source.

### 10.1 New shared indicator module

```text
src/indicators/gaussian-filter-trend.ts
src/indicators/gaussian-filter-trend.test.ts
```

Responsibilities:

- exact source selection;
- pole coefficient and recursive stages;
- raw and smoothed Efficiency Ratio;
- ATR width and adaptive multiplier;
- stepped line, persistent direction and turn events;
- batch-series and incremental-state implementations with parity tests.

### 10.2 New strategy module

```text
src/strategies/gaussian-filter-trend/
|-- parameters.ts
|-- signal.ts
|-- strategy.ts
|-- index.ts
|-- parameters.test.ts
|-- signal.test.ts
`-- strategy.test.ts
```

- `parameters.ts`: strict schema, presets, effective snapshot and validation.
- `signal.ts`: pure mapping from a confirmed trend observation to entry,
  reversal and stop-update intents.
- `strategy.ts`: closed-candle integration and prepared backtest evaluation.
- tests: source parity, causality, restart state and Long/Short symmetry.

### 10.3 Shared execution changes

The implementation requires two additive shared changes:

1. Make `close_and_reverse` genuinely submit the reverse entry in live mode
   after the close is confirmed, matching the backtest contract.
2. Add a strategy-owned protective-stop update intent so the Gaussian line can
   move the resting stop without embedding Gaussian calculations in the engine.

For example:

```ts
interface ProtectiveStopUpdateIntent {
  readonly action: "update_protective_stop";
  readonly symbol: string;
  readonly candleTimestamp: number;
  readonly stopLossPrice: number;
  readonly reason: string;
}
```

The engine and backtest runner must validate direction, tick normalization,
cost distance and never-backward movement before applying the update. Existing
strategies that do not emit this intent retain their current behavior.

### 10.4 Registry and database

Expected integration points:

- `src/strategies/strategy.registry.ts`: register strategy, parser and default
  parameters;
- `src/backtest/backtest-execution-configuration.ts`: snapshot the required
  opposite-signal and stop-management policy;
- engine and backtest parity tests for reversal and stop updates;
- an operational migration to seed a disabled strategy template and
  account-owned copies.

No new position-state column is expected. The active stop price already belongs
to the persisted open position, while Gaussian filter state can be rebuilt from
the required closed-candle warm-up history.

## 11. Backtest and reporting requirements

Use the existing execution cost, funding, liquidation, circuit-breaker and
market-regime reporting infrastructure. Add Gaussian-specific analytics:

The shared market-regime classifier is a separately versioned consumer of the
same Gaussian calculation. It evaluates `4h` structural direction with `1h`
confirmation for every strategy, persists historical report periods in
`backtest_regime_periods`, and advances incrementally from closed candles in
live mode. This analytics use must not create trades or otherwise couple the
Gaussian strategy to other strategy execution.

- bullish, bearish and neutral candle counts;
- total direction changes and flips per 30 days;
- median and percentile trend duration;
- average Efficiency Ratio by direction and at entry;
- average adaptive width multiplier and ATR width;
- entries, wins, losses and net PnL by direction;
- reversal exits versus protective-stop exits;
- stopped trades that received no same-trend re-entry;
- signal close to actual entry-fill slippage;
- return captured between confirmed direction changes;
- whipsaw count, defined by an opposite change inside a configured diagnostic
  horizon;
- performance and entry quality by market regime;
- fees, funding, profit factor, expectancy and maximum drawdown.

Reports must state whether results use:

- closed-candle signals or TradingView intrabar alerts;
- source-default, Fast Response, Smooth Trend or custom parameters;
- static initial line stop or dynamically updated line stop;
- close-only or close-and-reverse behavior.

Only the closed-candle, dynamically protected result is eligible for live
acceptance.

## 12. Verification plan

### 12.1 Indicator unit tests

- Exact `beta` and `alpha` for pole counts 1 through 4.
- Each recursive stage against hand-calculated fixtures.
- Price source mapping for `close`, `hl2` and `hlc3`.
- Efficiency Ratio equal to `0` for a flat path.
- Efficiency Ratio near `1` for a monotonic path.
- Adaptive multiplier interpolation at efficiency `0`, `0.5` and `1`.
- Fixed-width behavior when adaptive width is disabled.
- Trend line unchanged inside the deadband.
- Bullish and bearish steps outside the deadband.
- Direction persistence while the trend line is flat.
- Neutral-to-direction and direction-to-opposite turn events.
- Batch and incremental output equality.
- No future or forming-candle input.

### 12.2 Strategy and execution tests

- One Long signal on `turnedBull`, not on every bullish candle.
- One Short signal on `turnedBear`, not on every bearish candle.
- No same-side scale-in.
- No same-trend re-entry after a protective stop.
- Opposite signal closes and reverses exactly once.
- Reverse entry occurs only after close confirmation.
- Initial trend-line stop is on the protective side.
- Stop updates only in the favorable direction.
- Stop modification parity across backtest, paper and live paths.
- Restart warm-up reconstructs the same trend line and direction.
- Missing or stale candle data blocks new entries without removing protection.

### 12.3 Research gates

Before paper enablement:

- backtest each intended symbol independently;
- run chronological in-sample/out-of-sample and walk-forward evaluation;
- compare the three presets without selecting on the final holdout period;
- test the intended timeframe rather than mixing results across timeframes;
- stress taker fees, spread, slippage and funding;
- analyze whipsaw clusters and performance by market regime;
- reject a profile whose edge depends on one symbol, one month or unconfirmed
  intrabar state changes.

Before live enablement:

```text
shadow -> paper -> minimum-contract canary -> staged capital increase
```

A reversal failure, unconfirmed protective stop, stale data incident or
reconciliation mismatch requires manual review and returns the strategy to the
canary stage.

## 13. Customer decisions required before implementation

1. Confirm the strategy name `gaussian-filter-trend`.
2. Confirm the initial trigger timeframe and symbol list.
3. Confirm whether the first profile should use Default, Fast Response, Smooth
   Trend or custom parameters.
4. Confirm the allowed price sources. The recommended reproducible set is
   `close`, `hl2` and `hlc3`.
5. Confirm closed-candle signals even though an intrabar TradingView alert may
   arrive earlier and later disappear.
6. Confirm `close_and_reverse`, including paying an exit and a new entry fee on
   every direction flip.
7. Confirm the proposed Gaussian trend-line protective stop and its buffer.
8. Confirm that no take profit or break-even rule is added in version 1 because
   the source defines neither.
9. Confirm the maximum risk per trade, leverage, total margin ceiling and daily
   drawdown limit.
10. Confirm whether the customer has permission to port and deploy the supplied
    QuantAlgo source code.
