# London Open Liquidity and Macro Strategy V2

> **Status: implemented for research/backtest and exploratory paper testing;
> not approved for live trading.**
>
> V2 is a separate research contract. It must not overwrite the V1 strategy,
> its stored parameters, or its historical reports. Recommendations from the
> supplied memoranda and LB review are treated as hypotheses to test, not as
> validated performance claims.

## 1. Goal

V2 must answer two questions with auditable evidence:

1. Can a lower-timeframe London liquidity sweep model produce materially more
   qualified setups than V1 without losing positive expectancy after costs?
2. For every London session, can the system explain exactly why a trade was or
   was not triggered?

The strategy must not increase frequency by silently removing risk controls or
by tuning against the out-of-sample period.

## 2. Scope

V2 includes:

- BTC/USDT, ETH/USDT and SOL/USDT perpetual markets;
- a `5m` signal model;
- causal `1m` replay for entry, stop and target ordering;
- a London-open-gated observation window;
- ATR-normalized sweep detection;
- mandatory sweep and reclaim structure;
- weighted setup scoring instead of a hard displacement-only gate;
- a high-impact news blackout comparison;
- Asian-range structural targets;
- one immutable session decision per asset and London date;
- a daily human-readable report, including no-trade sessions; and
- a session-level backtest ledger and funnel summary.

V2 does not include:

- RSI, MACD, moving-average crossover or Bollinger entry filters;
- directional CPI surprise trading without actual, consensus and first-release
  data;
- order-book absorption claims reconstructed from OHLCV candles;
- automatic live deployment after a profitable backtest;
- increasing leverage or order size to improve headline profit; or
- changing or deleting V1 results.

## 3. Strategy identity

| Item | V2 contract |
|---|---|
| Research name | `London Open Liquidity and Macro Strategy V2` |
| Strategy identifier | `london-open-liquidity-macro-v2` |
| Version | `2` |
| Target assets | BTC, ETH and SOL perpetuals |
| Signal timeframe | `5m` |
| Execution-detail timeframe | `1m` |
| Regime timeframes | closed `1h` and `4h` |
| Maximum entries | one per asset per London session |
| Research position sizing | fixed `$1,000` notional |
| Eligible trading days | Monday through Friday |

Fixed research sizing keeps V1 and V2 comparable. Risk-based position sizing
is a later deployment decision and must not be used to manufacture backtest
profit.

## 4. Required data

No V2 performance claim is valid until all three assets have:

- at least three complete years of `1m`, `5m`, `1h` and `4h` candles;
- no unexplained gaps inside an evaluated session;
- exchange volume on every `5m` signal candle;
- symbol-specific tick size, quantity step and contract multiplier;
- recorded funding history for every holding interval; and
- a versioned high-impact economic calendar for the same period.

The `1m` series is execution detail. It must not create a signal that the `5m`
model could not know at the `5m` decision close.

## 5. Session models

### 5.1 V2 primary model: London-open-gated UTC window

The primary LB-derived model is:

```text
Asian reference range: [00:00 UTC, 06:00 UTC)
Observation window:    [08:00 Europe/London, 10:00 UTC)
Position close cutoff: 12:00 UTC
```

The London open is `07:00 UTC` during BST and `08:00 UTC` during GMT. The
Asian range still freezes at `06:00 UTC`, but pre-open sweeps are not V2
candidates and cannot produce an entry. This removes V1's separate pre-open
invalidation interval without allowing pre-open trades. A `5m` signal candle
must open no earlier than the London open; the market order fills on the next
`1m` candle after that signal closes.

This changes the meaning of existing `v2_wide_utc` runs that allowed entries
from `06:00 UTC`. Regenerate backtests before comparing trade counts or P&L;
older stored results are not directly comparable to the London-open-gated model.

### 5.2 Required sensitivity comparisons

The implementation must support named research runs for these definitions:

| Model | Asian range | Observation window | Purpose |
|---|---|---|---|
| `v2_wide_utc` | `00:00-06:00 UTC` | `08:00 Europe/London-10:00 UTC` | primary LB hypothesis, gated at London open |
| `v2_memo_utc` | `00:00-07:00 UTC` | later of `07:00 UTC` and `08:00 Europe/London`, until `09:00 UTC` | supplied memorandum contract, gated at London open |
| `v1_local_open` | `00:00-06:00 UTC` | first two hours after `08:00 Europe/London` | direct V1 comparator |

Only one session model may be active in a run. Results must be reported under
the full model name. A better in-sample result must not silently replace the
primary model.

### 5.3 Weekend reporting

Saturday and Sunday do not generate entries, but the daily report must still
record them as `not_eligible_weekend`. This lets a seven-day report schedule
account for every calendar date without presenting weekends as missing data.

## 6. Causal event definitions

### 6.1 Asian range

The Asian High and Asian Low use only completed `5m` candles whose open
timestamps belong to the configured half-open reference interval.

The range is frozen at the end of that interval. Later candles cannot rewrite
it.

### 6.2 ATR

Use Wilder `ATR(14)` on closed `5m` candles.

- Sweep thresholds use the ATR from the candle immediately before the sweep.
- Score components use values available at the candidate decision close.
- Stop buffers use the latest ATR available at entry decision time.
- Missing ATR produces `atr_not_ready`; it must never be replaced with zero.

### 6.3 Sweep

```text
High sweep: high >= AsianHigh + 0.05 * priorATR
Low sweep:  low  <= AsianLow  - 0.05 * priorATR
```

Sweep depth in ATR is recorded even when the session later fails.

### 6.4 Reclaim

```text
High reclaim: close < AsianHigh after a High sweep
Low reclaim:  close > AsianLow  after a Low sweep
```

A `5m` candle may perform both the sweep and reclaim because its completed OHLC
proves that price traded outside and closed back inside the range.

### 6.5 Opposite-side sweep

If both Asian boundaries are swept before a qualified entry, the reversal
candidate is invalidated as `both_sides_swept`. The report must retain the first
sweep side and both event timestamps.

## 7. V2 entry model

### 7.1 Mandatory core gate

A candidate cannot be scored until all of these are true:

1. the day is eligible;
2. all required session candles are complete;
3. one Asian boundary is swept by the ATR buffer;
4. price closes back inside the range; and
5. the opposite boundary has not been swept before the decision.

Sweep and reclaim remain structural requirements. They are not converted into
optional indicators.

### 7.2 Weighted quality score

V2 replaces V1's hard displacement gate with a score from `0` to `100`.

#### Rejection quality: 0 to 35 points

| Condition | Points |
|---|---:|
| Close barely returns inside the boundary, less than `0.05 ATR` | 15 |
| Close returns at least `0.05 ATR` inside the boundary | 25 |
| Close returns at least `0.15 ATR` inside and rejection wick is at least half the candle range | 35 |

Only the highest matching row applies.

#### Displacement quality: 0 to 30 points

| Condition | Points |
|---|---:|
| No directional displacement | 0 |
| Body at least `0.5 ATR` and directional close | 10 |
| Body at least `0.75 ATR`, close in outer 30%, breaks prior candle extreme | 20 |
| Body at least `1.0 ATR`, close in outer 20%, breaks prior candle extreme | 30 |

Displacement may occur on the reclaim candle or on a later completed `5m`
candle. The `1m` replay must prove event ordering before a same-candle entry is
accepted.

#### Relative volume: 0 to 20 points

Compare candidate `5m` volume with the prior completed 20-candle volume SMA:

| Volume ratio | Points |
|---|---:|
| Less than `1.0` | 0 |
| `1.0` to less than `1.25` | 5 |
| `1.25` to less than `1.5` | 10 |
| At least `1.5` | 20 |

Volume is a quality input, not a claim that OHLCV proves order-book absorption.

#### Regime context: 0 to 15 points

