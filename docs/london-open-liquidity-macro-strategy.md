# London Open Liquidity and Macro Strategy

> **Status: V1 price-action contract ready for implementation and backtesting;
> not approved for live trading.**
>
> This document combines the current London Open sweep/reclaim backtest contract
> with the liquidity, order-flow and macro concepts described in the supplied
> London Open playbook and CPI impact analysis. Concepts taken from the source
> documents are hypotheses until they pass causal backtesting, out-of-sample
> validation and paper trading.

## 1. Objective

Evaluate whether the arrival of European liquidity around the London market
open creates a repeatable trading edge in crypto perpetual markets.

The strategy studies two different post-open behaviors:

1. **Sweep reversal:** price raids an Asian-session extreme, reclaims the range
   and moves toward the opposite side.
2. **Breakout continuation:** price breaks an Asian-session extreme, holds
   outside the range and continues in the breakout direction.

The system must not assume that every sweep is a reversal. Reversal and
continuation are separate setups and must be measured independently.

## 2. Research sources and evidence limits

This specification summarizes concepts from:

- `London_Open_BTC_Effect_-_Top_Traders___Playbook.pdf`
- `London_Open_Crypto_CPI_Impact_Analysis.pdf`

The source documents describe market mechanisms, discretionary frameworks and
educational resources. They do not provide an auditable backtest, sample size
or statistically validated performance record. Statements such as "London
commonly sweeps the Asian range" are therefore research hypotheses, not proven
facts.

## 3. Market and timeframe contract

### 3.1 Baseline research configuration

| Item | Baseline value |
|---|---:|
| Instrument | ETH/USDT perpetual |
| Execution timeframe | `15m` |
| Trend-regime timeframes | closed `1h` and `4h` candles |
| Initial capital | `$5,000` |
| Target notional per trade | `$1,000` |
| Margin assumption | `$100` |
| Leverage assumption | `10x` |
| Maximum trades | one qualified entry per London session |

These values reproduce the current research convention. They are not universal
defaults for every asset. BTC, SOL and other perpetual feeds require separate
results and must not inherit ETH conclusions.

### 3.2 London Open clock

Use the IANA time zone `Europe/London` and define London Open as `08:00` local
time. The UTC timestamp therefore changes when daylight saving time changes.

Do not hardcode London Open to one UTC hour for the entire year. The supplied
documents use both `07:00 UTC` and `08:00 BST / UTC` descriptions, which are not
equivalent across summer and winter.

### 3.3 Asian reference range

The playbook proposes the absolute high and low formed during the Asian session
before London. The starting research definition is:

```text
Asian reference window: 00:00-06:00 UTC
Asian High: highest completed-candle high in the window
Asian Low:  lowest completed-candle low in the window
```

The reference range must be complete before any London signal is evaluated.
The London observation window must be defined relative to `08:00 Europe/London`,
with the first two hours after the open as the baseline test.

Alternative reference and observation windows are sensitivity variants. They
must not replace the baseline merely because they produce a better in-sample
result.

## 4. Market hypothesis

The combined hypothesis has three phases.

### Phase 1 - Asian range formation

Lower overnight participation may create a visible consolidation range. Stops
and breakout orders can accumulate above the Asian High and below the Asian
Low.

### Phase 2 - London liquidity event

European institutional, FX, derivatives and proprietary trading activity adds
volume around the open. Price may trade beyond an Asian extreme to access
resting liquidity.

Trading beyond an extreme is not enough to determine direction. It may be:

- a false breakout followed by a range reclaim; or
- genuine acceptance outside the range followed by trend expansion.

### Phase 3 - confirmation and expansion

The strategy waits for closed-candle evidence that distinguishes reversal from
continuation. Entry occurs only after the confirmation candle has closed.

## 5. Shared event definitions

### 5.1 ATR

Use a causal ATR calculated from closed execution-timeframe candles. The ATR
period and exact smoothing method must be snapshotted with each backtest.

### 5.2 Liquidity sweep

A boundary is swept only when price exceeds it by a volatility-adjusted
buffer:

```text
High sweep: candle high >= Asian High + 0.05 ATR
Low sweep:  candle low  <= Asian Low  - 0.05 ATR
```

