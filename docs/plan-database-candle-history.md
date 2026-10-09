# Plan D - Database-first candle history

> **Status: completed (2026-08-29).** PostgreSQL is the only source of
> candle history. The folder `data/` was removed together with `download-history`,
> `import-history`, `compress-history`, the `history-file.ts` reader module, and the
> `CANDLE_HISTORY_SOURCE`. There is no remaining file-reader fallback. The sections
> below reflect the final design and no longer describe a transitional period in
> which files and the database operate in parallel.

## 1. Objective

Candle history is in PostgreSQL and is the only source for:

- Backtests keyed by `symbol + strategy + date range`.
- Paper-trading and market-scanner warm-up from historical candles.
- Realtime signal processing on new candles while retaining data across restarts.
- KuCoin candle synchronization for these timeframes:
  `1m, 3m, 5m, 15m, 30m, 1h, 2h, 4h, 1d`.
- The initial bootstrap loads up to three years from the synchronization time.
- After the baseline exists, each daily run downloads and stores only newly missing candles.

For example, a synchronization on 2026-08-28 targets the range from 2023-08-28 through the latest closed candle. Calculate this range dynamically at runtime; do not hard-code dates in the source.

### 1.1 Meaning of "only store today's additions"

"Only store today's additions" means the incremental job downloads new candles after the last stored candle, usually covering the current UTC day. It does not mean deleting three years of history and retaining only one day.

Historical data must remain available for backtesting and warm-up. Each incremental run only `upsert`s new or missing ranges; it does not download the entire three-year range again.

## 2. Acceptance criteria

Migration is completed when all of the following conditions are met:

1. Every symbol in `is_active` has coverage in the database for nine target timeframes.
2. Bootstrap computes a rolling three-year range and stores only closed candles.
3. Rerunning the downloader creates no duplicates and resumes correctly after interruption.
4. The daily job requests only the missing range between the database and the latest closed candle.
5. Closed WebSocket candles enter a database write queue without delaying the signal handler.
6. Paper/realtime warm-up reads the database first; KuCoin REST is only a fallback for repairing gaps.
7. Backtest worker reads candles from database by symbol, timeframe and date range.
8. The same dataset produces stable backtest results across runs: trade count, entry/exit timestamps, PnL, and metrics change only when code or parameters change.
9. Telegram shows coverage, synchronization progress, success, and errors by symbol/timeframe.
10. A market-data database failure must not stop management of open positions; log and report the failure.
11. Once the database became the only reader, deleting raw files was a separate destructive operation requiring manual confirmation. The operator approved and completed it on 2026-08-29 after choosing to redownload history from KuCoin instead of importing files.

## 3. Non-goals of this migration

- Do not change strategy, indicator, entry, stop-loss, or take-profit logic.
- Do not enable live trading from Telegram.
- Do not interpolate missing candles or create synthetic candles.
- Do not add Redis or message broker to the downloader.
- Do not require TimescaleDB initially; add it only if standard PostgreSQL fails measured performance targets.

## 4. Current implementation

### Writers

- `scripts/sync-history.ts` calls `MarketDataSyncService` and is idempotent and resumable. It is the only command for loading history.
- `HistoryDownloadService` is the Telegram adapter for the same synchronization service.
- The WebSocket writer saves each closed candle to the `candles` table.

The old `download-history`, `import-history`, and `compress-history` scripts were deleted because they existed only to write or migrate files, and files are no longer part of the design.

### Readers

- Backtests read through `loadDataSetFromDatabase()` with keyset pagination; the worker, CLI (`scripts/support/backtest-cli.ts`), and batch runner share this path.
- Paper/realtime warm-up reads the database first; REST repairs gaps only through the sync service.
- Telegram coverage screen reads candles, not folders.

### Current data model

The `candles` table has this unique key:

```text
(symbol_id, timeframe, open_timestamp)
```

This key supports idempotent upserts and prevents duplicates. The migration extends metadata and access flow without changing candle identity.

## 5. Data size

A three-year range from 2023-08-28 to 2026-08-28 contains 1,096 days. If all nine timeframes are stored separately, each symbol has approximately:

| Timeframe | Candles / 3 years |
|---|---:|
| `1m` | 1,578,240 |
| `3m` | 526,080 |
| `5m` | 315,648 |
| `15m` | 105,216 |
| `30m` | 52,608 |
| `1h` | 26,304 |
| `2h` | 13,152 |
| `4h` | 6,576 |
| `1d` | 1,096 |
| Total | approximately 2,624,920 |

