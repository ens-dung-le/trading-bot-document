# Plan E - Negative DCA strategy for paper trade

## Timeframe portability review (2026-09-16)

Apply the [shared timeframe contract](strategy-timeframe-support.md). The
minute-based examples below remain the baseline; portability is pending.

Review: the parameter schema already accepts several timeframes, but
`strategy.ts` declares `[Timeframe.OneMinute]` and
`evaluateDcaStopStraddle` throws when the parameter differs from that array.
The current schema therefore overstates executable timeframe support.

Goal: Make `dca-stop-straddle` use a validated trigger `T` consistently.
Out of scope: Changing ladder multipliers, basket risk, OCO rules or live-mode
restrictions.
Steps:

1. In `src/strategies/dca-stop-straddle/parameters.ts`, reuse the shared enum.
   In `strategy.ts`, replace the fixed dependency/check with the resolved
   profile's `[T]` and validate that the requested closed series is available.
2. Preserve `initialOrderExpiryCandleCount` and `cooldownCandleCount` as `T`
   bars, including zero-as-no-expiry behavior. Keep ticker-driven paper fills
   distinct from candle-driven historical replay; changing `T` must not slow
   protective exchange orders or funding accrual to one update per bar.
3. Test hourly OCO activation, both stops touched in one candle, multiple DCA
   levels crossed, gap fills, basket TP/stop ordering, cooldown and restart.
   Preserve the existing conservative ambiguity policy; do not infer tick
   chronology from a large hourly range.

Verification: Static trace a basket closed on an hourly candle with cooldown
two: the configured two-bar condition must not become two minutes. Compare
identical available event streams across replay/paper and disclose execution
resolution when only OHLC is available. An hourly configuration must no longer
throw merely because the previous default dependency was `1m`.

Sizing remains a basket model: $100 initial margin at 10x requests $1,000
initial notional before rounding, and subsequent DCA legs grow independently.

## Original baseline specification

## 1. Objective

Build a paper-trade strategy with the proposed name `dca-stop-straddle`, operating in a cycle:

1. Without a position, simultaneously place a `buy stop` above the anchor and a `sell stop` below the anchor.
2. Whichever side the price touches first, open a position on that side and cancel the remaining order.
3. If the price moves in the right direction, close the entire basket at take profit.
4. If price moves adversely, add an entry in the same direction with quantity equal to the previous entry multiplied by the selected factor; the initial profiles are `1.5`, `1.68`, and `1.7`.
5. After each entry, recalculate the weighted-average entry and basket take profit, ensuring the target net profit remains positive after fees, funding, and the slippage buffer.
6. When the basket closes, wait for cooldown and then start a new cycle.

The first version is registered only for `paper` and `backtest`. It must not run when `TRADING_MODE=live`.

## 1.1 Confirmation behavior from reference MT5 settings

MT5 image is a behavioral reference, not a configuration that can copy whole numbers to crypto. Points that can be directly confirmed:

| MT5 settings/behavior | Value seen in photos | Crypto mapping |
|---|---:|---|
| `D1` | `30` point | Initial buy/sell stop distance equals basis points/rate |
| `V1` | `0.01` lots | Initial order size according to fixed USDT notional or balance rate |
| `N` | `20` | Configured maximum entry count, still limited by effective risk cap |
| `S1` | `135` points | Near-stage DCA spacing equals basis points/rate |
| `S2` | `235` points | Far-stage DCA spacing equals basis points/rate |
| `T1` | `70` points | Do not copy directly; Replace with cost-aware minimum net profit target |
| `E1` | `1.68` | `sizeMultiplier = 1.68` |
| `R1` | `0.0` | Fixed-size mode; balance-rate mode is not used in this setting |

The remaining abbreviated fields such as `P1`, `P2`, `H1`, `F1...F4`, `T2...T6`, `M1`, `M2`, and `N1` cannot be interpreted reliably from the image. Do not guess or implement them until their behavior is documented.

The actual volume string in the image is:

```text
0.01 -> 0.02 -> 0.03 -> 0.05 -> 0.08 -> 0.13 -> 0.22
```

This sequence applies `previousSize * 1.68` and then normalizes to the lot step. Seven buy orders have a weighted-average entry near `4601.21` and share TP `4601.91`, placing TP approximately `0.70` above the average and matching XAUUSD's `T1 = 70` point setting.

