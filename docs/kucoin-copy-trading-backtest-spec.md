# KuCoin copy-trading history storage and backtest specification

Date: 2026-09-22.
Baseline reviewed: main, commit 704daee.
Status: implementation proposal; no implementation is authorized by this document.
Owner decisions about capital settings remain explicit inputs, not accepted defaults.

## 1. Goal and scope

Goal: persist publicly available KuCoin lead-trader history, then run reproducible
3/6/12-month copy-trading simulations entirely from prepared database data.

This document supersedes the session-only storage, on-demand remote history loading,
and no-persistence clauses of `kucoin-lead-traders-history-spec.md`. Its source access
rules, decimal handling, field meanings and Telegram authorization still apply.

Required behavior:

- Database storage survives restart, Telegram expiry, navigation and trader removal
  from today's leaderboard.
- Initial preparation fetches all history that the source currently exposes for each
  selected KuCoin trader. History is not restricted to the requested backtest window.
- A backtest, history view or export never calls KuCoin, including when data is missing.
- A separate sync operation checks for new records. Existing data is reused; it is
  not downloaded again merely because the user starts another backtest.
- Retain source payloads, normalized fields, acquisition provenance and quality flags.
- Backtests pin immutable input versions and store their configuration and results.
- Run each selected trader independently with its own simulated capital. A comparison
  of several traders is a batch of independent runs, not a combined portfolio.

Out of scope: live copy orders, exchange account login, combined portfolios, automatic
historical top-N rotation, invented individual fills, SL/TP strategies, liquidation
reconstruction, a web viewer, XLSX, Hyperliquid, or changes to existing trading strategies.
No Pine indicator/strategy folder is involved.

## 2. What the result means

Name the first engine `kucoin_closed_position_replay_v1`. Every report must say:

> Approximate copy simulation from published closed-position summaries. Average
> prices and reported leverage can include information unavailable at entry.
> Open positions absent from the source history are not represented. This is not
> evidence that these entries could have been copied live at these prices.

The source supplies aggregate opening/closing timestamps, average prices and a closed
quantity. It does not provide a complete sequence of entry signals, scale-ins,
partial exits, leverage changes, balance changes or equity at entry. One source row
is therefore one synthetic lot in this model, not proof of one original position.
Repeated rows and related partial-close rows can exist; disclose this assumption.

Keep two result groups separate:

1. **Observed trader results:** source-reported PNL for rows closed in the period,
   with counts and validity/coverage disclosures. Do not call this account ROI.
2. **Simulated copy results:** calculated from the user's capital allocation and the
   price replay contract below. Do not scale the source PNL or reuse `pnlRatio` as
   the follower's return. Never subtract estimated fees from source PNL.

Selecting today's top traders and looking backward is a retrospective cohort study.
Store selection time and leaderboard version; do not present it as selecting the
same winners before their results were known.

## 3. Existing integration points

| Current file | Keep / change |
| --- | --- |
| `src/control/kucoin-lead-traders.client.ts` | Reuse access checks, source endpoints, throttling and page validation; expose bounded page reads for the sync service. |
| `src/control/kucoin-lead-trader-report.ts` | Preserve source field semantics; generate history export from DB batches. |
| `src/control/control-state.ts` | Retain only trader/version IDs, page cursor, draft parameters and job IDs. Remove full trader/history arrays from durable-data responsibilities. |
| `src/control/control-handler.service.ts` | Query repositories for browsing; queue sync/replay separately. |
| `src/control/menu-view.ts` | Render a bounded DB page and coverage/status; preserve existing menu conventions. |
| `src/notification/telegram-control-runtime.ts` | Inject repositories/services; stop constructing a history client for ordinary browsing. |
| `src/database/database.module.ts` | Register operational history writer/repository. |
| `src/database/market-data-database.module.ts` | Extend the restricted read connection for published KuCoin datasets only. |
| `src/database/backtest-database.module.ts` | Register dedicated copy-replay job/result tables and repositories. |
| `src/notification/telegram-document.service.ts` | Reuse file delivery. |

Do not force these jobs into the existing strategy-only `backtest_jobs` / `backtest_runs`
shape using fake symbols, timeframe, strategy configuration or candle counts. Their
required fields and existing metrics imply a candle-based equity simulation.
Reuse the queue locking and notification patterns, but keep a typed copy-replay
job/result model. Do not change the behavior of existing backtest workers.

## 4. Storage contract

