# Scalping Lower-Wick Reclaim V1 / V2 / V3 / V4 — specification and paper-test gate

## V4 — reversed V2 entries (2026-10-07)

`scalping-lower-wick-reclaim-v4` (configuration version 4) uses exactly V2's completed lower-wick sweep/reclaim signal and next-open validity/sizing checks, but enters **short** where V2 enters long. No upper-wick or new bearish signal is introduced. It retains 10x isolated leverage, maximum $50 initial margin and the 60-minute deadline. TP is 5% gross margin return (price −0.5%); SL is 2% gross margin loss (price +0.2%), before costs. Both prices are rounded down to the price tick from the actual entry fill. At entry 3000 and tick 0.01: TP 2985, SL 3006. SL wins if a candle touches both; gap and liquidation rules remain unchanged. Deferred short entries also reject an open at/below the reclaimed low, matching V2. Persisted order side controls restored exit direction. Migration `0083` exposes V4 in Backtest with paper availability disabled; account 1 continues ETH V1 without changing balance/history.

> Paper policy: account 1 continues ETH V1. V2/V3 are separate Backtest menu entries and remain inactive in paper. Version split requested on 2026-10-07; migrations `0081`/`0082` seed V2/V3 and persist separate SL rates and the entry reason.
> Local deployment verified on 2026-10-07: bot and backtest worker restarted, three menu choices and exit profiles checked in the running image, ETH V1 running at 10x with $5,000 starting balance. Build passed; 1,703 tests passed and 17 were skipped. No Scalping position/order was open during this deployment.
> Research hypothesis, not a claim of positive expectancy. Paper/backtest only; live orders are out of scope.

## Agreed contract

- Strategy IDs: `scalping-lower-wick-reclaim-v1`, `scalping-lower-wick-reclaim-v2`, and `scalping-lower-wick-reclaim-v3`; configuration versions 1, 2, and 3 respectively. Candidate symbols: BTC, ETH, and SOL perpetual USDT contracts already supported by the bot.
- Long/buy only. One open position and one pending entry per symbol; no short, DCA, scale-in, or immediate re-entry on the same signal.
- Closed `1m` candles form the signal. A qualifying signal enters no earlier than the next `1m` open. Never buy at the historical wick low or use an unfinished candle.
- Paper-test leverage: **10x isolated**. This is an explicit test setting, not an instruction to raise the global leverage cap for other strategies.
- All percentages below are gross return/loss on initial margin at 10x, before fees, funding, spread, and slippage. Targets use actual average entry fill. V3 inherits V2's 2% SL; only TP changes.

| Version | TP on margin | Price increase for TP | SL on margin | Price decrease for SL | Time exit |
| --- | ---: | ---: | ---: | ---: | --- |
| V1 | +2% | +0.2% | None | None | 60 minutes |
| V2 | +2% | +0.2% | −2% | −0.2% | 60 minutes |
| V3 | +5% | +0.5% | −2% | −0.2% | 60 minutes |

- V1 preserves the original TP and has no price SL; if TP has not filled, close at the one-hour deadline even at a loss. V2/V3 close on SL, TP, or the same deadline. Liquidation and account-level circuit breakers remain active for every version.

## Shared entry hypothesis

All comparisons use completed, timestamp-contiguous `1m` candles and exchange-normalized prices. These numeric thresholds are initial research assumptions, not optimized or externally validated settings.

1. **High volatility:** `ATR(14) / close` on the signal candle is strictly above the 75th percentile of valid ATR% observations in the preceding 1,440 minutes. Exclude the signal candle from that reference distribution. Require at least 95% valid reference observations; never interpolate missing candles into an ATR path.
2. **Downside sweep and reclaim:** `low < min(low of previous 20 closed candles)` and `close > that previous-20-candle low`. Those 20 candles must be contiguous. This distinguishes rejection of a recent low from an arbitrary long lower wick.
3. **Lower-wick shape:** `lowerWick = min(open, close) - low`, `body = abs(close - open)`, and `range = high - low`. Require positive range, `lowerWick >= 2 × max(body, priceTickSize)`, `lowerWick / range >= 0.50`, and `(close - low) / range >= 0.75`.
4. **Entry validity:** at the next `1m` open, the observed price must still be above the swept prior low. Reject an entry when the price/quote is stale, the candle is missing, quantity is below exchange minimum, margin/exposure checks fail, or the configured paper spread/slippage gate fails. Record the precise rejection reason; do not fill at the signal close or wick low.

