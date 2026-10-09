# Adaptive DCA V2 - design proposal

Status: design only. No strategy is registered or enabled by this document.

## 1. Scope and assumptions

- New strategy name: `adaptive-dca-v2`. Keep `dca-stop-straddle` unchanged.
- First validation market: ETH/USDT:USDT Paper at 1x, long only. An initial buy
  plus at most four additional buys make one basket of at most five fills.
- Signals use closed 15m candles and the last fully closed 1h trend candle.
  No 1m or 5m candle may create an entry or increase the position.
- Interpret the 30% limit as total basket initial margin divided by current
  account equity. At 1x this also caps position notional at 30% of equity.
  Only one Adaptive DCA basket may be active per account.
- Interpret 2:1 as planned net basket reward divided by planned net basket
  loss at the common stop, recalculated after every fill. Fees, estimated
  slippage and accrued funding count in both sides of this calculation.
- Interpret a smaller candle as a smaller closed-candle true range relative
  to ATR on the selected signal timeframe. The first implementation uses 15m;
  a later timeframe option must not silently change these thresholds.

These are design assumptions to confirm before implementation, not claims of
measured profitability.

## 2. When a basket starts

1. Use only closed candles. Require at least 55 closed 1h candles and 50
   closed 15m candles; missing data pauses new entries.
2. Require the last closed 1h candle above EMA50 and EMA50 rising versus five
   completed 1h candles earlier. This avoids starting a long DCA basket during
   an established decline.
3. On 15m, require a close crossing above EMA20 after the preceding close was
   below EMA20. Enter at the next executable quote/open, never at the already
   observed signal close in a historical replay.
4. Compute Wilder ATR14 and the signal candle's true range using only closed
   15m data. Reject a new entry if ATR is unavailable, the signal true range
   exceeds 2 ATR, the exchange minimum cannot be met, or the risk preflight
   fails. A large shock is not permission to buy a larger position.
5. Permit one active basket per account. Re-arm only after the
   previous basket exits and four complete 15m candles have passed.

## 3. Ladder and adaptive size

Freeze `A = ATR14` and first fill price `P0` when the basket opens. Planned
additional buy thresholds are `P0 - 0.35*A*i` for `i = 1, 2, 3, 4`.
The common protective stop is `S = P0 - 2*A`, which is 0.6 ATR below the
last threshold. If `S <= 0`, reject the basket. A 1h trend reversal or a
15m candle with true range over 2 ATR blocks further adds, while the existing
stop and TP stay active. Never widen `S` after entry.

Use slot ceilings of `[3%, 4%, 5%, 7%, 11%]` of equity at basket open. Their
sum is exactly 30%. For candidate fill `i` (zero-based), use its last closed
signal candle and current executable quote. Recheck limits against the actual
fill as soon as the exchange confirms it:

```text
adverse_fraction = clamp((P0 - fill_price) / (P0 - S), 0, 1)
loss_factor      = 0.5 + 0.5 * adverse_fraction
candle_factor    = clamp(true_range / A, 0.4, 1.0)
desired_margin_i = equity_at_open * slot_ceiling_i
                   * loss_factor * candle_factor
```

Thus later slots can buy more as the basket moves farther into loss, while a
small signal candle requests less capital. The size is never increased merely
because volatility is extreme. A higher ATR widens the stop and ladder, so
the 1% loss budget below automatically reduces allowed quantity. Quantities
are rounded down to the exchange
step and checked against minimum order size. The actual order is the smallest
of its desired size, free account margin, the remaining 30% account-wide cap,
and the remaining basket loss budget below. If rounding or caps make the next
quantity smaller than the previous filled leg, do not add on that candle.
Place at most one add on each closed 15m candle, even if price crosses several
thresholds. Consume skipped thresholds in a gap; do not stack their orders on
later candles at the same price. Fill at the next executable price, never at
the crossed threshold. A gap to or through `S` triggers the stop, not an add.

## 4. Risk, stop and target

