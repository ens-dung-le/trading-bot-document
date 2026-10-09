# Hyperliquid trading history — implementation spec

Updated: 2026-10-08. Status: implemented and integrated on 2026-10-08; automated checks and deployed HTTP smoke checks are recorded below. Live watchlists start empty until the operator adds addresses.

Goal: show current perpetual positions and reconstructed Long/Short history for persisted watched Hyperliquid wallets, with Telegram wallet selection and profit/loss summaries for 1 week, 1/3/6 months and 1 year.
Out of scope: automatic trading, copy trading, pending orders/order lifecycle, alerts, leaderboard reranking, cohort consensus, XLSX, paid providers and archive imports.
Steps for this documentation task:
1. Write this specification in `docs/hyperliquid-trading-history-spec.md`, covering data, schedules, reconstruction, UI and acceptance.
2. Link it from the earlier top-30 monitoring proposal and state which scope governs this implementation.
Verification: diff review of these two documents; static traces below verify Long/Short episode semantics. No Pine indicator/strategy folder is involved.

## 1. Scope and watchlist

The screenshots are layout references, not requests to implement their Alert, Compare Trader, Backtest or Copy Trader controls. The dashboard comprises wallet summary, PnL chart/statistics, Open positions and History. Both sections support Long and Short; direction and profit/loss use independent labels/colors (a profitable Short is green PnL).

Use a durable watchlist, independent of daily leaderboard changes. Provide add-address, optional label, pause/resume and list controls on the web page. Default network is mainnet. Validate 0x + 40 hexadecimal characters and normalize case. Re-adding a wallet resumes the existing record without duplicating its history. Pausing stops scheduling new fetches and retains records; in-flight results may finish, but show the wallet as paused. Resume repairs from committed cursors and explicitly records unrecoverable gaps.

The current Telegram top-30 selection can provide addresses for an explicit initial seed/import. It must not silently replace the watchlist, import on every page visit, or change its month-PnL ranking. No automatic seed is required. Each wallet/subaccount is a distinct account; do not merge presumed owners. Runtime means time since tracking began, not account age.

Target initial capacity: 30 watched wallets. Supported instrument identity is `(network, dex, coin)`, including discoverable supported HIP-3 perpetual venues. Discover venues and validate their state/fill/funding coverage. Surface unavailable venues individually; do not claim whole-wallet coverage if some are missing. Spot trades are excluded from perpetual history.

## 2. Sources and API validation gate

Use read-only public `POST https://api.hyperliquid.xyz/info`; no signing key or trading endpoint.

| Source | Use |
| --- | --- |
| `perpDexs`, venue-specific `meta` | Discover venues and instrument identity |
| `clearinghouseState`, correct `dex` | Authoritative current positions, margin/account scope |
| `userFillsByTime`, `aggregateByTime: false` | Executions for historical and incremental ingestion |
| `userFunding` | Funding payments with source timestamps |
| Account-mode/spot state endpoints when needed | Correctly scoped current equity for unified accounts |

Historical leverage is not generally supplied by fills. Store observed position-state leverage with its observation timestamp. Show historical leverage only when evidence supports it, with provenance; otherwise null. Never infer it from notional or assign today's leverage to old trades. Liquidation price is a current-state field, not a historical exit price.

The earlier research dated 2026-09-21 records REST allowance 1,200 weighted units/minute, `clearinghouseState` weight 2, history page limit 2,000, fills availability limited to the latest 10,000, and at most 10 unique users across user-specific WebSocket subscriptions per IP. These are planning assumptions requiring revalidation against current official documentation and permitted response samples before coding source contracts. Initial HTML documentation retrieval on 2026-10-08 returned HTTP 403. During implementation preparation, the official Markdown pages (append `.md`) were accessible and revalidated these rate/fill limits; a read-only `perpDexs` call also succeeded. Endpoint-specific fixtures and pagination still require acceptance checks.

Validate response schemas, application errors inside HTTP 200, endpoint venue scope, ordering/pagination, fill identity, fee/funding sign and currency, and account-mode valuation. Record permitted samples as sanitized fixtures. Do not choose a speculative alternate endpoint when a required capability fails; mark that capability unavailable and resolve its contract before acceptance. This release uses REST polling, with no realtime guarantee or WebSocket provider requirement.

## 3. Synchronization contract

