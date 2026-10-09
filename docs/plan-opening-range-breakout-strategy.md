# Plan - Opening Range Breakout Strategy

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). This
strategy is not registered/implemented; the review updates its design only.
The source's <=15m chart limit below remains a source-parity restriction.

Review: opening-range duration and session timestamps are clock-based. A
30-minute range cannot be reconstructed from a single hourly OHLC candle;
its high/low includes prices after the range has already ended.

Goal: Define broader session-valid trigger `T` independently from range source.
Out of scope: Claiming source parity above 15m or changing the session model.
Steps:

1. In the planned `src/strategies/opening-range-breakout/parameters.ts`, separate
   trigger `T` from the finer range-source interval and any finer execution
   interval. Require exact session/range boundary coverage; keep timezone and
   daylight-saving rules explicit. Resolve every required series together.
2. In planned signal evaluation, freeze the range using only closed source
   bars within `[sessionStart, rangeEnd)`. Trigger breakout checks only on
   candles entirely after the completed range and within the active session.
   Keep one-trade-per-session and retest counts on the declared signal series.
3. Test baseline intraday profiles and an explicit `T=1h` research profile
   with fine range data, including range end inside an hour, DST changes,
   session cutoff and stage-exit ordering with the declared execution series.

Verification: Static trace a 09:30-10:00 range built from minute candles. An
hourly 09:00-10:00 candle overlaps range formation and is ineligible; the
10:00-11:00 candle can first confirm a breakout at 11:00 if still in session.

Daily/weekly candles spanning the session cannot implement the same intraday
ORB. Reject such profiles explicitly. Supporting them needs a new multi-session
range model, not automatic rescaling of the 30-minute opening range.

## Original source and baseline specification

> **Status: draft for customer review; no strategy implementation is authorized yet.**
>
> Source: `ORB Algo | Flux Charts`, Pine Script v6, supplied by the customer.
>
> License: Mozilla Public License 2.0, copyright fluxchart. Any source file that
> directly ports covered code must preserve the MPL notice and attribution.

## 1. Objective

Add a new KuCoin Futures USDT-M strategy named `opening-range-breakout` that
turns the supplied TradingView indicator into a causal, reproducible trading
strategy for backtest, paper and live execution.

The strategy must:

- trade the active runtime symbol rather than a hard-coded market;
- build one opening range for each configured trading session;
- detect bullish and bearish breakouts from closed candles only;
- optionally require zero to three successful retests before entry;
- open at most one trade per session;
- protect every filled position with an exchange-side stop;
- support the three take-profit stages described by the source;
- preserve partial-exit progress and protective state across restarts;
- produce the same decisions from the same closed candles in backtest, paper
  and live modes;
- include fees, spread, slippage, funding and liquidation risk in reports;
- remain disabled for live trading until walk-forward and paper gates pass.

The supplied script is an `indicator()`, not a TradingView `strategy()`. Its
dashboard estimates outcomes but does not place orders or model exchange
execution. Visual parity is therefore not sufficient evidence of trading
parity.

## 2. Verified source behavior

### 2.1 Reference inputs

The Pine source exposes these algorithm inputs:

| Input | Source default | Allowed values |
|---|---:|---|
| Chart candle timeframe | chart-owned | `15m` or lower |
| ORB duration | `30m` | TradingView timeframe |
| Sensitivity | `Medium` | `High`, `Medium`, `Low`, `Lowest` |
| Breakout condition | `Close` | candle close or EMA |
| Take-profit method | `Dynamic` | dynamic EMA or ATR |
| EMA length | `9` | `4`, `9`, `13`, `20`, `34` |
| ATR length | `12` | constant in the source |
| Stop-loss method | `Balanced` | `Safer`, `Balanced`, `Risky` |
| Adaptive stop | enabled | enabled or disabled |
| Backtest exit ratios | `90% / 5% / 5%` | seven source presets |

The source contains a `Fixed` one-percent stop branch, but `Fixed` is not one
of the selectable `slMethod` input options. It is unreachable through the
published settings panel and must not be included in version 1 unless the
customer explicitly requests it.

### 2.2 Session and opening range

With the custom session disabled, a new ORB begins on TradingView's first
regular-session bar. With it enabled, a new ORB begins when the configured
session starts.

For a session starting at `S` with an ORB duration `D`:

```text
rangeEnd = S + D
ORB high = maximum high of candles whose open timestamp is in [S, rangeEnd)
ORB low  = minimum low  of candles whose open timestamp is in [S, rangeEnd)
```