### 4.1 Database placement

Use the operational database for source history and synchronization metadata, beside
other durable market data. Use `trading_backtest` for copy-replay jobs and results.
There are no foreign keys across databases: a result stores source IDs plus a manifest
hash and the complete input/configuration snapshot needed to identify its dataset.

The replay worker gets SELECT access to published history tables through the existing
restricted read connection and write access to its result database. It must not import
the exchange client or operational order/position repositories. Extend the actual DB
reader-role grants as part of deployment; dependency injection alone is not access control.

Source history is backed up and is not subject to the existing backtest-artifact cleanup.
Do not automatically delete raw history or versions referenced by runs. Restriction or
removal at the source updates access metadata without erasing previously collected public
records. The UI labels those records as archived data, not currently accessible history.

### 4.2 Source tables

Names below are implementation targets. Choose the next free migration number at
implementation time; do not assume today's highest number is still available.

| Table | Required content / keys |
| --- | --- |
| `kucoin_lead_traders` | PK `lead_config_id`; latest known name, source exchange, visibility, first/last seen; preserve traders absent from later lists. |
| `kucoin_leaderboard_snapshots` | Snapshot ID, fetched/completed times, request/sort, source totals, validation status, raw pages/profile summaries. Each membership has rank and source 30-day PNL/ROI/currency. |
| `kucoin_history_sync_runs` | ID, trader, mode, status, lease/fencing token, baseline version, raw profile before/after, timestamps, source counts, progress, failure code, retry metadata. |
| `kucoin_history_source_pages` | Immutable raw JSON payload, request/page metadata, canonical content hash, fetched time; unique `(sync_run_id, request_sequence)`. Successful pages and retry attempts remain distinguishable. |
| `kucoin_history_rows` | Immutable normalized row version with raw-row reference, optional verified source ID, fingerprint, parser version, field validity and all fields below. |
| `kucoin_history_versions` | ID, trader, parent version, sync run, publication time, row count, manifest hash, source observation bounds, full-sweep time, verification scope. |
| `kucoin_history_version_rows` | PK `(version_id, ordinal)`; references row version; contains indexed opened/closed times for range reads and preserves occurrence multiplicity. |

Store these fields without losing precision: symbol, direction, margin mode, leverage,
start/end time, close quantity, multiplier, average entry/close prices, reported margin,
PNL, PNL ratio, currency. Keep source numeric strings in raw JSON and use PostgreSQL
`numeric` plus Decimal for arithmetic; timestamps use UTC millisecond precision.
Invalid normalized values stay nullable with reason codes; raw values remain available.
Preserve unknown fields in raw payloads rather than dropping them during normalization.

Do not fabricate unavailable account equity, original allocation, fees, funding, SL/TP,
liquidation price or execution IDs. Source leverage/margin are reported values, not
necessarily the values at the original entry.

Required indexes: `(lead_config_id, published_at, id)` on versions;
`(version_id, closed_at, ordinal)` and `(version_id, opened_at, ordinal)` on memberships;
unique sync lease per trader; job status/created time indexes. Add a source-ID uniqueness
constraint only if the source actually supplies a verified stable identifier.

A fingerprint identifies content, not a trade. Two identical rows in one source response
remain two occurrences. Reusing one immutable payload is allowed, but two memberships
must survive. Page number and row offset describe provenance, not durable trade identity.
Canonical hashes use a versioned, deterministic JSON serialization. A dataset content
hash covers ordered row contents and occurrence counts, excluding acquisition timestamps,
job IDs and version IDs. Metadata has a separate manifest hash. A no-change sync updates
observation metadata without manufacturing a different history dataset.

### 4.3 Replay storage

Use `kucoin_copy_jobs`, `kucoin_copy_runs`, `kucoin_copy_trades` and
`kucoin_copy_balance_points` in the backtest database.

- Job: owner/user/chat/token fingerprint, idempotency key, pinned history version/hash,
  period, required parameters, status, lease/fencing token, progress and delivery state.
- Run: job ID, model/parser/report versions, code revision, source observation metadata,
  config hash, input digest, coverage decision, counters, metrics and limitations.
- Trade: source occurrence reference, simulated entry/exit, quantity, leverage, reserved
  margin, prices, costs, PNL, status and skip reason. Store every considered row's decision.
- Balance point: event time/order, realized wallet balance, reserved margin and available
  balance. These are not mark-to-market equity points.