| Task | Default cadence |
| --- | --- |
| Current positions for active wallets/venues | Every 30 seconds |
| New fills | Every 60 seconds |
| Funding | Every 5 minutes |
| Initial historical import | Once on first watch, bounded to available API history |
| History reconciliation | Daily 07:10 Asia/Ho_Chi_Minh and startup/resume recovery |
| Venue/account-mode discovery | Startup and daily; recover newly discovered venues |
| Browser refresh while visible | Every 15 seconds, database reads only |

Use a single weighted limiter across these tasks, bounded concurrency (initial maximum four requests), jitter and non-overlapping per-wallet/source tasks. Start with at most 720 weighted units/minute, adjust down for other processes on the same IP, and account for result-size weights. Honor Retry-After/429 with bounded exponential backoff. Live positions and incremental fills take priority over initial imports. If 30-wallet/venue load cannot meet cadence within verified limits, show actual delay and overdue work rather than exceeding limits. Cadences are targets, not guarantees.

Acquire a single worker leadership lease/advisory lock and release it on shutdown/connection loss. Commit page records and cursor/covered range atomically. Restart must replay safely. Bound job duration and use a fair queue so a busy wallet cannot starve others. A wallet/source failure does not cancel successful wallets.

Initial import has a fixed upper cutoff. Pagination retains the last timestamp inclusively and deduplicates immutable execution IDs. Split full time windows where supported; never blindly advance one millisecond. If all records at a saturated timestamp cannot be enumerated, record a gap and stop that range. Each incremental/reconciliation run uses a fixed end time and a 10-minute overlap from its committed cursor, not a complete redownload. Daily reconciliation reviews since the previous successful daily cutoff with overlap, including explicitly pending recoverable gaps. Late records trigger revision of affected episodes/statistics.

Capture current position state immediately even while historical import is incomplete. Store only validated complete wallet/venue responses as authoritative replacement sets. A successful empty set means flat for that venue; timeout, malformed or partial data never means flat. Retain the last valid state on failure. Reject older poll generations so delayed responses cannot overwrite newer accepted state. Snapshot timestamp is source timestamp when available; also persist request/receive times and their provenance.

Save each successful account/venue snapshot at the 30-second cadence initially. Position changes are retained; no retention/deletion policy is introduced. Monitor storage size during acceptance. Browser reads never trigger source downloads.

## 4. Coverage and freshness

Store independent fill and funding cursors/intervals per wallet and applicable source scope; venue coverage must be explicit even if an endpoint returns all venues together. States: `importing`, `verified_interval`, `partial`, `api_window_limited`, `gap`, `unavailable`, plus scheduling state `active/paused`.

Expose earliest/latest covered times, known missing ranges, last success/error and reason. Empty successful intervals are valid evidence; failures are not covered intervals. A newest fill timestamp alone does not prove coverage. Detect possible source truncation; when the boundary cannot be proven, mark completeness unknown. Never claim lifetime history or assume the wallet was flat before the earliest available fill.

Current state older than 90 seconds is stale; retain values with an explicit timestamp. Fills overdue by 180 seconds and funding overdue by 15 minutes are degraded. Paused wallets display paused state regardless of elapsed thresholds. An unavailable venue makes aggregate status partial. UI and API carry these statuses; absence of rows cannot stand in for zero activity or zero PnL.

## 5. Storage contracts

Use the operational PostgreSQL and existing repository/worker conventions; external positions must never enter bot order/execution tables. Use exact `numeric` and decimal strings/`decimal.js`, UTC timestamps, string-safe source IDs and raw payload evidence. Tables:

| Table | Essential fields/key |
| --- | --- |
| `hl_watched_wallets` | network/address unique, label, tracking state, tracked_since, created/updated |
| `hl_fills` | wallet + validated immutable execution identity, dex/coin, tid/oid/hash, time, side/dir, startPosition, quantity, price, closedPnl, signed fee, fee currency, raw payload |
| `hl_funding` | wallet + validated source funding identity, dex/coin, time, signed amount/currency, raw payload |
| `hl_position_snapshots` | wallet/venue, source/receive timestamps, poll generation, validity, full account envelope/positions |
| `hl_current_positions` | wallet/dex/coin unique, signed size, entry, notional, leverage/type, margin, liquidation, unrealized PnL, snapshot reference |
| `hl_position_episodes` | stable episode ID, wallet/dex/coin, Long/Short, opening/closing evidence, open/closed/incomplete, opening/closing time, weighted prices, cumulative opened/closed quantity/notional, peak size, realized PnL/fees/funding, coverage, derivation version/revision |
| `hl_episode_fills` | episode + fill + allocation part, opening/closing quantity, allocated fee and closedPnl; supports a flip affecting two episodes |
| `hl_sync_state` | wallet/source scope, committed cursors, covered intervals/gaps, job status/errors, last success and freshness |