Storing all nine timeframes adds approximately 2,395 candles per symbol each day.

Benchmark actual storage after loading the first symbol. The initial estimate with numeric columns and indexes is approximately 0.4-0.8 GB per symbol, depending on PostgreSQL storage and index bloat.

## 6. Target architecture

```text
                       +--------------------+
KuCoin REST ---------->| MarketHistorySync  |
                       +----------+---------+
                                  |
                                  | idempotent upsert
                                  v
                        +-------------------+
KuCoin WebSocket ------>| PostgreSQL candles|
                        +---------+---------+
                                  |
                    +-------------+-------------+
                    |                           |
                    v                           v
          Paper/realtime warm-up        Backtest worker
          latest candle window          date-range batches
```

PostgreSQL is the source of truth. REST and WebSocket are ingestion sources; strategies do not read them directly for historical data.

### 6.1 Database permissions

Keep three database connection roles clearly separated:

- `DATABASE_URL`: bot and history sync have the right to read/write operational data and market data.
- `MARKET_DATA_DATABASE_URL`: backtest worker only has permission to `SELECT` on `trading_symbols`, `candles`, `funding_rates` and sync coverage.
- `BACKTEST_DATABASE_URL`: backtest jobs, runs and results.

`MARKET_DATA_DATABASE_URL` may point to the same PostgreSQL database initially, but it must use a read-only role. The backtest worker must not import the complete `DatabaseModule`; `MarketDataDatabaseModule` exposes only the market-data schema and read repositories.

This rule allows workers to read candles without being able to write orders, positions or runtime settings.

## 7. Schema migration

Add an expand-only migration; do not remove existing columns or tables.

### 7.1 Extend the `candles` table

Add these columns:

```sql
source       TEXT NOT NULL DEFAULT 'exchange'
ingested_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
updated_at   TIMESTAMPTZ NOT NULL DEFAULT NOW()
```

The `source` value distinguishes:

- `exchange`: a native KuCoin candle.
- `aggregate`: a candle aggregated from a smaller timeframe.
- `legacy_import`: a row that existed before the migration, if provenance backfill is required.

Keep the existing primary key and lookup index. Add another index only when `EXPLAIN ANALYZE` shows that a backtest query needs it; do not create redundant indexes.

### 7.2 Add the `candle_sync_state` table

Each row represents one `symbol_id + timeframe` pair:

```text
symbol_id
timeframe
target_from_timestamp
target_to_timestamp
stored_from_timestamp
stored_to_timestamp
last_success_at
last_attempt_at
status                 idle | running | partial | succeeded | failed
last_error
missing_candle_count
version
```

The primary key is `(symbol_id, timeframe)`. Use `version` or an advisory lock to prevent two sync writers from processing the same symbol/timeframe pair.

This table supports resumability, Telegram progress, and distinguishing "no data" from "the job has not run yet."

### 7.3 Constraints

- Persist only closed candles: `is_closed = true`.
- Price and volume must be valid before insertion.
- `high >= open/close/low` and `low <= open/close/high` are validated in the service; You can add database checks after the data has been verified.
- All timestamps use UTC, and open timestamps are aligned to their timeframe.

## 8. New configuration

Add environment schema and `.env.example`:

```bash
MARKET_DATA_DATABASE_URL=postgres://market_reader:.../trading
MARKET_HISTORY_YEARS=3
MARKET_DATA_TIMEFRAMES=1m,3m,5m,15m,30m,1h,2h,4h,1d
MARKET_SYNC_ENABLED=true
MARKET_SYNC_HOUR_UTC=0
MARKET_SYNC_MINUTE_UTC=5
MARKET_SYNC_BATCH_SIZE=500
```

`MARKET_HISTORY_YEARS` and timeframe list must be validated at startup. Telegram is not allowed to change these values ​​because this is infrastructure/data policy.

## 9. Download and sync flow

### 9.1 Initial three-year bootstrap

For each active symbol and timeframe:

1. Read the exchange timestamp; do not use the local clock to determine the latest closed candle.
2. Calculate `targetTo` as the open timestamp of the latest closed candle.
3. Calculate `targetFrom` by going back three calendar years in UTC, then align according to the timeframe.
4. Read the coverage currently stored in the database.
5. Split missing ranges into chunks that respect KuCoin's limits.
6. Fetch each chunk, discard open candles, and validate OHLCV.
7. Upsert in batches of 500 rows by default.
8. Update sync state after each batch so the job can resume.
9. When complete, verify duplicates, boundaries, and gaps.
10. Mark `succeeded` when coverage is complete, or `partial` when KuCoin does not return the full three years.

The downloader must be idempotent: rerunning the same range only updates existing rows.

### 9.2 KuCoin history limits

Do not hard-code `earliestServedDate` as fact. Request the target range and record the earliest candle the exchange actually returns.

If KuCoin does not provide three years for a timeframe:

- Do not create synthetic candles.
- Record `partial` and the actual coverage in the database.
- Send a Telegram warning stating the timeframe and earliest available date.
- Block backtest ranges outside the available coverage.
- Add another data provider or archive only in a separately approved scope.

### 9.3 Native timeframe and aggregate fallback

At synchronization startup, validate each timeframe against `exchange.timeframes`.

- If KuCoin supports the timeframe, download native candles with `source=exchange`.
- If KuCoin does not support `3m` or `2h`, aggregate from the smallest available timeframe, preferably `1m`, with `source=aggregate`.
- Create an aggregate candle only when every constituent candle is present.
- If the base timeframe has a gap, leave the target candle missing and report the coverage gap; never aggregate an incomplete bar.

All aggregation must use the shared `CandleAggregatorService` so download and realtime paths cannot produce different candle definitions.

### 9.4 Daily incremental

Job runs every day after 00:05 UTC and also runs a catch-up as soon as the bot starts:

1. Read `MAX(open_timestamp)` for the symbol/timeframe.
2. Start from the next timestamp of the last candle.
3. End at the latest closed candle according to exchange time.
4. If no range is missing, finish without calling KuCoin.
5. If a range is missing, fetch and upsert only that range.
6. Verify gaps in the range that was just synchronized.

The daily job does not redownload three years by default and never deletes historical data.

### 9.5 Realtime persistence

Every closed candle from WebSocket is added to a write queue according to `symbol + timeframe`:

- The signal handler processes the candle immediately; database writes stay off the critical path.
- The queue preserves write order.
- Upserts ensure reconnects or resends do not create duplicates.
- Write failures retry with backoff and trigger a Telegram warning after the configured threshold.
- Daily incremental is the ultimate reconciliation if realtime writes fail or the bot crashes.

## 10. Repository and service changes

### 10.1 `CandleRepository`

Add or correct methods:

```text
upsertClosedMany(symbolId, timeframe, candles, source)
findLatest(symbolId, timeframe, count)
findRangeBatch(symbolId, timeframe, from, to, cursor, limit)
findCoverage(symbolId, timeframe)
findMissingRanges(symbolId, timeframe, from, to)
countRange(symbolId, timeframe, from, to)
```

Do not use an unbounded query to transfer more than one million rows. The backtest reader uses keyset pagination on `open_timestamp`, not offset pagination.

### 10.2 `MarketHistorySyncService`

The new service owns:

- Initial bootstrap.
- Incremental sync.
- Gap repair.
- Native/aggregate decision.
- Sync state and progress callback.
- Lock each symbol/timeframe pair so it has only one writer.

Telegram's `HistoryDownloadService` becomes an adapter that calls this service and does not write files.

### 10.3 CLI

Use one idempotent command instead of the old two-command download/import flow:

```bash
npm run sync-history -- --symbols active --years 3
npm run sync-history -- --symbol "BTC/USDT:USDT" --incremental
npm run verify-data-integrity -- --symbol "BTC/USDT:USDT" --timeframe 1m
```

`download-history` and `import-history` were removed. `verify-data-integrity` no longer accepts `--source` because there is only one source to verify.

Operational note: host-run scripts use the `127.0.0.1:5433` URLs from `.env`.
Docker Compose overrides those URLs with `postgres:5432` for processes running
inside the Compose network.

## 11. Paper and realtime flow

Replace `EngineSupervisorService.loadHistory()`:

1. Calculate the warm-up count of the strategy.
2. Read the latest candles from the database for the trigger and trend timeframes.
3. Verify candle count and gaps.
4. If data is missing, call incremental sync for the exact missing range.
5. Read the database again and load the result into `CandleStoreService`.
6. Start the WebSocket and allow signals only after warm-up succeeds.