Crypto must maintain the principle of "a common TP is profitable compared to the weighted average", but the TP price is calculated from the cost and tick size of each market instead of a fixed addition of 70 points.

## 1.2 ETHUSD photo: pending order available TP

Second MT5 photo taken with expert on `ETHUSD, M5`, bid `2435.59` and ask `2438.10`. The Toolbox shows exactly two pending orders, each of which already has `T/P` from the time of placement:

| Order | Entry price | T/P | Distance |
|---|---:|---:|---:|
| buy stop | `2440.79` | `2450.79` | `1000 point = 10.00` |
| sell stop | `2433.09` | `2423.09` | `1000 point = 10.00` |

`sell stop = bid − D1 = 2435.59 − 2.50` matches exactly, so `D1 = 250 point` is the distance from the current price to the stop entry. `T1 = 1000 point` is the distance from the entry to the TP, and the seven legs in the previous XAU image show that the TP is moved to `weighted average + T1` each time a new leg becomes available.

Converted to bps at 2435.59 (1 point = 0.01, so 1 bps ≈ 24 points):

| MT5 | Point | Crypto (bps) | Parameters |
|---|---:|---:|---|
| `D1` | 250 | 10 | `initialTriggerDistanceBps` |
| `S1` | 250 | 10 | `nearDcaStepBps` |
| `S2` | 350 | 14 | `farDcaStepBps` |
| `T1` | 1000 | 41 total | `takeProfitBps` 27 + fee 12 + buffer 2 |
| `E1` | — | — | `sizeMultiplier = 1.68` |
| `N` | 20 | — | `maximumEntryCount = 20` |
| `V1` | 0.01 lots | — | `initialOrderNotionalAmount = 25`, exactly 1 ETHUSDTM contract (0.01 ETH) |

This set is located at `mt5LikeEthDcaStopStraddlePaperParameters`. `farStepStartsAtEntry` still cannot be deduced from the image, so the value of the old profile remains unchanged.

Three behavior points that this image determines, and the strategy has been modified accordingly:

1. The two stop entries are pre-set and **have available TP** calculated from their trigger price, not from the anchor.
2. TP is recalculated from the weighted average **including the newly filled leg**, because that leg fills immediately after the triggering candle.
3. DCA legs are **not** placed as resting limit orders. Each rung is a distance threshold, and its leg is sent to market only when the candle close reaches that threshold. Only the two initial stop entries are resting orders.

TP is still not a resting reduce-only exchange order: it is stored on the position as `protectiveTakeProfitPrice`, and the basket closes with a market order when the candle close crosses the target. A resting TP limit would become stale as soon as a new leg fills, while the current approach is more conservative.

## 2. Mandatory warning

Negative DCA does not guarantee that the price will return to take profit. When the market is one-sided, quantity increases exponentially and margin can increase faster than price recovery.

So the first version must have:

- Limit the number of entries in a basket.
- Check in advance the total margin of the entire ladder.
- Limit basket notional and account exposure.
- Catastrophe stop is outside the last DCA level.
- Circuit breaker and manual stop.
- Paper-only guard in both menu and engine, not only hides Telegram button.

Do not implement the “hold indefinitely until TP” variant because that variant has no finite risk and can end in liquidation.

## 3. Acceptance criteria

The strategy is complete when:

1. The idle state correctly creates two initial stop orders belonging to the same OCO group.
2. The first touched order fills, and its sibling is canceled before an opposite position can open.
3. The initial fill creates a basket with `entryCount = 1` and a basket TP.
4. An adverse move reaching the near/far DCA spacing from the current entry creates the next same-side entry.
5. With multiplier `1.5`, the quantity sequence is `Q0, 1.5Q0, 2.25Q0, 3.375Q0...` after normalizing by quantity step.
6. With multiplier `1.68`, the normalized quantity sequence matches the MT5 profile pattern: `0.01, 0.02, 0.03, 0.05, 0.08, 0.13, 0.22...` when an equivalent quantity step is used.
7. With multiplier `1.7`, quantity sequence is `Q0, 1.7Q0, 2.89Q0, 4.913Q0...` after normalization.
8. After each fill, total quantity, weighted average entry, margin, liquidation estimate and basket TP are recalculated.
9. Basket TP is also recalculated when funding/cost state changes, even if there is no new fill.
10. If the basket TP fills, the paper model closes the entire quantity as reduce-only and net PnL is greater than or equal to the configured target.
11. TP is the valid price tick closest to the weighted average but still exceeds target net PnL after fee/funding/slippage.
12. Do not add entries that exceed the configured/effective maximum entry count, maximum basket margin or exposure limit.
13. Backtest and paper replay with candles create the same order sequence, fills, average entry and PnL within the cost model limits.
14. Restart does not cause the entry legs to be lost or the bot to regenerate initial OCO while the basket is open.
15. Strategy cannot be launched in live mode.

