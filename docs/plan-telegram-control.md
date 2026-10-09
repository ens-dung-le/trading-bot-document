# Plan part C — Controlling bots with Telegram

Part A builds and verifies the engine with fixture strategies. Part B connects the real strategy and prepares it for live operation. Part C turns Telegram from a **one-way notification channel** into a **two-way control channel**: select a symbol and strategy, run date-range backtests, enable paper trading, and receive report files.

This document continues with the same convention as the original plan: full name without abbreviation, English comments, all numbers are parameters, not hardwired.

---

## C1. Principle — chat controls are only allowed to reduce risk

This is the most important section of the entire document and should be finalized before writing any line of code.

Telegram is a **weak** authentication channel: all it takes is the phone being unlocked, the Telegram Web session being taken over, or the SIM being swapped for a stranger to type commands into the bot. Therefore, the rights of the chat channel are divided into two groups, and the boundary cannot be expanded:

| Group | Content | Why |
|---|---|---|
| **Allowed** | View status, pause, kill switch, select a symbol from the configured list, select a registered strategy, run backtests, enable or disable paper trading | These operations are read-only, reduce risk, or use simulated funds |
| **Never** | Turn on live mode, increase leverage, increase `riskPerTradeRate`, loosen `maximumMarginUsageRate`, loosen circuit breaker threshold, fix api key, add symbol not in whitelist | These are operations that make the risk **increase**. They can only be changed using environment variables and restarting the process, which requires access to the server |

Specific consequences:

- `TRADING_MODE=live` can only be set in `.env`. On Telegram, the live button **does not exist** — not hidden, but not in the code.
- List of symbols displayed on the keyboard taken from table `trading_symbols` is `is_active`. Users select from a list, do not type freely.
- Backtest runs in a separate process, does not have a trading api key, so even if there is an error, orders cannot be placed.

### C1.1 Each thing has exactly one source of truth

Four environment keys `TRADING_SYMBOLS`, `TRIGGER_TIMEFRAME`, `TREND_TIMEFRAME`, `STRATEGY_NAME` **dropped**. They describe things that now have a more accurate home, and each thing has two sources—and the two sources sooner or later diverge, often when no one is paying attention.

Specific case: user changes to `ETH/USDT:USDT` via Telegram, three weeks later someone restarts the process, `.env` still records `BTC/USDT:USDT`. Which symbol does the bot run? That question shouldn't exist.

| Concern | Source of truth | Who can change |
|---|---|---|
| List of allowed symbols | `trading_symbols` vs. `is_active = true` | Migration or script, unchanged from Telegram |
| Symbol is running | `bot_runtime_settings.active_symbol` | Telegram |
| Strategy and version running | `bot_runtime_settings.active_strategy_name` and `active_strategy_version` | Telegram |
| Trigger and trend timeframes | `strategy_configurations.parameters` of the active row | Stored with the parameter set and not independently editable |
| Parameter strategy | `strategy_configurations.parameters` | Script after setting parameters through out-of-sample |
| Run mode, leverage, risk per order, circuit breaker threshold | `.env` | Only people with access to the server |
| API key, connection string, Telegram configuration | `.env` | Only people with access to the server |

Rule: **`.env` stores safety and infrastructure settings; the database stores strategy settings.** With that boundary, everything Telegram must never increase is controlled by `.env`, so no second authorization mechanism is needed for those settings.

`.env` after removing four keys:

```bash
# Infrastructure and safety - server changes only
NODE_ENV, LOG_LEVEL, TRADING_MODE
DATABASE_URL, BACKTEST_DATABASE_URL
EXCHANGE_NAME, EXCHANGE_API_KEY, EXCHANGE_API_SECRET
EXCHANGE_API_PASSPHRASE, EXCHANGE_USE_SANDBOX
STARTING_BALANCE, RISK_PER_TRADE_RATE
DEFAULT_LEVERAGE, MAXIMUM_ALLOWED_LEVERAGE, MARGIN_MODE
ROLLING_DRAWDOWN_LIMIT_RATE, MAXIMUM_OPEN_POSITION_COUNT
TELEGRAM_*

# Removed: TRADING_SYMBOLS, TRIGGER_TIMEFRAME, TREND_TIMEFRAME, STRATEGY_NAME
```

**Cold start.** Removing these `.env` keys also removes the fallback used when `bot_runtime_settings` is empty. Replace it with an `is_default` column on `strategy_configurations`; exactly one row may be `true`:

- If `bot_runtime_settings` is empty and one configuration row has `is_default`, use that row, write it to `bot_runtime_settings`, and start normally.
- If `bot_runtime_settings` is empty and no row has `is_default`, keep the engine `Paused`, register no streams, and send `No active configuration. Open /menu and select a strategy.`

The second case is correct, not an error: a bot without an active configuration must remain idle and report the problem instead of guessing.

Also keep the same two rules from part A: reply only to `TELEGRAM_ALLOWED_USER_IDS`. Ignore non-whitelisted users **silently** because even a rejection would confirm that the bot exists.

**One operator.** `TELEGRAM_ALLOWED_USER_IDS` has only one identifier. Therefore, the first version **does not require optimistic locking**: without two people pressing buttons concurrently, there is no write conflict to resolve.

Two things remain for even one user:

- `bot_runtime_settings.version` increments on every change. It is an audit sequence number, not a value used to reject an operation.
- `control_audit_log` is still written. With one operator it does not need to distinguish who made a change, but it still records what changed and when. If a losing streak begins immediately after a strategy change at 02:00, this log connects the two events. It costs one row insert per operation.

The day a second person is added to the white list, the optimistic key must return, and `expectedVersion` already has a column to use.

---

## C2. Biggest architectural problem: backtest cannot run in the same process as the engine

A two-year one-minute candle backtest is about 1.05 million bars, each bar calls `evaluate` once. That loop takes up CPU **continuously for several minutes to several tens of minutes**, and Node runs one thread. If you run it in the engine process, during that time:

- Closed-candle events queue up without being handled.
- Trailing-stop updates cannot be submitted.
- The `/stop` command is not processed, so the kill switch is unavailable when it is needed most.

The stop loss remains resting at the exchange, so the position retains basic protection, but everything else freezes. This failure appears with long real datasets and may not surface in tests using only a few hundred candles.

**So: three separate processes.**

```
┌────────────────────────┐        ┌──────────────────────────┐
│  trading-bot           │        │  backtest-worker         │
│  engine + telegram     │        │  backtest jobs only      │
│  loop must stay fast   │        │  may consume CPU freely  │
│  has trading API keys  │        │  has NO API keys         │
└───────────┬────────────┘        └────────────┬─────────────┘
            │                                  │
            │   write jobs, read status         │  claim jobs, write progress
            │                                  │
            │                                  │
            ▼                                  ▼
      ┌───────────────┐              ┌──────────────────────┐
      │  trading      │              │  trading_backtest    │
      │ positions/orders│             │  jobs, runs, trades  │
      └───────────────┘              └──────────────────────┘
            ▲                                  ▲
            └──────── only the bot process ─────┘
                     connects to both
```

Worker opens only one connection, to `trading_backtest`. Details in section C8.1.

The queue is in Postgres, not Redis: the project already has Postgres, the number of jobs a day can be counted on the fingers of one hand, and adding another infrastructure just to queue five jobs is no return on operating costs.

Worker retrieves job using `SELECT ... FOR UPDATE SKIP LOCKED LIMIT 1` so that two workers never receive the same job.

In `docker-compose.yml`, add a `backtest-worker` service that uses the same image but a different command and **different environment variables**: do not pass `EXCHANGE_API_KEY` to it. A process without API credentials cannot place exchange orders even if the code is defective.

---

## C3. The menu state machine is a pure function

A multi-step menu can easily turn into a bunch of nested `if` clinging to global variables, and then it can't be tested. Apply the same discipline used for strategy: separate the decision part into pure functions, the network call part is outside.