The first candle opening at or after `rangeEnd` may create a breakout. The
source does not enforce a separate entry cutoff, so a breakout can occur late
in the session.

The custom-session end value is not used as an entry cutoff in the Pine logic.
Only the next session start closes the previous ORB. Production behavior must
make this explicit rather than infer an entry window from the displayed
session text.

### 2.3 Breakout detection

After the opening range is complete:

```text
conditionPrice = candle close                         when breakoutCondition = Close
conditionPrice = EMA((high + low) / 2, emaLength)    when breakoutCondition = EMA

conditionPrice > ORB high -> bullish breakout candidate
conditionPrice < ORB low  -> bearish breakout candidate
```

The EMA is calculated from candle midpoint `(high + low) / 2`, not from close.

A bullish candidate fails when a later candle closes below the ORB high. A
bearish candidate fails when a later candle closes above the ORB low. A failed
candidate returns the session to the waiting-for-breakout state, so another
candidate may form during the same session if no entry has occurred.

### 2.4 Retest and sensitivity

The source maps sensitivity to the required retest count:

```text
High   -> 0 retests
Medium -> 1 retest
Low    -> 2 retests
Lowest -> 3 retests
```

A bullish retest requires all of the following on a candle after the breakout
candle:

```text
close > ORB high
low   < ORB high
```

A bearish retest is symmetric:

```text
close < ORB low
high  > ORB low
```

At `High` sensitivity, entry is signalled on the same closed candle that
establishes the breakout. At the other sensitivities, entry is signalled on the
closed candle that satisfies the final required retest.

### 2.5 Entry

The indicator records the signal candle's close as its entry price and records
ATR(12) from that same closed candle. It permits only one entry per ORB session.

A real bot cannot fill retroactively at a candle close after using that close
to decide. The proposed bot contract is therefore:

```text
signal candle timestamp = open timestamp of the confirmed closed candle
decision timestamp      = signal candle open + trigger timeframe
reference price         = signal candle close
order type              = market
backtest fill           = next trigger candle open plus the execution-cost model
live fill               = actual confirmed exchange fill
```

This causal difference must be visible in parity fixtures and reports.

## 3. Stop-loss model

Let:

```text
H = ORB high
L = ORB low
C = (H + L) / 2
```

The source stop prices are:

| Method | Long stop | Short stop |
|---|---:|---:|
| `Safer` | `(C + H) / 2` | `(C + L) / 2` |
| `Balanced` | `C` | `C` |
| `Risky` | `(C + L) / 2` | `(C + H) / 2` |

The stop is calculated from the completed opening range and normalized to the
symbol's price tick. The shared risk-to-stop sizing layer then derives quantity
from the confirmed risk budget, contract multiplier, quantity step and minimum
quantity.

An entry must be rejected when:

- the stop is on the wrong side of the reference or expected fill price;
- the stop distance is zero or outside the configured safety ceiling;
- the calculated quantity is below the exchange minimum;
- margin, leverage, liquidation-buffer or account exposure checks fail;
- an exchange-side protective stop cannot be confirmed after the fill.

## 4. Take-profit model

### 4.1 ATR method

The source freezes ATR at entry and defines:

```text
Long TP1 = entry anchor + 0.75 * entry ATR
Long TP2 = entry anchor + 1.50 * entry ATR
Long TP3 = entry anchor + 2.25 * entry ATR

Short TP1 = entry anchor - 0.75 * entry ATR
Short TP2 = entry anchor - 1.50 * entry ATR
Short TP3 = entry anchor - 2.25 * entry ATR
```

`atrTotalMult` is `1.0` in the supplied source and is not user-configurable.

The indicator does not submit partial orders. Its dashboard weights each target
as if that share of the original position had been closed there. The bot will
turn those weights into real reduce-only target quantities, and they must sum
to exactly `1.0`. The source default is:

```text
TP1 = 90 percent
TP2 = 5 percent
TP3 = 5 percent
```

When adaptive stop is enabled, the stop for the remaining quantity moves to
break even after TP1 is confirmed. The live break-even price must cover entry
fee, estimated exit fee, accumulated funding and the configured slippage
buffer; it must not blindly use the raw entry price.

TP3 closes any remaining quantity and completes the session trade.

### 4.2 Dynamic EMA method

For a long, the source considers the EMA profitable when it is above the entry
anchor and creates a target event when the candle closes below that EMA. The
short rule is symmetric.