The `0.05 ATR` buffer is the current baseline hypothesis and must be included
in sensitivity testing.

### 5.3 Range reclaim

```text
High reclaim: after a High sweep, a closed candle closes below Asian High
Low reclaim:  after a Low sweep, a closed candle closes above Asian Low
```

A wick through the range boundary without a qualifying close is not a reclaim.

### 5.4 Directional displacement

The current baseline displacement candle must:

1. have a body of at least `0.8 ATR`;
2. close within the outer `25%` of its candle range in the intended direction;
3. break the prior candle's high for bullish displacement or low for bearish
   displacement; and
4. be fully closed before the entry decision.

### 5.5 Market Structure Shift

The source playbook proposes a Market Structure Shift after the liquidity
sweep. This is not part of the current baseline result and must initially be
tested as an additional filter.

A future implementation must define the exact causal swing-point algorithm.
It must not label a pivot using future candles that were unavailable at the
decision time.

## 6. Setup A - sweep reversal

### 6.1 Long reversal

A Long candidate requires this sequence:

1. Price sweeps below the Asian Low by the configured ATR buffer.
2. A closed candle reclaims the Asian Low.
3. Bullish displacement appears after the sweep.
4. If the MSS filter is enabled, bullish structure shift is confirmed.
5. The macro and order-flow policies do not block the signal.
6. Enter at the next `15m` candle open.

### 6.2 Short reversal

A Short candidate requires the mirrored sequence:

1. Price sweeps above the Asian High by the configured ATR buffer.
2. A closed candle reclaims the Asian High.
3. Bearish displacement appears after the sweep.
4. If the MSS filter is enabled, bearish structure shift is confirmed.
5. The macro and order-flow policies do not block the signal.
6. Enter at the next `15m` candle open.

### 6.3 Reversal invalidation

Reject a reversal candidate when:

- the signal occurs outside the configured observation window;
- price has not closed back inside the Asian range;
- displacement occurred before the sweep;
- the required candle or higher-timeframe regime candle is still forming;
- the stop price is invalid or the exchange cannot protect the position; or
- another qualified trade has already been accepted for the session.

## 7. Setup B - breakout continuation

Continuation is a separate research branch. It must not share the reversal
label or be combined with reversal performance.

### 7.1 Long continuation

A Long candidate requires:

1. Price closes above the Asian High.
2. Price demonstrates acceptance rather than an immediate reclaim.
3. A retest holds at or above the broken boundary.
4. Bullish displacement confirms renewed expansion.
5. Optional spot/order-flow evidence supports genuine buying.
6. Enter at the next closed-decision candle's following open.

### 7.2 Short continuation

Use the mirrored conditions below the Asian Low.

### 7.3 Acceptance definition

The exact number of closes required outside the range and the allowed retest
depth are not defined by the source documents. Test fixed, predeclared variants
such as one versus two outside closes. Do not select a definition after viewing
the full-period result without an independent holdout.

## 8. Gaussian market-regime context

Use the existing causal Gaussian `1h/4h` classifier described in
`docs/market-regime-analysis-spec.md`.

The baseline reporting groups trades by regime without changing execution. A
separate filtered model may retain only:

- Long trades classified as `uptrend`; and
- Short trades classified as `downtrend`.

Both source candles must be closed by the `15m` decision time. Gaussian
alignment is not proof that the entry is valid, and a result based on a small
filtered sample must be labeled accordingly.

The following comparisons must remain separate:

1. no regime execution filter;
2. strict Gaussian direction alignment;
3. sideways or countertrend subsets for diagnosis; and
4. any future minimum-confidence rule.

## 9. Order-flow confirmation

The playbook recommends using market microstructure to distinguish a genuine
breakout from a liquidity raid. Candidate features include:

- Cumulative Volume Delta (`CVD`);
- Open Interest (`OI`) change;
- spot volume versus perpetual volume;
- funding rate and funding direction;
- aggressive market buying or selling; and
- passive limit-order absorption near the range boundary.

These fields are optional research features, not mandatory baseline inputs.
Each must have a timestamped historical source and causal availability rules.

Examples of testable interpretations include:

- price up + positive CVD + rising OI: possible new Long participation;
- price up + negative or flat CVD: possible passive sell absorption or weak
  breakout;
- price down + falling OI: possible liquidation or position closing rather
  than durable new Short exposure; and
- an extreme funding imbalance against the signal: possible crowded-position
  risk.

The interpretation itself must not be hardcoded until its incremental value is
demonstrated against the price-only baseline.

## 10. Macro-event context

### 10.1 CPI surprise classification

Classify each release from the timestamp and values supplied by an authoritative
economic calendar:

```text
surprise = actual CPI - consensus CPI

surprise > tolerance  -> hot
abs(surprise) <= tolerance -> in line
surprise < -tolerance -> cool
```

The tolerance, CPI series and revision policy must be explicit. Store the
actual event timestamp in UTC rather than assuming a fixed local release hour.

### 10.2 Expected macro bias

| CPI scenario | Expected transmission | Research bias |
|---|---|---|
| Hotter than consensus | yields and USD may strengthen | bearish crypto bias |
| In line | limited policy repricing | whipsaw or mean-reversion risk |
| Cooler than consensus | USD may weaken and liquidity expectations improve | bullish crypto bias |

This matrix is a context hypothesis, not an automatic order instruction.
Actual market reaction can differ from the textbook expectation.

### 10.3 Cross-asset confirmation

For each signal, snapshot the latest causally available state of:

- DXY or a reproducible USD proxy;
- EUR/USD and GBP/USD;
- sovereign yields when reliable history is available;
- DAX and FTSE direction;
- S&P 500 and Nasdaq futures; and
- options implied volatility when available.

A potential risk-on confirmation is a weaker USD with stronger equity futures.
A potential risk-off confirmation is a stronger USD with weaker equity
futures. Conflicting conditions should be reported separately instead of being
silently mapped to bullish or bearish.

### 10.4 Event-day groups

Backtest at least these mutually identifiable groups:

1. normal day with no selected CPI release;
2. UK or European CPI day;
3. US CPI day;
4. hot, in-line and cool surprise groups;
5. signal before versus after the release; and
6. macro-aligned versus macro-conflicting signal.

Do not combine pre-release positioning with post-release reaction. They expose
the strategy to different information and liquidity conditions.

## 11. Entry, stop and exit contract

### 11.1 Entry

All required conditions are evaluated at a closed `15m` candle. A qualified
trade enters at the next `15m` open. The backtest must include spread and
slippage rather than assuming a fill at the signal close.

### 11.2 Protective stop

For the baseline reversal setup:

```text
Long stop  = sweep low  - 0.1 ATR
Short stop = sweep high + 0.1 ATR
```

For continuation trades, the stop belongs beyond the confirmed failed-retest
level. Continuation stop logic must be versioned separately from reversal stop
logic.

### 11.3 Profit target

The baseline target is `2R`, where `1R` is the entry-to-stop risk. Additional
research may compare structural targets such as:

- Asian range midpoint;
- opposite Asian range boundary;
- session VWAP; or
- partial profit followed by a trailing exit.

Structural targets are sensitivity variants until validated. They must not be
mixed into the baseline trade ledger.

### 11.4 Time exit

Close any baseline position still open after `16` execution candles, equal to
four hours on `15m`. The backtest must close at the defined executable price
and apply exit costs.

### 11.5 Same-candle ambiguity

If a candle can touch both stop and target, candle OHLC does not reveal the
intrabar order. The backtest must use a declared conservative policy or replay
lower-timeframe data. The chosen policy must be visible in the report.

## 12. Position sizing and costs

The baseline quantity is:

```text
quantity = target notional / executable entry price
```

The strategy must report the actual entry-to-stop dollar risk. A `$100` margin
allocation does not mean every losing trade loses exactly `$100`.

Net P&L must include:

- entry and exit trading fees;
- bid/ask spread;
- slippage;
- funding paid or received while the position is open; and
- any exchange-specific minimum size or tick rounding.

The current research convention uses a `0.06%` taker fee on each side, a
`0.01%` half-spread assumption and one tick of slippage. Every report must show
the effective assumptions rather than relying on hidden defaults.

## 13. Backtest experiment matrix