Validate fill key using source samples; prefer immutable execution ID scoped to wallet/instrument. Transaction hash alone is insufficient for partial fills. Reject/quarantine conflicting payloads for an existing identity rather than silently replacing facts. Funding deduplication needs its own validated identity. Index wallet/instrument/time and deterministic history ordering. Allocate no migration number until implementation; choose the next free sequence then.

Episodes/statistics are derived evidence, not source truth. Publish recalculations transactionally, preserving stable episode links where evidence is unchanged. Position details and chart/history reads bind to the same data revision. Funding assignments derive from actual position ownership at event time; ambiguous boundary events remain unallocated and partial, never duplicated across episodes.

## 6. Long/Short reconstruction

Order fills by source event time and validated execution ordering (not lexical string-ID order). Use `startPosition` and signed executed delta: Buy is positive, Sell negative. Buy may close a Short; Sell may close a Long. Validate resulting position and source direction/closedPnl instead of inferring direction from Buy/Sell alone. Ambiguous ordering or continuity mismatches create a gap/incomplete episode until repaired.

An episode is one continuous nonzero position of one sign. Zero -> positive opens Long; zero -> negative opens Short. Same-sign increases/decreases remain in that episode. Reaching zero closes it. Crossing zero splits the fill at zero into a close and a new opening with the opposite sign. Allocate the flip fee proportionally to closed/opened executed quantity; its source closedPnl belongs to the closing episode. Funding is independent. Liquidation executions participate with their source provenance; do not invent liquidation events from missing position rows.

Initial nonzero `startPosition` creates an incomplete episode with unknown opening time and opening cost. Record observed realized amounts but exclude it from complete-episode win/loss counts. Preserve current authoritative entry separately from reconstructed episode entry. A snapshot disappearance cannot fabricate an exit price/closedPnl: show current flat, with pending history repair for the unfinished episode.

Historical display entry is quantity-weighted average of all opening/increase fills in the episode, explicitly labeled as such; exit is quantity-weighted average of closing/reduction fills. It is not necessarily the final exchange average entry. Historical Size is total opening notional (sum opening quantity × fill price), not current or peak size; detail exposes those separately. Current Size shows absolute base quantity and snapshot notional USD. Preserve null for unknown fields.

Static traces / required fixtures:

| Executions | Expected result |
| --- | --- |
| Buy 2 from 0; Sell 1 | One open Long, current quantity +1; partial close not a completed trade |
| Sell 2 from 0; Buy 1; Buy 1 | One closed Short, opened quantity 2; two reductions, one completed trade |
| Long +2; Sell 3 | Close Long 2; open Short 1; closing/opening fee shares 2/3 and 1/3 |
| Short -2; Buy 3 | Close Short 2; open Long 1 with the same allocation rule |
| First observed startPosition -5; Buy 2 | Incomplete Short with remaining -3; opening time/entry unknown |
| Position poll fails | Existing current state remains, later becomes stale; no fabricated closure |

## 7. PnL, chart and statistics

Define wallet window statistics over event timestamps within `[start,end)`; do not sum whole episodes merely because their close occurs in the window. Required windows are `1w`, `1m`, `3m`, `6m`, `1y`; web may also show All available. Default selected window is `1m`. All available means stored covered history, never lifetime by default.

Freeze one UTC `asOf` for a summary bundle. `1w` is the preceding seven days; month/year windows subtract 1/3/6 calendar months or 1 calendar year in UTC, clamping the day to the last valid day in the target month and preserving time. Thus 2026-10-08 05:00 UTC gives starts 2026-10-01, 2026-09-08, 2026-07-08, 2026-04-08 and 2025-10-08, each at 05:00 UTC. A March 31 monthly start clamps to February 28/29. These are rolling periods, not completed calendar months. Display the exact interval in Vietnam time; do not mix rolling UTC windows and local midnight cutoffs.

Compute all five windows from the same committed data revision/asOf, using the same calculation function in bot and web. Persisted raw events remain authoritative; indexed event queries are sufficient initially, without introducing a separate aggregation worker. Any later cache must include wallet, window bounds, filters, calculation version and data revision in its key.