## 4. Non-goals

- Do not hedge long and short at the same time on the same symbol.
- Do not allow arbitrary user-entered multipliers; accept only validated `1.5`, `1.68`, or `1.7` profiles in the first version.
- Do not change the multiplier while a basket is open.
- Do not add trailing stop, break-even stop or exit indicator to the first version.
- Do not automatically optimize DCA spacing.
- Do not set DCA according to volatility in the first version; spacing uses two fixed-rate stages and is recorded in parameters.
- Leg-by-leg partial closes are not supported; take profit closes the entire basket.
- Do not perform live execution until paper/backtest parity and soak-test pass.

## 5. Current engine gaps

This is not just another strategy file. The current engine has limitations that must be addressed at the layer that owns each responsibility:

- Backtest keeps only one `pendingEntry` and rejects the second pending entry.
- Backtest returns `Position scaling is not implemented` when there is an entry in the same direction.
- Realtime engine ignores signals in the same direction when it has a position.
- Trading engine is sending market orders even though `Signal` has declared stop/limit entry.
- Paper executor fills order immediately upon submission, does not hold stop/limit order waiting for candle trigger.
- `OpenPosition` only has aggregate quantity/price, does not have entry legs and last entry price.
- Executor does not have an OCO group or basket take-profit lifecycle.
Plan must extend these capabilities in a reusable way, without stuffing DCA logic directly into `TradingEngineService`.

## 6. State machine

```text
Idle
  |
  | arm initial buy/sell stop
  v
Armed (two OCO entries)
  | buy stop fills                  | sell stop fills
  v                                 v
LongBasket                       ShortBasket
  | TP fills                        | TP fills
  |                                 |
  +-------------> Closing <---------+
                      |
                      v
                   Cooldown
                      |
                      v
                     Idle

LongBasket / ShortBasket
  |
  | adverse price reaches next DCA level
  v
same basket with one additional entry and a recalculated TP

Any state
  |
  | maximum risk, catastrophe stop, circuit breaker or manual stop
  v
Paused / Closed
```

### 6.1 State invariants

- `Idle`: no pending DCA orders and no position.
- `Armed`: exactly two initial entry orders, one buy and one sell, total `orderGroupId`.
- `LongBasket`: only buy entry legs; no sell entry must exist.
- `ShortBasket`: only sell entry legs; buy entry cannot exist.
- A basket can only have a maximum of one next-DCA order and one basket-TP order.
- Basket TP quantity is always equal to current total basket quantity.
- Entry count cannot be reduced; reset only when basket is closed.

## 7. Parameters

Create `src/strategies/dca-stop-straddle/parameters.ts` with validated schema:

```ts
interface DcaStopStraddleParameters {
  triggerTimeframe: Timeframe;

  initialTriggerDistanceBps: number;
  initialSizingMode: "fixed_notional" | "balance_rate";
  initialOrderNotionalAmount: number;
  initialOrderBalanceRate: number;

  nearDcaStepBps: number;
  farDcaStepBps: number;
  farStepStartsAtEntry: number;
  sizeMultiplier: 1.5 | 1.68 | 1.7;
  maximumEntryCount: number;

  targetNetProfitRate: number;
  minimumNetProfitAmount: number;
  slippageBufferRate: number;
  exitFeeAssumption: "worst_case_taker";

  catastropheStopBufferRate: number;
  maximumBasketMarginRate: number;

  initialOrderExpiryCandleCount: number;
  cooldownCandleCount: number;
}
```

### 7.1 Meaning

