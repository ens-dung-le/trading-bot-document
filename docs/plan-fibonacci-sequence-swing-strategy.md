# Fibonacci Sequence Swing strategy specification

Status: implemented for Backtest and its menu, 2026-10-05. Paper and live trading are disabled.

Source: [ChartPrime Pine indicator](https://www.tradingview.com/script/ZjjAH1Fu-Fibonacci-Sequence-Swing-ChartPrime/) and [ChartPrime walkthrough](https://www.youtube.com/watch?v=ZqrzwodB-Po). The video illustrates retests and breakouts but does not define mechanical entries, exits, or performance claims. The trading rules below are this strategy's testable choices.

Goal: Add one Fibonacci Sequence Swing retest strategy to the bot's existing backtest lifecycle.
Out of scope: Changing the ChartPrime indicator, adding breakout entries, or enabling live trading.
Steps:
1. Implement closed-candle swing snapshots and the long/short retest signal in one isolated strategy module.
2. Apply the specified stop, target, time exit, and margin-based sizing through the existing execution and risk layers.
3. Register the strategy for backtesting and verify causal replay against concrete bar sequences.
Verification: Static trace the examples below, review the diff for isolated strategy changes, and compare backtest fills with the stored pre-signal snapshots. A TradingView Pine compile is not applicable to the TypeScript implementation.

## Source calculation and causal state

- Use the source's `depth = 150` on fully closed candles. A new high has `high == highest(high, 150)`; a new low has `low == lowest(low, 150)`. Maintain the latest high, latest low, their bar times, and the last extreme's direction as in the source.
- Reject a bar that qualifies as both a new high and a new low. Require both anchors and a positive range before producing levels.
- At each candle close, store the direction, anchors, and levels known at that moment. Future changes to an active swing must not rewrite earlier snapshots or trade decisions. The current extreme is confirmed only as a closed-bar observation, not as a future-confirmed pivot.
- For an up swing, the entry level is `lowAnchor + 13 / 21 * (highAnchor - lowAnchor)`. For a down swing, it is `highAnchor - 13 / 21 * (highAnchor - lowAnchor)`. Both are approximately a 38.1% retracement from the latest directional extreme. The upper target boundary is the known high anchor for Long; the lower target boundary is the known low anchor for Short.
- Do not require a minimum retest count. The source's display count includes every candle whose high and low straddle a level, including consecutive candles, and is not a count of independent reactions.

## Entry

- Signal timeframes: `15m`, `30m`, `1h`, `4h`, and `1d`; evaluate each profile independently. No candle below `15m` is an entry input.
- Evaluate candle `t` against the level and direction stored after candle `t-1`. Long requires up direction, `close[t-1] > level`, `low[t] <= level`, and `close[t] > level`. Short requires down direction, `close[t-1] < level`, `high[t] >= level`, and `close[t] < level`.
- Reject a signal if candle `t` changes swing direction or qualifies as both a new high and new low. The signal expires at the next candle open; enter at that open only if the stop and target remain valid. There is no pending entry order to cancel.
- At most one open position per symbol for this strategy. Do not enter again while that position is open.

## Protective exit and target

- Compute `ATR(14)` from candles closed through signal candle `t`. Long stop: `low[t] - 0.5 * ATR(14)`; Short stop: `high[t] + 0.5 * ATR(14)`. Round prices to exchange tick size away from the entry. Reject a trade if next-open entry is already beyond its stop or the stop distance is nonpositive.
- Let `R = abs(entry - stop)`. Long target: `entry + 2R`; Short target: `entry - 2R`. Accept only if that target is strictly before the stored swing high for Long or swing low for Short. Otherwise skip the entry. The 2:1 ratio is gross price distance; report fees, funding, spread, slippage, and net reward/risk separately.
- SL and TP are active from entry. In OHLC replay, if both are touched within the same candle and their order cannot be established, count the stop first. Do not infer a favorable intrabar path.

## Time exit and US market calendar

- For each entry, record the first scheduled US regular equity-market close strictly after entry, using the `America/New_York` calendar with daylight saving, holidays, and early closes. A trade entered after that day's close uses the next scheduled close.
- Time exit is eligible only after both (a) eight complete signal-timeframe candles since entry and (b) that recorded US market close. Close at the next executable open after the first completed candle satisfying both conditions. SL/TP can close the position sooner.
- Treat crypto as a continuously traded market. The US market close is a time anchor, not a crypto exchange shutdown. Include applicable funding for positions held across settlement times.

## Margin and risk

- Requested opening margin is **38 USDT**, not 38 USDT notional. Read leverage from the backtest run configuration and store it in each run snapshot; requested notional before contract rounding is `38 * leverage` USDT. Do not multiply realized PnL by leverage again after sizing.
- The seeded Backtest menu defaults to `5x` leverage so 38 USDT margin can meet BTC's current one-contract minimum; the operator can select a different allowed leverage for an individual run. Exchange minimums and the account risk cap can still skip a setup.
- Cap expected stop loss, including estimated round-trip fees and slippage, at 0.5% of account equity. Reduce margin below 38 USDT if necessary; skip if the resulting order is below the exchange minimum or liquidation can precede the protective stop. Never silently increase margin above 38 USDT.
- Apply the existing symbol contract multiplier, quantity step, minimum quantity, leverage limit, and margin checks. The same settings and costs must be used for every compared backtest profile.

## Static traces

- Up swing low `100`, high `121`: stored Long level is `113`. If the previous close is `115` and the next candle has low `112` and close `114`, it signals Long for the following open. A later high cannot retroactively move this signal's level.
- Down swing high `121`, low `100`: stored Short level is `108`. If the previous close is `106` and the next candle has high `109` and close `107`, it signals Short for the following open.
- On `15m`, an entry at 10:00 New York reaches eight completed candles around 12:00; if SL/TP remain untouched, the time exit waits until the scheduled US close. An entry after the US close waits for the next scheduled close as well as eight completed candles.