TP1 additionally requires the EMA to be at least `0.20%` from the entry
anchor. TP2 and TP3 each require the EMA to advance at least `0.075%` beyond
the previous target price.

The Pine dashboard assumes the exit occurred at the EMA value on the same
candle used to detect the condition. That price is not causally executable. In
the bot:

- the condition is evaluated only after the candle closes;
- the partial reduce-only market order is sent at the next executable moment;
- backtest and report use the simulated or confirmed fill, not the historical
  EMA value;
- the EMA value remains stored as the decision reference for audit.

Dynamic TP will therefore have signal parity with the indicator but not its
optimistic dashboard fill price.

### 4.3 Target anchor decision

The Pine source anchors targets to the signal candle close. A production bot
normally anchors protective orders to the confirmed entry fill so gaps and
slippage do not silently change the reward-to-risk ratio.

Recommended live behavior is `confirmed_fill_price`. A separate
`reference_signal_price` compatibility mode may be retained for TradingView
comparison, but it must not be the live default. This remains a customer
approval item.

## 5. Intrabar execution policy

OHLC candles do not reveal whether the high or low happened first. The source
processes a TP before the stop and suppresses the stop on any candle that marks
a TP. That is optimistic when both levels are touched.

The bot must use this deterministic policy:

1. Use lower-timeframe candles to resolve order when complete lower-timeframe
   history is available.
2. Otherwise, if liquidation, stop and target are all possible inside one bar,
   use the existing conservative precedence: liquidation, then stop, then TP.
3. When several resting TP levels are crossed with no competing stop event,
   fill each crossed target in price order up to the remaining quantity.
4. Record the number of ambiguous candles in the report.

The TradingView compatibility report may additionally calculate the source's
TP-first estimate, but acceptance and live decisions must use the conservative
result.

## 6. Session rollover

The Pine code discovers a new session and then records the previous candle's
close as the old trade's exit. That is retrospective and cannot be reproduced
live.

The bot must instead:

- stop accepting new candidates at the configured entry cutoff, if enabled;
- cancel any unfilled strategy-owned entry order at session rollover;
- close the remaining position reduce-only at the first executable price on or
  after the configured session-close boundary;
- never backdate an order or fill to the previous candle;
- classify the close reason as `session_exit` in strategy telemetry and as an
  explicit position close reason in the shared trade report.

For a continuously traded crypto market, the proposed reference session is one
UTC calendar day. A different market-session timezone requires an explicit
configuration and DST policy.

## 7. Proposed version-1 profiles

Two profiles should be kept separate so source comparison is not confused with
production approval.

### 7.1 TradingView comparison profile

```text
triggerTimeframe          = 15m
sessionStartMinuteUtc     = 0
sessionDurationMinutes    = 1440
openingRangeMinutes       = 30
sensitivity               = medium
breakoutCondition         = close
takeProfitMethod          = dynamic
emaLength                 = 9
atrLength                 = 12
stopLossMethod            = balanced
adaptiveStopLoss          = true
takeProfitExitRates       = [0.90, 0.05, 0.05]
targetAnchor              = reference_signal_price
entryCutoffMinute         = disabled
```

This profile exists for behavioral comparison. It is not automatically safe
for live trading. The `15m` chart and midnight UTC session are bot adapter
choices; the Pine defaults the other fields but reads chart timeframe and
regular-session boundaries from TradingView.

### 7.2 Live-candidate profile

The live candidate should start with:

```text
triggerTimeframe          = 15m
sessionStartMinuteUtc     = 0
sessionDurationMinutes    = 1440
openingRangeMinutes       = 30
targetAnchor              = confirmed_fill_price
sameBarPolicy             = conservative
marginMode                = isolated
```

Sensitivity, breakout condition, TP method, exit ratios, entry cutoff and risk
budget remain pending backtest evidence. ATR targets are the recommended first
live candidate because they can rest at the exchange immediately after a fill;
dynamic EMA exits depend on the bot receiving and processing a later closed
candle.

## 8. Parameter contract

The strategy parameter snapshot should contain:

| Group | Fields | Validation |
|---|---|---|
| Runtime | `triggerTimeframe`, optional `defaultSymbol`, `defaultLeverage` | trigger must be `1m`, `3m`, `5m` or `15m` |
| Session | `sessionStartMinuteUtc`, `sessionDurationMinutes`, optional `entryCutoffMinute` | integer minute boundaries; no overlap |
| Opening range | `openingRangeMinutes` | multiple of trigger duration; shorter than session |
| Breakout | `sensitivity`, `breakoutCondition`, `emaLength` | strict enums; EMA positive |
| Volatility | `atrLength` | positive integer |
| Stop | `stopLossMethod`, `adaptiveStopLoss`, `breakEvenSlippageBufferRate` | strict enum; non-negative buffer |
| Targets | `takeProfitMethod`, `takeProfit1AtrMultiple`, `takeProfit2AtrMultiple`, `takeProfit3AtrMultiple` | positive and strictly increasing |
| Dynamic targets | `minimumDynamicProfitRate`, `minimumDynamicProfitIncrementRate` | finite positive rates |
| Partial exits | `takeProfit1ExitRate`, `takeProfit2ExitRate`, `takeProfit3ExitRate` | each in `[0,1]`; sum exactly `1` |
| Fill model | `targetAnchor`, `sameBarPolicy` | strict enums |
| Shared risk | `maximumHoldingCandleCount`, stop/trailing/volatility switches | explicit values in snapshot |

Source constants must be represented as named defaults rather than unexplained
numbers inside evaluation code.

## 9. Warm-up and data contract

The strategy reads one trigger timeframe only. It must never use a forming
candle.

To reconstruct an arbitrary evaluation start safely, the loader needs enough
history for both indicators and the current session:

```text
indicatorWarmUp = max(emaLength, atrLength + 1)
sessionBars     = ceil(sessionDurationMinutes / triggerMinutes)
requiredWarmUp = indicatorWarmUp + sessionBars
```

The extra session ensures a backtest beginning in the middle of a UTC day can
rebuild that day's ORB state without pretending the first loaded candle was the
session start.

Missing candles inside the opening range are not acceptable. A missing range
candle changes `H` or `L` and therefore every downstream entry, stop, quantity
and target. The history synchronizer should attempt repair before queueing; the
run must fail if the opening range remains incomplete.

## 10. Current repository assessment

The repository already contains an MPL-attributed calculation prototype:

```text
src/indicators/opening-range-breakout.ts
src/indicators/opening-range-breakout.test.ts
```

It is useful as a parity fixture, but it is not yet a registered strategy or a
complete live execution design.

Before reuse, implementation must address these gaps:

- its current defaults (`15m`, four-hour ORB, High, EMA breakout, ATR targets,
  EMA 34) differ from the supplied Pine defaults;
- its session is fixed to midnight UTC and has no configurable cutoff;
- TP events do not reduce position quantity;
- session exit is recorded retrospectively at the previous candle close;
- the source-style TP-first same-bar ordering is not conservative;
- no exit-plan state is persisted for restart recovery;
- calling the full-series calculator on every live candle would be
  unnecessarily expensive; live evaluation needs incremental session state,
  while backtest should use `prepareBacktest` or a single chronological pass.

The existing prototype must not be silently presented as source-default parity
or as a live-ready strategy.

## 11. Required implementation structure

### 11.1 New strategy-owned module

