# Plan - Box Strategy

## Long-only risk review (2026-09-24)

The application strategy accepts long setups only. Reversal-short and
retest-short detection remains specified for research comparison, but those
setups do not create orders.

This is a risk filter, not a change to entry, stop loss, take profit, reward/risk
or position sizing. The decision follows a BTC/ETH backtest over rolling 1, 3,
6 and 12 month windows ending 2026-09-24: short trades were the dominant loss
source across the tested 15m, 1h and 4h profiles. The long-only variant must be
reported separately from the original two-sided baseline.

Twelve-month comparison on a 5,000 USDT account, after modeled fees, spread,
slippage and available funding. BTC funding history was unavailable, so BTC
figures remain before funding; ETH includes recorded funding events:

| Market / signal | Two-sided return / PF / max DD | Long-only return / PF / max DD | Long-only trades |
|---|---:|---:|---:|
| BTC 15m | -8.06% / 0.85 / -17.67% | -0.83% / 0.97 / -10.90% | 78 |
| ETH 15m | -15.57% / 0.74 / -19.33% | -9.68% / 0.73 / -18.10% | 83 |
| BTC 1h | +7.73% / 1.31 / -6.10% | +14.08% / 2.19 / -2.75% | 38 |
| ETH 1h | +3.12% / 1.13 / -8.03% | +8.70% / 1.73 / -3.46% | 38 |
| BTC 4h | +7.50% / 2.39 / -2.22% | +7.39% / 3.16 / -1.77% | 15 |
| ETH 4h | +0.78% / 1.08 / -4.84% | +4.02% / 2.35 / -2.47% | 11 |

Use 1h as the primary long-horizon research profile. It was positive for both
assets over 1, 6 and 12 months, with a losing 3-month sub-window that prevents
claiming continuous profitability. Keep 4h as supporting evidence only because
its sample has at most 15 long trades in one year. Do not approve 15m for
long-horizon use: removing shorts reduced drawdown but did not produce a stable
12-month edge. These rolling windows share the same end date and are robustness
views, not independent out-of-sample tests.

Keep the existing fixed TP and structural SL. On the 12-month 1h long-only
sample, BTC and ETH both won 14 of 38 trades; average net win was 3.76 and 2.97
times average net loss respectively. The strategy therefore depends on letting
the less frequent boundary-to-boundary winners run. Do not add break-even,
trailing, partial exits or a time exit without a separate controlled backtest.
Keep fixed planned risk at 25 USDT and the 10,000 USDT notional cap.

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). Existing
research results and the baseline execution model below retain their original
meaning. The application runner now separates signal candles from execution
candles for this strategy.

Review: `boxStrategySignalTimeframes` currently contains `1m`, `5m`, `15m`,
`1h`, `4h`. The strategy always needs `15m` candles to build the previous UTC
day. Worker/Telegram resolve that dynamically. The application uses the signal
timeframe itself for `1m` and `5m`; `15m`, `1h`, and `4h` signals use `15m`
execution candles, matching the higher-timeframe research runner.

Goal: Support the declared intraday `T` values while retaining the previous-day box.
Out of scope: Changing the box to a previous-week/month range or changing the
fixed-risk sizing model to a fixed-notional model.
Steps:

1. In `src/strategies/box-strategy/parameters.ts`, keep the supported intraday
   intervals aligned within a UTC day. Reuse
   `boxStrategyRequiredTimeframes` for all loaders; require `[T, 15m]` with
   duplicates removed.
2. In `signal.ts` and prepared evaluation, require the same complete 96-bar
   previous-day source. Keep pivot lookback, three-bar order expiry and signal
   ATR on `T`; simulate fills and exits on the execution timeframe and reset
   box/session state at the actual UTC boundary.
3. Test `1m`, `5m`, `15m`, `1h`, `4h` profiles for dependency parity, missing
   prior-day candles, expiry and next-open orders.

Verification: Static trace a 1h signal closing at 09:00: an order cannot fill
before 09:00, and three signal bars expire according to hourly boundaries,
not three `15m` execution bars. Repeat across midnight using the documented
box-reset/order-cancellation policy.

Daily/weekly triggers cannot retain an intraday box that expires each UTC
day. Reject them for this model. A different reference period and order
lifetime would require a separate strategy definition. A `12h` signal near
midnight may correctly be ineligible under the existing daily cutoff rules.