```ts
// src/control/control-state.ts

/**
 * One screen of the Telegram control menu.
 *
 * The screen is part of the state rather than being inferred from the last
 * message, because a user can scroll up and press a button on an old message.
 */
export const ControlScreen = {
  MainMenu: "main_menu",
  SymbolList: "symbol_list",
  StrategyList: "strategy_list",
  FeatureChoice: "feature_choice",
  BacktestPeriod: "backtest_period",
  BacktestCustomPeriod: "backtest_custom_period",
  BacktestConfirm: "backtest_confirm",
  PaperConfirm: "paper_confirm",
} as const;

/** What the user has chosen so far. Nothing here has been acted on yet. */
export interface RunDraft {
  readonly symbol: string | null;
  readonly strategyName: string | null;
  readonly feature: "backtest" | "paper" | null;
  readonly periodStartTimestamp: number | null;
  readonly periodEndTimestamp: number | null;
  /**
   * Not chosen by the user. It comes from the selected strategy configuration,
   * because a parameter set is tuned for one timeframe and means something
   * different on another. It is kept in the draft only so that the period
   * screen can show how many bars the run will have. See C5.2.
   */
  readonly triggerTimeframe: Timeframe | null;
}

export interface ControlState {
  readonly screen: ControlScreen;
  readonly draft: RunDraft;
  /**
   * Identifier of the message the menu lives in. The menu edits that one
   * message instead of sending a new one per press, so the chat does not fill
   * up with dead keyboards the user can still tap.
   */
  readonly menuMessageId: number | null;
}

/**
 * Everything the reducer is allowed to know about the outside world.
 *
 * Passed in as data so the reducer stays pure: the same state and the same
 * event always produce the same result, and every branch can be tested without
 * a database or a network.
 */
export interface ControlContext {
  readonly availableSymbols: readonly string[];
  readonly availableStrategyNames: readonly string[];
  readonly openPositionCount: number;
  readonly tradingMode: TradingMode;
  readonly runningJobCount: number;
  /** Stored candle coverage per symbol, used to refuse an impossible period. */
  readonly coverageBySymbol: ReadonlyMap<string, DataCoverage>;
  /** Passed in rather than read from the clock, so the reducer stays pure. */
  readonly nowTimestamp: number;
}

export type ControlEvent =
  | { readonly kind: "command"; readonly text: string }
  | { readonly kind: "callback"; readonly data: string; readonly callbackId: string }
  | { readonly kind: "text"; readonly text: string };

/**
 * What the caller has to do after the state changed.
 *
 * The reducer never performs these itself. That is what keeps it testable and
 * what makes the ordering of side effects visible in one place.
 */
export type ControlEffect =
  | { readonly kind: "render_menu"; readonly view: MenuView }
  | { readonly kind: "answer_callback"; readonly callbackId: string; readonly text: string }
  | { readonly kind: "enqueue_backtest_job"; readonly request: BacktestJobRequest }
  | { readonly kind: "apply_paper_configuration"; readonly symbol: string; readonly strategyName: string }
  | { readonly kind: "send_text"; readonly text: string };

export function reduceControl(
  state: ControlState,
  event: ControlEvent,
  context: ControlContext,
): { readonly state: ControlState; readonly effects: readonly ControlEffect[] };
```

Three technical conditions of Telegram must be designed according to, and cannot be avoided:

1. **`callback_data` maximum 64 bytes.** Cannot fit json in there. Use shortcodes with prefix: `sym:BTCUSDT`, `str:sma-cross`, `per:90d`, `go:backtest`. The shortcode mapping to the full value is at `ControlContext`, not the button.
2. **The old button can still be clicked.** The user scrolls up and clicks on yesterday's menu again. Each time you build a menu, add a `menuVersion` to `callback_data` (`sym:BTCUSDT:v7`); version difference, reply "this menu is out of date" and redraw the current menu, instead of silently applying the selection to a different context.
3. **Click twice.** Telegram resends `callback_query` when the network is unstable. Save the processed `callbackId` in a limited set of a few hundred elements and ignore duplicates. Otherwise, a shaky click of the "run backtest" button could create two jobs.

Every `callback_query` must call `answerCallbackQuery` within a few seconds, even if rejected, otherwise the button will cycle forever on the user's machine.

---

## C4. Screen structure

All text sent by the bot is **English**. The document is written in Vietnamese, but the message is not: the index names, status names, and parameter names are all in English, so mixing the two languages ​​in a forty-character widescreen display just makes it harder to read.

```
/menu
┌─────────────────────────────────────────────┐
│ paper · BTC/USDT:USDT · sma-cross (v1, 1m)  │
│ state: running · open positions: 0          │
├─────────────────────────────────────────────┤
│ [ Symbol ]     [ Strategy ]                 │
│ [ Backtest ]   [ Paper trade ]              │
│ [ Status ]     [ Pause ]                    │
│ [ Kill switch ]                             │
└─────────────────────────────────────────────┘
```

The first line always shows the running configuration, because the phone screen is small and users often forget which symbol they are on.

**Select symbol** — `ControlScreen.SymbolList`

```
Select symbol · active: BTC/USDT:USDT

[ • BTC/USDT:USDT  · 1.05M candles ]
[   ETH/USDT:USDT  · 1.05M candles ]
[   SOL/USDT:USDT  · 420k candles  ]
[   AVAX/USDT:USDT · no history    ]
[ ← Back ]
```

Dot marks the running symbol. Each button displays the number of saved candles, so users can immediately know which symbols have not yet loaded their history instead of reporting an error after selecting them.

The current stage runs **one symbol at a time** to compare multiple strategies on the same data set, so this screen is only used when switching to another market. It still has to be there, because changing symbols by editing the configuration and then restarting is an operation that no one wants to do while the market is running.

### C4.1 Four contracts on KuCoin futures

Whitelist of four USDT-margined perpetual contracts:

| CCXT symbol | Exchange contract code | Notes |
|---|---|---|
| `BTC/USDT:USDT` | `XBTUSDTM` | KuCoin uses `XBT` and not `BTC` in its contract code |
| `ETH/USDT:USDT` | `ETHUSDTM` | |
| `SOL/USDT:USDT` | `SOLUSDTM` | |
| `AVAX/USDT:USDT` | `AVAXUSDTM` | Liquidity is noticeably thinner than the above three |

These four rows are loaded into `trading_symbols` by a one-time script, and **all specifications come from `loadMarkets()`, not manual input**: `priceTickSize`, `quantityStepSize`, `minimumQuantity`, `contractMultiplier`, and `maximumLeverage`. An exchange may change a price or quantity increment without notice; stale hand-entered values would cause orders to be rejected.

**Biggest trap: KuCoin futures places orders by contract number, not coin number.**

A `XBTUSDTM` contract is not a bitcoin — it is a `contractMultiplier` bitcoin, and that number differs between the four symbols. Meanwhile `position-sizing.service.ts` currently calculates:

```
quantity = riskAmount / stopLossDistance
```

outputs a number of **coin units**, then rounds according to `quantityStepSize`. Sending that number directly to KuCoin is sending the wrong unit, and wrong by a factor of several dozen to several hundred times depending on the contract - meaning either rejection, or opening a position much larger than expected.

This requires an explicit conversion step before sending the order:

```ts
/**
 * Convert a size in base currency into the number of contracts the exchange
 * expects.
 *
 * Formula: contractCount = floor(baseQuantity / contractMultiplier)
 *
 * KuCoin futures orders are denominated in contracts, and one contract is a
 * fixed amount of the base currency that differs per symbol. Sending a size in
 * base units is not slightly wrong, it is wrong by the multiplier.
 *
 * Rounding down and never up: a position larger than the one the risk
 * calculation allowed is the failure this whole layer exists to prevent.
 */
export function toContractCount(
  baseQuantity: number,
  contractMultiplier: number,
): number;
```

And vice versa when reading the position, because `fetchPositions` pays the number of contracts and the profit and loss calculation is done in coins.

Consequence: the real risk of an order is **contractually quantized**. If a contract is already larger than the amount of risk allowed, then the smallest order that can be placed exceeds the expected risk — in which case the signal must be **rejected**, not rounded up. This actually happens with small accounts and tight stops, so it belongs to `risk-guard.service.ts` and not a theoretical case.