Main chart/headline: cumulative **net realized trading PnL** over the selected window = source closedPnl - signed trading fees. Gains = sum positive gross closedPnl; losses = sum negative gross closedPnl; show them as gross and show fees separately, including signed rebates. Volume = sum absolute fill quantity × price, counting opening and closing executions once each. All values use the same window/filter/revision.

Funding is a separate signed metric. Also show net realized including funding only when currency conversion and funding coverage support it; otherwise null/partial with reason. Source non-USD fees/funding require verified timestamped conversion; preserve original currencies and mark USD totals partial until resolved. Do not fabricate conversions. Each complete episode shows gross closedPnl, allocated fees, funding and net values with evidence/coverage.

Chart is a realized-event curve, not equity, unrealized PnL, or cash-flow-adjusted return. No deposit/withdrawal ledger import or ROI/drawdown is needed for this chart. Current unrealized PnL is displayed separately from realized statistics. Historical entry-to-exit performance should use source closedPnl rather than recomputing a guessed cost basis.

Optional source fields remain null when unavailable. Current Equity shows a clearly defined account scope after account-mode validation; do not sum repeated unified collateral across venues or label equity as available balance. Last trade is latest confirmed fill; Runtime is tracked duration. No unsupported lifetime account metric appears.

## 8. Web/API behavior

Reuse `apps/backtest-report-viewer` with `/hyperliquid` and wallet detail `/hyperliquid/[address]`, server-side calls to the private Hyperliquid worker API backed by operational PostgreSQL. Telegram and the worker API use the same statistics/repository implementation, avoiding duplicated PnL logic. The private API requires a server-only bearer key and explicit scope; browser requests are authenticated through the viewer and never receive that key. The wallet selector shows label/address, tracking/coverage and last update. The watchlist selects one wallet for the detail dashboard. Monetary summaries and positions/history are scoped to that selected wallet; there is no implicit all-wallet monetary aggregation or netting across addresses/venues.

Open positions columns: wallet (aggregate view), age or unknown, L/Long or S/Short, venue/coin, current average entry, base quantity + USD notional, leverage/type, liquidation price or —, unrealized PnL, update status. Detail shows margin, last accepted snapshot and episode/fill links. Sort defaults to notional descending with stable identity tie-breaker.

History displays confirmed CLOSED positions only, newest closed time first with stable ID tie-breaker. A closure requires execution evidence reaching exact zero. Unclosed incomplete fragments never appear. Confirmed closures with missing opening evidence retain unknown opening time/entry and partial coverage. Columns: wallet, closed time (explicit header), direction/coin/venue, opening-average entry, total opening notional, observed historical leverage or —, net realized trading PnL/coverage. Detail shows open/close timestamps, exit average, quantities, gross/fees/funding, holding duration and completeness reasons; raw fills remain internal source evidence. Open episodes appear in the Open panel, with partial realized PnL in detail; their events still contribute to wallet realized statistics.

Filters: wallet, venue/coin, All/Long/Short, selected time window. History filter uses confirmed closed time; wallet statistics use event time as defined above. Label this distinction. Changing a table-only sort does not alter statistics. The selected period applies to chart/statistics. Direction/coin/venue controls inside History are explicitly table-only filters and do not change wallet-level chart/statistics; no hidden direction filter is applied to unknown/unallocated facts.

Read API namespace `/api/hyperliquid`: wallets, wallet summary, positions, closed position summaries, position detail and chart/statistics. Return decimal strings, UTC timestamps, asOf/revision, coverage and freshness. Use server-side cursor pagination, default 40/max 100 rows, whitelisted sorts and stable tie-breakers; fix a revision/cutoff across pages and require refresh if no longer available. Reject invalid addresses/query bounds; parameterize queries. Distinguish loading, truly empty, importing, failed, stale and paused states. Hidden pages stop refreshing; resume reads on focus. Mobile preserves access to all fields through horizontal scroll/detail.

The web feature is a tab/button inside the existing Backtest Report Viewer, with navigation back to backtest; no separate web application is created. The existing Telegram top-30 List/Export behavior remains unchanged; add a separate watched-wallet history action as specified below. Source credentials/DB URLs stay server-side. Use a separate read-only database role for reads; explicit watchlist write endpoints use a narrow write role and the same authenticated/private access boundary. Validate request origin/CSRF for writes and input limits; protect both page and APIs before exposing the viewer. If the viewer has no existing authentication, keep it private behind the deployment access boundary and document enforcement; do not ship public watchlist writes by default.