## Original baseline specification

> Status: research specification; not approved for live trading.
>
> Source: https://www.youtube.com/watch?v=R7XkHW9i65g

## 1. Objective and profit mechanism

Trade rejection and breakout-retest patterns around the previous UTC day's
high and low on BTC/ETH KuCoin USDT-margined perpetual futures.

- **Reversal:** enter near a rejected box boundary and target the opposite
  boundary. Seek a move across the range while limiting initial risk to the
  rejection cluster.
- **Continuation:** after a confirmed break, enter when the old boundary holds
  on a retest and target the nearest confirmed swing level in the trade direction.
- **Confirmation:** a rejection candle creates a candidate; a subsequent break
  of its extreme triggers entry. Boundary contact alone is not an entry.
- **Payoff filter:** require expected reward/risk after execution costs of at
  least 1.5. A large target does not imply a high probability of reaching it.

The profitability hypothesis is positive realized expectancy:

```text
expectancy = winProbability * averageNetWin
           - (1 - winProbability) * averageNetLoss

B = averageNetWin / averageNetLoss
breakEvenWinProbability = 1 / (1 + B)
```

Average net loss is a positive magnitude. At a constant realized B=1.5,
break-even win probability is 40%. The entry RR threshold alone does not
establish the realized payoff ratio or profitability.

## 2. Baseline parameters and data

| Parameter | Value |
|---|---|
| Exchange / product | kucoinfutures / linear USDT perpetuals |
| Symbols | BTC/USDT:USDT, ETH/USDT:USDT |
| Signal timeframe | 1m, 5m, 15m, 1h or 4h, evaluated independently |
| Execution timeframe | Signal timeframe for 1m/5m; otherwise 15m |
| Box session | UTC calendar day, including weekends |
| ATR | Wilder ATR14 from the candle before the signal |
| Boundary tolerance / TP offset | max(2 ticks, 0.20 * ATR) |
| Stop buffer | max(2 ticks, 0.10 * ATR) |
| Rejection wick | At least 1 body and 40% of candle range |
| Opposite-color lookback | 3 candles before the signal |
| Stop cluster | Signal candle and its 2 preceding candles |
| Pending lifetime | 3 signal candles, capped at UTC day-end |
| Breakout-state lifetime | 12 signal candles after the breakout |
| Pivot confirmation / lookback | 2 left, 2 right / 288 signal candles |
| Minimum expected net RR | 1.5, before funding |
| Risk budget / notional cap | 25 USDT / 10,000 USDT per trade |
| Position limit | One pending or position per symbol/timeframe experiment |

Use closed candles from the same exchange and product. Require aligned,
strictly increasing timestamps, no duplicates, and finite positive, consistent
OHLC values. Do not interpolate missing candles.

Read tick size, contract multiplier, quantity step and minimum quantity from
the symbol specification; coin quantity is not contract count. There is no
additional trend, volume, session or multi-timeframe filter.

## 3. Box, volatility and rejection calculations

Let s be the latest closed signal candle, t its open timestamp, T the signal
interval, and DAY 24 hours. Candle fields are O,H,L,C.

```text
D   = floor(t / DAY) * DAY
PDH = max(high) over [D-DAY, D)
PDL = min(low)  over [D-DAY, D)
MID = (PDH + PDL) / 2

TR[i]  = max(H[i]-L[i], abs(H[i]-C[i-1]), abs(L[i]-C[i-1]))
ATR[i] = (13 * ATR[i-1] + TR[i]) / 14
A      = ATR[s-1]
z      = max(2*tick, 0.20*A)
b      = max(2*tick, 0.10*A)

body  = abs(C-O)
range = H-L
upper = H-max(O,C)
lower = min(O,C)-L
touches(level) = H >= level-z AND L <= level+z
```

Build the previous-day box from exactly 96 consecutive closed 15m candles.
Freeze it throughout the current day. If the immediately preceding day is
incomplete, skip signals; do not substitute an older day or a rolling 24h range.

For each continuous signal-candle segment, initialize TR with H-L and ATR
with the mean of its first 14 TR values. Restart ATR warm-up after a missing
signal candle. Require a valid ATR[s-1] and three consecutive candles before s.