The first implementation should use the existing ATR utility, candle-contiguity checks, symbol specification, risk/cost model, and strategy registry. Do not create a second generic wick indicator just for this strategy.

## Position size and exits

- Paper starting balance: **$5,000**; this is the selected account's existing balance setting, not a request to reset its history. Initial margin budget: **$50 per entry**, hence a hard $500 notional ceiling at 10x. Plan with a 0.5% next-open/spread buffer, then normalize contract quantity **down** to the exchange step; the paper executor checks the actual fill against the $50 ceiling and rejects a gap beyond it. Reject if the normalized quantity is below the minimum; never round up merely to force a trade. The existing account-wide exposure and circuit-breaker limits remain in force.
- V1: arm only `target = roundPriceUp(actualAverageEntryPrice × 1.002, priceTickSize)`. V2: arm `stop = roundPriceUp(actualAverageEntryPrice × 0.998, priceTickSize)` and the same TP. V3: use the same V2 stop and `target = roundPriceUp(actualAverageEntryPrice × 1.005, priceTickSize)`. All orders are reduce-only for the filled quantity. If a required order cannot be armed, close the position immediately and log the failure. An SL or TP fill cancels its sibling. If both levels are touched in the same 1-minute candle and their order is unknown, use SL first. Liquidation remains the repository's first-priority conservative outcome.
- The 60-minute deadline is `firstEntryFilledAt + 3,600,000 ms`, independent of process restarts and candle count. If SL or TP filled before the deadline, cancel the time exit. If the deadline occurs first, cancel both protective orders and submit one idempotent reduce-only market close. A delayed/missing price must not mark a fictitious fill: block new entries, retry the close, and report the overdue position.
- Paper event timestamps and the exchange's actual fill price, not the strategy's reference close, determine elapsed time and realized P&L. Time exit may lose money. No rule waits for recovery or extends the hour because the position is negative.
- If a 1-minute replay candle could contain both the TP touch and 60-minute deadline but their intrabar order is unknowable, choose the conservative outcome (time exit at the first executable price at/after the deadline) and flag the ambiguity. Prefer timestamped fills/quotes when available. Simulate liquidation before assuming a later TP or time exit.

## Cost and risk reality check

The repository's default KuCoin futures paper assumptions are taker fee 0.06% per side, maker fee 0.02% per side, plus spread and slippage. Its resting TP is modeled as a stop-triggered market exit, so the relevant fees are taker on both sides. A 0.2% price target leaves 0.08% of notional after those two fees **before** spread/slippage/funding. Report gross price move, gross margin return, all costs, and net margin return separately. Do not call a TP hit a profitable trade unless net P&L is positive.

V1's one-hour limit caps holding time, not loss; it can lose much more than 2% margin before the deadline. V2/V3's 2% margin SL is a trigger threshold, not a guaranteed net loss cap. Costs and a gap through the stop can produce a larger loss. At $50 initial margin, planned gross TP is approximately $1 for V1/V2 or $2.50 for V3; planned gross SL loss is about $1 for V2/V3. Paper acceptance requires a review of liquidation, maximum adverse excursion, exit losses, drawdown, and exposure. Existing circuit breakers remain active.

## Implementation and activation gate