**Unequal liquidity.** `defaultLiquidityRequirements.minimumQuoteVolume24Hours` is currently 30 million USDT. BTC and ETH far exceed that threshold all the time; SOL often exceeds; AVAX has periods of no overtaking. This is not a reason to lower the threshold — it is the right threshold — but a reason to accept that AVAX will be rejected during those periods, and that reason should be clearly visible in `signals.rejection_reason` instead of making users think the strategy is broken.

**Select strategy** — `ControlScreen.StrategyList`

Reading list from table `strategy_configurations` is `is_enabled`, **not reading from `strategyRegistry`**. This difference is important:

- Registry contains both `fixture-*` and `benchmark-random-entry`. They are there to test the engine, not for someone to run a paper trade using `every-bar-flip` and burn fees all day. Reading from the configuration panel they never make it into the menu, without an exclusion list that can be forgotten to update.
- A configuration row carries the **validated parameter set** and its **timeframe**. The menu therefore shows one entry for each `(strategy, version)` pair, not one entry per strategy implementation.
- Registry has only one thing left to do: map `strategy_name` to settings. What is allowed is a question of data, not code.

```
Select strategy · active: sma-cross v1

[ • sma-cross v1 · 1m/5m · warm-up 320 ]
[   sma-cross v2 · 1m/5m · warm-up 380 ]
[   range-breakout v1 · 5m/30m · warm-up 210 ]
[ ← Back ]
```

Adding a strategy to the menu means inserting a row into `strategy_configurations`. That is done through a migration or script after the parameter set passes out-of-sample validation, never from Telegram.

**Select feature** — `ControlScreen.FeatureChoice`

```
BTC/USDT:USDT · sma-cross v1 · 1m/5m

[ Run backtest ]  [ Start paper trade ]
[ ← Back ]
```

---

## C5. Run backtest from Telegram

### C5.1 Select a date range

```
Select period · sma-cross v1 · 1m

[ 30 days ]   [ 90 days ]
[ 1 year ]    [ 2 years ]
[ All stored data ]
[ In-sample 70% ]  [ Out-of-sample 30% ]
[ Enter dates ]
[ ← Back ]
```

Corresponding `callback_data` code: `per:30d`, `per:90d`, `per:1y`, `per:2y`, `per:all`, `per:in`, `per:out`, `per:custom`. All presets count backward from the **last candle in stock**, not from the mechanical clock: if the history stops at the last month, then "1 year" must be the twelve months ending last month, not the twelve months ending today with an empty last month.

**Hard ceiling: two years.** `maximumBacktestSpanDays = 730` is a configuration parameter, and it applies to **all** inputs, not just preset buttons:

| Entrance | When surpassing the ceiling |
|---|---|
| Preset button | No node is longer than two years |
| Complete data | Cut back to the last two years, clearly stating that it was cut |
| Manual entry | Refuse with reason and longest allowed period |
| In-sample 70% | Take 70% of the cutoff to two years, not 70% of the entire inventory |

If you just block the preset button and let the hand input box go straight, that ceiling isn't a ceiling — it's just a suggestion, and a tired person at two in the morning will type for about five years and then wait forty minutes.

Manually input the correct format: `2024-01-01 2024-06-30`, always interpreted as UTC. Any other format is rejected with examples, don't try to guess. Guessing the date format is a surefire way to one day run a backtest that's a month wrong without anyone knowing.

For periods longer than two years, use the command line: `npm run backtest -- --from ... --to ...` has no such ceiling because the operator is present to monitor the run and accept the waiting time.

**Presets that are longer than the existing data will be truncated, and must be stated.** Choosing "2 years" when there is only 14 months in stock is normal. The bot cuts to the actual amount and displays it clearly on the confirmation screen, without silently changing the number:

```
Requested : 2 years
Actual    : 2025-06-14 → 2026-08-27 (1 year 2 months)
That is all the stored history for BTC/USDT:USDT 1m.
```

If you don't say it, three months later, looking back at a report that says "2 years" but actually ran for 14 months, the number in there cannot be compared to any other report.

**The length changes the order of magnitude of the job, not just the number.** One-minute candles:

| Presets | Approximate number of bars | Estimated run time | Equity points saved |
|---|---|---|---|
| 30 days | 43 000 | under 1 minute | 720 |
| 90 days | 130 000 | 1 to 3 minutes | 2 160 |
| 1 year | 526 000 | 5 to 10 minutes | 8 760 |
| 2 years | 1 052 000 | 10 to 20 minutes | 17 520 |

Three consequences follow, all reflected in the following sections:

1. `maximumJobMinutes` default 30 is still tight with two year preset. Let it stretch: `maximumJobMinutes = max(30, ceil(totalCandleCount / 20000))`, which is about 53 minutes of being naked for two years. A fixed constant that either kills a valid job, or leaves the job hanging for too long.
2. The equity curve is stored at one point per hour rather than one point per candle (section C6.1), so a two-year preset leaves 17,520 rows instead of more than one million. It is still excluded from the default Telegram attachments because thousands of numeric rows are not useful on a phone. All metrics, including drawdown, are calculated from the full-resolution series before downsampling.
3. Benchmark randomized entries from section 12.1. The original plan replays **the entire** loop several hundred times. With two years of data, 200 runs process more than 200 million bars and can take many hours. The confirmation screen must therefore choose the random-run count, with defaults that **decrease as the test period grows**: 200 runs for up to 90 days, 50 for one year, and 20 for two years. It must note that a percentile based on 20 samples is much coarser than one based on 200. Operators who need a tighter estimate can run the CLI benchmark overnight.

**The out-of-sample button must have a warning.** According to section 7 of the original plan, an out-of-sample evaluation should be run **once**. Repeated evaluation and subsequent parameter adjustment turn that period into in-sample data. The bot therefore counts out-of-sample runs for each `(strategy, version, symbol)` tuple in `backtest_jobs` and displays the count directly:

```
Warning: this parameter set has already been run
out-of-sample 2 times. Every rerun makes the
out-of-sample number less meaningful.

[ Run anyway ]  [ Cancel ]
```

The bot does not enforce the one-run rule—command-line runs remain possible—but it records the count and warns the user.

### C5.2 Timeframe is decided by strategy, the time period must match it

**Telegram does not allow timeframe selection.** Timeframe is a property of the parameter set, not a choice of the button clicker.

Reason: consider a strategy with `fastLookbackPeriod: 20`, `slowLookbackPeriod: 60`, and `stopLossMultiplier: 2` tuned on one-minute candles. Running the same numbers on hourly candles makes a 20-bar lookback almost one day instead of 20 minutes. That is **another strategy**, and its result says nothing about the parameter set running live. A timeframe selector in the menu would make this invalid comparison too easy.

Each row in `strategy_configurations` therefore carries its own timeframe:

```jsonc
// strategy_configurations.parameters
{
  "triggerTimeframe": "1m",
  "trendTimeframe": "5m",
  "fastLookbackPeriod": 20,
  "slowLookbackPeriod": 60,
  "averageTrueRangeLookbackPeriod": 14,
  "stopLossMultiplier": 2,
  "rewardToRiskRatio": 2
}
```

Selecting `sma-cross v1` also selects `1m/5m`. Running the same idea on a 15-minute timeframe requires a **different configuration row** with parameters tuned separately for that timeframe in stage B4—not the same values with a different timeframe label.

Determining which timeframes are effective belongs to strategy optimization, not the control menu.

**But the number of bars still has to be checked.** The time period multiplied by the strategy's timeframe gives the number of bars, and the number of bars determines whether the job runs for one minute or forty minutes, as well as whether the result is meaningful or just noise:

```
totalCandleCount = periodDays * 1440 / timeframeMinutes
```

| Period | 1m | 5m | 15m | 1h |
|---|---|---|---|---|
| 30 days | 43 200 | 8 640 | 2 880 | 720 |
| 90 days | 129 600 | 25 920 | 8 640 | 2 160 |
| 1 year | 525 600 | 105 120 | 35 040 | 8 760 |
| 2 years | 1 051 200 | 210 240 | 70 080 | 17 520 |

Both extremes of the acceptable bar-count range fail differently, and neither should be treated as an ordinary warning:

- **Too heavy.** Two years of one-minute candles exceed one million bars, take roughly fifteen to twenty minutes to replay, and the randomized-entry benchmark multiplies that workload twentyfold.
- **Too thin.** Thirty days of hourly candles contain only 720 bars. After a 320-bar warm-up, roughly 400 bars remain, often producing only a handful of trades. The backtest can still print profit factor and Sharpe ratio, but those estimates are mostly noise. This failure is more dangerous because the report looks normal.

**The acceptable range** is configurable, not hard-coded:

```ts
/**
 * The band of bar counts a backtest started from Telegram has to land in.
 *
 * Below the lower bound the run is too short to say anything: the warm-up eats
 * most of it and the few remaining trades are noise that still gets printed as
 * a profit factor. Above the upper bound the job stops being something anyone
 * waits for on a phone, and the random entry benchmark multiplies it further.
 *
 * The command line has no such band. Someone sitting in front of the machine
 * can wait forty minutes and knows why they are waiting.
 */
export const telegramBacktestBarBand = {
  minimumBarCount: 20_000,
  recommendedMaximumBarCount: 300_000,
  hardMaximumBarCount: 1_100_000,
} as const;
```

**Consequences of the 300,000-bar threshold for a one-minute strategy.** As shown above, 90 days contain about 130,000 bars and can proceed directly. One year (526,000) and two years (1.05 million) both exceed the recommended threshold and **require an additional confirmation step**.

That is correct behavior: the operator should knowingly approve an eight- or eighteen-minute CPU-intensive run. The validation step prints the bar count and estimated runtime, then asks for confirmation:

```
1 year at 1m is 525 600 bars, about 8 minutes.
That is above the 300 000 bar comfort limit.

[ Run anyway ]  [ Pick a shorter period ]
```

The `hardMaximumBarCount` threshold of 1.1 million is just above two years of one-minute candles. The time-span ceiling in C5.1 and the bar-count ceiling therefore meet at the same boundary. This is intentional: conflicting limits could display an enabled period button that is rejected only after it is clicked.

**The period menu is labeled using the selected strategy's timeframe.** Because the timeframe is fixed, each period is either displayed normally, marked as heavy, or blocked when it contains too few bars.

With `sma-cross v1` on timeframe `1m`:

```
Select period · sma-cross v1 · 1m

[ 30 days · 43k bars · ~30s ]
[ 90 days · 130k bars · ~2m ]
[ 1 year  · 526k bars · ~8m ]   ⚠ heavy
[ 2 years · 1.05M bars · ~18m ] ⚠ heavy
```

With a `1h` configuration, the same list changes:

```
Select period · range-breakout v1 · 1h

[ 30 days · 720 bars · too few ]     ← blocked
[ 90 days · 2 160 bars · too few ]   ← blocked
[ 1 year  · 8 760 bars · too few ]   ← blocked
[ 2 years · 17 520 bars · ~20s ]
```

A period below `minimumBarCount` is **disabled** and displays the reason. A period above `recommendedMaximumBarCount` remains selectable but requires an additional confirmation.

In the second example, the hourly configuration has only one valid period. That is useful information: the parameter set needs more data, so the operator should synchronize additional history instead of running an undersized sample.

**Warm-up must also be expressed in elapsed time.** A 320-bar warm-up is about five and a half hours on `1m`, but about thirteen days on `1h`. The threshold in C5.3 remains `warmUpCandleCount * 3`, while the screen displays both bar count and elapsed time because `warm-up 13 days` is more immediately meaningful than `warm-up 320 candles`.

**Acceptance thresholds must be timeframe-specific.** Section 15.3 of the original plan requires at least 600 trades. Two years of hourly data contain only 17,520 bars, and a crossover strategy may generate roughly one hundred trades. It would fail a threshold calibrated for `1m` even if its edge were valid. `AcceptanceThresholds` therefore needs a profile per timeframe, and every report must identify the profile used.

**Results across timeframes are not directly comparable.** Per-trade friction may be the same while typical price movement per trade differs substantially, so profit factor on `1m` and `1h` describes different conditions. Rankings in `run-batch-backtest.ts` must therefore be **grouped by timeframe**; comparisons should be between strategies on the same symbol and timeframe.

### C5.3 Check data before loading

Before creating a job, check it on the spot, because it's much cheaper than running it for ten minutes and then reporting an error:

| Check | Refuse when | Message |
|---|---|---|
| Is there any data | Table `candles` has no rows for `(symbol, timeframe)` | `No stored history. Run sync-history first.` |
| Requested dates covered | The first candle is later than `from`, or the last candle is earlier than `to` | `Stored range is 2025-06-14 to 2026-08-27.` |
| Gap rate acceptable | The missing-candle rate exceeds `maximumMissingCandleRate`, suggested default 0.1% | `1 240 candles missing (0.47%), first gap at 2025-11-02T03:14Z.` |
| Enough warm-up | The number of candles in the range is less than `warmUpCandleCount * 3` | `Period too short: warm-up would use most of it.` |
| Secondary timeframe available | Strategy needs `5m` but it has not been loaded | `Trend timeframe 5m has no stored history.` |

The last check deserves emphasis: a 30-day `1m` backtest has 43,200 bars, so a 320-bar warm-up is small. A two-day backtest has only 2,880 bars; warm-up consumes more than 10%, leaving too little opportunity for a meaningful trade sample. `warmUpCandleCount * 3` is an absolute minimum, not evidence that the sample is sufficient.

### C5.4 Confirmation screen

Confirmation prints out **all assumptions**, in the spirit of section 8 of the original plan: a result without cost assumptions is unreadable.

```
Confirm backtest

Symbol        : BTC/USDT:USDT
Strategy      : sma-cross v1
Timeframe     : 1m (trend 5m)
Period        : 2024-01-01 to 2024-06-30 (UTC)
Candles       : 261 216 · 12 missing (0.005%)
Warm-up       : 320 candles (5h 20m at 1m)
Usable        : 260 896 candles for signals
Gate profile  : thresholds for 1m
Taker fee     : 0.060% per side
Slippage      : 1 tick · half spread 0.010%
Leverage      : 3 · isolated
Risk per trade: 1.00%
Random entry  : 50 runs (scaled to period length)
Estimated     : 3 to 5 minutes

[ Run ]  [ Cancel ]
[ Change random entry runs ]
```

Estimate runtime from the requested bar count and the average processing time per bar from previous runs stored in `backtest_jobs`. When no history exists, display `no estimate yet`.

### C5.5 Progress

Worker writes `progress_rate` to the job table every 1% of bars. The bot process reads that table every 10 seconds and **edits** a single message:

```
Backtest running · 43%
▓▓▓▓▓▓▓▓▓░░░░░░░░░░░  112 400 / 261 216 bars
elapsed 2m 10s · about 2m 50s left
```

Edit messages instead of sending new ones, and every 10 seconds instead of every percentage: Telegram's limit is about 20 messages per minute to a group, and a long job will eat up that limit and block important alerts lining up behind. The queue at `message-queue.service.ts` must treat progress information as **cancelable**: if the queue is accumulating, discard the old progress version, only keep the latest version.

If a job runs too long, the worker terminates it after `maximumJobMinutes` and marks it `failed` with a reason. A hung job would otherwise block every later job because the queue has concurrency one. As described in C5.1, this timeout **scales with bar count**: `max(30, ceil(totalCandleCount / 20000))`, about 30 minutes for a short backtest and 53 minutes near the two-year `1m` ceiling.

---

## C6. Export the resulting file

When the job is finished, the bot sends **two things**, in this order:

**A, short summary in the message.** The limit for a Telegram message is 4096 characters, while `summary.txt` is currently longer than that. So the message only contains the part read on the phone:

```
Backtest finished · sma-cross v1 · BTC/USDT:USDT · 1m
2024-01-01 to 2024-06-30

Net profit    : +3 842.51 USDT  (+38.43%)
Trades        : 412 · 43.20% winners
Profit factor : 1.51
Max drawdown  : -13.80%
Sharpe        : 1.42
Liquidations  : 0
Friction      : 29.0% of gross profit

Gate: FAILED (2 of 10 criteria)
  x Sharpe 1.42 below 1.8
  x Trades 412 below 600

Buy and hold over the same period: +121.60%
```