REST is no longer a direct engine reader. Only the sync service calls REST to fill the database.

If the database is unavailable:

- A new engine with no open position must pause and notify the operator.
- An engine already managing a position must not crash; WebSocket processing continues to manage exits in memory while persistence retries in the background.
- Do not open new entries unless warm-up and coverage are confirmed.

## 12. Backtest flow

### 12.1 Precheck

Before enqueue or claim job:

- Check coverage for every `strategy.requiredTimeframes` entry.
- The requested date range must be inside the stored range.
- The missing-candle rate must remain below the accepted threshold.
- If coverage is insufficient, reply on Telegram with the first and last available dates.

### 12.2 Worker access

The backtest worker imports the read-only `MarketDataDatabaseModule` and `BacktestDatabaseModule`. It has no exchange API key and no repository that writes operational data.

### 12.3 Dataset loader

Replace `readHistory(rawDirectory, ...)` with a database loader:

- Query by `symbol_id + timeframe + open_timestamp range`.
- Read ordered batches in ascending timestamp order.
- Reject gaps, duplicates, and unfinished candles.
- Apply the in-sample/out-of-sample split only after loading completes.
- Read funding history from the database over the same date range.

In the first pass, the runner may continue receiving arrays to keep the migration scope narrow. Before rollout, benchmark memory with 1.6 million `1m` candles. If it exceeds the memory budget, refactor the runner to stream data in a separate stage rather than mixing that work into the source migration.

## 13. Telegram flow

Update the history menu:

- `Sync 3 years`: run bootstrap for the selected symbol and all nine timeframes.
- `Sync today`: incremental run, usually ends immediately if there is no gap.
- `Coverage`: show first/last candle, count, missing count and status by timeframe.
- Allow only one download at a time to avoid races and exchange rate-limit pressure.
- Progress messages are edited, do not spam new messages.
- Disable the backtest button when required timeframes lack coverage.

Result messages must clearly state coverage:

```text
BTC/USDT:USDT
1m  2025-01-01 -> 2026-08-28  partial
5m  2025-01-01 -> 2026-08-28  partial
1h  2024-10-01 -> 2026-08-28  partial
1d  2023-08-28 -> 2026-08-28  complete
```

## 14. Migration sequence

### Phase D0 - Baseline

- Run typecheck and current test.
- Select a canary symbol, defaulting to `BTC/USDT:USDT`.
- Record count/first/last for the current source and a baseline backtest for comparison after each phase.

### Phase D1 - Expand schema

- Add candle metadata and `candle_sync_state`.
- Add the read-only market-data module and role.
- Add configuration, repository methods, and integration tests.
- Readers still operate.

### Phase D2 - Database writer

- Implement sync service, CLI bootstrap, resume and verifier.
- Bootstrap the canary symbol for all nine timeframes.
- Compare database coverage and counts with sampled requested ranges.
- Bootstrap the remaining active symbols only after the canary passes.

### Phase D3 - Realtime dual write

- WebSocket continues to load candles into RAM and synchronizes the database upsert queue.
- Daily incremental repair.
- Monitor write latency, errors, and gaps for at least seven days.

### Phase D4 - Paper reader switch

- Shadow-read database and REST, compare latest candle windows.
- Switch paper warm-up to the database when the compared windows match.

### Phase D5 - Backtest reader switch

- The worker reads the database through the read-only role.
- Compare datasets, trades, and metrics across equivalent runs.
- Switch Telegram backtests to the database after parity passes.

### Phase D6 - Contract (effective 2026-08-29)

- Delete the file downloader, importer, compressor, and file-reader module.
- Remove `CANDLE_HISTORY_SOURCE` and `DATA_DIRECTORY`; no source-selection branch remains.
- Delete `data/` after the user chooses the method to download from KuCoin instead of import from file.
- Telegram's CLI, worker and screen coverage share the same reader.

## 15. Rollback

There is no file fallback: raw files and the file reader were removed. Rollback now means recovering data, not switching sources:

1. Disable `MARKET_SYNC_ENABLED` to stop the writer and scheduler.
2. Change the warm-up paper to the old REST path if the database market data cannot be read.
3. If data is corrupt or missing, rerun `sync-history` for the exact gap; idempotent upserts make retries safe.
4. If the database is lost, restore the PostgreSQL backup. This is why PostgreSQL backups became mandatory after files were removed.
5. Investigate using sync state and logs, fix the issue, then resume idempotently.