- Maximum planned net loss at `S`: 1% of current account equity per basket.
  Before the first order and each add, project the loss of all filled legs
  plus the proposed leg at `S`, including entry/exit fees, slippage and known
  funding. Shrink or reject the proposed leg if this exceeds the limit.
- Maximum total Adaptive DCA initial margin: 30% of the lower of equity at
  basket open and current equity. Check the whole account and free margin, not
  just the symbol. If a falling account equity makes an existing basket exceed
  the cap, cancel adds and close it as a separately reported safety exit.
  One-time leverage is 1x during initial Paper validation.
- After every fill, let `R` be the positive estimated net loss if all current
  quantity exits at `S`. Set one basket TP price where estimated net PnL is
  `+2*R`. Recompute from actual filled quantity and weighted entry; round the
  TP away from the average price. If a valid TP cannot be placed, reject the
  add and keep the existing protection.

  For a long basket with filled cost `C = sum(quantity * fill_price)`, total
  quantity `Q`, paid entry fees `F`, accrued funding `U`, estimated stop costs
  `K_stop`, estimated target slippage `K_tp`, and exit fee rate `f`:

  ```text
  R      = C - Q*S + F + U + K_stop
  TP_raw = (C + F + U + K_tp + 2*R) / (Q * (1 - f))
  ```

  `K_stop` includes stop exit fees and slippage. Recompute after a funding
  payment or fee change as well as after a fill. Reject a basket if `R <= 0`.
- Keep the stop exchange-protected throughout the basket. Update its quantity
  after each fill. As in the current DCA engine, a current market quote may
  trigger the stored basket TP and send a reduce-only market close; 15m bars
  still control entries and adds. Never leave excess closing quantity resting.
  A manual halt or exchange failure is reported as an exceptional exit and
  may realize a ratio different from 2:1.
- If a closed candle could have touched a newly placed add, TP and stop in
  unknown order, historical replay takes the conservative stop-first path.
  Report ambiguous candles separately. Planned 2:1 is not a guaranteed fill
  ratio after a gap, slippage or funding.

## 5. Static trace

With equity 1,000 USDT, `P0 = 100`, `A = 2`, 1x and a true-range factor of 1:

- Buy thresholds are 99.30, 98.60, 97.90 and 97.20; common stop is 96.00.
- Initial order requests `1,000 * 3% * 0.5 = 15` USDT of margin.
- First add at 99.30 has `adverse_fraction = 0.175`, so it requests
  `1,000 * 4% * 0.5875 = 23.50` USDT, subject to risk checks.
- If that add candle has true range only 0.5 ATR, its request becomes 11.75
  USDT; because that is smaller than the initial 15 USDT leg, no add occurs.
- Even if all five slot ceilings are used, their margin totals 300 USDT.
  Every proposed fill must also keep worst-case loss at the stop <= 10 USDT.
- If all five candles have factor 1 and fills occur exactly at the thresholds,
  requested margin is about 219.13 USDT. Gross risk at the stop is about 4.46
  USDT, weighted entry about 98.00, and the gross 2R target about 101.99.
  Costs raise that target further. Even after averaging down, a 2R basket may
  need price to recover above the initial 100 entry.
- TP is solved from the actual weighted basket and actual risk `R`; it is not
  a fixed percentage above the newest buy or a promise of 2:1 realized PnL.

## 6. Validation and implementation gate

Implement this as a separate Paper/backtest strategy, reusing the existing
Wilder ATR, candle, order rounding, basket accounting and cost models. Store
the frozen ladder, stop, all fills, ATR, signal candle range, size factors,
equity snapshot, risk check and TP revision in a recoverable basket record.
An engine restart must restore the same basket and never repeat an add.

Before Paper activation, backtest ETH on at least one year of available 15m
and 1h history. Compare against equal-sized five-leg DCA and the existing
fixed-multiplier DCA with the same 30% capital and 1% loss caps. Include fees,
slippage, funding, missed candles, gap-through-stop fills and exchange steps.
Report out-of-sample net PnL, drawdown, net expectancy, realized win/loss
ratio, stop frequency, maximum capital used and the number of rejected adds.
Do not infer a profitable strategy from a theoretical 2:1 target alone.