Run the following models over the same data coverage:

| Model | Purpose |
|---|---|
| Time-agnostic baseline | measure whether London timing adds value |
| London sweep reversal | measure the price-only core setup |
| London breakout continuation | measure genuine range expansion |
| Reversal + Gaussian alignment | test directional regime filtering |
| Continuation + Gaussian alignment | test trend participation |
| Setup + MSS | test structural confirmation |
| Setup + order flow | test microstructure confirmation |
| Setup + macro context | test CPI/DXY/equity alignment |
| Full combined model | test incremental value after all filters |

Every stricter model must be compared with its immediate parent, not only with
the weakest baseline. The comparison must show both performance improvement and
the number of trades removed.

## 14. Required reporting

### 14.1 Event funnel

For each complete London session report:

1. eligible market-open windows;
2. sessions with any Asian High/Low sweep;
3. sessions with a reclaim;
4. sessions with a breakout hold;
5. sessions with displacement;
6. qualified entries; and
7. Gaussian, MSS, order-flow and macro-filtered entries.

### 14.2 Performance metrics

Report at minimum:

- trade count and active-session rate;
- net P&L and ending balance;
- expectancy in USD and `R`;
- win rate and profit factor;
- maximum drawdown;
- average holding time;
- fees, funding and estimated execution costs;
- Long versus Short performance;
- reversal versus continuation performance;
- MAE and MFE after entry; and
- stability by month, chronological half and market regime.

### 14.3 Macro and order-flow breakdowns

Where data is available, include:

- CPI day versus normal day;
- hot, in-line and cool CPI results;
- DXY-aligned versus conflicting trades;
- equity-futures-aligned versus conflicting trades;
- rising versus falling OI;
- CVD-confirmed versus unconfirmed signals; and
- spot-supported versus perpetual-only movement.

Subsets with fewer than the report's minimum validation sample must be labeled
`low sample`. Positive expectancy from a handful of trades is not sufficient
for deployment.

## 15. Causality and data-quality rules

1. Use only candles that were closed at the decision timestamp.
2. Convert session clocks using IANA time zones and historical daylight-saving
   rules.
3. Store economic events with their original timestamp, consensus, actual and
   revision status.
4. Never join a signal to a macro value published later.
5. Validate that `15m`, `1h` and `4h` histories have no unexplained gaps.
6. Snapshot all parameters, data versions and model versions with the run.
7. Ensure backtest, paper and live execution share the same signal evaluator.
8. Keep asset results isolated; one asset must not provide candles or regime
   state to another.

## 16. Validation and deployment gates

The strategy remains research-only until it passes all applicable gates:

1. positive expectancy after realistic costs;
2. profit factor above `1.0` with a non-trivial margin of safety;
3. acceptable drawdown under the configured capital and sizing model;
4. enough trades to avoid a conclusion driven by a few outliers;
5. reasonable performance in both chronological halves;
6. no single month, direction or market regime explains nearly all profit;
7. an untouched out-of-sample or walk-forward period remains positive;
8. parameter neighborhoods behave consistently rather than showing one sharp
   optimum; and
9. paper execution confirms timing, fills, protection orders and data quality.

## 17. Execution checklist

### Before London Open

- Resolve `08:00 Europe/London` to the correct UTC timestamp.
- Confirm the Asian reference window is complete.
- Record Asian High, Asian Low and current ATR.
- Load the latest closed `1h/4h` Gaussian regime.
- Identify relevant CPI or other scheduled macro events.
- Confirm all required market and auxiliary data are fresh.

### During the observation window

- Detect a buffered sweep of the Asian High or Low.
- Determine whether price reclaims the range or holds outside it.
- Wait for the required displacement and optional MSS confirmation.
- Apply the selected regime, order-flow and macro policy.
- Accept no more than one qualified entry for the session.

### At entry

- Enter only at the next executable candle open.
- Calculate quantity from target notional and rounded exchange constraints.
- Place the protective stop immediately.
- Record the initial dollar risk and `2R` target.

### After entry

- Process stop, target, funding and time exit causally.
- Record MAE, MFE, fees, slippage and exit reason.
- Do not re-enter during the same session under the baseline contract.

## 18. Current interpretation