```text
bearishRejection =
  range > 0 AND C < O
  AND upper >= body AND upper >= 0.40*range
  AND C <= L + 0.50*range
  AND any(C[i] > O[i] for i in s-3..s-1)

bullishRejection =
  range > 0 AND C > O
  AND lower >= body AND lower >= 0.40*range
  AND C >= L + 0.50*range
  AND any(C[i] < O[i] for i in s-3..s-1)
```

Doji candles are not rejection signals. Inclusive wick thresholds are
intentional; the box-location inequalities below are strict.

## 4. Entry candidates

Evaluate at signal close in the following priority order; select at most one.
priorCloseIn(a,b) means any close among s-3..s-1 lies strictly inside (a,b).
The opposite-color candle and the prior-close candle need not be the same candle.

| Priority / setup | Required conditions | Valid planned entry |
|---|---|---|
| 1. reversal-short | bearishRejection; touches(PDH); MID < C < PDH; priorCloseIn(MID,PDH) | MID < entry < PDH |
| 2. reversal-long | bullishRejection; touches(PDL); PDL < C < MID; priorCloseIn(PDL,MID) | PDL < entry < MID |
| 3. retest-short | bearishRejection; active earlier breakdown; touches(PDL); C < PDL | 0 < entry < PDL |
| 4. retest-long | bullishRejection; active earlier breakout; touches(PDH); C > PDH | entry > PDH |

### Breakout state

```text
create up state:   C[k] > PDH + z[k]
create down state: C[k] < PDL - z[k]
state stores:     openTimestamp[k], z[k]

eligible retest:   0 < t-openTimestamp[k] <= 12*T
invalidate up:    C[s] < PDH - z[k]
invalidate down:  C[s] > PDL + z[k]
```

Reset both states at a new UTC day or signal-data gap. Expire/invalidate old
states before evaluating candidates; create new states only after candidate
evaluation. The breakout candle cannot also be its own retest.

Do not refresh an active same-direction state on subsequent outside closes.
After expiry/invalidation, the current candle may establish a new state.
Executing a trade does not consume the breakout state. State invalidation uses
the stored z[k]; the current retest uses z[s] for boundary contact.

## 5. Entry, stop loss and take profit

roundUp and roundDown round to the symbol's price tick.

| Direction | Stop-market entry trigger | Initial stop loss |
|---|---|---|
| Long | roundUp(H[s]+tick) | roundDown(min(L[s-2..s])-b) |
| Short | roundDown(L[s]-tick) | roundUp(max(H[s-2..s])+b) |

| Setup | Fixed take profit |
|---|---|
| Reversal long | roundDown(PDH-z) |
| Reversal short | roundUp(PDL+z) |
| Retest long | roundDown(pivotHigh-z) |
| Retest short | roundUp(pivotLow+z) |

TP uses the boundary offset z, not the stop buffer b.

A pivot is a strict extreme relative to two left and two right candles, with
all five consecutive. A pivot at j becomes available only when j+2 closes;
confirmation at the signal close is allowed. Retain pivots whose open timestamp
is strictly greater than `t-288*T`; confirmed pivots survive day changes and
gaps until they age out.

For retest long, choose the lowest confirmed pivot high strictly above entry.
For retest short, choose the highest confirmed pivot low strictly below entry.
Choose by price proximity, not recency. The lookback spans 3/12/48 days for
15m/1h/4h respectively.

Reject a candidate if its planned entry is outside the allowed region, the
required pivot is absent, the target is not finite, or barriers fail:

```text
Long:  stop < entry < target
Short: target < entry < stop
```

If the nearest pivot produces an invalid target after the offset, reject the
candidate instead of selecting a farther pivot. Freeze entry, stop and target
when the setup is created. No partial exits, trailing, break-even, stop widening
or maximum holding time. A filled position retains its original barriers
across UTC day changes.

## 6. Execution costs, reward/risk and sizing

Baseline assumptions: taker fee f=0.0006 per side, half-spread h=0.0001,
slippage n=1 tick per side. A cost-stress run uses h=0.0002,n=3, same fees.

```text
buyFill(price)  = price*(1+h) + tick*n
sellFill(price) = price*(1-h) - tick*n
```

Apply costs in the order direction: long entry buys and exits sell; short
entry sells and exits buy. Both SL and TP use market-exit costs, not maker-limit
costs. Triggers are tick-rounded; effective modeled fills are not rounded again.

Let E,S,P be expected cost-adjusted entry, stop and target fills; d=1 for
long and d=-1 for short.