The final lines are often overlooked but are the most important: acceptance-gate results and the comparison with buy-and-hold. A profit figure alone is not sufficient evidence.

**Second, attached files.** Use the Bot API's `sendDocument`. Reports are **not stored on disk**: their source data is stored in `backtest_runs`, `backtest_trades`, and `backtest_equity_points`, and files are generated from those rows when requested.

| File | Always send | Word construction |
|---|---|---|
| `summary.txt` | Yes | Column `summary_text` |
| `benchmark.txt` | Yes, if you run benchmark | Column `benchmark_text` |
| `parameters.json` | Yes | Column `strategy_parameters` |
| `monthly-breakdown.txt` | Yes | Column `monthly_breakdown_text` |
| `trades.csv.gz` | Yes | Table `backtest_trades`, compressed in memory at send |
| `equity-curve.csv.gz` | No default | Table `backtest_equity_points`, sent when pressing `Send equity curve` |

Generating files on demand preserves **one source of truth**. A persisted export can diverge from the database as soon as report formatting changes, creating a second copy that is difficult to reproduce consistently.

Practical consequences: the bot runs on any machine, any container, after many redeployments, it still sends out the same report. Previously, the report was located in `data/reports/` of the machine that ran the job, so a rebuild on an empty volume kept all the indicators intact but lost the report.

The Bot API upload limit is 50 MB. With one equity point per hour (section C6.1), two years produce 17,520 rows—about 700 KB uncompressed and much less after gzip—so the `Send equity curve` action remains safely below the limit.

If sending the file fails, **do not lose the results**: everything is already in the database. The bot returns the run label and ID with a `Retry sending` button, and the next click rebuilds the file from the same stored rows. The principle from part A still holds: a failed notification must not damage the result it reports.

### C6.1 Storage and cleanup

Reports grow monotonically. The heavy portion is not the text but the child tables: a one-year `1m` run produces about 526,000 equity points if every candle is stored, and ten runs exceed five million rows at a resolution a human cannot meaningfully inspect.

**Downsample when storing.** Save **one equity point per hour**, plus the final point. Always retain the endpoint because it carries the ending balance; omitting it would make a partial-month curve disagree with the summary. Calculate all metrics—especially drawdown—from the full-resolution series **before** downsampling. Store the retained resolution in `equity_point_interval_seconds` so later charts state what they represent.

**Two cleaning mechanisms, running independently of each other.**

**One, sort by age.** `reportRetentionDays = 30`. The task runs hourly, cleaning reports of jobs that are `succeeded` older than 30 days and **unpinned**. Pin with the `Keep` button on the resulting message, save to `backtest_jobs.is_pinned`.

**Second, clean up by storage usage.** Age alone is insufficient: ten one-year runs created on the same day are all new, so age-based cleanup removes none of them. Measure storage using `pg_total_relation_size` for the three report tables—the actual bytes used, not an estimate based on row counts:

| Threshold | Suggested value | Action |
|---|---|---|
| Warning | exceed `artifactWarningGigabytes` = **2 GB** | Send a `warning` notification with current usage; delete nothing |
| Incremental cleanup | exceed `artifactCriticalGigabytes` = **5 GB** | Delete artifacts **from the oldest run forward**, one run at a time, until usage returns to the warning threshold; send a message for every cleaned run |

The two milestones are **absolute gigabytes, not percentages**, same argument as before: what determines whether the database can still work or not is the actual number of bytes, not the ratio on a disk that no one has measured.

Four constraints of incremental cleanup, missing any of them will result in unintended data loss:

1. **Never clean pinned reports**, even if the database is large. If only pinned reports remain, stop and send a `critical` warning so an operator can decide what to retain.
2. **Do not clean up reports of jobs currently `running` or `queued`.**
3. **Sort in ascending order `finished_at`**, oldest first, not by size. Clearing the biggest one releases faster but is often the most important result.
4. **Record in log and send to Telegram each cleaned run**, with `run_label`. If you look for a report three months later and can't find it, you have to know when it was removed and for what reason.

**Cleanup removes heavy artifacts, not the run itself.** Delete rows from `backtest_trades` and `backtest_equity_points`, set the three report-text columns to `NULL`, and record `artifacts_deleted_at`. Keep the `backtest_runs` row and its metrics: those compact values are what allow completed runs to remain comparable over time.

The `Retry sending` action for a cleaned run must respond with `The report of job 41 was removed by the retention cleanup. The metrics are still in the database.` rather than fail silently.

### C6.2 Two csv files are compressed when sending

**Decision: `trades.csv` and `equity-curve.csv` are built from the database and compressed in memory immediately before sending, without writing temporary files.**

These two files are repeating decimal numbers — timestamps increase steadily, prices fluctuate within a narrow range, the same `entry_reason` repeats hundreds of times. That's exactly the type of data gzip handles best: about one-fifth left.

Sizes after the equity-curve downsampling defined in C6.1:

| File | Two years of 1m candles, approximate | After compression | Notes |
|---|---|---|---|
| `equity-curve.csv` | ~0.7 MB | ~0.15 MB | Two years of 1m candles, stored as one row per hour |
| `trades.csv` | 0.2 to 2 MB | 0.05 to 0.4 MB | Depending on the number of orders |

**The other four text files are not compressed**: `summary.txt`, `benchmark.txt`, `monthly-breakdown.txt`, `parameters.json`. They are only a few kilobytes, so compressing them does not save anything significant, but loses exactly what is most valuable: viewing them directly in Telegram without having to download and decompress them.

**In-memory compression is sufficient, unlike in the old design.** Previously, a two-year full-resolution equity curve contained hundreds of thousands of rows, so holding it all in memory for compression caused a large peak while the worker was already busy. At one point per hour it contains about seventeen thousand rows; `gzipSync` can process that quickly using only `node:zlib`.

**Shared header.** Define the column list once in `report.service.ts`; both CSV constructors, one using an in-memory run result and one using persisted rows, must reuse it. Duplicating the format allows the two exports to drift silently when a column moves.

**Required fields must be persisted.** Add `leverage`, `slippage_cost`, `gross_profit_and_loss`, `was_break_even_stop_armed`, and `holding_candle_count` to `backtest_trades`. Once database rows are the only source, a downloaded trade list without leverage and slippage cannot be used to audit the run.

### C6.3 Candle history is also in the database

Reports are not the largest data set. Candle history grows as follows:

| Content | Row count |
|---|---|
| One symbol, 1m, two years | 1.05 million |
| Four symbols, 1m, two years | 4.2 million |
| Add 5m frame to all four | 840 thousand |

Unlike reports, history is **never cleaned up**. It is permanent market data; reloading takes hours because of exchange rate limits and depends on how much historical data the exchange still serves, so storage grows monotonically.

**Decision: history lives in the `candles` table, with no copy on disk.** The previous design stored monthly `.csv.gz` files. See `docs/plan-database-candle-history.md` for the migration path. The final design removes those files instead of maintaining a parallel copy for three reasons:

**One, two sources inevitably drift.** If the coverage monitor reads the file directory while the worker replays the database table, the UI can accept a date range based on monthly files that the worker never opens, then fail several minutes later.

**Two, database upserts are idempotent.** Rerunning `sync-history` over an existing range requires indexed conflict checks, and an interrupted run leaves committed candles without falsely marking the entire range complete. A file workflow has to approximate this behavior with `.tmp` files and renames.

**Three, gaps are detected by query.** `verify-data-integrity` queries the table directly instead of inferring coverage from which monthly files exist.

In return, there is a price that must be clearly stated: **Postgres backup becomes mandatory**. While the history is still on file, a corrupted database still leaves enough data behind to rerun the backtest. Now the database is unique, so a backup schedule and a recovery attempt are a mandatory part of the operation, not a reserve.

**The cleanup task in C6.1 should never touch table `candles`.** It only cleans up three report tables. The report can be reconstructed in a run of a few minutes; Reload history takes hours. When the database is so large that the only way is to delete the history, that's when people need to decide, not the machine.