### 8.1 Telegram wallet/history flow

Reuse the existing Hyperliquid menu and control state/effect/session patterns:

`Main menu -> Hyperliquid -> Wallet history -> select watched wallet -> summary -> select period`

The selection list includes all watched wallets, including flat/paused/importing wallets; current top-30 List only includes active positions and is therefore not a suitable history selector. Paginate five wallets per page and show label/short address plus state. Each wallet has an internal selection callback and a separate Hypurrscan URL; clicking the selection must open the bot summary, not redirect to an explorer.

Store selected wallet ID, period, list page and summary bundle ID in the user's existing control session. Use compact opaque IDs in callback data within Telegram's 64-byte limit; resolve to canonical wallet identity server-side. Reuse menu versions/session ownership checks, reject expired/unauthorized callbacks and escape labels/addresses. Wallet read permissions follow the existing allowed-user/runtime boundary; do not introduce unauthenticated access or silently share another runtime's watchlist.

Summary content:

- Wallet label/address, chosen interval and data-as-of timestamp.
- Net realized trading PnL, gross gains, gross losses and signed fees.
- Signed funding and net including funding when covered; otherwise explicit unavailable/partial status.
- Executed volume, distinct perpetual instruments traded in the interval, and last confirmed trade.
- Current Long count, Short count and current unrealized PnL, labeled current/asOf rather than historical period performance; mark partial if any venue is stale/unavailable.
- Requested vs covered interval, known gaps and freshness. Never render a missing metric as zero.

Buttons: `1 tuần | 1 tháng`, `3 tháng | 6 tháng | 1 năm`, `So sánh các kỳ`, `Xem biểu đồ & history`, `Làm mới`, `Chọn ví khác`, `Quay lại`. Default shows 1 month. Compare renders the five windows with net trading PnL and per-window completeness from one bundle; details remain in the selected-window view. Keep messages within Telegram's size limit and edit the existing menu through the established service rather than flooding chat.

Period changes and Compare read the frozen bundle, making cross-window values comparable. Refresh creates a new bundle from committed DB state; it never requests Hyper directly, reranks traders or queues a full import. Bind bundles to user/runtime, expire after 15 minutes and on session expiry; expired actions ask to refresh. In-memory bundle loss on restart requires refresh, while financial history stays durable in DB. Expose collecting/empty/error states rather than falling back to leaderboard PnL.

The chart/history button opens the existing planned viewer's canonical wallet route with whitelisted `period`, `asOf` and `revision` parameters, using an absolute configured HTTPS viewer URL and preserving its access controls. Do not put credentials or bearer tokens in the URL. If a revision expires or newer repairs prevent rendering it, require an explicit refresh and explain the change. Viewer uses the same interval/calculation contract. If no viewer URL is configured, explain that chart view is unavailable while keeping the bot summary usable. This release delivers the summary inside Telegram and the full chart/history layout in the web viewer; generating or sending chart images is not required.

### 8.2 Availability of 3/6/12-month history

Required time buttons do not imply that the public API can retrieve that much history for every wallet. A high-activity wallet may exhaust the latest-fill window within days. For each window, return `requestedStart/End`, verified coverage intervals/gaps, observed amounts and `complete/partial/unavailable` for fills, funding and currency conversion independently. A partial amount is labeled "observed PnL / incomplete period", never presented as the definitive period total. Complete covered windows with no events may correctly show zero.

Onboarding first imports available public history, records the boundary, and retains incremental facts thereafter. No annual total extrapolated from recent trades, leaderboard month PnL or currently open positions is allowed. Source portfolio PnL series are not a substitute for this fill-based trading PnL definition. To guarantee older complete periods immediately, a separate archive/indexer data source is necessary: first validate wallet/venue filtering, coverage dates, execution identity, fees/funding, continuity and cost. No paid service or archive download is included or authorized by this spec. Historical gaps must be repaired before a period is relabeled complete; prospective accumulation alone cannot repair past missing ranges.

## 9. Ordered implementation plan and files