- `initialTriggerDistanceBps`: crypto equivalent of initial stop distance; basis points instead of GOLD points.
- `initialSizingMode`: choose fixed USDT notional like `V1` or balance-rate like `R1`.
- `initialOrderNotionalAmount`: initial USDT notional when using fixed mode.
- `initialOrderBalanceRate`: balance part used for initial order when using rate mode.
- `nearDcaStepBps`: crypto equivalent of `S1`, used for close entries.
- `farDcaStepBps`: crypto equivalent of `S2`, used for deeper entries.
- `farStepStartsAtEntry`: entry sequence starts switching from near spacing to far spacing.
- `sizeMultiplier`: multiplier of new entry compared to previous entry.
- `maximumEntryCount`: includes initial entry.
- `targetNetProfitRate`: target net profit calculated on total entry notional, not raw price distance.
- `minimumNetProfitAmount`: minimum positive USDT level after cost.
- `slippageBufferRate`: additional buffer so that TP not only breaks even on the ideal model.
- `exitFeeAssumption`: always use taker fee for planned exit to avoid overly optimistic assessments.
- `catastropheStopBufferRate`: approximately outside the final planned DCA level.
- `maximumBasketMarginRate`: maximum total planned margin compared to account balance.

### 7.2 Validation

- `sizeMultiplier` is just `1.5`, `1.68` or `1.7`.
- All active rates must be finite and positive; An unchecked sizing field may be equal to `0`.
- `maximumEntryCount >= 1` and must have a hard configuration ceiling from the server.
- Fixed mode requires `initialOrderNotionalAmount > 0` and ignores balance rate.
- Balance-rate mode requires `0 < initialOrderBalanceRate <= server safety ceiling` and ignores fixed amount.
- `initialOrderNotionalAmount`, `initialOrderBalanceRate` and `maximumBasketMarginRate` must not exceed the safety policy in `.env`.
- `farStepStartsAtEntry` must be in `2..maximumEntryCount`.
- Catastrophe stop must be farther than the last DCA level but still before the liquidation safety boundary.
- If the entire planned ladder does not fit risk guard, then do not arm initial OCO.

### 7.3 Crypto unit instead of GOLD point

All distance settings use basis points:

```text
distanceRate  = distanceBps / 10_000
distancePrice = referencePrice * distanceRate
```

The price is then normalized to `priceTickSize`. Because BTC, ETH, and altcoins have different price scales, do not use one absolute point distance across all symbols.

Do not directly copy `D1=30`, `S1=135`, `S2=235`, or `T1=70` into default basis-point values. Those MT5 values are XAUUSD points; they validate the parameter structure, not appropriate spacing for crypto markets.

### 7.4 MT5-like crypto profile template

Profile is structurally equivalent, not an accepted set of numbers:

```text
profileName                  = mt5-like-1.68
initialSizingMode            = fixed_notional
sizeMultiplier               = 1.68
configuredMaximumEntryCount  = 20
initialTriggerDistanceBps    = TBD by symbol backtest
nearDcaStepBps               = TBD by symbol backtest
farDcaStepBps                = TBD by symbol backtest
farStepStartsAtEntry         = TBD after behavior is confirmed
targetNetProfitRate          = smallest tested positive target above modeled cost
minimumNetProfitAmount       = configured positive USDT floor
exitFeeAssumption            = worst_case_taker
```

Do not seed `TBD` values by directly converting GOLD points to crypto bps. Each symbol/configuration version must have its own set of backtests.

## 8. Entry ladder

### 8.1 Initial OCO

When the cycle starts at anchor price `A`:

```text
initialRate = initialTriggerDistanceBps / 10_000
buyStop     = A * (1 + initialRate)
sellStop    = A * (1 - initialRate)
```

The two orders are stop-market and have the same `orderGroupId` with policy `cancel_siblings_on_fill`.

Anchor is taken from the close of the candle that was closed at the time of arm. Do not use forming candles.

### 8.2 DCA levels

Spacing has two stages similar to the `S1/S2` structure in the reference setting:

```text
stepBps(n) = n < farStepStartsAtEntry ? nearDcaStepBps : farDcaStepBps
stepRate(n) = stepBps(n) / 10_000
```

After initial long fill at `P0`:

```text
P1 = P0 * (1 - stepRate(1))
P2 = P1 * (1 - stepRate(2))
...
```

