# London Open Liquidity and Macro Strategy V3 — recorded-news confirmation

> Research/backtest only, including the bot's Backtest menu. V2, its paper worker and its historical reports remain unchanged.
> This is a testable hypothesis, not evidence that news trading is profitable.

## Contract

- Identifier: `london-open-liquidity-macro-v3`; 5m decisions and immediate next 1m execution.
- Retain V2's London morning branch, sizing and 12:00 UTC exit. Add one independent New York news-Short opportunity per symbol and New York trading day.
- A news item is eligible only when it is explicitly classified as high-impact US macro, has `live_first_seen` timing, is not historical backfill, and has a real `firstSeenAt` timestamp. A scheduled calendar event, publication timestamp alone, or a retrospectively imported headline is **not** proof that the bot knew the news. The current RSS feed is CLARITY Act focused; it is not a complete macro feed.
- Eligible news must first be recorded between 08:00 and 10:00 `America/New_York` on a weekday. Confirmation must close by 10:30 New York time. These are V3 research assumptions, not LB-approved live settings. DST is resolved using the IANA time zone, not a fixed UTC offset.
- Freeze the pre-news reference at the last completed 5m candle before `firstSeenAt`: its close and the minimum low of the preceding six completed, contiguous 5m candles. The pre-news reference must be fresh, and post-news 5m candles must remain contiguous through confirmation. A signal candle must **open at or after** `firstSeenAt`; a bar already in progress when news arrived cannot qualify.
- **Liquidity break:** a completed 5m close below the frozen six-bar low. **Weakness:** two consecutive post-news bearish 5m candles with falling closes, the second closing below the frozen pre-news close. Either confirms a Short. The news item alone never sends an order.
- Enter at the first 1m open after the confirming 5m close; missing that immediate 1m candle means no trade. Stop is the highest high from the first post-news 5m candle through confirmation plus `0.1 × ATR(14)`. At the actual 1m fill, recalculate fixed-notional quantity and the 2R target below that fill; reject the entry if the fill invalidates the stop or produces a nonpositive target. Exit any remaining New York position at 12:00 New York time.
- No Long liquidation is inferred from a news headline. “Sell” here means an **independent Short entry when flat**, not a forced sale of a V2 Long. One New York Short maximum per symbol/day; an existing position or pending entry blocks it.
- Record one New York decision row per evaluated weekday with New York market coverage, including `no_recorded_news`, `no_confirmation`, `data_gap`, `execution_data_missing`, `risk_rejected`, `trade_open`, or `trade_entered`. Record the event ID, first-seen time, frozen liquidity level, confirmation rule and decision time when applicable.

## Data and comparison

The CLI accepts an immutable JSON array through `--recorded-news-file`. Each item includes `eventId`, `title`, `firstSeenAt` (Unix milliseconds), `timingQuality`, `isHistoricalBackfill`, `region`, `impact`, and `category`; the run label includes a short SHA-256 hash of that file.

The bot Backtest worker instead reads `news_events` through its read-only market-data connection, joined to explicit `news_event_classifications`. It selects only `live_first_seen`, non-backfilled events classified `US` / `high` / `macro`; a title match or calendar schedule alone cannot add a classification. There are currently no classified macro records, so menu runs can compare the unchanged London branch but will report `no_recorded_news` for the New York branch until a verified macro feed and classification process supplies events. Zero events is **not** evidence that no macro release occurred. Do not interpret an empty-feed V3 run as validation of the news hypothesis.

Compare V2 and V3 on the same price period, costs and symbol specification. Report morning V2 trades separately from V3 New York news trades; do not claim improved frequency or P&L until the recorded-event coverage and out-of-sample comparison are checked.