---

## C7. Enable paper trade from Telegram

This is the most dangerous part of part C, because it touches the live engine.

The engine now registers the websocket **once at startup** in `bootstrap.service.ts`. Changing symbols at runtime requires something that doesn't already exist: the ability to stop and rebuild the register without restarting the process. Add `engine-supervisor.service.ts` to keep machine state:

```
Stopped ──start──► WarmingUp ──enough candles──► Running
                       │                   │
                       │                pause│resume
                       │                   ▼
                       └──────────────► Paused
                                           │
                                        halt│
                                           ▼
                                        Halted (manual resume only)
```

**Process to change symbols or strategies, do not reverse order:**

```
1. Reject immediately if a position is open.
     → `1 open position on BTC/USDT:USDT. Close it before switching.`
     -> buttons [ Close position and switch ]  [ Cancel ]
2. Transition to `Paused` and stop producing new signals.
3. Cancel every unfilled entry order for the old symbol.
4. Close the old symbol's WebSocket stream.
5. Write the new configuration to `bot_runtime_settings` with the actor.
6. Transition to `WarmingUp`: load history for the new symbol through REST and verify gaps.
7. Transition to `Running` only after warm-up is complete.
8. Send a notification with the new configuration and loaded candle count.
```

Step 1 has no exceptions. Changing strategy while a position is open creates an **orphaned** position: it was opened under the old strategy but would then be managed by a new strategy that was never tested for that situation. Exchange-resting stops still protect it, but trailing stops, indicator exits, and time exits would all lose their intended context.

Step 6 is no exception: the bot **does not enter orders during warm-up**, exactly as in section 13.6 of the original plan.

**Recovery after reboot.** `bot_runtime_settings` is read in `bootstrap.service.ts`, after the reconciliation step and before opening the stream. Users who changed to `ETH/USDT:USDT` via Telegram and then the process died, then when turned back on it had to return to `ETH/USDT:USDT`, with no other values ​​competing with it - this is the reason why the four environment keys in section C1.1 were removed.

If the table is empty, follow the cold-start rule in C1.1: use the single row marked `is_default`; otherwise keep the engine `Paused` and report that configuration is required.

**Multi-symbol.** Initial version only allowed **one symbol active at a time**. Multiple symbols at the same time entail the correlation exposure limit in section 14.4 of the original plan, for which the correlation matrix is ​​currently passed as empty. Opening multiple symbols before having that matrix is ​​opening three seemingly independent positions while actually being one triple position.

---

## C8. Additional Schema

### C8.1 Two separate databases

Backtest writes to a **separate database**, not shared with operational data. The operational database will also be used for other purposes, and mixing two types of data with completely different life cycles in one place is something that will be very difficult to untangle later.

| | `trading` (operation) | `trading_backtest` |
|---|---|---|
| Table | `trading_symbols`, `strategy_configurations`, `signals`, `orders`, `fills`, `positions`, `bot_runtime_settings`, `control_audit_log`, `candles`, `funding_rates`, `calendar_events` | `backtest_jobs`, `backtest_runs`, `backtest_trades`, `backtest_equity_points`, `strategy_candidates` |
| Who recorded | Bot progress | Bot (job placement) and worker (result) process |
| What if I lose it | Lost track of real positions and orders. **Must back up** | Run again yes. Low priority backup |
| Growth | Slow and steady | Every large batch or run adds hundreds of thousands of equity rows |

Place two databases **on the same Postgres server**. "Database alone" is enough to separate the boundary, but building two servers just doubles the memory without giving anything in return. If you want to completely separate from another computer, just change the connection string.

`.env` therefore has an additional key:

```bash
DATABASE_URL=postgres://trading:trading@127.0.0.1:5433/trading
BACKTEST_DATABASE_URL=postgres://trading:trading@127.0.0.1:5433/trading_backtest
```

**Most important technical consequence: there is no foreign key between the two databases.** Postgres cannot cross-reference, so these two columns of `backtest_runs` no longer exist as foreign keys:

```sql
strategy_configuration_id INTEGER REFERENCES strategy_configurations (id)
symbol_id                 INTEGER REFERENCES trading_symbols (id)
```

There are two ways to handle it, and the second way is more correct **even if the two tables are in the same database**:

1. Duplicate the reference table to the backtest database and synchronize. Maintaining foreign keys, in exchange for having an additional synchronization mechanism to fail.
2. **Snapshot the values directly on the run row.** Use no cross-database foreign key or synchronization.

Choose option two for a reason more important than database separation: a foreign key to `strategy_configurations` points to **an editable row**. If someone changes the version 1 parameters later, every old report referencing that row would misrepresent what it actually ran. A snapshot keeps the exact executed values permanently, matching the role of `parameters.json` in section 8 of the original plan.

So `backtest_runs` and `backtest_jobs` carry a copy:

```sql
    -- Snapshot captured when the run starts. Intentionally not a foreign key:
    -- the source configuration may change, but the report must remain immutable.
    strategy_name       TEXT        NOT NULL,
    strategy_version    INTEGER     NOT NULL,
    strategy_parameters JSONB       NOT NULL,
    unified_symbol      TEXT        NOT NULL,
    exchange_symbol     TEXT        NOT NULL,
    symbol_specification JSONB      NOT NULL,   -- tick, step, multiplier, maintenance margin
    cost_model          JSONB       NOT NULL,   -- fees, spread, slippage, latency
```

**Worker does not need a database to operate.** The snapshot is already in the job stream, so the worker only opens a connection to `trading_backtest` and reads the history file. Combined with the fact that it is not given an api key in section C2, the worker becomes a process that **cannot touch anything on the real account** — not because of discipline in writing the code, but because it has no way to touch it.

Migration splits into two directories, run with two separate commands:

```
migrations/
├── operational/     # runs on trading
└── backtest/        # runs on trading_backtest
```