Goal: deliver the watched-wallet Long/Short dashboard and Telegram multi-period profit/loss summary defined here.
Out of scope: items listed at the top; no indicator, strategy or KuCoin edits.
Steps:
1. Add `migrations/operational/<next>_hyperliquid_history.sql`, `src/hyperliquid/source.types.ts` and `hyperliquid.repository.ts` with focused tests for schema, identities and atomic cursor commits.
2. Add `src/hyperliquid/hyperliquid-info.client.ts` and source fixtures/tests for validated venue/account state, fill/funding pagination and weighted requests; reuse existing Decimal/address parsing conventions without unrelated control-file refactors.
3. Add `src/hyperliquid/history-sync.service.ts` and `position-episodes.ts` with tests for coverage, idempotency, late repairs, both directions and flip allocation.
4. Add `src/hyperliquid/worker.ts`, `hyperliquid.configuration.ts`, `scripts/run-hyperliquid-worker.ts` and its `package.json` command; implement cadence, leadership, priorities, restart and shutdown tests. Deployment configuration changes only where required to run this worker.
5. Add `src/hyperliquid/statistics.ts` with hand-calculated event-window tests; add viewer server queries/operational DB configuration, `/api/hyperliquid/` read/write routes and `/hyperliquid/` pages/components with focused read/filter/access tests.
6. Update `src/control/control-state.ts`, `menu-view.ts`, `control-handler.service.ts` and focused existing tests to add watched-wallet selection, owned summary bundles, five period buttons, Compare and viewer link; use the same statistics/repository service as the web and preserve top-30 List/Export.
7. Document worker/viewer configuration and run acceptance in relevant existing README files, covering permitted live samples, Telegram summaries and visual verification of both Long and Short.
Verification: static traces in section 6, diff review limited to the listed Hyperliquid/integration paths, focused unit/database tests, bot/viewer typechecks, viewer build and manual UI/live-source checks below. This is a new non-Pine feature spanning backend/web, not an indicator tweak.

## 10. Acceptance checklist

- Telegram wallet selection includes flat/paused wallets; explorer links remain separate. Two users' selections/bundles do not interfere; stale/expired callbacks cannot select a different wallet.
- All five required periods use one asOf/revision; bot/web totals match decimal-exactly. Test leap year, month-end clamp, UTC/Vietnam display, and events exactly at start (included) or end (excluded).
- Static period trace: for asOf 2026-10-08 05:00 UTC, net event +100 on September 20 and -40 on October 5 yields 1w=-40 and 1m=+60 with full coverage; with only October coverage, the monthly observed -40 is partial, never a complete monthly result.
- Switching periods/Compare makes no exchange call. Refresh reads DB only; viewer opens the selected wallet/window, and missing URL or expired revision has an explicit state.
- One-year/three-month buttons remain usable with partial coverage labels; unavailable amounts are not zero, recent portfolio/leaderboard PnL is not substituted, and no provider purchase/import happens implicitly.
- Long and Short have equal field/filter coverage; closing Buy never mislabeled opening Long, and closing Sell never mislabeled opening Short.
- Complete/partial closes and both flip directions match the traces; allocations conserve quantity, fees and source realized PnL without double-counting volume.
- Replaying pages or restarting after commit leaves counts/PnL unchanged; crash before commit replays both data and cursor safely.
- Same-timestamp saturation, source-window truncation, first nonzero startPosition and late conflicting events cannot produce false complete history.
- Successful empty venue state removes its positions; timeout/partial failure preserves them. Old responses cannot overwrite newer state; missing fills leave history repair pending.
- Funding deduplication/assignment and non-USD amounts expose partialness when uncertain; historical leverage and opening age remain unknown without evidence.
- One wallet failure does not block the rest. Actual 30-wallet/venue workload stays inside revalidated request limits; cadence misses and API errors are visible.
- Daily reconciliation and restart/resume recovery execute from durable state; paused wallets retain history. An observation run covers restart, rate-limit handling and one daily cycle (24 hours for live acceptance).
- Chart/statistics reconcile to stored event facts for each range; example closedPnl +100/-30 and fees +5/-1 gives gain 100, loss -30, signed fees 4, trading net 66; signed funding -2 gives including-funding net 64 when covered.
- History cursor pages have no duplicates/skips within a fixed revision; repaired data requires consistent refresh. User-controlled text is escaped; invalid filters/access are rejected.
- Browser refresh only reads DB, watchlist changes persist across restart, no signing/private trading action occurs. Existing backtest viewer and Telegram List/Export still pass their relevant checks.
- Desktop/mobile visually verify Short with positive PnL, Long with negative PnL, empty/importing/stale/partial states, detail rows and pagination. Never use color alone to identify direction.