Do not drop columns or tables during a normal rollback.

## 16. Test plan

### Unit tests

- Calculate exactly three calendar years across leap years.
- Align `targetFrom/targetTo` to launch timeframe.
- Accept only closed candles according to the exchange clock.
- Idempotent merge and duplicate replacement.
- Missing-range detection.
- Aggregate only when all constituent candles are present.
- Daily incremental does not fetch when the database is current.
- Resume batch usage after partial failure.

### Database integration tests

- Upserting the same candle twice leaves one row.
- Range pagination neither loses nor duplicates boundary rows.
- Coverage and counts are correct by symbol and timeframe.
- Concurrent sync only one writer is locked.
- Read-only worker role cannot insert/update operational tables.

### End-to-end tests

- Bootstrap the canary symbol, interrupt it, rerun it, and complete successfully.
- Stop the bot for one day, restart it, and catch up exactly the missing candles.
- Confirm database paper warm-up produces the same candle window as REST.
- Confirm two backtests with identical parameters and ranges produce identical trades and metrics.
- Telegram coverage and progress without callback/message limits.

### Performance gates

- Bootstrap stays within exchange rate limits.
- Realtime database writes stay off the signal critical path.
- The latest warm-up query completes within the target of one second.
- A three-year `1m` backtest does not run out of memory and completes within an acceptable time.
- `EXPLAIN ANALYZE` must use composite candle index.

## 17. Observability and operations

Required minimum logs and metrics:

- `market_sync_requested_range`
- `market_sync_received_count`
- `market_sync_upserted_count`
- `market_sync_missing_count`
- `market_sync_duration_ms`
- `market_sync_retry_count`
- `realtime_candle_write_lag_ms`
- `market_data_coverage_start/end`

Alert when:

- Daily sync failed.
- Realtime write queue is longer than two timeframe bars.
- Coverage has a new gap.
- KuCoin returns an earliest candle later than the three-year target.
- Database free space is low.

## 18. Files expected to change during implementation

### Schema/config

- `src/database/schema/market-data.schema.ts`
- `migrations/operational/<next>_database_candle_history.sql`
- `src/configuration/environment.schema.ts`
- `.env.example`
- `docker-compose.yml`

### Market data

- `src/database/repositories/candle.repository.ts`
- `src/market-data/history-download.service.ts`
- `src/market-data/market-data-sync.service.ts` (new)
- `src/market-data/realtime-candle-writer.service.ts` (new if the queue does not fit cleanly in the sync service)
- `src/market-data/market-data-database.module.ts` (new, read-only surface for workers)
- `src/market-data/websocket.service.ts`

### Paper/realtime

- `src/engine/engine-supervisor.service.ts`
- `src/engine/engine.module.ts`
- `src/main.ts`

### Backtest

- `scripts/support/backtest-setup.ts`
- `scripts/support/backtest-database-source.ts` (new)
- `scripts/support/backtest-cli.ts` (new, common for `run-backtest` and `run-batch-backtest`)
- `scripts/run-backtest-worker.ts`
- `src/backtest/backtest-job.service.ts`
- `src/control/data-coverage.service.ts`

### CLI/Telegram/tests

- `scripts/sync-history.ts` (new)
- `scripts/verify-data-integrity.ts`
- Delete: `scripts/download-history.ts`, `scripts/import-history.ts`, `scripts/compress-history.ts`, `scripts/support/history-file.ts`, `src/market-data/history-file.ts`, `src/common/utilities/compressed-csv.ts`
- `package.json`
- `src/control/control-handler.service.ts`
- `src/control/control-state.ts`
- `src/control/menu-view.ts`
- Related unit and integration tests.

## 19. Stop condition

Conditions for stopping at Phase D5 before D6:

- Database is the main reader of paper/realtime and backtest.
- Initial bootstrap and daily incremental synchronization meet acceptance criteria.
- Backtests produce stable results across repeated runs.
- The data recovery path (`sync-history` and Postgres backup) has been tested.

Phase D6—deleting raw data—must never run automatically. It is a separate operational decision. The operator approved it on 2026-08-29 after choosing to redownload history from KuCoin, making PostgreSQL backups the only protection for candle history.