After initial short fill:

```text
P1 = P0 * (1 + stepRate(1))
P2 = P1 * (1 + stepRate(2))
...
```

Long basket uses buy-limit at lower DCA levels. Short basket uses sell-limit at higher DCA levels.

Place the next DCA order only after the current fill has been recorded and the basket state has updated.

### 8.3 Quantity sequence

```text
Q0 = normalized initial quantity
Qn = normalizeQuantity(Q(n-1) * sizeMultiplier)
```

Do not recalculate each level from the new balance because that makes the sequence no longer have the correct exponential level of the previous entry.

Before arming, build the entire planned ladder and check:

```text
plannedMargin = sum(Qn * Pn / leverage)
plannedNotional = sum(Qn * Pn)
```

If the planned margin/notional exceeds the limit, refuse the cycle before initial fill instead of receiving the first entry and then discovering that DCA cannot continue.

### 8.4 Configured max and effective max

Setting MT5 has `N=20`, but crypto does not allow 20 entries by default just because the config says 20.

With 20 entries before rounding:

| Multiplier | Last entry / Q0 | Total quantity / Q0 |
|---:|---:|---:|
| `1.5` | about `2,216.84` | about `6,648.51` |
| `1.68` | about `19,093.10` | about `47,169.71` |
| `1.7` | about `23,907.24` | about `58,059.02` |

Therefore:

```text
effectiveMaximumEntryCount = min(
  configuredMaximumEntryCount,
  largestEntryCountPassingFullLadderRiskPreflight
)
```

Telegram must display both configured max and effective max. If effective max is lower than configured max, it is a normal risk decision, not an error.

## 9. Basket accounting

Each fill is saved as a leg:

```ts
interface PositionEntryLeg {
  sequence: number;
  clientOrderId: string;
  side: OrderSide;
  quantity: number;
  fillPrice: number;
  feeAmount: number;
  filledAtTimestamp: number;
}
```

Aggregate basket:

```text
totalQuantity = sum(qi)
costBasis     = sum(pi * qi)
averageEntry  = costBasis / totalQuantity
entryFees     = sum(entryFeei)
```

After each fill, you must recompute:

- Total quantity.
- Average entry.
- Initial margin.
- Estimated liquidation price.
- Total entry fees.
- Next DCA price.
- Basket take profit.
- Catastrophe stop coverage for total quantity.

## 10. Basket take-profit is positive after expenses

The goal is not just to place the TP above/below the average entry. TP must cover:

- All entry fees.
- Estimated exit fee.
- Funding has accumulated.
- Slippage buffer.
- Target net profit.

Set:

```text
Q = total quantity
C = sum(entryPrice_i * quantity_i)
E = total entry fees
F = accumulated funding paid; received funding is represented as a negative value
S = slippage buffer amount
R = max(minimumNetProfitAmount, totalEntryNotional * targetNetProfitRate)
f = expected exit fee rate
```

### 10.1 Long basket

Net PnL at exit price `T`:

```text
net = T*Q - C - E - T*Q*f - F - S
```

Minimum TP:

```text
T >= (C + E + F + S + R) / (Q * (1 - f))
```

### 10.2 Short basket

```text
net = C - T*Q - E - T*Q*f - F - S
```

Maximum TP:

```text
T <= (C - E - F - S - R) / (Q * (1 + f))
```

The final TP price must be normalized according to the tick size in a way that does not reduce the target profit:

- Long TP round up.
- Short TP round down.

After normalization, the calculator must recalculate the estimated net PnL and reject if it is still smaller than the target.

### 10.3 Fee model for KuCoin crypto

Basket TP uses realistic costs and conservative assumptions:

- Each filled leg uses the actual fee amount recorded by the paper fill model.
- Initial stop-market is charged taker fee.
- DCA limit uses maker fee only when model filling proves the order has rested; If not clear, use taker fee.
- Planned basket exit always uses taker fee in the first version.
- Funding has been added/subtracted directly.
- Slippage buffer calculated on total exit notional.
- Quantity is converted/normalized via contract multiplier before calculating PnL.

If the fee schedule changes or a new funding event appears while the basket is open, the TP must be recomputed and replaced.

### 10.4 Select the nearest sufficient TP tick

After getting raw TP from the formula:

1. Long round up tick; short round down tick.
2. Recalculate net PnL using the correct fill/cost model.
3. If net PnL has not reached `R`, move each tick further in a beneficial direction.
4. Select the first tick to reach the target.

The result is the TP closest to the weighted-average entry but still has a positive expected net PnL according to the target, instead of copying GOLD's `70 point` distance.

## 11. Catastrophe stop and risk guard

Since the negative DCA has no stop near the entry, the strategy uses a pre-calculated basket catastrophe stop.

For example:

```text
longStop  = lastPlannedDcaPrice * (1 - catastropheStopBufferRate)
shortStop = lastPlannedDcaPrice * (1 + catastropheStopBufferRate)
```

Risk guard must be run twice:

1. Preflight the entire planned ladder before arming.
2. Before each scale fill, use current balance, used margin and actual fill prices.

If the next scale is rejected by risk guard:

- Cancel next DCA order.
- Keep basket TP and current catastrophe stop.
- Mark basket `scaling_exhausted`.
- Send Telegram warning.
- Do not silently reduce quantity because it will change the backtested strategy.

Catastrophe stop must always be far from the liquidation price according to the safety buffer. If not, the cycle cannot be armed.

## 12. Order lifecycle

### 12.1 Capability needs to be added

Extend execution model to support:

- Multiple pending entry orders on the same symbol.
- `orderGroupId` and OCO cancellation.
- Stop/limit order is actually pending in the paper executor.
- Same-side position scaling.
- A reduce-only basket TP that can be replaced after each scale fill.
- Catastrophe stop quantity is updated according to total basket quantity.

### 12.2 Paper order processing

Add candle handling step in paper executor/order book:

```text
processClosedCandle(symbol, candle) -> execution events
```

Order fill rules:

- Buy stop fill when candle high touches trigger.
- Sell stop fill when candle low touches trigger.
- Buy limit fill when candle low touches limit.
- Sell limit fill when candle high touches limit.
- Gap through stop using candle open as intended market reference before execution cost.
- Limit gap in the favorable direction can be filled at open, then apply uniform rule of fill simulator.

Paper executor cannot fill stop/limit right at `submitOrder()` as currently.

### 12.3 OCO atomicity

When an initial order fills:

1. Record fill.
2. Mark OCO group triggered.
3. Cancel sibling order.
4. Only then create basket position and next DCA order.

In paper/backtest, this sequence must be atomic in a reducer step so it cannot fill both long and short because the two handlers run together.

## 13. Intrabar ambiguity

OHLC candle does not indicate whether the high or low occurred first. Two cases need to be determined by policy:

### 13.1 Both buy stop and sell stop are touched in the same candle

Default policy: `reject_ambiguous_cycle`.

- Can't guess which side happens first.
- Cancel both simulated orders.
- Logs an ambiguous event to the report.
- Rearm from the next candle after cooldown.

Paper realtime if there is a real tick/order event, use fill that happened first. Backtest candle-only uses the above policy to not create fake results.

### 13.2 A DCA level and basket TP are touched in the same candle

Default backtest policy: `adverse_fill_before_take_profit`.

1. Fill DCA order.
2. Recompute basket average and TP.
3. Then check for new TP in the remaining range according to the conservative model.

If intrabar order cannot be proven, report marked with ambiguous count. Policy must be written to `parameters.json`.

## 14. Strategy contract

Does not change the behavior of existing single-entry strategies.

Minimum signal/context expansion:

```ts
interface StrategyContext<P> {
  // existing fields
  pendingOrders: readonly PendingOrderView[];
  positionEntries: readonly PositionEntryLeg[];
}

interface EntryInstruction {
  role: "initial" | "scale";
  orderGroupId: string | null;
  cancelSiblingsOnFill: boolean;
  quantityMode:
    | { kind: "initial_notional_rate"; rate: number }
    | { kind: "previous_fill_multiplier"; multiplier: 1.5 | 1.68 | 1.7 };
}
```

Can be represented by optional metadata on `Signal` or a discriminated signal union. Select union if optional fields cause the current signal to have many invalid combinations.

Pure strategy decision:

- When does arm initial OCO.
- Initial stop prices.
- When do you need next scale entry.
- Entry role and multiplier.

Engine/execution decision:

- Quantity normalization.
- Risk approval.
- Fill lifecycle.
- Aggregate position accounting.
- Basket TP replacement.

## 15. Persistence model

To restart without losing the basket, add an entry legs table or expand the schema equivalent:

```text
position_entries
  id
  position_id
  sequence
  order_id
  side
  quantity
  fill_price
  fee_amount
  filled_at
  UNIQUE(position_id, sequence)
```

Expand order metadata:

```text
order_group_id
entry_role            initial | scale | basket_tp | catastrophe_stop
entry_sequence
parent_position_id
```

Position aggregate continues to hold:

- Total quantity.
- Average entry.
- Basket take profit.
- Catastrophe stop.
- Accumulated funding.

When restarting paper session:

1. Load aggregate position.
2. Load ordered entry legs.
3. Load pending next-DCA/TP/stop orders.
4. Recompute basket invariants.
5. Refuse new entries if stored aggregate does not match legs.

Migration only expands schema; Do not change the old position data.

## 16. Backtest changes

Refactor single `pendingEntry` into a pending-order collection with deterministic processing order.

Backtest position must support:

- Entry legs.
- Same-side quantity addition.
- Weighted average entry.
- Fees/slippage on each fill.
- Funding on aggregate quantity over time.
- Margin/liquidation recompute after scale.
- Basket TP changes after each fill.
- A trade report representing the entire basket, including child legs.

Do not just turn on `allowsPositionScaling=true` and then cancel the current rejection. Need proof that accounting, liquidation, fees and reporting are all correct.

### 16.1 New Report metrics

- Basket count.
- Average and maximum entry count.
- Distribution level closed TP.
- Maximum basket notional.
- Maximum margin usage.
- Maximum adverse excursion before recovery.
- Average recovery time.
- `scaling_exhausted` count.
- Catastrophe-stop count.
- Liquidation count.
- Ambiguous candle count.
- Net profit after all fees/funding/slippage.

## 17. Paper/realtime flow

Order per candle:

1. Feed full closed candle into paper order book.
2. Resolve pending stop/limit/TP fills and OCO effects.
3. Update basket aggregate and persistence.
4. Recompute/replace TP and catastrophe stop if there is scale fill.
5. Update funding/equity/circuit breaker.
6. Evaluate strategy to arm a new order for the next candle.

This avoids the strategy of sending signals based on the old position state before the current candle's pending order is filled.

## 18. Telegram flow

Strategy appears under separate configurations, for example:

```text
dca-stop-straddle v1 · multiplier 1.5 · paper only
dca-stop-straddle v2 · multiplier 1.68 · MT5-like · paper only
dca-stop-straddle v3 · multiplier 1.7 · paper only
```

Do not allow multiplier changes during an active basket. Select a persisted strategy version and parameter set before arming the cycle.

Notice:

### Armed

```text
DCA cycle armed
Anchor: 100.00
Buy stop: 101.00
Sell stop: 99.00
Initial size: 100 USDT
Maximum entries: 5
Effective entries: 4
Planned maximum margin: 24.3%
```

### Initial fill

```text
DCA basket opened LONG
Entry 1: 101.00 x 0.99
Opposite sell stop canceled
Next DCA: 98.98
Basket TP: 101.47
```

### Scale fill

```text
DCA entry 2/5 filled
Fill: 98.98 x 1.48
Total quantity: 2.47
Average entry: 99.79
New basket TP: 100.24
Margin usage: 12.8%
```

### Close

```text
DCA basket closed at TP
Entries: 2
Gross PnL: +1.18 USDT
Fees/funding/slippage: -0.42 USDT
Net PnL: +0.76 USDT
```

`/status` and the paper menu display side, level, total quantity, average entry, next DCA, basket TP, catastrophe stop and unrealized PnL.

## 19. Implementation phases

### Phase E0 - Pure basket math

- Implement parameter validation.
- Implement ladder calculator.
- Implement weighted average and basket TP calculator.
- Implement pure basket state reducer.
- Property tests for multiplier, cost coverage and rounding.

Telegram or executor is not connected at this phase.

### Phase E1 - Pending paper orders and OCO

- Paper executor holds stop/limit orders instead of filling immediately.
- Process candle to execution events.
- Supports multiple pending entries and OCO group.
- Test gap, expiry, sibling cancellation and ambiguous candle.