## 11. References

- Earlier broader proposal: [Hyperliquid top-30 monitoring](hyperliquid-top30-trader-monitor-plan.md). This specification governs watched-wallet positions/history implementation; daily cohorts, streaming, consensus and exports from that proposal are not implicit requirements here.
- [Official info endpoint](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint)
- [Official rate/user limits](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits)
- Existing snapshot client: `src/control/hyperliquid-leaderboard.client.ts`; current report store is in-memory and is not durable history storage.

## 12. Implementation and current verification record

Work split across `codex/hyperliquid-data`, `codex/hyperliquid-telegram`,
`codex/hyperliquid-web` and `codex/hyperliquid-integration` managed worktrees.
Existing uncommitted repository work was copied into isolated baseline snapshots
and preserved during integration; local secrets were not committed.

The deployed worker uses adaptive weighted REST scheduling: native/active
positions target 30s, inactive HIP-3 venues 5min, fills 60s and funding 5min,
scaled under the shared budget. Live discovery found 11 venues, so the original
all-venues/30-wallet/30s target cannot fit the official quota. Stale/error/partial
labels expose actual state rather than implying realtime coverage. Equity and
historical leverage remain unavailable where no validated evidence exists.

Verification on 2026-10-08: 1,770 bot/backend tests passed (17 unrelated tests
skipped), including dedicated PostgreSQL, immutable event replay/rollback,
history continuation/truncation, real localhost HTTP authentication/revision
and database-to-Telegram/API PnL reconciliation. Viewer: 23 tests passed, bot
and viewer typechecks/builds passed. Sanitized populated live-source fixtures
confirm actual position/fill/funding parsing. Migration 0085 applied; bot,
worker and existing viewer restarted. HTTP smoke checks cover existing
backtest health, private page/API access and rejected invalid/cross-origin
writes; the Docker Host/origin mismatch found during deployment was corrected
with a regression test.

Not yet verified: a 24-hour observation with real operator watchlist addresses
and desktop/mobile interaction through a browser. Computer-use connectors
were unavailable (native service startup failure/Chrome auth token unavailable);
component rendering tests and deployed HTTP checks passed, but do not constitute
a manual visual check. No provider/archive was purchased or imported.

Login follow-up (2026-10-08): embedded browsers may not present HTTP Basic
challenges. Added a real login page using the existing credentials and signed
8-hour HttpOnly sessions, with direct API authorization rechecks and same-origin
form validation. Viewer suite now has 28 passing tests; deployed HTTP checks
verified page redirect, form login, private cookie API reads and anonymous API
rejection. Backtest pages remain accessible as before.

### Closed-position materialization correction (2026-10-09)

Migration `0086_hyperliquid_position_history.sql` adds `hl_position_history` for finalized closed summaries and `hl_history_materialization` for per-wallet/venue/coin source fingerprint, calculation version and evidence. Bot/web history and detail read these stored summaries without reconstructing episodes. Repeated unchanged syncs and restarts skip reconstruction. New or corrected fills/funding rebuild only the affected instrument tail, preserving finalized unaffected prefixes transactionally. Startup backfills existing watched wallets from stored facts.

Fills at the same millisecond are ordered using exact `startPosition` → signed resulting-position chains; `tid` is not chronological. Ambiguous chains retain a coverage gap instead of manufacturing positions. Each history row shows Long/Short, open/close timestamps, weighted entry/exit, quantity/notional, holding duration, gross PnL, fees, funding/net and completeness. No raw-fill list is shown. Open positions remain in their separate panel. Wallet period statistics continue to use event timestamps.

### History recovery and precise margin/date display (2026-10-09)

Do not let non-perpetual outcome/spot settlements invalidate a perpetual fill page. Historical backfill failures must not block retrieval of recent perpetual executions; retain explicit coverage gaps and recoverable cursors. Validate both Long and Short closed positions across native and HIP-3 instruments against stored source executions.

UI dates use full year and UTC+07 (Asia/Ho_Chi_Minh). Position opening means the first execution opening from zero; closing means the execution returning to zero (or the closing portion of a flip). These are execution dates, not order submission dates; never relabel them as order creation without order-source evidence. Current collateral displays exchange `marginUsed`, labeled position margin: for cross margin this is not isolated assigned collateral or wallet equity. Historical leverage/margin must have snapshot evidence within that episode, with observation time/type; absent evidence displays unavailable, never inferred from current leverage or notional.