Pin actual row membership, not just a query against a mutable `latest` view. A publication
between job creation and execution must not alter the selected input. Final results and
job success are committed together in the result DB; fencing prevents stale workers from
overwriting a newer attempt. Failed attempts must not become successful partial reports.

## 5. Preparation and incremental sync

### 5.1 Initial acquisition

1. Persist the full validated leaderboard and the chosen cohort IDs. Keep `KU` only;
   use the existing numeric 30-day PNL ranking, not a new scoring model.
2. For each selected trader, acquire a DB lease, fetch profile/access, and load every
   history page. Stage bounded batches in DB; never require a complete in-memory array.
3. Retain existing timeouts, response limits and serialized request spacing. Scope the
   outbound queue to the sync worker, not one queue per Telegram user. A page failure,
   changed count, repeated page or changed first page leaves an incomplete sync.
4. Validate counts and recheck access/first page. In one transaction publish the immutable
   version and advance the trader's latest published pointer. Incomplete stages do not
   replace an earlier published version.
5. Record both full-sweep completion and source-retention uncertainty. All pages observed
   is not a lifetime-history or 12-month guarantee.

Resume requires checking that pagination still refers to the same source sequence.
Never blindly continue a saved page number after restart: new records shift pages.
If continuity cannot be established, retain staged data and require a new acquisition
attempt; do not merge inconsistent pages and declare success.

### 5.2 Incremental acquisition contract

The reviewed endpoint supports `currentPage` / `pageSize`. A time cursor and a stable
closed-trade ID are not currently verified. Do not invent `since`, `startTime` or cursor
parameters. Implementation starts with recorded-fixture checks of actual source behavior.

Backtests make **zero source requests**. Sync may read overlapping source rows to locate
the boundary. No automatic full-history scan is triggered by a backtest, menu refresh,
or an ambiguous incremental boundary.

If a verified stable-ID/time-cursor contract becomes available, implement that contract
with an overlap window, idempotent row versions and explicit revision handling. Otherwise
the following conservative page-based protocol is the baseline:

1. Pin the previous published version; fetch profile, counts and newest pages in order.
2. Validate descending close-time order. Read complete equal-timestamp groups at the
   proposed boundary. Match an ordered sequence of row fingerprints and multiplicities
   against the prefix of the stored version; never match on timestamp or one row alone.
3. Require an unambiguous contiguous overlap covering at least two complete timestamp
   groups, and check `new_count + previous_count == source_total`. For a very small
   baseline, compare the entire baseline. Require a matching prefix in two first-page
   observations around the operation. Bound work by the existing operation limits.
4. A new version can reference the unchanged prior suffix and append the newly observed
   occurrences only when these checks pass. Replaying the same response must add zero
   occurrences. Preserve the original suffix observation timestamps.
5. Publish with `verification_scope = head_overlap`, not `full_sweep`. This relies on an
   append-at-head assumption and does not verify older rows were unchanged. Each report
   retains the baseline full-sweep time and this limitation.
6. Ambiguous duplicate sequences, count decreases, unexplained count changes, an altered
   overlap, source ordering changes or no boundary before the deadline produce
   `needs_reconciliation`. Preserve raw staging and the previous usable version; do not
   guess inserts/updates or remove archived rows.

Passing these checks cannot prove absence of arbitrary old source edits. This is a
documented source limitation, not exactly-once exchange event ingestion. If the user
requires source-wide verification, an explicit reconciliation sweep is necessary; it is
separate from normal replay and never silently starts. Such a sweep publishes a fresh
observed snapshot instead of unioning guessed trade identities. Older versions survive.
If a full reconciliation exposes a shorter/pruned source history, retain it as a new
source observation without silently replacing the active archive with the shorter set.
Expose archived versions for replay. Combining a pruned snapshot with older archived
occurrences requires resolved identity evidence; otherwise leave the archive stale and
report the unresolved update instead of inventing a complete merged history.

An unchanged head marks `head_checked_at`; it does not refresh every historical row's
observation time. Record original totals, observed oldest/newest dates and access status
independently. Source pruning must not silently erase the archive.

Sync jobs are durable and idempotent, with one active job per trader. Rate-limit failures
use bounded retries and source Retry-After. An expired lease can be reclaimed with a new
fencing token. The initial scope provides explicit `Prepare history` / `Sync new history`
actions and a CLI; no new recurring scheduler is required for offline backtesting.