```text
riskPerCoin   = d*(E-S) + f*(E+S)
rewardPerCoin = d*(P-E) - f*(E+P)
netRR         = rewardPerCoin / riskPerCoin
accept        = riskPerCoin > 0 AND netRR >= 1.5

rawCoin     = min(25/riskPerCoin, 10000/E)
contracts   = floor((rawCoin/contractMultiplier)/quantityStep)*quantityStep
quantity    = contracts * contractMultiplier
plannedRisk = quantity * riskPerCoin
```

Reject quantity below the exchange minimum; never round up to qualify.
Use a fixed 25 USDT planned risk budget without compounding. The 10,000 USDT
limit is planned notional, not leverage or an equity-based sizing rule.

Freeze quantity before entry. A gap fill does not retroactively resize or
cancel an already triggered order; realized loss/notional may exceed planned
limits. The baseline model excludes funding, margin, liquidation and separate
latency simulation. Its netRR and P&L must therefore be labeled before funding.

## 7. Signal timing and order lifecycle

```text
availableAt = t+T
expiresAt   = min(t+4*T, D+DAY)
validWindow = [availableAt, expiresAt)
```

No entry may occur inside the signal candle. Reject availableAt >= expiresAt,
including a signal closing exactly at UTC midnight. Lifetime is three signal
candles, not three 15m execution candles.

Allow one pending order or position per symbol/timeframe experiment. Discard
new signals while busy; do not queue them, replace pending orders, add exposure
or reverse positions.

Process each execution candle in this order:

1. On an execution-data gap, clear pending orders and mark an open observation
   data-gap with unknown exit/P&L. This is censoring, not a simulated fill.
2. Cancel pending orders at or after expiry.
3. If idle, accept only a setup available at this candle's open timestamp,
   subject to RR and sizing checks.
4. Cancel pending long if open <= SL, or pending short if open >= SL, before
   checking entry. Otherwise check entry; if entry does not trigger but SL is
   touched, cancel pending.
5. Long triggers when high >= entry and fills from max(entry,open); short
   triggers when low <= entry and fills from min(entry,open). Apply costs.
   A gap beyond the allowed entry region still fills and is marked ambiguous.
6. Check position exits, including on the entry candle. If entry and SL are
   both touched, assume entry then SL and flag ambiguity. If both SL and TP
   are touched, SL wins and the trade is flagged ambiguous.
7. Close the entire position on exit. Do not reuse a signal discarded earlier
   in that same execution candle.

Before costs, stop fills use min(open,SL) for long and max(open,SL) for
short. Target fills use the fixed target without favorable gap improvement.
If only entry and TP are touched, the simulator records a same-candle target;
OHLC cannot establish the actual intrabar sequence. The ambiguity flag does
not resolve every possible ordering uncertainty.

A signal-data gap resets signal warm-up and breakout state but does not censor
a position if execution candles remain continuous. At sample-end, an open
position is omitted from completed trades and counted in
`censoredOpenPositionCount` rather than force-closed. Exit, fees, P&L and R are
not manufactured from the last close. Filled positions are not closed merely
because the day changes.

For a completed trade:

```text
fees = f * quantity * (actualEntry + actualExit)
netExFunding = d*(actualExit-actualEntry)*quantity - fees
realizedR = netExFunding / plannedRisk
```

## 8. Verification cases

- **Causality and expiry:** a 1h signal opening at 08:00 and closing at 09:00
  can first fill on the 09:00 execution candle; pending expires before the
  12:00 candle is evaluated, not at 09:45. A signal closing at 00:00 UTC is rejected.
- **Barrier calculation:** PDH=100000, PDL=96000, ATR=200, tick=0.1 gives
  z=40 and b=20. A valid short signal with H=100060, L=99880 and cluster
  maximum H=100060 gives entry=99879.9, SL=100080 and TP=96040.
- **Ambiguous fill:** buy trigger=100, SL=98, TP=106 and execution OHLC
  99/107/97/101 produces entry then SL, with adverse costs and ambiguity flagged.
- **No look-ahead:** appending future candles must not change earlier setups;
  a pivot cannot be used before its second right candle closes.
- **Rejection gates:** missing prior-day box, netRR < 1.5, invalid barriers or
  below-minimum contract quantity must produce no pending order.

Reference implementation: `scripts/research-box-strategy/model.ts`.
Regression cases: `scripts/research-box-strategy/model.test.ts`.