```sql
-- ============================================================
-- Telegram control
-- ============================================================

-- Active runtime configuration. Exactly one row, enforced by CHECK so nobody
-- accidentally inserts a second row and makes two processes read different configurations.
CREATE TABLE bot_runtime_settings (
    id                  INTEGER     PRIMARY KEY DEFAULT 1 CHECK (id = 1),
    active_symbol       TEXT        NOT NULL,
    active_strategy_name TEXT       NOT NULL,
    active_strategy_version INTEGER NOT NULL DEFAULT 1,
    -- running, paused, or halted. Halted requires a manual resume command.
    engine_state        TEXT        NOT NULL DEFAULT 'running',
    -- Incremented on every change. With one operator this is only a revision
    -- number for log correlation. If a second operator is allowed, it becomes
    -- an optimistic-lock key: commands include the version they observed and
    -- are rejected when it no longer matches.
    version             INTEGER     NOT NULL DEFAULT 1,
    updated_by_user_id  BIGINT,
    updated_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Exactly one row may be the default. It is used only for the first startup
-- while bot_runtime_settings is still empty.
ALTER TABLE strategy_configurations
    ADD COLUMN is_default BOOLEAN NOT NULL DEFAULT FALSE;

CREATE UNIQUE INDEX strategy_configurations_single_default_index
    ON strategy_configurations ((TRUE)) WHERE is_default;

CREATE TYPE backtest_job_status AS ENUM (
    'queued', 'running', 'succeeded', 'failed', 'canceled'
);

-- Stored in the trading_backtest database.
CREATE TABLE backtest_jobs (
    id                  BIGSERIAL PRIMARY KEY,
    -- The user who pressed the button, for auditing and result delivery.
    requested_by_user_id BIGINT     NOT NULL,
    chat_id             TEXT        NOT NULL,
    -- Snapshot, not a foreign key: the source table is in another database and
    -- its row may be edited after the job has run.
    unified_symbol      TEXT        NOT NULL,
    exchange_symbol     TEXT        NOT NULL,
    symbol_specification JSONB      NOT NULL,
    strategy_name       TEXT        NOT NULL,
    strategy_version    INTEGER     NOT NULL,
    strategy_parameters JSONB       NOT NULL,
    cost_model          JSONB       NOT NULL,
    timeframe           TEXT        NOT NULL,
    period_start        TIMESTAMPTZ NOT NULL,
    period_end          TIMESTAMPTZ NOT NULL,
    -- in, out, or all. Stored to count how often a parameter set has run
    -- out-of-sample, because the result loses validity after the first run.
    sample_selection    TEXT        NOT NULL DEFAULT 'all',
    status              backtest_job_status NOT NULL DEFAULT 'queued',
    -- From 0 to 1. The worker writes after each one percent of bars.
    progress_rate       NUMERIC(5, 4) NOT NULL DEFAULT 0,
    total_candle_count  INTEGER,
    -- References backtest_runs after success. This foreign key is valid because
    -- both tables are in trading_backtest.
    backtest_run_id     BIGINT      REFERENCES backtest_runs (id),
    -- No column points to a directory: backtest_run_id identifies the report.
    -- Cleanup records backtest_runs.artifacts_deleted_at and keeps the run row,
    -- so metrics are retained when heavy artifacts are removed.
    error_message       TEXT,
    -- Protects the result from age-based and capacity-based cleanup.
    is_pinned           BOOLEAN     NOT NULL DEFAULT FALSE,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    started_at          TIMESTAMPTZ,
    finished_at         TIMESTAMPTZ
);

-- The worker claims the oldest queued job. SKIP LOCKED prevents two workers
-- from ever claiming the same job.
CREATE INDEX backtest_jobs_queue_index
    ON backtest_jobs (status, created_at) WHERE status = 'queued';

-- Four allowed contracts. Only symbols are listed here; the loader fills all
-- specifications from loadMarkets(), never by hand, because the exchange may
-- change a price step without notice.
INSERT INTO trading_symbols (
    exchange_name, exchange_symbol, unified_symbol,
    base_currency, quote_currency,
    price_tick_size, quantity_step_size, minimum_quantity
) VALUES
    ('kucoinfutures', 'XBTUSDTM',  'BTC/USDT:USDT',  'BTC',  'USDT', 0, 0, 0),
    ('kucoinfutures', 'ETHUSDTM',  'ETH/USDT:USDT',  'ETH',  'USDT', 0, 0, 0),
    ('kucoinfutures', 'SOLUSDTM',  'SOL/USDT:USDT',  'SOL',  'USDT', 0, 0, 0),
    ('kucoinfutures', 'AVAXUSDTM', 'AVAX/USDT:USDT', 'AVAX', 'USDT', 0, 0, 0)
ON CONFLICT (exchange_name, exchange_symbol) DO NOTHING;

-- Every control action, including rejected actions.
-- With one operator it does not answer "who changed it," but it still answers
-- "when did it change"; if a losing streak starts after a strategy change,
-- this is the record that connects the two events.
CREATE TABLE control_audit_log (
    id                  BIGSERIAL PRIMARY KEY,
    user_id             BIGINT      NOT NULL,
    action              TEXT        NOT NULL,   -- select_symbol, run_backtest, kill_switch, ...
    payload             JSONB,
    was_accepted        BOOLEAN     NOT NULL,
    rejection_reason    TEXT,
    created_at          TIMESTAMPTZ NOT NULL DEFAULT NOW()
);
```

Menu session table **does not** save to Postgres. It is a temporary state of a conversation. If lost, the user just needs to retype `/menu`. Keep in memory for a period of 30 minutes, up to a few dozen sessions.

---

## C9. Additional folders

```
src/
├── control/                          # control logic; reducer has no side effects
│   ├── control.module.ts
│   ├── control-state.ts              # pure reducer and menu state machine
│   ├── control-state.test.ts
│   ├── menu-view.ts                  # builds content and keyboards; pure
│   ├── menu-view.test.ts
│   ├── control-session.store.ts      # expiring in-memory sessions
│   ├── runtime-settings.service.ts   # reads/writes bot_runtime_settings; optimistic lock
│   └── control-handler.service.ts    # connects reducer to Telegram and the engine
│
├── engine/
│   └── engine-supervisor.service.ts  # start, pause, resume, runtime symbol changes
│
├── backtest/
│   ├── backtest-job.service.ts       # queueing, status reads, data checks
│   └── backtest-job.worker.ts        # claim/run/progress loop
│
├── notification/
│   ├── telegram-keyboard.ts          # builds inline keyboards and encodes callback_data
│   ├── telegram-document.service.ts  # sendDocument from buffers; size limits
│   └── telegram-command.service.ts   # extended to accept callback_query
│
└── database/
    ├── database.module.ts             # operational and backtest connections
    ├── backtest-database.module.ts    # separate connection used by the worker
    ├── schema/backtest/               # trading_backtest tables
    └── repositories/
        ├── backtest-job.repository.ts
        └── runtime-settings.repository.ts

scripts/
└── run-backtest-worker.ts            # worker process entry point
```

---

## C10. Marginal cases must be handled from the beginning

| Situation | Processing |
|---|---|
| Job finished while Telegram was dead | The results are in the database and on disk. The bot sends again when reconnecting, based on `finished_at` and the sent | flag
| Worker died midway | The job is `running` too `maximumJobMinutes` is marked `failed` when the worker restarts, don't let it hang forever |
| User clicks "Run" twice | Block with processed `callbackId`, and by binding an identical `queued` job to the same user |
| Change symbol while an entry order is still pending | Cancel the pending order before closing the stream, following the sequence in C7 |
| Symbol selected but no history yet | Block on the selection screen, display the number of candles right on the | button
| `summary.txt` sending failed | Send text link with retry button, no results lost |
| User types date in wrong format | Refuse with examples, don't guess |
| The backtest runs when the engine is in position | Allowed: worker is another process, does not touch the engine. This is the reason for process separation |
| A job is queued while another job is running | Enqueue it, report its queue position, and do not run it in parallel |
| Preset is longer than existing data | Cut to the actual range and clearly displayed on the confirmation screen, do not silently change the number |
| Preset 2 years with random benchmark | Decrease default runs by length, specifying coarser percentiles |
| Enter a period longer than two years | Deny with longest allowed range, suggest running with command line |
| Strategy has no row in `strategy_configurations` | Do not show it in the menu. Runnable configurations are controlled by data, not only by the registry |
| `bot_runtime_settings` is empty and no row has `is_default` | Keep the engine `Paused`, register no stream, and clearly request configuration selection; do not guess |
| Symbol in `bot_runtime_settings` has been set `is_active = false` | Refuse to start engine, send warning `critical`. A symbol is removed from the whitelist for a reason |
| A contract is larger than the allowable risk amount Reject the signal, don't round up. Really happens with small accounts and narrow stops |
| AVAX fails to reach 24-hour volume threshold | Reject the signal and clearly state the reason in `signals.rejection_reason`, so as not to be misunderstood as a failed strategy |
| The selected range yields less than 20 000 bars in the timeframe of strategy | Block completely, with suggestions for a longer interval, or download more history |
| Free storage is below the warning threshold | Send a warning with current usage; delete nothing |
| Database backtest cannot connect | The bot still runs the engine normally, only the backtest feature says it cannot be used. A dead secondary database cannot be taken with it the part holding the | position
| The strategy configuration was edited after the job ran | The old report is unchanged, because run carries a parameter snapshot, not a foreign key |
| Free storage is below the critical threshold | Delete oldest report artifacts first, skip pinned runs, and report each deletion |
| There are only pinned folders left but the disk is still full | Stop deletion, warning level `critical`, waiting for someone to decide |
| Press `Retry sending` for the job to be cleared | Answer clearly that the report has been cleaned up according to retention, the index is still in the database |
| Press `Retry sending` for a job without run | The answer is that the job cannot save any runs, but it does not fail silently |
| History download interrupted midway | Upsert idempotent, run again at the right place; There is no state that looks like it's done but isn't done |
| Historical flaw | Detected by query on table `candles`, counted and reported, not filled with self-created candles |
| The database is so large that the history has to be deleted | The cleanup task does not touch table `candles`. Warning `critical` and waiting for someone to decide |
| Worker died while running | Run and all sublines are written after the replay is complete, so a failed job does not leave a report that looks like it was done |

---

## C11. Test