The strongest practical contribution of the source material is not a promise
that London Open is profitable. It is a clearer classification problem:

```text
Asian liquidity is taken
        |
        +-- range reclaimed -> test reversal
        |
        +-- price accepted outside range -> test continuation
```

The current price-only sweep/reclaim/displacement model covers the reversal
branch. The next useful research step is to add the continuation branch and
then measure the incremental value of MSS, order flow and macro context one at
a time.

No filter should be promoted because it improves headline profit while reducing
the sample to an unreliable number of trades. Deployment decisions must be
based on expectancy, drawdown, sample size, stability and out-of-sample
evidence together.

## 19. Locked V1 implementation contract

This section resolves the implementation choices left open elsewhere in the
document. Where an earlier research section offers multiple alternatives, this
section owns the V1 baseline. Any different behavior is a separately named
sensitivity model and must not silently replace the baseline.

### 19.1 V1 scope

V1 implements only the price-action sweep-reversal setup on ETH/USDT
perpetuals.

V1 includes:

- causal `15m` sweep, reclaim and displacement detection;
- fixed-notional entries;
- fixed initial stop, `2R` target and time exit;
- fees, spread, slippage, funding and liquidation handling from the shared
  backtest engine;
- Gaussian `1h/4h` regime attribution for reporting; and
- a separately reported strict Gaussian-aligned model.

V1 does not use MSS, breakout continuation, CPI surprise values, DXY, equity
futures, CVD, OI, historical order-book depth or spot/perpetual volume as entry
conditions. These remain later research layers and must not block V1 delivery.

### 19.2 Locked V1 parameters

| Parameter | V1 value |
|---|---:|
| Strategy name | `london-open-sweep-reversal` |
| Strategy version | `london-open-sweep-reversal-v1` |
| Symbol | `ETH/USDT:USDT` exchange canonical equivalent |
| Trigger timeframe | `15m` |
| ATR | Wilder `ATR(14)` on `15m` |
| London time zone | `Europe/London` |
| London open | `08:00` local |
| Eligible session days | Monday-Friday by London local date |
| Asian range | `00:00-06:00 UTC` completed candles |
| Observation window | first eight `15m` candles from London Open |
| Sweep buffer | `0.05 ATR` |
| Displacement body | minimum `0.8 ATR` |
| Displacement close location | outer `25%` of candle range |
| Stop buffer | `0.1 ATR` |
| Reward-to-risk target | `2R` |
| Maximum holding time | `16` completed `15m` candles after entry |
| Maximum entries | one per London local date |
| Starting balance | `$5,000` |
| Entry notional | `$1,000` |
| Default leverage | `10x` |
| Baseline Gaussian policy | reporting only |

Crypto trades continuously, but the V1 research sample uses London weekdays
to represent institutional market-open sessions. Exchange trading days are not
removed for UK public holidays. Holiday exclusion, if tested, is a separate
calendar sensitivity model.

### 19.3 Session identity and candle membership

Each session is keyed by the calendar date containing `08:00` in
`Europe/London`.

Use half-open timestamp intervals:

```text
Asian range:       [00:00 UTC, 06:00 UTC)
Observation:       [London Open, London Open + 2 hours)
```

A candle belongs to a window by its open timestamp. Every included candle must
be complete. If any expected `15m` candle in the Asian range or observation
history is missing, mark the session `data_gap` and do not trade it.

### 19.4 Pre-open sweep policy

The V1 hypothesis is specifically that London takes liquidity that still
exists at the open. Therefore:

1. Freeze the Asian High and Low at `06:00 UTC`.
2. Inspect every completed `15m` candle between `06:00 UTC` and London Open.
3. If either frozen boundary is breached by the configured sweep buffer before
   London Open, mark the session `pre_open_swept` and skip the entire session.

Skipping the complete session is intentionally conservative and avoids
relabeling an already-consumed liquidity level as a London-generated sweep.
Alternative policies, such as invalidating only the swept side or extending
the range to the open, require separately named sensitivity runs.

### 19.5 ATR sampling rules

Use Wilder ATR with period `14`.

- A sweep threshold uses the ATR value from the closed candle immediately
  preceding the potential sweep candle. This prevents the sweep candle's own
  range from moving the threshold it is being tested against.