| Latest causal `1h/4h` context | Points |
|---|---:|
| Direction conflicts with the reversal and confidence is at least `0.50` | 0 |
| Unknown, low confidence, or mixed direction | 5 |
| Sideways regime | 10 |
| Direction aligns with the reversal and confidence is at least `0.50` | 15 |

Regime never blocks a setup by itself. Every bucket must remain visible in the
report so that the score can be audited.

### 7.3 Entry threshold

The initial V2 research threshold is:

```text
minimumEntryScore = 60
```

The threshold is a predeclared starting hypothesis. Sensitivity runs may test
`50`, `60` and `70`, but they must share the same data, costs and exits. The
selected threshold must be frozen before the out-of-sample run.

### 7.4 Entry timing

When the score reaches the threshold:

1. record the completed `5m` decision candle;
2. use `1m` candles only to confirm intrabar order and executable prices;
3. enter at the first executable price after the decision became known; and
4. reject the entry if required `1m` coverage is missing.

No historical candle may enter at a price that occurred before the confirming
information was available.

## 8. Macro risk shield

V2 macro handling is a calendar risk shield, not a directional macro model.

A candidate that otherwise reaches the score threshold is rejected when its
entry timestamp is from 15 minutes before through 15 minutes after a stored
high-impact event in the configured UK, EU or US event set.

The report must record:

- event identifier and name;
- affected country or region;
- scheduled release timestamp;
- stored impact level;
- blackout start and end; and
- whether the candidate would have entered with the shield disabled.

If there is no qualified setup, a nearby event is recorded as context but must
not become the primary no-trade reason. For example, a session with no sweep is
`no_sweep`, not `macro_blackout`.

Actual-versus-consensus directional macro scoring remains out of scope until
the system stores immutable first-release values and ingestion timestamps.

## 9. Risk and exit contract

### 9.1 Stop loss

For a Long, place the initial stop below the lowest price of the sweep sequence
by `0.1 ATR`. For a Short, place it above the highest price by `0.1 ATR`.

Reject the setup as `risk_rejected` if normalization puts the stop on the wrong
side of the entry or produces a non-positive risk distance.

### 9.2 Structural targets

V2 uses the Asian range rather than a fixed full-position `2R` target:

- TP1 is the Asian range midpoint;
- TP1 closes 50% of the initial quantity;
- after TP1, the stop for the remaining quantity moves to entry plus the
  configured cost buffer;
- TP2 is the opposite Asian boundary; and
- any remaining quantity closes at `12:00 UTC`.

If the entry is already beyond the midpoint in the trade direction, TP1 is not
valid. The report must mark `midpoint_already_passed`, and the run must use the
predeclared fallback target of `1R` for TP1 rather than invent a target after
the result is known.

### 9.3 Intrabar precedence

Use complete `1m` candles to determine stop, TP1 and TP2 order. If stop and a
target are touched within the same `1m` candle and no finer data exists, choose
the stop-first outcome.

### 9.4 Costs

Every report snapshots:

- taker and maker fee rates;
- half spread;
- slippage in ticks;
- funding payments;
- leverage;
- contract multiplier; and
- quantity and price rounding.

The reference cost run must use the shared exchange cost model. A separate
stress run doubles spread and slippage without changing strategy parameters.

## 10. Session state machine

Each asset and London date owns one state machine:

```text
scheduled
  -> not_eligible
  -> data_ready
  -> waiting_for_sweep
  -> low_swept  -> low_reclaimed  -> scored -> entry_qualified
  -> high_swept -> high_reclaimed -> scored -> entry_qualified
  -> blocked
  -> expired
  -> trade_entered
  -> trade_closed
```

State transitions are append-only audit events. A later candle may advance the
state but may not erase the earlier sweep, reclaim, score or rejection reason.

## 11. Final decision reasons

Every asset and calendar date must end with exactly one primary decision reason:

| Reason | Meaning |
|---|---|
| `not_eligible_weekend` | Saturday or Sunday |
| `data_gap` | required `1m`, `5m`, `1h` or `4h` coverage is incomplete |
| `atr_not_ready` | required causal ATR is unavailable |
| `no_sweep` | neither Asian boundary was swept during the observation window |
| `both_sides_swept` | both boundaries were swept before entry qualification |
| `swept_not_reclaimed` | a boundary was swept but never closed back inside |
| `score_below_threshold` | reclaim occurred but final score was below the entry threshold |
| `macro_blackout` | score qualified, but a high-impact event blocked entry |
| `risk_rejected` | stop or sizing rules made the trade invalid |
| `execution_data_missing` | signal qualified, but causal `1m` execution replay was unavailable |
| `entry_not_filled` | a valid order did not receive a causal fill |
| `trade_entered` | an entry was filled |

The primary reason is the furthest terminal stage reached. Secondary flags may
record additional context, but they must not replace the primary reason.

## 12. Persistent session ledger

The database, not an exported file, is the source of truth. One immutable
session record per asset and date must include at least:

### Identity and configuration

- strategy identifier and version;
- run or live-session identifier;
- London date and UTC timestamps;
- asset and exchange symbol;
- session model name;
- parameter snapshot and source-code revision; and
- data-source and calendar versions.

### Range and market context

- Asian range start and end;
- Asian High, Asian Low and midpoint;
- ATR at range close and at decision;
- latest causal `1h/4h` regime, confidence and source timestamps;
- nearby high-impact events; and
- data-gap details, if any.

### Funnel events

- first sweep side, timestamp, price and ATR depth;
- reclaim timestamp, price and distance inside the range;
- opposite sweep timestamp, if present;
- displacement measurements;
- candidate volume and volume-SMA ratio;
- each score component and total score;
- score threshold;
- macro decision;
- risk decision;
- execution decision; and
- primary final decision reason.

### Trade outcome

When a trade is entered, also store:

- direction, entry timestamp and price;
- initial quantity and notional;
- stop, TP1, TP2 and close cutoff;
- partial-fill and partial-exit quantities;
- breakeven activation;
- exit timestamps, prices and reasons;
- gross P&L, fees, slippage, funding and net P&L;
- MAE, MFE and holding time; and
- realized result in USD, basis points and initial `R`.

## 13. Daily report for LB

### Paper menu control

Selecting V2 from the Telegram Paper trade menu activates the dedicated V2
paper worker for BTC, ETH and SOL. The main paper engine does not place V2
orders. The selected symbol in the menu does not narrow the worker to one asset.
Pause blocks new V2 entries; an already-open simulated position continues to
its exit. Switching to another strategy or using the kill switch asks the
worker to close any open V2 paper position at the next available public quote.
Activation after a signal does not replay that signal into a late entry, so
select V2 before the London open for a full-session paper test.

Paper entry prices use the public market quote plus modeled execution costs;
no exchange order is sent. The worker polls exchange data rather than receiving
guaranteed tick-by-tick fills. Missing candles, API timeouts and unverified
macro-calendar coverage are disclosed in the daily report and limit what the
paper result can establish.

### 13.1 Delivery contract

After the configured position close cutoff has passed and every entered trade
has reached a terminal state, the bot generates and sends one report for the
London date:

```text
london-open-v2-daily-YYYY-MM-DD.txt
```

The report contains one section each for BTC, ETH and SOL. It is generated from
stored session-ledger rows at send time; the rendered file is not a second
source of truth.

If sending fails, trading and stored evaluation results are unaffected. A retry
must rebuild the same report from the same immutable rows.

### 13.2 Required report header

```text
LONDON OPEN LIQUIDITY AND MACRO V2 - DAILY DECISION REPORT
London date:
Session model:
Strategy version:
Source revision:
Generated at:
Assets evaluated:
Trades entered:
No-trade sessions:
```

### 13.3 Required per-asset section

Each asset section must state, in this order:

1. final decision: `TRADE` or `NO TRADE`;
2. primary reason code and plain-language explanation;
3. Asian High, Asian Low, midpoint and range width;
4. sweep result and timestamp;
5. reclaim result and timestamp;
6. score breakdown and threshold;
7. macro-event decision;
8. risk and execution decision; and
9. trade result, when applicable.