Same discipline as part A: the decision part is a pure function so it can be tested, the network call part is replaced with a dummy version.

**Menu reducer.** Use table-driven tests where each case is `(state, event, context) -> (new state, effects)`. Include stale-menu buttons, symbols without data, invalid date input, confirming while another job runs, and changing symbols while a position is open.

**Fake Telegram.** A class replacing `fetch` that only records calls. Check: the menu edits old messages but does not send new ones, all `callback_query`s are `answerCallbackQuery`, `callback_data` does not exceed 64 bytes, and no message contains a string similar to an api key.

**Split database.** Worker is only given connection string to `trading_backtest`; Run a complete job and test it without any of the `trading` tables. Edit `strategy_configurations` after the job is completed and then read `backtest_runs` again, the numbers must remain the same.

**Job queue.** Two fake workers who receive the same job must receive two different jobs. The suspended job is marked `failed`. A failed job does not block the next job.

**End to end.** Enqueue a `fixture-single-trade` job on synthetic data, run the real worker, and verify that `backtest_runs` contains a row, all expected report artifacts exist, and `send_document` is emitted with the correct target. The fixture is fast while still exercising the complete path.

**History in database.** Upserting the same candle twice leaves one row. Keyset pagination by `open_timestamp` is lossless and does not duplicate boundary rows. `verify-data-integrity` detects the expected number of missing candles in a deliberately gapped range. Of two concurrent writers for the same symbol/timeframe pair, only one may hold the lock.

**Write a compressed file.** Generate `equity-curve.csv.gz`, decompress it, and compare every CSV row with the source data. Also verify that the four text artifacts remain plain text and that the reader accepts both `.csv.gz` and legacy `.csv` files.

**Cleanup.** Simulate storage usage through an injected function; no real database is required. Below the warning threshold, delete nothing. Above the critical threshold, clean by ascending `finished_at` and stop when usage reaches the warning threshold. Never clean pinned reports. If only pinned reports remain, emit a `critical` warning and stop. After cleanup, `artifacts_deleted_at` is set while the `backtest_runs` row and its original metrics remain.

**Supervisor.** Changing symbols while flat executes all eight steps in order. A switch is rejected while a position is open. After restart, the supervisor reloads `bot_runtime_settings`. If that table is empty and no configuration row has `is_default`, the engine remains `Paused` and does not choose a configuration implicitly.

**Contract quantity.** Test `toContractCount` with the `contractMultiplier` for each of the four contracts and compare it with sandbox `loadMarkets()` metadata. Verify that outbound quantity is expressed in contracts and that quantity returned by `fetchPositions` is converted to base-asset units before PnL calculation. If even one contract exceeds the allowed risk, reject the signal instead of rounding up.

---

## C12. Part C roadmap

| Phase | Content | When will it be finished |
|---|---|---|
| C0 | Finalize the questions in section C13 | Record it in writing, do not make implicit decisions in the code |
| C0b | Remove four environment keys, load four symbols from `loadMarkets()`, add `is_default`, convert contract number | The cold start is correct according to C1.1, and a real order on the sandbox matches the expected volume |
| C0c | Move candle history completely to the `candles` table and remove the file downloader/importer | `sync-history` loads all four symbols, `verify-data-integrity` reports no gaps, and a repeated backtest produces the same metrics and trade count |
| C1 | `control-state.ts` + `menu-view.ts` + test | Reducer goes through all menu branches, not connected to Telegram |
| C2 | Get `callback_query`, build inline keyboard, fake Telegram | `/menu` is visible and clickable, all buttons respond |
| C2b | Read strategies from `strategy_configurations`, use each row's timeframe, enforce the bar-count range in C5.2 | The menu shows only persisted configurations and blocks ranges with too few bars |
| C3 | `bot_runtime_settings` + select symbol and strategy, not applied | Selection saved, survives reboot |
| C4 | `engine-supervisor.service.ts` + apply configuration to paper | Change symbol while running, all eight steps, with warm-up |
| C5 | Database `trading_backtest`, job table, separate worker process | Queue jobs from Telegram, workers run with only one connection and no api keys, engine does not slow down |
| C6 | Progress, results, send build files from database | Receive `summary.txt` on phone, progress does not touch rate limit, 2 year preset runs without being cut off due to time running out, run saves all text and two subtables |
| C6b | Cleanup after 30 days plus two storage watermarks | Simulate a full database: delete oldest artifacts first, preserve pinned runs, and report each deletion |
| C7 | Auditing and edge cases in C10 | Every row in table C10 has a corresponding test |

**Gate of part C: phase C5.** Before having a separate worker process, absolutely do not connect the "run backtest" node to the running engine. One test run with long data is enough to leave the kill switch unresponsive for ten minutes, and that's ten minutes no one wants to experience.

---

## C13. Closed

| # | Decision | Influence on design |
|---|---|---|
| 1 | **Bot messages and documentation are in English.** | All strings in `message-template.ts`, `menu-view.ts`, and project documentation use consistent English trading terminology. |
| 2 | **One operator** | Omit optimistic locking from the first version. Keep `control_audit_log` because it records what changed and when, and retain the `version` column for a future multi-operator workflow. |
| 3 | **One symbol, many strategies** | There is no need for the correlation matrix in section 14.4. The main way is to change strategies on the same symbol, so the eight-step process in C7 must be smooth and flat before changing |
| 4 | **Telegram only selects strategies already in the config** | Menu reading from `strategy_configurations` is `is_enabled`, not reading `strategyRegistry`. Fixtures and automatic benchmarks don't make it to the menu, no need for an exclusion list |
| 5 | **Do not select timeframe on Telegram** | The timeframe belongs to the configuration row. The same idea on another timeframe is **another configuration row** with its own tuned parameters; timeframe selection belongs to strategy optimization |
| 6 | **`reportRetentionDays = 30`, plus storage-based cleanup** | Two independent mechanisms in C6.1: age-based retention and storage watermarks that remove oldest artifacts first without touching pinned runs |
| 7 | **Two-year backtest ceiling** | `maximumBacktestSpanDays = 730`, applies to both presets, "all data" and manual entry |
| 8 | **Reports live in the database**; two CSV files are generated and compressed when sent | Section C6.2. There is no report directory. The equity curve stores one point per hour, producing 17,520 rows over two years; all metrics are calculated from the full series before downsampling |
| 9 | **Whitelist of four KuCoin futures contracts**: BTC, ETH, SOL, AVAX | Section C4.1. Load into `trading_symbols` using script, read specifications from `loadMarkets()`. Following the step of converting contract numbers, AVAX will be rejected during periods of thin liquidity |
| 10 | **Remove `TRADING_SYMBOLS`, `TRIGGER_TIMEFRAME`, `TREND_TIMEFRAME`, `STRATEGY_NAME`** from `.env` | Section C1.1. `.env` retains security and infrastructure settings; the database owns strategy configuration. Cold start relies on `strategy_configurations.is_default`, with no fallback value in `.env`. |
| 11 | **Candle history is in table `candles`**, no more copies on disk | Section C6.3. A single source for coverage, workers, and CLI. In return **Postgres backups become mandatory**: the database is the only copy of history. The cleanup task never touches this table |
| 12 | **Bar range `20 000` to `300 000`** | Section C5.2. For a `1m` strategy, one-year and two-year periods exceed the recommended threshold and intentionally require an additional confirmation step. |
| 13 | **`artifactWarningGigabytes = 2`, `artifactCriticalGigabytes = 5`**, absolute gigabytes, not percentage | Section C6.1. Measured by `pg_total_relation_size` of the three report tables, the actual number of bytes they occupy |
| 14 | **Backtest writes to private database `trading_backtest`** | Section C8.1. There are no cross-database foreign keys, so `backtest_runs` carries a snapshot of parameters and contract specifications. Workers need only one connection and cannot access operational data. |

There are no more items to open. Phase C0 is complete, starting from C0b.

The only two values in this document not yet measured on real hardware are the bar-count thresholds in decision 12 and the estimated runtimes in C5.2. They do not block implementation: both are configuration values, and the first real run in phase C5 will provide evidence for tuning them.