All three strategies are registered for `1m` backtests and share one entry implementation. Paper queues the next-open entry, enforces the $50 actual-fill margin ceiling, and uses the version's exit profile. TP and SL rates and the original entry reason are persisted on pending orders; null SL rate means V1 has no price stop. This prevents V3's 5% TP from silently widening its SL to 5% after restart. Protective orders and the original fill timestamp survive restart. The 10x leverage allowance applies only to these paper profiles; the no-SL allowance additionally requires V1. Startup reconciliation permits an open paper V1 without SL only when its Scalping identity and resting TP are present. V2/V3 and ordinary strategies still require a stop. Live execution remains blocked.

Historical local 2026-10-03–2026-10-07 backtests **before SL was added**, each starting at $5,000 with 10x and the configured KuCoin cost assumptions, gave (these are not results for the new SL variant):

| Symbol | Trades | Net P&L | Run |
| --- | ---: | ---: | ---: |
| BTC | 0 | $0.00 | 386 |
| ETH | 3 TP | +$1.02 | 385 |
| SOL | 3 (2 wins) | −$2.50 | 387 |

All three failed the repository's generic acceptance gate; its 600-trade minimum and 2–5x leverage range are not tailored to this experimental 10x paper profile. The tiny sample and negative SOL result remain substantive warnings. ETH's three trades each used about $48.55–$48.87 initial margin. No backtest liquidation occurred in this short window, which does not establish liquidation safety.

ETH-only exploratory paper testing remains on V1 for account 1: $5,000 starting balance, 10x isolated, $50 maximum margin per entry. Its account-specific `paperTradingEnabled` flag stays true. V2/V3 are seeded with paper availability false and appear in Backtest; they do not start automatically. No balance or trading history is reset. An in-position process-restart check and per-filter daily diagnostics remain outstanding before claiming full paper acceptance. Synthetic replay and persistence tests cover all three exit profiles.

## Acceptance cases and required report

- A sweep that closes back above the old low, passes the wick/ATR rules, and has a valid next open creates exactly one long. A long wick without reclaim, a wick on an unfinished candle, a gap in the latest 20 minutes, stale quote, or failed 24-hour coverage creates none, each with its own reason.
- At an actual fill of 100.00 and 10x: V1 has TP 100.20 and no SL; V2 has TP 100.20 and SL 99.80; V3 has TP 100.50 and SL 99.80. A V2/V3 candle touching both uses SL. A later candle opening below SL fills at the worse open plus costs. On identical entry candles, V3 must not take V2's smaller TP, and V1 must not inherit either version's SL.
- A position filled at 12:03:20 UTC and still open at 13:03:20 UTC is submitted for market close, even if it is losing. A restart at 12:40 does not restart the timer. TP before the deadline prevents a second close; a simultaneous/ambiguous case uses the conservative replay rule.
- Reconcile paper and backtest on the same candles and cost assumptions. Report, by symbol and day: evaluated candles, each entry-filter pass/rejection count, orders/fills, SL versus TP versus time-exit versus liquidation counts, gross/net P&L, fees, funding, spread/slippage assumption, maximum adverse excursion, and any missing-data/late-close incidents. Include open positions with unrealized P&L; do not count them as realized wins.
- Before expanding beyond the operator-approved ETH exploratory paper run, verify the SL/TP bracket end-to-end, confirm an in-position restart preserves both orders and the deadline, and review net P&L and drawdown. A startup with no open position is not an in-position restart test.

## References

- [CMC Markets: hammer candle, context, confirmation, and failure risk](https://www.cmcmarkets.com/en-gb/technical-analysis/hammer-candlestick-patterns). The source explains why a lower wick is only a possible rejection signal, not proof of a reversal.
- [Fidelity: Average True Range](https://www.fidelity.com/learning-center/trading-investing/technical-analysis/technical-indicator-guide/atr). ATR measures volatility; the percentile gate above is this strategy's proposed rule, not Fidelity's endorsement.
- [CME Group: the 2% rule](https://www.cmegroup.com/education/courses/trade-and-risk-management/the-2-percent-rule). The exact 1% paper-margin sizing above is a conservative experiment choice, not a profitability guarantee or a CME-prescribed figure.