Example no-trade summary:

```text
ETH/USDT: NO TRADE
Reason: score_below_threshold
Asian range: 2,340.20 - 2,381.70
Sweep: Asian Low at 08:35 UTC, depth 0.08 ATR
Reclaim: confirmed at 08:40 UTC
Score: 50 / 60
  Rejection: 25 / 35
  Displacement: 10 / 30
  Volume: 10 / 20
  Regime: 5 / 15
Macro shield: clear
Explanation: price reclaimed the range, but the combined setup quality did not
reach the predeclared entry threshold.
```

Example trade summary:

```text
BTC/USDT: TRADE LONG
Reason: trade_entered
Sweep: Asian Low at 08:10 UTC, depth 0.17 ATR
Reclaim: confirmed at 08:15 UTC
Score: 80 / 60
Macro shield: clear
Entry: 08:20 UTC at 63,120.0
Stop: 62,850.0
TP1: Asian midpoint, 50% closed
TP2: Asian High
Final exit: TP2
Net result: +8.40 USDT, +0.84R after costs
```

### 13.4 Daily funnel footer

The report ends with a three-asset reconciliation:

```text
Assets scheduled: 3
Data complete: 3
Sweeps: 2
Reclaims: 2
Score qualified: 1
Macro blocked: 0
Risk/execution rejected: 0
Trades entered: 1
```

Every count must be derivable from the asset sections.

### 13.5 Immediate V2 paper trade messages

The dedicated V2 paper worker sends a Telegram message to its account's chat
after each newly persisted trade transition for BTC, ETH or SOL. This applies
only to V2 paper trades, not V1, generic paper strategies, backtests or live
orders. Daily decision reports remain separate and are still sent on no-trade
days; an immediate message never replaces or changes the session ledger.

Required messages, in transition order:

1. **Entry:** London date, symbol, LONG/SHORT (paper buy/sell), UTC fill time,
   simulated entry price, quantity/notional, stop, TP1 and TP2. P&L is not
   realized yet and must not be shown as a gain or loss.
2. **TP1 partial exit:** UTC fill time, simulated exit price, quantity closed,
   quantity remaining and realized P&L for the closed portion after allocated
   entry/exit fees and funding. State that the remainder is still open and show
   its current stop. TP1 is emitted once, including when TP1 and TP2 are
   reached on the same confirmed candle.
3. **Final exit:** UTC fill time, simulated exit price, remaining quantity
   closed, reason (`take_profit`, `stop_loss`, `time_exit`, `operator_halt` or
   `strategy_switch`) and total trade net P&L in USDT after fees and funding.
   If TP1 was reached first, the total includes both exits. A stop after TP1
   may still leave a positive total; the message must report the calculated
   number rather than label every stop a loss.

If a stop and TP1 are both touched within one `1m` candle, the existing
conservative stop-first rule applies and no TP1 message is sent.

Prices and P&L must use the same modeled fills and costs as the stored V2 paper
trade, not a fresh quote at send time. Label every message **PAPER / SIMULATED**:
no exchange order was sent. Entry is based on a public quote after the closed
`5m` signal; TP/SL and other candle-driven exits are confirmed from `1m`
candles. The worker polls and may be delayed by data sync or Telegram, so
"immediate" means after the transition is confirmed and stored, not a
guaranteed tick-time alert.

For the TP1 message, allocate the entry fee and funding accrued by TP1 in
proportion to the quantity closed; subtract that portion's exit fee. The final
message takes total net P&L directly from the stored trade outcome, not from
the sum of rounded message amounts.

Each notification has a durable identity scoped to account, symbol, London
date and transition (`entry`, `tp1`, `final_exit`). Save the notification work
with the session transition, send only after commit, and retry an unsent item
after a transient Telegram failure or worker restart. Repeated polls must not
create repeated notifications. Telegram may accept a message just before the
worker crashes without recording its receipt, so delivery is at-least-once,
not a promise of exactly-once. Notification failure must not affect trade
state or suppress the daily report. Do not backfill old transitions when this
feature is enabled; an already-open trade receives only future exits.