### Phase E2 - Position scaling

- Entry legs and aggregate position accounting.
- Risk preflight for the entire ladder.
- Same-side scale fill.
- Basket TP/catastrophe stop replacement.
- Persistence and restart reconciliation.

### Phase E3 - Backtest parity

- Pending-order collection in backtest.
- Basket accounting and reports.
- Replay fixtures with hand-calculated answers.
- Paper replay and backtest should give the same result.

### Phase E4 - Strategy registry and Telegram

- Add `dca-stop-straddle` to the registry.
- Seed configuration multiplier `1.5`, `1.68` and `1.7`; profile `1.68` uses the same near/far spacing structure as the reference setting.
- Only show/start when mode is paper.
- Add notifications/status.

### Phase E5 - Soak test

- Run minimal paper through many regimes: strong trend, range, gap, high volatility.
- Monitor margin ladder, recovery time and ambiguous events.
- Do not open live scope during this phase.

## 20. Required test fixtures

1. Buy stop is touched first, sell sibling is canceled.
2. Sell stop is touched first, buy sibling is canceled.
3. Both stops hit the same candle, the cycle is marked as ambiguous.
4. Initial entry touches TP without DCA.
5. Long basket DCA two levels then TP.
6. Short basket DCA two levels then TP.
7. Quantity sequence multiplier `1.5`.
8. Quantity sequence multiplier `1.68` normalizes the sequence like MT5 reference.
9. Quantity sequence multiplier `1.7`.
10. Near/far spacing switches correctly at `farStepStartsAtEntry`.
11. Tick/quantity rounding still reaches target net profit.
12. Entry fees + conservative exit fee + funding move TP in the right direction.
13. Funding changes when the basket is open and TP is replaced.
14. Maker/taker uncertainty uses worst-case taker instead of making TP without cost.
15. DCA and TP touch the same candle using correct precedence.
16. Configured max `20` but effective max is lowered according to preflight margin.
17. Max entries reached did not create a new order.
18. Planned ladder exceeds margin so initial OCO cannot arm.
19. Actual fill causes risk guard to refuse next scale.
20. Catastrophe stop before liquidation.
21. Restart restores correct legs and pending orders.
22. Manual stop, cancel OCO/DCA orders and close basket.
23. Live mode rejects strategy even though configuration exists.

## 21. Files expected to change

### Strategy and math

- `src/strategies/dca-stop-straddle/strategy.ts` (new)
- `src/strategies/dca-stop-straddle/parameters.ts` (new)
- `src/strategies/dca-stop-straddle/basket-calculator.ts` (new)
- `src/strategies/dca-stop-straddle/basket-state.ts` (new)
- `src/strategies/strategy.registry.ts`
- `src/strategies/strategy.interface.ts`
- `src/common/types/signal.type.ts`
- `src/common/types/position.type.ts`

### Execution/engine

- `src/execution/executor.interface.ts`
- `src/execution/paper-executor.service.ts`
- `src/execution/order-router.service.ts`
- `src/execution/position-tracker.service.ts`
- `src/engine/trading-engine.service.ts`
- `src/risk/risk-guard.service.ts`

### Backtest/report

- `src/backtest/backtest-runner.service.ts`
- `src/backtest/fill-simulator.service.ts`
- `src/backtest/backtest.types.ts`
- `src/backtest/report.service.ts`
- `src/backtest/metrics.service.ts`

### Database/Telegram

- `src/database/schema/trading.schema.ts`
- `src/database/repositories/order.repository.ts`
- `src/database/repositories/position.repository.ts`
- `migrations/operational/<next>_dca_basket_entries.sql`
- `src/control/menu-view.ts`
- `src/control/control-handler.service.ts`
- `src/notification/message-template.ts`

## 22. Stop condition

Stop scope when:

- Initial OCO, same-side scaling and basket TP work in paper.
- Multiplier `1.5`, `1.68` and `1.7` are validated and tested.
- Basket TP proves net positive following modeled costs in fixtures.
- Backtest/paper parity passed.
- Risk preflight, max entries, catastrophe stop and live-mode guard all pass.
- Telegram shows full basket lifecycle.

Do not continue to live executor, parameter optimization or adaptive DCA spacing without specific requirements and paper evidence.