- A displacement threshold uses the ATR value from the closed candle
  immediately preceding the potential displacement candle.
- The stop buffer uses the latest ATR value available at the displacement
  candle close.
- If a required ATR value is not ready, the event cannot advance state.

All ATR comparisons use unrounded prices. Exchange tick rounding is applied
only when creating executable stop and target prices.

### 19.6 V1 session state machine

Each eligible session starts in `waiting_for_sweep` and may advance only in the
order below:

```text
waiting_for_sweep
  -> low_swept  -> low_reclaimed  -> bullish_displacement -> entry_pending
  -> high_swept -> high_reclaimed -> bearish_displacement -> entry_pending
  -> invalidated
  -> expired
```

The state rules are:

1. The first qualifying sweep locks the candidate side.
2. A Low sweep creates only a potential Long reversal. A High sweep creates
   only a potential Short reversal.
3. If the opposite Asian boundary is swept before an entry is confirmed,
   invalidate the session as `both_sides_swept`.
4. Reclaim and displacement must occur no later than the final observation
   candle.
5. The first fully qualified setup owns the session. Ignore later signals for
   that London local date.
6. If the state has not reached `entry_pending` when the observation window
   ends, mark it `no_qualified_signal`.

### 19.7 Same-candle event rules

A sweep and reclaim may occur on the same `15m` candle because the final high,
low and close prove that price traded beyond the boundary and closed back
inside it.

V1 does not allow the same candle to count as both the reclaim and the required
displacement. The displacement must be a later completed candle. This avoids
assuming an intrabar ordering that cannot be recovered from `15m` OHLC.

A future lower-timeframe replay may introduce a separately versioned
single-candle model when `1m` data proves the sweep occurred before the
displacement break.

### 19.8 Exact reclaim and displacement rules

For a Long reversal:

```text
lowSweep = low <= AsianLow - 0.05 * priorATR
lowReclaim = close > AsianLow
bullishBody = close - open
bullishCloseLocation = (close - low) / (high - low)

bullishDisplacement =
  bullishBody >= 0.8 * priorATR
  and bullishCloseLocation >= 0.75
  and high > previousCandle.high
```

For a Short reversal:

```text
highSweep = high >= AsianHigh + 0.05 * priorATR
highReclaim = close < AsianHigh
bearishBody = open - close
bearishCloseLocation = (high - close) / (high - low)

bearishDisplacement =
  bearishBody >= 0.8 * priorATR
  and bearishCloseLocation >= 0.75
  and low < previousCandle.low
```

When `high == low`, close location is undefined and the candle cannot be a
displacement candle. Equality at the Asian boundary is not a reclaim; the
close must be strictly inside the range.

### 19.9 Signal and entry timing

The displacement candle creates a market-entry signal at its close. The order
fills at the next `15m` candle open using the shared market-order cost model.

If the final observation candle creates the signal, its immediately following
candle may fill the entry even though that fill timestamp is exactly at the end
of the two-hour observation window. The signal, not the fill, must occur inside
the observation window.

Cancel the pending entry and record `missing_entry_candle` when the next candle
is unavailable. Do not fill at the signal close or a later non-adjacent candle.

### 19.10 Fixed-notional sizing

The strategy must declare shared initial sizing as:

```text
mode = fixed_notional
parameterName = orderAmountUsd
orderAmountUsd = 1000
```

The intended base quantity is:

```text
signalReferencePrice = displacement candle close
rawQuantity = 1000 / signalReferencePrice
```

The engine converts this quantity through the exchange contract multiplier and
rounds down to the permitted contract step. Actual notional and initial margin
are reported after rounding and the next-open fill. Because quantity is frozen
when the signal is accepted, a gap between the signal close and next open may
make filled notional differ slightly from `$1,000`.

At `10x`, the target initial margin is approximately `$100`; margin is derived
from actual filled notional and is not a second independent sizing input. Reject
the entry when the rounded quantity is zero or when shared balance, exposure,
margin or liquidation safety checks fail.

### 19.11 Stop and target construction

At the displacement close, construct the unrounded protective levels:

```text
signalReferencePrice = displacement candle close

Long stop  = lowest price of the qualifying Low-sweep event - 0.1 * signalATR
Short stop = highest price of the qualifying High-sweep event + 0.1 * signalATR

riskDistance = abs(signalReferencePrice - stop)

Long target  = signalReferencePrice + 2 * riskDistance
Short target = signalReferencePrice - 2 * riskDistance
```

The next-open fill is unknown when the closed displacement candle creates the
signal. V1 therefore calculates the stop and nominal `2R` target from the
signal close, then keeps those precomputed resting levels after the actual fill.
This matches the existing strategy interface and keeps backtest and live order
behavior identical. The report must show the signal reference price, actual
fill price and realized reward-to-risk after costs.

Reject the signal before entry when:

- stop, signal reference price and target are not strictly ordered for the side;
- stop distance exceeds the shared maximum-stop-distance guard;
- stop distance is too small to cover the shared minimum execution-cost
  multiple; or
- exchange price normalization would place the stop on the wrong side of the
  entry.

### 19.12 Exit ordering and fill policy

The stop and target are treated as resting exchange orders from the entry fill.

For every post-entry `15m` candle:

1. apply any funding events whose timestamps belong to the holding interval;
2. evaluate liquidation, protective stop and take profit using the levels that
   were already active when the candle opened;
3. if both stop and target are touched in the same candle and lower-timeframe
   replay is unavailable, choose the stop-loss outcome;
4. otherwise execute the single touched resting level using the shared fill and
   fee model; and
5. only after intrabar exits are ruled out, evaluate the time exit.

After `16` complete holding candles, submit a market exit that fills at the
next available `15m` open. If the dataset ends first, close at the final
available candle close and label the result `end_of_data`, separate from a
normal time exit.

The report must disclose that the stop-first same-candle rule is conservative.
A separately named `1m` replay may replace this assumption when complete lower
timeframe coverage exists.

### 19.13 Gaussian policy

The V1 core model does not block trades based on Gaussian regime. At every
signal, attach the latest causal `1h/4h` regime whose source candles have closed
by the displacement close.

Produce two ledgers from the same core signals:

1. `London Open sweep reversal`: all V1-qualified trades.
2. `London Open + Gaussian aligned`: Long only in `uptrend`, Short only in
   `downtrend`.

The aligned ledger is a diagnostic filtered subset, not a second signal search.
Unknown or warm-up regimes remain in the core model and are excluded from the
strict aligned subset.

### 19.14 News handling in V1

The current `calendar_events` data supports only event name, timestamp and
impact. It does not support a causal hot/in-line/cool CPI surprise calculation.

Therefore V1 provides two price-identical comparisons:

1. `news_blackout_disabled`: no calendar entry filter; and
2. `news_blackout_enabled`: reject a new entry from five minutes before through
   fifteen minutes after a stored high-impact event.

The blackout comparison must not be labeled a CPI-direction model. CPI
surprise and cross-asset confirmation remain V3 work.

### 19.15 Execution-cost contract

Snapshot the exact shared cost configuration with every run. The reference
profile is:

- taker fee: `0.06%` of notional per side;
- half-spread: `0.01%` per side;
- slippage: one exchange tick per side; and
- recorded funding events during the holding period.

Also run a named cost-stress profile with doubled modeled spread and slippage.
Do not overwrite the reference profile with an alternative fee assumption
after results are known.

### 19.16 Time-agnostic baseline

The comparison baseline uses the same event definitions, ATR rules, execution,
stop, target, time exit and costs, but removes the London-time requirement.

For each weekday UTC date:

1. evaluate twelve fixed anchors at `00:00`, `02:00`, ..., `22:00 UTC`;
2. build a six-hour reference range ending at each anchor;
3. observe the following two hours;
4. accept only the first qualified setup for that UTC date; and
5. apply the same one-trade-per-day rule.

This baseline measures whether concentrating the setup around London improves
quality relative to similar opportunities throughout the day.

### 19.17 Required V1 event reasons

Each eligible session must end with exactly one primary reason:

- `data_gap`;
- `pre_open_swept`;
- `no_sweep`;
- `swept_not_reclaimed`;
- `reclaimed_no_displacement`;
- `both_sides_swept`;
- `news_blackout`;
- `risk_rejected`;
- `missing_entry_candle`; or
- `trade_entered`.

This event ledger allows the report to reconcile every eligible London date
without treating no-trade sessions as missing data.

### 19.18 V1 minimum verification scenarios

Before running the full database backtest, automated tests must cover at least:

1. London Open resolves to different UTC hours on a winter and summer date.
2. A missing Asian candle invalidates the session.
3. A pre-open sweep skips the session.
4. A sweep without reclaim produces no trade.
5. A same-candle sweep and reclaim may advance state.
6. The reclaim candle cannot also be the V1 displacement candle.
7. A later qualifying displacement enters at the next candle open.
8. A sweep of the opposite boundary before entry invalidates the session.
9. A final-window displacement may fill on the immediately following candle.
10. Stop and target on the same candle resolve to the stop.
11. Time exit fills at the open after sixteen complete holding candles.
12. A second setup on the same London date is ignored.
13. Fixed-notional quantity respects contract multiplier and step size.
14. Appending future candles does not change any earlier signal.
15. Prepared historical evaluation and incremental live evaluation produce the
    same signals.

## 20. Deferred research contracts

The following models require additional definitions or stored data. They must
not be presented as included in V1.

### 20.1 V2 breakout continuation

Use these predeclared starting hypotheses when V2 begins:

- breakout close exceeds the Asian boundary by `0.05 ATR`;
- acceptance requires two consecutive closes outside the range;
- a retest must occur within four subsequent `15m` candles;
- the retest may penetrate the boundary by at most `0.1 ATR` and must close
  outside the range;
- directional displacement after the retest uses the V1 `0.8 ATR` and outer
  `25%` rules;
- stop is `0.1 ATR` beyond the retest extreme;
- target is `2R`; and
- maximum holding time remains sixteen `15m` candles.

These are fixed research defaults, not validated trading parameters. V2 must
have its own event ledger and report, separate from sweep reversal.

### 20.2 V2 MSS filter

The starting causal MSS definition uses confirmed `2-left/2-right` pivots. A
pivot becomes available only after both right-side candles have closed.

After a Low sweep and reclaim, bullish MSS requires a close above the latest
confirmed swing high that was known at that decision timestamp. Bearish MSS is
the mirrored close below the latest confirmed swing low.

Test MSS as an optional child model of V1. Do not replace the price-only V1
baseline.

### 20.3 V3 macro-event data

Directional CPI testing requires a new immutable event record containing at
least:

- event identifier, country, category and release timestamp;
- actual, consensus and previous values;
- unit, seasonal-adjustment metadata and source;
- first-published values and any later revision; and
- ingestion timestamp proving when the value became available.

The CPI surprise tolerance and selected CPI series must be versioned before a
directional macro backtest is run.

### 20.4 V3 cross-asset data

DXY, EUR/USD, GBP/USD, yields and equity futures require timestamp-aligned
historical feeds with their own trading calendars and gap policies. A value is
eligible only when it was published or its source bar had closed by the crypto
decision timestamp.

### 20.5 V3 order-flow data

CVD, OI, aggressor-side volume and absorption cannot be reconstructed reliably
from OHLCV candles. Their use requires stored trade prints, open-interest
snapshots and, for absorption claims, historical order-book snapshots or an
explicit proxy model.

No missing V3 field may be silently replaced with zero, neutral or a fabricated
value. Runs without verified coverage must omit the corresponding model.

## 21. Definition of ready

Implementation may begin when all items below are true:

- the V1 parameter snapshot is accepted without hidden defaults;
- London time conversion and weekday eligibility are covered by tests;
- the database contains complete `15m`, `1h`, `4h` and funding coverage for the
  selected interval;
- symbol tick size, contract multiplier and quantity step are available;
- reference and stressed cost profiles are frozen;
- the time-agnostic baseline is enabled in the same run;
- the event ledger can reconcile every eligible session; and
- V2/V3 features are confirmed as out of scope for the first implementation.

Meeting this definition makes the V1 bot deterministic and backtestable. It
does not make it live-ready; live approval still requires the validation and
deployment gates in section 16.