Acceptance checks cover entry, TP1, TP2, stop, time/operator exits, TP1 and
TP2 on one `1m` candle, restart/retry deduplication, and reconciliation of the
final message's net P&L with the session ledger.

## 14. Backtest report artifacts

A backtest must not create one physical file per historical day. It stores one
session-ledger row per asset/day and generates these downloadable documents:

| File | Purpose |
|---|---|
| `backtest-result.txt` | performance, costs, acceptance gates and model identity |
| `london-open-funnel-summary.txt` | counts and conversion rates at every funnel stage |
| `london-open-session-ledger.csv` | one row per asset and calendar date, including no-trade reasons |
| `trades.csv` | entered trades and realized outcomes |
| `parameters.json` | immutable V2 parameter snapshot |

The funnel summary must show both counts and percentages:

```text
eligible sessions
-> complete data
-> any sweep
-> reclaim
-> score qualified
-> macro clear
-> risk accepted
-> entry filled
```

It must also show the terminal reason distribution by asset, month, direction
and selected market regime.

## 15. Frequency and performance reporting

Report these metrics separately for BTC, ETH, SOL and the combined portfolio:

- eligible sessions;
- sweep rate;
- reclaim rate after sweep;
- score-pass rate after reclaim;
- trade count per month and year;
- unique portfolio signal days;
- simultaneous multi-asset signal days;
- win rate and payoff ratio;
- expectancy in USD, basis points and `R`;
- profit factor;
- net profit after all costs;
- friction as a share of gross profit;
- maximum drawdown;
- TP1, TP2, stop and time-exit rates;
- Long versus Short performance;
- performance by session model, score band and regime; and
- in-sample versus out-of-sample degradation.

Frequency is diagnostic, not sufficient evidence of edge. A model must not pass
because it meets a trade-count target while losing money after costs.

## 16. Research sequence

Run the experiments in this order over identical data and costs:

1. V1 replay on the expanded data set.
2. V2 core gate with score recorded but no score threshold.
3. V2 threshold `50`, `60` and `70` sensitivity on in-sample data.
4. V2 with and without the macro blackout.
5. V2 wide, memorandum and V1 session-model comparison.
6. V2 structural exits versus the V1 fixed `2R` exit.
7. Cost-stress replay.
8. Freeze one candidate and run the untouched chronological out-of-sample set.

Each child model must report the exact sessions and trades added or removed
relative to its immediate parent.

## 17. Validation split

Use chronological data only:

- first 60%: parameter research;
- next 20%: validation and candidate selection;
- final 20%: untouched out-of-sample evaluation.

The final 20% may be run once for a frozen parameter set. If parameters change
after seeing that result, the period is no longer out-of-sample and a new later
period is required.

## 18. Promotion criteria

V2 remains research-only unless all of these pass:

1. every scheduled asset/day has exactly one terminal reason;
2. funnel totals reconcile with no unexplained missing sessions;
3. at least three years of complete required data are evaluated;
4. the combined portfolio has at least 100 completed out-of-sample trades;
5. out-of-sample expectancy after costs is greater than zero;
6. out-of-sample portfolio profit factor is at least `1.5`;
7. no enabled asset has profit factor below `1.2`;
8. out-of-sample net expectancy degrades by no more than 20% from validation;
9. maximum drawdown is no worse than 12%;
10. friction consumes less than 40% of gross profit;
11. no single trade contributes more than 20% of total net profit;
12. no single month or regime contributes more than 50% of total net profit;
13. the cost-stress run remains net profitable; and
14. paper execution reproduces the same session decisions and reason codes.

The LB frequency expectation of approximately one setup per asset per week is
a research target, not a pass condition. The report must disclose the measured
frequency even when it is materially lower or higher.

## 19. Static decision traces

### 19.1 Qualified Long