## 6. Coverage, quality and period selection

Period presets mean calendar months: subtract 3, 6 or 12 months from an explicit UTC
`as_of` timestamp, clamp invalid month days, and persist the exact `[from, to)` interval.
Do not recalculate `now` when a queued job starts. Display timestamps and history
observation time; do not automatically shorten a requested year to available months.

Keep independent facts:

- acquisition status: full sweep, head overlap, incomplete or restricted;
- observed earliest/latest entry and close, row counts and field-error counts;
- requested-period support: observed span shorter / source coverage unknown;
- freshness: last full sweep, latest head check and latest published version.

Oldest/newest trade dates do not prove continuous coverage: an interval with no trades
can be inactivity or missing source history. Even a full sweep carries retention-unknown
status unless the source supplies a verifiable coverage contract.

Preflight rejects an unpublished dataset, missing version, invalid period/configuration
or a dataset changed relative to its manifest. A limited-data simulation may run only
after the user sees and accepts its stored coverage/quality disclosure. A known short
span must be reported as partial; never label it a complete 12-month result.

Validate separately for each calculation:

- Source-PNL statistics require a valid close time, finite PNL and known currency;
  aggregate currencies separately and never silently convert them.
- Replay requires symbol, LONG/SHORT, valid ordered times (`open < close`), positive
  finite entry/exit prices, positive multiplier, positive closed quantity, valid leverage
  and verified linear USDT-settled contract metadata. Keep unsupported contracts archived.
- Holding-time metrics require ordered times. Missing/invalid fields become explicit
  exclusions, not zero values. `isValid` from the current parser is insufficient.

Preflight also counts rows with missing/unusable timestamps across the pinned dataset;
range predicates must not hide these rows from the quality disclosure simply because
their membership in the requested period cannot be determined.

Only the scope of observed closed summaries is measurable. Unknown still-open source
positions, unreported partial exits and inaccessible records cannot be counted as zero.

## 7. Simulation inputs and financial contract

### 7.1 Capital settings to confirm

The user has not supplied numeric capital/risk settings. These must be explicit required
run inputs; no production defaults are accepted by this spec:

| Input | Requirement |
| --- | --- |
| `initial_balance_usdt` | Finite, greater than zero. |
| `margin_per_position_usdt` | Finite, positive, no greater than initial balance. Fixed synthetic margin per source row. |
| `maximum_leverage` | Positive cap. A source row above the cap is skipped with a reason, not silently resized. |
| `taker_fee_rate` | Explicit nonnegative assumed rate for each side; zero must be intentional. |
| `slippage_bps_per_side` | Explicit nonnegative adverse execution-price adjustment below 10,000 bps. |

Fixed margin is the proposed minimum implementation, pending the owner's sizing choice.
It avoids claiming to know equity while positions are open. The conversational example
of 1,000 USDT and 5% **equity** is not adopted: true equity-based sizing needs additional
historical valuation data and a different contract. If chosen, revise this section before
implementing the sizing engine; source ingestion and DB read paths do not depend on it.
Capital examples in tests are arithmetic fixtures, not accepted trading settings.

One trader per run. Every source row becomes a separate synthetic lot; overlapping long
and short rows are kept separate, not exchange-netted. Report this approximation. Use
the source's reported leverage subject to the configured cap; source margin mode is
displayed, but cross-margin behavior is not reconstructed.

### 7.2 Event inclusion and ordering

For replay, select every known source row with `from <= opened_at < to`, including known
rows that close at or after `to`. Create entry and exit events; process only events before
`to`. An exit exactly at `to` belongs to the next interval.

Rows opened before `from` are excluded as carry-in positions; report their known count.
Rows still open at `to` retain margin and are reported as unresolved. Do not use their
future exit price/PNL within the window or drop them merely because their close is later.
Rows absent from closed-history altogether remain an explicitly unmeasurable exposure.

At the same timestamp: process exits first, then entries ordered by pinned source
occurrence ordinal. This is a deterministic tie convention, not known exchange ordering.
Do not choose entry priority using future PNL, duration, exit price or winner status.

### 7.3 Sizing, reservation and settlement

Use Decimal through sizing, cost and PNL calculation. Persist contract multiplier,
quantity step/minimum and their provenance in the run manifest. If only present-day
contract rules are available, mark their historical applicability as assumed. Missing
contract data produces `unsupported_contract`, never a network request from the runner.

