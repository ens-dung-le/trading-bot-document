# Fibonacci Sequence Swing V2 — backtest contract

Goal: Test retest and confirmed breakout entries against Fibonacci sequence levels, with a configurable swing depth and a visual touch-count overlay.
Out of scope: Changing V1 results or rules, using level brightness as an entry filter, and paper/live trading.
Steps:
1. Register `fibonacci-sequence-swing-v2` separately from the V1 `fibonacci-sequence-swing` profile.
2. Add a `swingDepth` backtest input from 20 to 500 closed signal candles, default 150, and freeze it in each job's parameter snapshot.
3. Keep V1's 13/21 retest entry; add breakout entries after a completed candle closes through a sequence level in the current swing direction.
4. Display the active 0/21, 1/21, 2/21, 3/21, 5/21, 8/21, 13/21, and 21/21 levels in the Backtest Report Viewer, with brightness based on candle touches.
Verification: Static trace the Long and Short breakout tests, run TypeScript tests/builds for bot and viewer, and review the diff for V1 behavior changes.

## Causal signal and execution

- Swing anchors use the same rolling-high/rolling-low rules as V1, but over `swingDepth` closed candles. A data gap resets the swing; no level is available until a fresh depth has completed.
- The decision candle uses the **prior closed swing**. Long breakouts require an up swing, previous close at/below a level, and decision close above it. Shorts mirror this in a down swing. When one candle crosses several levels, use the furthest level crossed in the trend direction that still has a next level beyond the close. A 13/21 retest takes precedence if both patterns occur on one candle.
- Enter at the next executable candle open. Breakout TP is the next sequence level in the trade direction; reject an entry if that open has already reached/passed TP. Retest TP remains V1's 2R target. Stops remain a 0.5 ATR buffer beyond the signal candle, and V1 margin, cost, liquidation, account-risk, and time-exit rules remain in force.
- Brightness is visual context only: count closed candles in the latest depth whose high-low range intersects each displayed level; consecutive candles count separately. Selecting a trade displays the prior swing snapshot used by its signal, avoiding later levels being presented as if known then. No institutional participation is inferred from touch count.
- Viewer lines show the selected trade's snapshot or the latest loaded snapshot, not a historical sequence of every prior swing. Missing candle coverage can leave the overlay empty; it must not synthesize a level.