1. Monday data is complete and the Asian range freezes at `06:00 UTC`.
2. After `08:00 Europe/London`, price sweeps below the Asian Low by `0.12 ATR` and closes inside.
3. Rejection scores 25, displacement 20, volume 10 and regime 5.
4. Total score is `60`, meeting the threshold.
5. No high-impact event belongs to the blackout interval.
6. `1m` replay confirms the entry occurs after the reclaim information exists.
7. Final reason is `trade_entered`.

### 19.2 No sweep

1. Tuesday data is complete.
2. Every `5m` high remains below the buffered Asian High and every low remains
   above the buffered Asian Low through the observation window.
3. No candidate reaches reclaim or scoring.
4. Final reason is `no_sweep`.

### 19.3 Qualified setup blocked by news

1. Wednesday price sweeps and reclaims the Asian High.
2. The Short candidate scores `75`, above the threshold.
3. A stored high-impact UK release is scheduled eight minutes after the entry
   decision.
4. The entry belongs to the configured blackout interval.
5. The report records the hypothetical unblocked score and event details.
6. Final reason is `macro_blackout`.

## 20. Definition of done for implementation

The later implementation task is complete only when:

- V1 behavior and reports remain unchanged;
- V2 uses one evaluator in backtest, paper and live modes;
- all state transitions are causal and deterministic;
- every calendar date produces one session-ledger row per configured asset;
- daily reports can be regenerated from stored rows;
- backtest funnel counts reconcile exactly;
- automated tests cover every primary decision reason;
- the three static traces above pass; and
- no V2 parameter is promoted from an out-of-sample result.

## 21. Paper-test rollout notes

- The opt-in `london-v2-paper` worker observes BTC, ETH and SOL in one simulated
  portfolio. It uses the account's paper balance and the active run's selected
  leverage and V2 parameters. Invalid active parameters pause new entries.
  It does not write the main bot's orders, positions or strategy settings.
- Apply operational migrations through `0074_london_v2_calendar_coverage.sql`
  before starting the worker. Set
  `LONDON_V2_PAPER_ACCOUNT_ID` and `LONDON_V2_PAPER_START_DATE` explicitly.
  The worker reads public market data and does not receive exchange trading keys.
- The current V2 entry gate remains `08:00 Europe/London`. A signal or quote
  before that local opening time is not a valid V2 entry.
- `LONDON_V2_PAPER_MACRO_MODE=observe_unverified` permits exploratory paper
  entries without verified calendar coverage. Such sessions and reports say
  `macro_shield_unverified`; they do not validate the macro filter. `strict`
  blocks entries until coverage is verified. Import releases into
  `calendar_events` with `region` and `calendar_version`, then record one
  `calendar_coverage` row for each configured region and `UNCLASSIFIED` (when
  enabled), for each UTC date. Use the same nonempty version and an actual
  verification timestamp for every row. A coverage row attests that the source
  was checked for the whole UTC day, including days with no releases; merely
  inserting a few events does not verify coverage.
- Paper entries use a fetched quote after the closed `5m` signal. Complete `1m`
  candles drive conservative stop-first exit replay. Historical `1m` gaps
  before the quote entry are disclosed but do not block paper observation.
  Missing execution candles after entry leave the outcome and P&L unknown.
  The worker retries missing candles and replays the session after repair;
  it does not finalize an unresolved trade or send its daily report. If the
  exchange cannot restore a missing candle, the operator must import a
  verified OHLC candle and let the worker replay before the margin can be
  released. It never substitutes a current quote for an unknown historical
  TP/SL result.
- After a restart, the worker revisits one older incomplete UTC date per
  polling cycle, alongside yesterday and today. Historical signals never
  become new Paper fills because the one-minute entry window has elapsed.
- Entry, TP1, final exit and operator-exit messages are queued with stable
  deduplication keys and retried on delivery failure. The daily report is
  rebuilt from session rows after cutoff. A crash after Telegram accepts a
  message but before the receipt is saved can still cause a duplicate.