At entry, with balance `B`, reserved margin `R`, desired margin `M`, leverage `L`,
source average entry `P`, multiplier `m`, contract step `step`, slippage `s = bps/10000`:

```text
entry_fill = P * (1 + s) for LONG; P * (1 - s) for SHORT
contracts = floor_to_step(M * L / (entry_fill * m), step)
base_quantity = contracts * m
actual_margin = base_quantity * entry_fill / L
entry_fee = base_quantity * entry_fill * taker_fee_rate
available_balance = B - R
```

If below the minimum quantity or `actual_margin + entry_fee > available_balance`, skip
the entire synthetic lot; no partial fill, queueing or automatic margin shrink. A skipped
entry has no later settlement. Otherwise deduct entry fee from `B` and add actual margin
to `R`. Reserving margin is not itself a loss and does not reduce `B`.

At exit, for source average close `C`:

```text
exit_fill = C * (1 - s) for LONG; C * (1 + s) for SHORT
direction = +1 for LONG; -1 for SHORT
gross_pnl = direction * base_quantity * (exit_fill - entry_fill)
exit_fee = base_quantity * exit_fill * taker_fee_rate
trade_pnl_after_assumed_fees = gross_pnl - entry_fee - exit_fee
B = B + gross_pnl - exit_fee
R = R - actual_margin
```

Slippage is already in fill prices; never subtract it a second time. Report its estimated
cost as attribution relative to the same quantity at source prices, not an additional
debit. Fees are estimates for this synthetic account, independent of source PNL.

Funding, copier profit share, spread beyond configured slippage and actual latency are
not modeled in v1. Store `funding_mode = excluded`, `profit_share_mode = excluded` and
`latency_mode = not_modeled`; their amounts are N/A, not measured zeros. Name the result
`PNL after assumed trading fees; funding/profit share excluded`, not all-in net profit.
Do not reuse the existing 80ms latency setting without market data to price that delay.

There is no intra-position mark-to-market or liquidation model. If observed settlement
would exhaust reserved margin or make balance/available balance negative, flag
`margin_model_breached`, stop new entries, settle already admitted known exits for
diagnostics, and suppress a valid-copy-return conclusion. Do not cap the loss at margin
and label the resulting number exchange-accurate. Even without a detected breach,
intratrade liquidation remains unverified.

### 7.4 Metrics and boundary results

- Source: reported PNL by currency, wins/losses/breakevens, long/short and coin counts,
  holding durations and monthly close-based summaries, each with its eligible-row count.
- Replay: admitted/skipped/closed/unresolved counts, skip reasons, final realized balance,
  reserved/available balance, trading fees, PNL, per-coin/direction/month breakdowns.
- Win rate = wins / closed valid simulated lots, including breakevens in the denominator.
  Profit factor = positive trade PNL / absolute negative trade PNL. No trades -> N/A;
  positive gains and zero losses -> unbounded; all breakeven -> N/A. Store status rather
  than JSON Infinity/NaN.
- Realized-balance drawdown is calculated on entry-fee and exit events, including the
  initial balance point; it must not be named equity drawdown. Do not produce Sharpe,
  Sortino, annualized account ROI, liquidation count or MFE/MAE from absent data.
- A conditional model return `(B - B0) / B0` is shown only if there are no unresolved
  admitted lots and no margin-model breach, with model/coverage disclosures attached.
  Otherwise show realized balance change and unresolved exposure, with total return N/A.
- Monthly accounting includes entry fees in their entry month and exit PNL/fees in the
  exit month. Sum of monthly balance movements must reconcile to `B - B0`. Trade-based
  monthly statistics use close month and are labeled separately.

The same source version, parser/model versions, contract assumptions, event ordering and
configuration must yield identical decisions and Decimal results. Changing today's source
data must not change an already-pinned run.

## 8. Telegram and command-line behavior

Preserve the current `KuCoin Lead Traders` menu entry and English UI language.

1. `List traders` reads the last stored leaderboard. Empty DB shows `Prepare history`.
   Display stored rank and observation time; no automatic source fetch on navigation.
2. Trader detail reads a DB page with profile, stored counts, source visibility, history
   observation span, quality and last sync. Separate `Sync new history`, `History`,
   `Export history CSV` and `Backtest` actions.
3. Backtest chooses 3/6/12 months, displays exact dates and capital/cost inputs, then
   shows preflight disclosures and queues the accepted configuration. No data -> clear
   preparation status and a separate sync action; no implicit download.