```text
src/strategies/opening-range-breakout/
|-- parameters.ts
|-- session-state.ts
|-- signal.ts
|-- strategy.ts
|-- index.ts
|-- parameters.test.ts
|-- session-state.test.ts
|-- signal.test.ts
`-- strategy.test.ts
```

- `parameters.ts`: strict schema, defaults and cross-field validation.
- `session-state.ts`: incremental opening range, breakout, retest and TP-stage
  state machine.
- `signal.ts`: pure entry and dynamic-exit decisions.
- `strategy.ts`: integration with closed-candle context and prepared backtest.
- tests: long/short symmetry, every sensitivity, failed breakouts, session
  boundaries, causal entries and restart reconstruction.

### 11.2 Existing indicator module

Refactor `src/indicators/opening-range-breakout.ts` into reusable formulas or
retain it as an independent TradingView-parity oracle. Any directly ported code
must keep its MPL header. Strategy-specific execution and account decisions do
not belong in this indicator module.

### 11.3 Shared partial-exit capability

The current shared contract holds one full-position `takeProfitPrice`. Exact
three-stage execution therefore requires an additive shared capability, for
example:

```ts
interface TakeProfitTargetPlan {
  readonly targetId: "tp1" | "tp2" | "tp3";
  readonly price: number;
  readonly originalQuantityRate: number;
}
```

Required shared changes include:

- signal or post-fill execution-plan contract for three targets;
- position state containing original quantity, remaining quantity and completed
  target IDs;
- paper and live executor support for reduce-only partial closes;
- protective-stop quantity reconciliation after every partial fill;
- backtest support for partial realized PnL, fees and remaining exposure;
- Telegram messages for each TP stage and stop modification;
- restart reconciliation against open position and open reduce-only orders.

This must be additive. Existing strategies with one take-profit price must keep
their current behavior.

### 11.4 Persistence

Do not overload DCA `basket_state` with ORB exit progress. Add a dedicated,
versioned position execution-plan field or table. The stored state must include:

```text
strategy name and version
session start timestamp
original entry quantity
remaining quantity
entry ATR and target anchor
three target prices and quantities
completed target IDs
current protective stop
adaptive-stop activation timestamp
```

A schema migration is expected for live-safe partial-exit persistence. A second
operational migration should seed the strategy template as disabled by default.

## 12. Backtest and reporting requirements

Use the existing cost model, funding history, liquidation simulation, risk
controls and market-regime analytics. Add ORB-specific report fields:

- total sessions and sessions with complete opening-range data;
- sessions with no breakout, failed breakout and confirmed breakout;
- entry count and no-trade session rate;
- long/short split and performance;
- entries by sensitivity/retest count;
- TP1, TP2 and TP3 reach rates;
- quantity and PnL realized at each target;
- stop-loss, session-exit, manual-exit and end-of-data counts;
- adaptive-stop activation and break-even outcomes;
- average ORB width as price rate and ATR multiple;
- signal-close to actual-fill slippage;
- ambiguous same-bar event count;
- performance and entry quality by market regime;
- maximum drawdown, profit factor, expectancy and fee/funding contribution.

The original dashboard's `Wins`, `Losses` and `Total Profit %` may be emitted as
a separate compatibility section, but they must not replace the bot's
quantity-weighted, cost-aware performance metrics.

## 13. Verification plan

### 13.1 Unit and parity fixtures

- Pine defaults and all input mappings.
- Opening-range high/low boundaries.
- EMA calculated from midpoint rather than close.
- Long/short breakout symmetry.
- Zero, one, two and three retest paths.
- Failed breakout followed by a valid second candidate.
- One entry maximum per session.
- All stop formulas.
- ATR and dynamic target sequences.
- Partial quantities and rounding remainder assigned to TP3.
- Adaptive stop only after confirmed TP1 fill.
- Conservative same-bar stop/target ordering.
- No use of a forming or future candle.
- Causal session exit with no backdated fill.

### 13.2 Cross-mode parity

Given the same candles, costs and fills:

```text
backtest decisions = paper decisions = live strategy decisions
```

Executor-specific fill prices may differ, but the triggering candle, direction,
ORB boundaries, stop method, TP plan and decision reason must match.

### 13.3 Research gates

Before paper enablement:

- backtest at least one full market cycle per intended symbol;
- keep all indicator calculations causal and versioned;
- run chronological in-sample/out-of-sample and walk-forward evaluation;
- compare source-default and live-candidate profiles without choosing on the
  final holdout period;
- report results separately by uptrend, sideways and downtrend regimes;
- stress fees, spread and slippage above the expected live cost;
- reject configurations whose result depends mainly on optimistic same-bar
  ordering or one exceptional session.

Before live enablement:

```text
shadow -> paper -> minimum-contract canary -> staged capital increase
```

Any severe reconciliation, stop-placement or partial-exit failure returns the
strategy to the canary stage and requires manual review.

## 14. Customer decisions required before implementation

1. Confirm the strategy name `opening-range-breakout`.
2. Confirm the live trigger timeframe: `1m`, `3m`, `5m` or `15m`.
3. Confirm the session start, timezone and daylight-saving policy. The proposed
   crypto default is `00:00 UTC` with no DST.
4. Confirm whether the opening range should use the Pine default `30m` or the
   existing prototype's customized `4h` range.
5. Confirm whether entries remain possible all session or stop after a defined
   cutoff.
6. Confirm whether version 1 must support both TP methods or begin with the ATR
   method recommended for live safety.
7. Confirm target anchoring: source-compatible signal close or recommended
   confirmed fill.
8. Confirm partial-exit ratios. The Pine default is `90% / 5% / 5%`.
9. Confirm whether conservative stop-first same-bar behavior is accepted when
   lower-timeframe ordering is unavailable.
10. Confirm the maximum live risk per trade and maximum daily drawdown. These
    must not be inferred from the indicator.
11. Confirm the initial symbol list for backtest, paper and live rollout.