4. A separate process executes the durable job. Telegram remains responsive to Stop/Back.
   Completed jobs can be opened after restart. Delivery state is keyed by the existing
   owner/chat/token-fingerprint conventions; unauthorized users cannot retrieve runs.
5. Deliver a concise summary plus CSV details using the existing document sender. Export
   source history and simulated results separately. Result exports include manifest/config,
   source vs simulated fields, admitted/skipped lots, balance points and limitations.

Session state holds IDs, cursors and parameter drafts only. Rendering may use one bounded
page of rows temporarily; the runner may hold active synthetic lots in memory while it
computes. The requirement is no session-owned history archive, not zero working memory.

Provide CLI entry points `sync-kucoin-lead-history`, `run-kucoin-copy-backtest` and
`run-kucoin-copy-worker`. Sync takes explicit trader IDs or a pinned leaderboard selection;
replay takes a published version, period and required financial inputs. Replay has no
`fetch-missing` option. Batch comparison reuses one independent job per trader.

Reuse DB queue claim/locking patterns; unique request IDs prevent repeated Telegram
callbacks from launching duplicate jobs. Recover expired leases safely. Result delivery
is retryable; Telegram delivery is not guaranteed exactly-once across a network timeout.

## 9. Performance and data durability acceptance

- Re-running a prepared dataset performs zero KuCoin requests and zero source-history
  writes. Test with an HTTP implementation that throws on any invocation.
- All history queries filter pinned version + interval using the specified indexes;
  use keyset pagination for views/exports and stream ordered events in bounded batches.
- Do not parse all raw JSON or sort all historical rows on every Telegram page render.
  Raw payloads serve provenance; normalized indexed rows serve computations.
- No result cache, Redis or precomputed aggregate service is needed for v1. Benchmark
  the DB read/replay/export separately on a documented fixture/hardware before adding one.
- Validate 100,000 synthetic rows for bounded source-page memory and stable query plans;
  report measured timings, not an invented latency guarantee. Active-lot memory scales
  with concurrency, not the full historical archive.
- Database restart and bot restart preserve published history and jobs. Backups include
  source/version tables, not only generated reports. Partial sync never hides a good version.

## 10. Implementation plan

Goal: implement prepared-history copy replay with persistent source data and offline runs.
Out of scope: live execution and the features excluded in section 1.

Steps:

1. **Record source contracts and implement page acquisition.** Update
   `src/control/kucoin-lead-traders.client.ts` and its tests for bounded page output,
   raw payload/provenance and field-specific validation. Add recorded fixtures under
   `src/copy-trading/kucoin/fixtures/` for ordering, duplicates, access and revisions.
   Verify actual incremental assumptions before enabling head-overlap publication.
2. **Add durable source schema and access.** Add an operational migration,
   `src/database/schema/kucoin-history.schema.ts`,
   `src/database/repositories/kucoin-history.repository.ts` and its read-only counterpart.
   Register schema/providers in operational and market-data module/token files; implement
   leases, staging, immutable publication, manifest checks and indexed range reads.
3. **Implement acquisition jobs.** Add `src/copy-trading/kucoin/history-sync.service.ts`
   and `scripts/sync-kucoin-lead-history.ts`. Implement bootstrap, conservative incremental
   checks, idempotency and explicit reconciliation; add script entries in `package.json`.
4. **Implement deterministic replay and metrics.** After the owner sizing decision,
   add `src/copy-trading/kucoin/copy-replay.types.ts`, `copy-replay.service.ts` and
   `copy-replay-report.ts`. Implement the period/quality gates and equations in sections
   6-7, with no remote-client dependency and no forced use of candle equity metrics.
5. **Persist jobs/results and run them outside Telegram.** Add a backtest migration,
   `src/database/schema/kucoin-copy.schema.ts`, job/run repositories and registrations in
   the backtest module/token. Add `scripts/run-kucoin-copy-backtest.ts` and
   `scripts/run-kucoin-copy-worker.ts`; preserve existing strategy worker behavior.
6. **Switch browsing/export to DB and wire replay controls.** Update the existing control
   state/handler/view/report and runtime files listed in section 3. Wire new dependencies
   through `src/main.ts` and `src/notification/telegram-paper-runtime.ts`. Add only
   feature-specific progress/result delivery, with account isolation and stale callback guards.
7. **Verify and deploy the complete path.** Apply migrations to test DBs, grant reader
   access, run fixture and DB integration tests, then prepare one public trader dataset
   through the sync path. Disconnect source access and run all three periods, export,
   restart, rerun and compare outputs. Document measured coverage and timing.

Verification: diff review and static traces below for this specification; TypeScript,
fixture, database-integration and manual Telegram checks for implementation. No TradingView
check applies because this feature changes TypeScript/data flows, not Pine scripts.

Each implementation step includes adjacent tests for the behavior it introduces. The
file list above is the intended feature boundary, not permission for unrelated refactoring.
Update the deployment setup only as needed to launch the chosen worker and grant reads.

## 11. Required acceptance scenarios

| Scenario | Expected result |
| --- | --- |
| Bootstrap 100 + 27 rows | Publish all 127 occurrences, raw and normalized; no Telegram-history archive. |
| Same sync repeated | No additional logical occurrences; unchanged published content hash. |
| New rows shift pages | Correct prefix/overlap match or explicit reconciliation status; no page-number dedupe. |
| Identical legitimate rows | Preserve both occurrences and their count across versions. |
| Equal close times across pages | Read complete boundary group; no timestamp-only checkpoint loss. |
| Source revises/deletes an overlap row | Do not guess identity; stage and require reconciliation. |
| Undetected old-tail change after head sync | Report head-only verification and original full-sweep time; never claim full-source freshness. |
| Failure/restart mid-sync | Prior published version still readable; resumptions validate source continuity. |
| Source access becomes hidden | Sync stops; previously collected archive remains labeled with current restriction. |
| Missing PNL, invalid prices or reversed dates | Field-specific exclusions counted; raw row retained. |
| 12-month request with short/unknown coverage | Explicit disclosure and recorded acceptance; no fabricated full-year label. |
| Backtest with network disabled | Succeeds on prepared inputs, including history/export reads. |
| Concurrent sync and replay | Replay remains pinned; result digest unchanged. |
| Long/short with multiplier and fee | Correct sign, quantity, margin reservation and fee treatment. |
| Nonzero slippage and entry/exit fees | Adverse fill-price changes and both fees reconcile to balance; slippage is not debited twice. |
| No free margin / high leverage / unsupported contract | Deterministic skip with no later exit settlement. |
| Entry before start / close at or beyond end | Carry-in exclusion / unresolved lot; future prices never settle in-period PNL. |
| Equal-time exit and entry | Exit releases margin first, then deterministic source-order admission. |
| No trades, breakeven, no losses | Defined N/A/unbounded handling, no NaN/Infinity serialization. |
| Loss exhausts margin | Model breach flag, no new entries, no valid-copy-return conclusion. |
| Restart after completed job | Same report remains available; no refetch or financial rerun on export. |
| Cross-account request for job | Rejected using owner and runtime authorization. |

Static trace fixtures (not production risk defaults):

1. B0=1,000, M=50, L=5, zero costs, LONG 100 -> 110, multiplier=1 and
   quantity step=0.1. Quantity=2.5, reserve=50, balance stays 1,000 at entry;
   close PNL=25, balance=1,025, reserve=0. Matching SHORT 100 -> 90 also earns 25.
2. B0=100, M=50, L=1, zero costs, two simultaneous admitted lots reserve 100;
   a third entry is skipped. One lot closes +5 before another equal-time entry;
   balance=105, reserve=50, available=55, so that new 50-margin entry is admitted.
3. A known lot opens one day before `to` and closes one day after it. Charge only its
   entry fee within the period, retain reserved margin, and report unresolved exposure.
   Do not count its future winner/loss, release its margin or publish total account return.

## 12. Decisions before implementation

- Confirm capital/sizing policy and actual run values. Fixed margin is the proposal;
  equity-percentage sizing requires a revised valuation contract. No implicit 1,000/5%
  production preset is approved.
- Accept the published-summary approximation and its missing live signals, open-position,
  funding/profit-share and liquidation information. A causal live-copy backtest needs a
  different historical event dataset; persistent DB storage alone cannot create it.
- Validate source ordering/identity behavior with recorded evidence. If the conservative
  incremental boundary cannot be established, bootstrap/offline replay can ship, while
  incremental publication stays disabled with an explicit status; do not silently fall
  back to downloading all history on each run.

These decisions constrain the sizing/source adapters. They do not block creating this
specification or preparing the independent storage and offline-query implementation plan.
