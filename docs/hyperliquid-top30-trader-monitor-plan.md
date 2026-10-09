# Hyperliquid top-30 trader monitoring plan

Date: 2026-09-21.
Status: proposed plan for review; implementation and paid infrastructure are not authorized.

Scope update (2026-10-08): the current watched-wallet Long/Short positions and
trading-history implementation is specified in
[Hyperliquid trading history](hyperliquid-trading-history-spec.md). That spec
uses durable watchlists and REST polling. The broader daily-cohort, streaming,
consensus and XLSX proposal below remains separate and is not an implicit
requirement of the trading-history implementation.

Goal: persist a daily top-30 wallet cohort and its trade history, calculate wallet performance daily, list currently positioned cohort wallets in Telegram, and export their coin direction, concentration and entry-price summary to XLSX, with the web viewer available for detailed browsing.

Out of scope: automatic trading, changing bot strategies, copy trading, KuCoin changes, a full in-chat coin analytics dashboard, alerts, and continuous leaderboard reranking.

Steps for this documentation task:
1. Create this document in `docs/` with data contracts, schedules, storage, and ordered implementation steps.
2. Review the document against the current worker/database patterns and official Hyperliquid limits.

Verification: diff review of this document only; implementation acceptance checks appear below. No Pine indicator or strategy folder is involved.

## 1. Agreed behavior and explicit defaults

| Work | Frequency | Result |
| --- | --- | --- |
| Download leaderboard and select top 30 | Once per daily cycle | Immutable leaderboard snapshot and cohort membership in PostgreSQL |
| Initial historical import for a new wallet | Once, then recover missing ranges | Available fills, funding, cash movements, and coverage metadata |
| History reconciliation | Daily | Append missing records; do not redownload all stored history |
| Wallet performance | Daily after reconciliation | Versioned daily metrics calculated from persisted data |
| Open positions | Stream while a wallet is in the active cohort | Current Long/Short exposure and position snapshots |
| New orders, updates, cancellations and fills | Stream while in active cohort | Order intent separately from executed trades |
| Cohort coin aggregates | On accepted live-state changes | Direction counts, gross/net exposure and separate Long/Short entry prices |
| Web summary, coin and wallet details | On opening or pressing Refresh; summary may refresh every 15 seconds while visible | Read persisted snapshots without refreshing the leaderboard or downloading history |
| Telegram List | User presses List traders | Freeze a report snapshot and list only cohort wallets with verified open perpetual positions, linked to Hypurrscan |
| Telegram Export | User presses Export XLSX after List | Send a workbook covering the entire listed snapshot, not only the visible page |
| Web entry | User presses Open web viewer | Open the configured viewer URL independently of List/Export |

Default schedule: 07:10 Asia/Ho_Chi_Minh, after the UTC daily boundary. The
performance day is the preceding `[00:00, 24:00)` UTC interval; timestamps in
storage are UTC. This is a proposed implementation default, not an exchange
requirement. Store the reporting timezone and cutoff with each daily run.

Daily history reconciliation complements immediate persistence of new fills.
Waiting until tomorrow to save fills risks losing them from the API window on
busy wallets. No continuous requests for old history or leaderboard updates
are required; bounded reconnect repairs are allowed.

Keep history and daily performance for every wallet ever admitted to a cohort.
Only the active 30 receive live subscriptions. Former members get daily history
reconciliation and metrics while their records remain tracked; their growing
daily workload must stay within the shared request budget and report overdue
work explicitly. Do not silently delete or stop tracking them when they leave.

## 2. Define the group before calculating consensus

Default selection for review:

1. Fetch and store the full public leaderboard once, including original ranks,
   address, account value, and day/week/month/all-time performance.
2. Sort numerically by the source's **month PNL**, descending, tie-breaking by
   normalized wallet address. No custom score and no intraday reranking.
3. Inspect candidates in that order. Exclude identified agent addresses and
   vaults from this directional-trader cohort. Accept user accounts or resolved
   subaccounts with verifiable perpetual activity: a current perp position or
   a confirmed perp fill in the preceding seven days.
4. Skip unverifiable candidates with a recorded reason; keep searching in source
   rank order until 30 eligible accounts are found. Show fewer than 30 if the
   verification budget is exhausted; never fill the group with guessed traders.
5. Freeze membership until the next successful daily cohort activation.

This is **top eligible accounts by leaderboard month PNL**, not a claim to the
30 highest perpetual-only performers on Hyperliquid. The general leaderboard
can mix account activity; preserve its metric scope. Store `portfolio` perp
series separately where supplied. Local recalculated performance is a separate
output and does not silently replace the selection rule.

Do not scan/enrich tens of thousands of profiles with expensive API requests
just to recalculate a global ranking. If the user later wants ranking by local
net perp PNL or return, that is a selection-rule revision with explicit coverage.

An account is not a proven independent person or directional trader. Market
makers and hedgers can have real fills. Record role, parent/master links and
observed behavior; do not claim ownership or off-platform exposure is known.
For known master/subaccount families, retain only the highest-ranked eligible
account in the voting cohort and record the excluded linkage. Do not sum a
master and its subaccounts implicitly. Unlinked wallets may still share an owner.

Identity is `(network, address)`. Instruments are `(network, dex, coin)`; identical
tickers across venues/contracts are not automatically the same instrument.
Include supported HyperCore perpetual venues, including HIP-3. Spot balances
are used for account valuation when needed, not counted as Long perp votes.

## 3. Source contract and the 30-wallet streaming constraint

REST: `POST https://api.hyperliquid.xyz/info`.
WebSocket: `wss://api.hyperliquid.xyz/ws`.
Leaderboard feed observed working:
`https://stats-data.hyperliquid.xyz/Mainnet/leaderboard`.
The leaderboard feed is a website data source; validate its schema and freshness
rather than treating it as a documented, guaranteed API.

| Purpose | API / subscription |
| --- | --- |
| Role and parent links | `userRole`, `subAccounts`, `vaultDetails` when applicable |
| Venue discovery | `perpDexs`, venue-specific `meta` |
| Account valuation mode | `userAbstraction`, `spotClearinghouseState`, perp margin state |
| Source performance reference | `portfolio` |
| Historical executed trades | `userFillsByTime`, `aggregateByTime: false` |
| Funding / external cash movements | `userFunding`, `userNonFundingLedgerUpdates` |
| Position baseline | `clearinghouseState` with the correct `dex` |
| Order baseline | `openOrders` or `frontendOpenOrders`, with the correct `dex` |
| Positions live | `allDexsClearinghouseState`, or validated venue-specific state subscriptions |
| Executions live | `userFills` |
| Order lifecycle live | `orderUpdates`; order snapshots repair the current open-order set |

### Verified limits on 2026-09-21

Official limits apply per IP, not independently to each socket:

- REST: 1,200 weighted units per minute.
- At most 10 WebSocket connections and 1,000 subscriptions.
- **At most 10 unique users across user-specific WebSocket subscriptions.**
- At most 30 new WebSocket connections/minute and 2,000 outgoing WS messages/minute.
- `clearinghouseState` weight 2; `userRole` weight 60; most other documented
  info requests weight 20. Large history responses add weight per returned items.
- `userFills`: at most 2,000 recent fills. `userFillsByTime`: at most 2,000 per
  response and access only to the 10,000 most recent fills.
- `historicalOrders`: at most 2,000 recent orders; it is not an unlimited replay log.

Therefore **30-wallet full realtime is not supported by the public user-stream
API on one IP**. Three sockets do not remove the 10-user limit. Do not rotate
subscriptions and call the result continuous coverage; do not evade the limit
through proxy/IP rotation.

Recommended target: a legitimate streaming provider or an approved expanded
service that explicitly supports at least 30 wallet addresses and the required
venues, order lifecycle, fills, position states, replay and timestamps. No vendor
has been verified or purchased in this research. A node/indexer is an alternative
only after confirming it exposes the required events and state, and estimating
its operational cost; it is not assumed to be a ready-made replacement.

The first implementation step must validate this source. Until it passes, do
not promise or roll out 30-wallet realtime. Daily history/database work can
proceed independently after implementation approval.

Optional fallback requiring a product decision: stream at most 10 wallets and
poll the remaining wallets with explicit `polled` status. This cannot guarantee
every newly placed/cancelled order; orders may appear and disappear between
polls. It is not equivalent to the requested full realtime scope.

## 4. History import and coverage

### Initial import

- On first admission, store the wallet and current baselines, then start its
  stream and historical import concurrently with deduplication. A large history
  import must not prevent collecting new events.
- Request the oldest available fills through a fixed upper cutoff. Page using
  source timestamps and immutable execution identities; persist raw payloads.
- Keep the last timestamp inclusive and deduplicate. Do not blindly add one
  millisecond, which can skip fills sharing that timestamp. Split bounded time
  windows when possible. If a full page at one timestamp prevents progress,
  record a gap and use a supported archive/indexer instead of looping or
  claiming completeness.
- Import funding, ledger movements and available source equity/PNL series with
  independent cursors. A fill cursor does not prove ledger coverage.
- Detect the 10,000-fill boundary. Label it `api_window_limited`, not lifetime
  history. Never infer a flat starting position because earlier fills are absent.

For history outside the API window, official docs reference
`s3://hl-mainnet-node-data/node_fills_by_block` and older node datasets. Add an
archive-source adapter only if needed and approved, with date/address filtering,
source provenance and schema checks. Retrieval can incur data-transfer costs;
do not download an unbounded archive or claim it is automatically complete.
No paid archive/provider use is authorized by this document.

### Daily reconciliation

- For each tracked wallet/source, start from its committed watermark with a
  small overlap, suggested 10 minutes. End at the fixed daily cutoff.
- Reconcile REST records with already-stored live fills; do not insert another
  trade simply because it came from a second transport.
- Commit validated pages and watermarks together. A crash before commit must
  safely replay the page. Store empty covered ranges separately from failures.
- A missing interval that has fallen outside the API window remains a known
  gap until repaired from an adequate source. Current positions alone cannot
  recover missed historical fills or cancelled order intentions.
- Late records trigger a new revision of affected daily metrics, not a silent
  change to an already published immutable calculation.

Coverage states: `verified_interval`, `partial`, `api_window_limited`, `gap`,
`unavailable`. "Verified interval" is bounded to a source and time range, never
an unconditional whole-account history claim.

## 5. PostgreSQL storage

Use the existing operational PostgreSQL and repository/worker pattern.
Use a dedicated `hl_` table prefix. Do not write these external positions into
the bot's execution/position tables. This is independent of the KuCoin feature.

| Table | Key and essential content |
| --- | --- |
| `hl_wallets` | `(network,address)`; role, known parent, first-seen/last-checked, tracking state |
| `hl_daily_runs` | `(network,report_date,run_kind,revision)`; cutoff, source time, status, raw leaderboard snapshot, errors |
| `hl_cohort_members` | `(cohort_id,address)`; source/eligible rank, metric scope/value, selection reason, activation interval |
| `hl_fills` | Wallet-scoped source execution identity; dex/coin, tid, oid, hash, event time, side, dir, startPosition, size, price, closedPnl, fee/currency, raw payload |
| `hl_ledger_events` | Wallet + source event identity; funding/transfers/deposits/withdrawals, currency, amount, time, provenance |
| `hl_orders` | `(network,address,dex,oid)`; current status, original/remaining size, side, reduce-only/trigger attributes, latest source version |
| `hl_order_events` | Wallet/order + event identity/version; immutable status transitions and raw payload |
| `hl_position_snapshots` | Snapshot ID, wallet/dex, source/receive time, connection generation, complete/partial coverage, account-state envelope and positions |
| `hl_current_positions` | `(network,address,dex,coin)`; latest position from a validated complete baseline/state sequence |
| `hl_account_points` | Wallet + source scope + timestamp; valuation mode, equity, source PNL, raw portfolio/account point |
| `hl_sync_state` | Wallet/source/venue; watermark, coverage intervals, failures, connection/gap state |
| `hl_wallet_daily_metrics` | Wallet/date/window/calculation version/revision; realized/net PNL, equity/return, sample counts and coverage |
| `hl_report_snapshots` | Opaque report ID; runtime/user ownership, immutable list/export bundle, cohort/source IDs, creation/expiry and schema version |
| `hl_cohort_coin_metrics` | Cohort/instrument/calculation timestamp; long/short counts, notionals, average/median entries, coverage, recent position changes |

Use PostgreSQL exact `numeric` and `decimal.js` for money/quantities. Preserve
source IDs as strings or database big integers, not unsafe JavaScript numbers.
Normalize addresses, preserve original instrument identifiers, and index wallet
and event time for history queries. Keep raw evidence alongside normalized data.

Fill deduplication must be validated against the chosen source: prefer its
immutable execution ID scoped to wallet and instrument (`tid` in the official
fill schema). Do not use transaction hash alone; one transaction/order can
contain many partial fills. Duplicate REST/WS events upsert idempotently;
conflicting payloads for the same execution identity require reconciliation.

Position snapshots store whole validated account/venue envelopes so a known
empty set can be distinguished from an omitted venue. Never zero all positions
because a partial message, timeout or parse failure returned nothing.

Persist changes to position quantity/entry/leverage and regular checkpoints;
coalesce pure mark/PNL updates to at most one snapshot per wallet/dex/minute.
Keep latest accepted state current. Persist executed trades and order lifecycle
events without this coalescing. Track storage growth; no automatic history
deletion is part of this plan.

## 6. Daily performance calculations

Store both reported source performance and locally calculated performance with
explicitly different labels. Do not replace local metrics with leaderboard ROI
when local history is incomplete.

Compute daily and trailing 7/30-day metrics from PostgreSQL:

- Executed perpetual volume, fill count, active instruments and activity dates.
- Gross realized PNL from source `closedPnl`.
- Net realized PNL = gross realized PNL - signed trading fees + signed funding,
  after verifying source sign conventions. Rebates and charges keep their signs.
- Mark-to-market PNL includes change in unrealized PNL; realized PNL alone is
  not complete performance for a wallet with open positions.
- Account PNL check: ending equity - beginning equity - net external inflows,
  for matching account scope/currency and valid boundary valuations.
- Completed position episodes, win rate, profit factor and holding time only
  for episodes whose opening and closing history is actually observed.
- Sampled drawdown and cash-flow-adjusted return only when valid valuation and
  cash-flow coverage supports them; otherwise store `null` and the reason.

Entry/exit fills must distinguish increasing, reducing, closing and flipping a
position. A flip closes the old episode and opens a new one; partial closes are
not separate wins. Do not count 100 partial fills as 100 completed trades.
Persist the derivation version and reprocess affected episodes after repairs.

Account modes matter: unified/portfolio-margin account balances need the
appropriate spot/account valuation data. Do not sum repeated account equity
across perp venues or add collateral twice. Convert non-USD fees/collateral only
with timestamped prices; missing conversions make the affected metric partial.

For return, prefer time-weighted subperiod returns when valuations at external
cash-flow boundaries exist. Daily endpoints alone do not support exact TWR with
intraday deposits. A separately labeled Modified Dietz estimate is allowed only
with complete timestamped external flows and a valid positive denominator.
Do not publish infinite ROI/profit factor or fabricate zero when undefined.

The nightly result records `report_date`, `as_of`, `calculated_at`, source scope,
data interval, coverage status, sample counts, calculation version and revision.
Existing historical equity/PNL series have sampling limits; label drawdown as
sampled, not maximum intraday drawdown unless sufficient observations exist.

## 7. Realtime lifecycle and recovery

1. Load the active cohort and durable watermarks from PostgreSQL at startup.
2. Establish supported subscriptions, buffer incoming events, acquire position
   and open-order baselines, then reconcile the overlap. Use provider sequence
   numbers/snapshot boundaries if available. Without them, re-fetch and compare
   baselines before declaring the state current; do not assume REST and WS
   form one atomic snapshot.
3. Store order updates and fills independently. Buy can close a Short; Sell can
   close a Long. An unfilled order is intention, never an exposure vote.
4. Use position state as the authority for current exposure, fills to explain
   changes, and order events to describe pending intentions.
5. Reject older state versions; retain valid late historical events without
   reverting current positions. Use connection generations to invalidate old
   callbacks after reconnect.
6. Reconnect with backoff/jitter, re-subscribe, repair fills since the committed
   cursor and refresh full position/open-order baselines. A connected socket
   does not prove an individual wallet's state is synchronized.
7. Track heartbeat, source/receive timestamps, baseline time and coverage per
   wallet/venue. Suggested health rule: after 30 seconds without required
   transport liveness, mark affected streams degraded; a quiet wallet need not
   trade every 30 seconds. Reconcile baselines at least every five minutes,
   staggered and within the global budget.

When a complete newer snapshot removes a position, record the observed absence;
confirm the close event/time from fills where available. Never invent an exact
close time from the polling/receipt time.

At cohort change, subscribe/bootstrap entrants before publishing the new
cohort if source capacity permits overlap. Capacity must cover the overlap or
use an explicit degraded handoff; do not claim uninterrupted coverage during
an unavoidable unsubscribe/bootstrap interval. Leaving the cohort is a
membership change, not a trade closure. Keep the old cohort if activation fails.

Use one independent worker with a reserved-connection PostgreSQL advisory lock,
following the existing news-impact worker pattern. Abort work when leadership
is lost. The daily job must not block live ingestion or Telegram/execution.
No exchange signing key or order-placement capability is needed.

## 8. Coin consensus and entry-price outputs

For each instrument and cohort, include only valid current states, and expose:

- Selected wallet count, observed fresh wallet count, and wallets holding the
  instrument. Always show these denominators alongside Long/Short votes.
- `long_count`, `short_count`, `flat_count`, `unknown_count` with defined scope.
- Long and Short gross notionals, plus net notional.
- Long vote share = `long_count / (long_count + short_count)` when nonzero.
- Capital bias = `(long_notional - short_notional) / (long_notional + short_notional)`.
- Separate Long/Short weighted entry prices:
  `sum(abs(base_size) * entry_price) / sum(abs(base_size))`.
- Separate median wallet entry prices so a single whale does not define the
  apparent shared entry zone. These are open-position cost bases, not desired
  future entry prices or average prices across all historical trades.
- Instrument concentration by gross cohort notional and by wallet participation.
- New/increased/reduced/closed/flipped exposure over 15m, 1h and 24h, based on
  quantity/fills, not notional changes caused only by market price movement.

Keep order intentions in separate fields with status, size and reduce-only
semantics. They never increase Long/Short holdings or executed volume until filled.

Only publish a complete-cohort label if all required wallet/venue states are
healthy. Otherwise publish observed values with `partial` and counts; do not
extrapolate stale/unknown wallets into the missing group. Never label 2/2 Long
as 100% agreement among 30 traders.

Rolling changes use a stable membership set. At a daily cohort rollover,
bootstrap entrant state but mark pre-admission live windows unavailable unless
reconstructed with adequate coverage. Do not interpret a changing roster as
capital entering/leaving a coin. Persist cohort ID on all derived outputs.

### 8.1. Web viewer with a Telegram launch button

The web viewer remains available for detailed browsing. The Telegram List and
Export flow in section 8.2 works independently and does not require a deployed
web viewer. A full in-chat coin analytics dashboard is not required. Extend the
existing Next.js application in `apps/backtest-report-viewer/` with a `/hyperliquid` page; preserve
its existing backtest pages and visual conventions. Do not create another web
application or implement the same analytics screens inside Telegram.

Web flow: `Telegram main menu -> Hyperliquid Top 30 -> Open web viewer -> web
summary -> coin or wallet detail -> Back`. The separate List/Export flow is
defined in section 8.2.

#### Summary table

Display the cohort's instruments in a readable desktop table and responsive
mobile layout with these columns:

- Coin and venue, keeping distinct contracts separate.
- Number of Long and Short wallets, participation and unknown counts.
- Total Long and Short position values with explicit currency/units.
- Separate weighted average Long and Short entry prices.
- Recent exposure changes for `15m`, `1h`, or `24h`; default `1h`. Describe
  increased/reduced/opened/closed positions with counts, not a buy/sell command.

The header shows cohort date/ID, snapshot time, selected window, fresh-wallet
coverage and source mode (streamed, polled or degraded). Missing values show
`N/A` with a reason; zero exposure is not a substitute for missing data.

Default sort is gross position value descending, with instrument ID as a
stable tie-breaker. Use 25 rows per page, server-side pagination and consistent
snapshot IDs across pages. Provide Refresh and the three window selectors.
The summary's first page may refresh from the database every 15 seconds while
the page is visible; pause background-tab requests and refresh on return.
Freeze snapshots while browsing later pages or details to prevent rows jumping
between pages. Explicit Refresh loads a new snapshot and resets pagination.
No browser request starts leaderboard/history jobs or calls exchange APIs.

#### Coin and wallet details

Selecting a coin shows all participating wallets, paginated at 25 per page:
address, Long/Short, size/value, average entry, reported leverage, unrealized
PNL and freshness. Show participation/unknown totals and separate Long/Short
median entry prices. Preserve coin/venue identity and the summary snapshot.

A separate Details action opens a wallet's latest persisted daily performance (with report
date, metric scope and coverage), current position snapshot, and stored executed
trade history. History is paginated at 50 records per page with a stable cursor;
include instrument, source time, direction when verified, quantity, price,
realized PNL and fees when present. Explain fill records versus derived position
episodes; do not present each partial fill as a separate completed position.
Show the stored history's time range and gaps, not a lifetime-history promise.
Do not fetch additional remote history merely because a user opens this page.

Use shortened addresses in tables with access to the full address and the
Hypurrscan link `https://hypurrscan.io/address/{address}`. Back restores the prior
selection/page/snapshot. An expired snapshot requires Refresh instead of
combining old totals with new wallet details.
Daily metrics and live positions may have different timestamps; show both.

#### Telegram entry and web access

Reuse an existing Hyperliquid menu entry when present; otherwise add one
`Hyperliquid Top 30` button opening the action panel from section 8.2. Its
`Open web viewer` action uses a normal Telegram inline URL button, not a new
Telegram Mini App or Web App authentication flow. The menu button model and
keyboard serializer currently support callback data only: extend them to a
callback-or-URL union, preserving existing actions and enforcing exactly one
button action. The 64-byte callback limit does not apply to URL button targets.

Configure an absolute viewer URL such as `https://<viewer-host>/hyperliquid` on
the server. If it is missing or invalid, explain that the viewer is not configured
rather than emit a broken link. A phone cannot open the server's localhost URL.
Reuse the current Telegram authorization and never place bot tokens, database
credentials or reusable login secrets in the URL. A forwarded URL is not proof
of authorization to the viewer: enforce access independently at the web layer.

Extend the viewer's existing server-side query pattern with a separate
Hyperliquid read-only DB connection/role limited to the necessary `hl_` tables
or views. Keep database credentials off the browser. Protect both the page and
its API routes using the deployment's authenticated reverse proxy/session;
if none exists, establishing it is a release prerequisite. Do not assume the
existing local viewer already provides authentication.

The existing viewer README keeps its Compose deployment separate. Complete
its documented integration review for authentication, reverse proxying and
read-only DB access before exposing a reachable viewer; this plan does not
authorize publishing the service or adding it to root Compose automatically.

Distinguish no cohort yet, unavailable worker/database, stale/partial snapshot,
and a valid flat cohort. Keep the last valid snapshot visibly stale after an
error instead of rendering zeroes. No automatic Telegram pushes or trade
controls belong to this viewer. XLSX export is in scope through the separate
Telegram action below; additional charting is out of scope.

### 8.2. Telegram List first, separate XLSX export

This is the required primary bot flow for the user's latest request:

```text
Main menu -> Hyperliquid Top 30
               [List traders] [Export XLSX]
               [Open web viewer] [Back]

List traders -> wallets currently holding positions (paginated)
                wallet address -> Hypurrscan
                [Previous] [Next] [Refresh list] [Export XLSX] [Back]
```

`List traders` and `Export XLSX` are separate callback actions. Before a valid
list snapshot exists for this authorized user/runtime, Export is disabled or
answers `List traders first.` It must not quietly create a new unseen list.
Listing alone never generates or sends a spreadsheet. Opening a wallet never
starts an export or enables copying/trading.

#### Which wallets appear

- Use the fixed daily cohort of at most 30, preserving their cohort ranks.
- Display a wallet only if a valid, fresh position state shows at least one
  nonzero perpetual position. A pending order, spot holding, past fill or high
  historical PNL alone does not qualify as currently in a position.
- List five wallets per page; provide a clickable address, daily cohort rank,
  currently held coins/directions, observed gross position value, and source
  timestamp/coverage. If a wallet has too many coins, show a short preview and
  label the remaining count; all positions remain available in the workbook.
- Use `https://hypurrscan.io/address/{address}` for every clickable wallet
  address, including workbook links. Validate a 42-character hexadecimal
  address before constructing this fixed-host URL. Use the full address in
  the workbook; shortened Telegram labels must not hide the link destination.
- Do not replace flat wallets with accounts outside the daily top 30. For
  example, 12 positioned wallets in the cohort means a list of 12, not a newly
  selected top 30. A holder whose entry was days ago still qualifies.
- Distinguish confirmed active, confirmed flat, and unknown/incomplete wallets.
  A wallet with one verified position but incomplete venue coverage may appear
  as confirmed active with incomplete exposure; do not include stale positions
  in numerical totals. No observed position plus incomplete coverage is unknown,
  not flat, and is not listed as active.

Header: cohort date/ID, `Active A / Selected N`, known-flat and unknown counts,
report snapshot time and coverage. A zero-active validated snapshot shows
`No verified open positions in this cohort.` Export may produce the same empty
report with headers and coverage metadata. If the DB/worker has no usable
snapshot, show unavailable and keep Export disabled.

#### Freeze the exact report behind the list

Create one immutable report bundle from a consistent DB read, containing cohort
membership, eligibility/coverage, all listed wallets and their verified position
rows, corresponding coin aggregates, and applicable daily performance revisions.
Include 15m/1h/24h change-window inputs/results and coverage with the bundle;
do not query mutable latest state again while producing the workbook.

Persist the bundle in `hl_report_snapshots` with an opaque report ID, originating
Telegram account/runtime and authorized user, cohort ID, source snapshot IDs,
created/as-of times, expiry and bundle/calculation version. Retain it for 24 hours
by default; this is transient report data, not deletion of stored trade history.
The existing menu session may expire sooner. Validate both ownership and session
state on every export. A selected report remains available to an in-progress
export until it completes or fails, even if a later List creates another report.

Paging does not create a new snapshot. Refresh list creates a new report and
resets to page 1; Export binds to the report shown on that menu, across all pages.
Stale callbacks or an expired/missing report ask the user to List again instead
of exporting a different cohort. A new daily cohort must not silently replace
an older listed report: its original cohort date/time stay on the file and caption.

Opening List reads existing data only. Neither List nor Export initiates remote
history downloads, leaderboard refreshes, or performance recalculation.

#### Workbook content: required sheets

File name: `hyperliquid-top30_<cohort-date>_<snapshot-UTC>_<short-report-id>.xlsx`.
Use a real Office Open XML workbook, not CSV/HTML renamed to `.xlsx`.

| Sheet | Required content |
| --- | --- |
| `Coin Summary` (first sheet) | The required Long/Short result table described below, default recent window 1h, with visible coverage and snapshot metadata. It must never be omitted. |
| `Active Traders` | Exactly every listed active wallet across all Telegram pages: rank, full address, Hypurrscan hyperlink, open-position count, Long/Short gross values, held instruments and coverage. Include available daily/7d/30d performance with its own cutoff and source/local metric labels; unavailable values are blank with reasons. |
| `Open Positions` | Each verified position used in the aggregates: wallet/link, venue/coin, Long/Short, signed/base size, notional/currency, average entry, leverage, margin and unrealized PNL when available, source time and quality. |
| `Recent Changes` | Per-coin new/increased/reduced/closed/flipped position changes over 15m/1h/24h, including quantitative evidence and coverage; separate order intention from executed/confirmed position changes. |
| `Report Info` | Report/cohort IDs, selected/active/flat/unknown counts, source snapshot times, latest daily metric dates, definitions, formula denominators, missing-data reasons and history/stream limitations. Include a coverage roster of all selected cohort addresses so omitted flat/unknown wallets are explainable. |

`Coin Summary` must contain at least:

| Column | Definition |
| --- | --- |
| Coin / venue | Exact instrument identity; do not combine different contracts sharing a ticker. |
| Long traders / Short traders | Distinct wallets with verified positions on each side. |
| Long % / Short % | Side count divided by total positioned wallets for that instrument. |
| Participation % | Positioned wallets for that instrument / selected cohort count, even though the list displays only active wallets. |
| Long value / Short value | Sum of current verified position notionals on each side, in a declared common valuation currency. |
| Long capital % / Short capital % | Side notional / total gross notional for that instrument. |
| Net capital bias % | (Long value - Short value) / (Long value + Short value). |
| Coin concentration % | Instrument gross notional / gross notional across all observed cohort instruments in the same valuation scope. Label partial coverage. |
| Avg entry Long / Avg entry Short | Separate base-quantity-weighted entry prices of current positions. |
| Recent trend (1h) | Evidence-based description of recent position changes, not an instruction to buy/sell. |
| Long exposure change / Short exposure change (1h) | Signed changes in absolute side quantities, valued at one fixed reference price per instrument for both interval endpoints; distinguish these from mark-driven notional changes. |
| Coverage / as-of | Known/unknown counts, window validity and source snapshot time. |

The image supplied by the user is a layout/content reference, not live data.
Implement the same essentials: coin, Long/Short counts, Long/Short values,
separate average entries and recent direction, with the additional ratio columns.
Zero denominators or missing valuations produce blank numeric cells plus `N/A`
reasons, not misleading 0%, infinity or spreadsheet errors. Do not sum unlike
quote currencies without an as-of conversion and a coverage label.

For trend descriptions, calculate changes for the entire fixed cohort in the
window, including wallets that closed everything and no longer appear on List.
Otherwise closures would disappear from the trend. Include recently closed
instruments in Recent Changes even when absent from the current-position summary.
Use signed side exposure changes with Decimal-safe precision and these distinct
cases: positive Long with zero Short change -> `Increasing Long`; positive
Short with zero Long change -> `Increasing Short`; both positive ->
`Adding both sides`; negative sides with no positive side -> `Reducing exposure`;
positive Long and negative Short -> `Shift toward Long`, and the reverse ->
`Shift toward Short`. Both changes zero -> `No net position change`; incomplete
windows -> `Insufficient data`. Expose underlying counts/amounts in Recent Changes rather
than claiming the label proves future price direction. Price moves alone must
not create a trend. Entrant/exit cohort membership is not a trade event.

Freeze headers, enable table filters, size columns, and use readable date,
currency and percentage formats. Distinguish Long/Short through labels as well
as color. Store numeric measures as numeric cells (percent ratios as fractions
with percentage formatting), preserving exact source decimals in the report
bundle; round only for presentation within Excel precision. Untrusted names and
addresses are literal text, never formulas. Use explicit hyperlink objects for
Hypurrscan URLs; no macros or external workbook links.

#### Generation and delivery

Use an explicit maintained XLSX writer compatible with the bot's supported Node
version, proposed `exceljs`, after dependency review. Generate from the immutable
bundle using the same aggregate calculation contract as the viewer, not a second
spreadsheet-only implementation of Long/Short logic. Do not require Python,
Microsoft Excel or an open web viewer at bot runtime.

Acknowledge Export immediately, capture the report ID, then generate without
blocking Telegram command handling or the trading loop. Use one export task per
user/report with bounded concurrency and duplicate-callback protection. Reuse
`TelegramDocumentService.sendDocuments` and the originating account's delivery
configuration; never fan out another user's requested workbook.

Send the XLSX as a Telegram document with cohort/snapshot/coverage in the caption
or accompanying message. Respect the delivery service's upload-size limit and
bound generation memory/time. If generation/upload fails or exceeds limits,
report it and offer Retry for the same report; never claim success or drop the
mandatory summary sheet to fit. Do not include unbounded lifetime trade-history
rows by default. The sheets above are the required report; further sheets must
not replace the requested result table.

## 9. Daily job sequence and resource control

1. Acquire a dated run record; one successful normal run per report date.
   Startup executes a missed current run without fetching the leaderboard
   repeatedly. Retries/revisions retain explicit lineage.
   Recalculate missed performance dates only from available historical data;
   never backdate today's leaderboard into a missing historical cohort. Persist
   actual selection and activation times to prevent future-data leakage.
2. Capture and validate the daily leaderboard once. Persist the raw response,
   retrieval time and source timestamp if available; retrieval time alone does
   not prove source freshness. Candidate enrichment reuses this snapshot.
3. Reconcile history, ledger and valuation coverage through the cutoff for
   tracked wallets; run first imports for new candidates under a bounded queue.
4. Calculate and persist daily metrics from committed records with per-wallet
   completion status. One wallet failure must not erase others' results.
5. Build the new cohort and initialize live coverage. Atomically activate its
   membership only after validation, or retain the previous cohort as stale.
   An incomplete initial history can coexist with healthy live positions if
   performance is clearly marked partial.
6. Record completion, outstanding repairs and stream coverage. No automatic
   chat notifications are required by this scope.

Use one weighted request limiter across live repair, candidate checks and daily
history. Start below 60% of the official IP allowance (720 weight/minute),
account for response-size weights, and prioritize live recovery over archives.
Reserve capacity for other processes sharing the IP. `429`/`Retry-After` reduces
load; failures use bounded backoff, never concurrent retry storms.

For scale: on the default perp venue, polling 30 position states every ten
seconds would cost about 360 weight/minute. Polling their open orders once a
minute adds about 600 before history/fees and extra venues. This is why casual
REST polling is not a substitute for a properly sized 30-wallet stream.

## 10. Ordered implementation steps after approval

All paths below are planned; this turn adds only this document. Main feature
folder: `src/hyperliquid/`. Shared changes are limited to worker registration,
database migrations, deployment wiring, the existing web viewer and the Telegram
List/Export actions and web launch button required by this feature.

| Step | Concrete work and proposed files | Verification / completion gate |
| --- | --- | --- |
| 1. Validate realtime source | Define source capability contract in `src/hyperliquid/source.types.ts`; inspect a candidate provider's supported streams/replay and measure a permitted sample. Record source choice and actual cost/limits in this doc. | Confirm 30 simultaneous addresses, all required venues, positions AND order lifecycle/fills, and gap recovery. Do not buy or deploy services without approval. Failure blocks full-realtime release, not the independent daily components. |
| 2. Add storage | Add `migrations/operational/<next>_hyperliquid_monitoring.sql` using the next free sequence, plus `src/hyperliquid/hyperliquid.repository.ts` and its tests; use existing `postgres` transaction pattern. | Dedicated test DB proves unique keys, exact decimals, restart-safe page/cursor commits, revisions and no execution-table writes. |
| 3. Implement read-only clients | Add `src/hyperliquid/hyperliquid-info.client.ts`, `leaderboard.client.ts`, and source schemas/limiter; validate network/address/dex/time/decimal fields. | Fixtures cover official response shapes, application errors, schema changes, malformed records and weighted backoff. |
| 4. Daily cohort selection | Add `src/hyperliquid/cohort.service.ts`; persist leaderboard once, verify candidates, resolve known family duplicates and freeze membership. | Deterministic rank/ties, source-scope labeling, exclusions, fewer-than-30 handling and restart using the same daily snapshot. |
| 5. History ingestion | Add `src/hyperliquid/history-sync.service.ts`; implement first import, overlap reconciliation, independent cursors and gap states. Add archive adapter only if required and approved. | Duplicate WS/REST fills, same-timestamp pagination, 10k truncation, page replay and crash recovery preserve records without false completeness. |
| 6. Position and order stream | Add `src/hyperliquid/live-monitor.service.ts` and the approved streaming adapter; persist new fills immediately, reconcile snapshots and manage membership handoff. | Out-of-order events, duplicate snapshots, partial fills, cancelled/trigger orders, reconnect gaps and quiet-wallet health. Validate all 30 under permitted capacity before acceptance. |
| 7. Performance engine | Add `src/hyperliquid/performance.ts` and `performance.service.ts`; derive episodes, fees/funding, daily/7d/30d metrics and coverage-aware return/drawdown. | Fixtures with deposits, missing equity, rebates, partial closes, flips, truncated openings and unified accounts produce explicit correct/undefined metrics. |
| 8. Cohort aggregation | Add `src/hyperliquid/consensus.ts` and `consensus.service.ts`; derive per-instrument votes, exposure, concentration and entry prices from DB/current state. | Hand-calculated Long/Short examples, whale-dominance case, 2/30 participation, stale venue, changing prices with constant size, and cohort rollover. |
| 9. Worker lifecycle | Add `src/hyperliquid/hyperliquid.configuration.ts`, `worker.ts`, `scripts/run-hyperliquid-worker.ts`; register script in `package.json` and optional service in `docker-compose.yml`. | One leader, daily catch-up, bounded parallel work, graceful shutdown, failure isolation and supported Node runtime. Choose a direct maintained WS dependency if required; do not rely on a transitive dependency or Node 24-only API while Node 20 remains supported. |
| 10. Web viewer | Add `apps/backtest-report-viewer/app/hyperliquid/page.tsx`, matching detail pages, read-only routes under `app/api/hyperliquid/`, and queries under `server/hyperliquid/`. Add dedicated read-only DB configuration and access enforcement through the reviewed deployment. | Summary/coin/wallet/history views use consistent DB snapshots and server pagination; validate mobile/desktop rendering, timestamp/coverage labels, authentication on page and API, errors and empty states. No exchange calls from viewing. |
| 11. Telegram List and links | Update the existing Hyperliquid menu in `src/control/menu-view.ts`, `control-state.ts`, `control-handler.service.ts` and keyboard serialization. Reuse existing Hyperliquid report/client helpers where present; bind read-only report snapshot queries and preserve the web URL action. | List precedes Export, only currently positioned cohort wallets appear, ranks remain fixed, all wallet links target Hypurrscan, and all pages bind to one owned snapshot. Flat/unknown states, empty lists, expired callbacks and existing menu actions are tested. |
| 12. XLSX report and delivery | Add `src/hyperliquid/xlsx-report.ts` with a reviewed XLSX dependency and workbook read-back tests; persist owned report bundles through the repository and wire the separate Export effect to `TelegramDocumentService.sendDocuments`. | First sheet contains every required summary column; active wallets/positions/totals reconcile to the list bundle across all pages. Percent denominators, trend math, hyperlink targets, literal text cells, empty/partial reports, retry, expiry, size limits and user/account isolation pass. No exchange refresh or trading action occurs. |
| 13. End-to-end acceptance | Add integration tests and document worker/viewer/export configuration in the relevant READMEs; inspect stored metrics and live states, List on Telegram, export XLSX and open it in Excel or LibreOffice, and browse the web viewer. | Bot/viewer typechecks, focused tests, viewer build, dedicated DB tests, workbook structure/read-back plus visual inspection, and permitted 24-hour observation including one daily turnover. Existing backtest viewer still works; no real orders submitted. |

Order dependencies: storage/contracts precede clients and ingestion; performance
depends on committed historical/valuation data; consensus depends on validated
live states. Live-source approval is mandatory before step 6 is finalized.
Where steps 4/5 interact, persist pending candidates and let bounded history
verification complete selection rather than inventing activity from an empty API.

The user has requested this plan document with separate Telegram List/Export
actions, Hypurrscan wallet links, an XLSX consensus report and the existing
planned web viewer as the presentation scope. This document update does not authorize
implementation or deployment. Any "only buy/only short" execution rule remains
a separate scope decision.

## 11. Acceptance examples

1. Daily refresh: repeated scheduler/startup attempts reuse one dated leaderboard
   snapshot; the active 30 do not change intraday. A failed new cohort leaves the
   previous group visible with its original timestamp and degraded status.
2. Trade evidence: a new Buy order changes pending intent only. Its fill marked
   Close Short reduces Short exposure. Only a resulting positive position counts
   as Long. Same execution arriving over REST later adds no extra trade/fee.
3. Entry-price math: two Long positions of 2 units at 100 and 1 unit at 130 have
   weighted entry 110. A Short at 120 has its own average and is not mixed in.
4. Coverage: 18 Long and 4 Short among 30 healthy wallets means 81.82% of the 22
   participating wallets lean Long, participation is 22/30, and 8 are flat for
   that instrument. Missing wallets appear as unknown, not flat.
5. Performance: opening equity 1,000, net deposit 500 and ending equity 1,550
   implies account PNL 50, not 550, for the same valuation scope. ROI remains
   undefined without sufficient flow timing/valuation support for its method.
6. History: 10,000 available fills with older trading evidence is window-limited.
   The system can report observed-period PNL but cannot claim lifetime win rate.
7. Recovery: after a disconnect, current state stays stale until revalidated;
   an unrecoverable cancelled order leaves an order-history gap even if current
   open orders and positions are successfully repaired.
8. Viewer: 26 instruments remain reachable across two summary pages at 25 per
   page; 127 stored fills are reachable across three history pages at 50 per
   page. Detail navigation retains snapshot identity; Refresh never starts a
   leaderboard/history job. Mobile and desktop expose the same records.
9. Web freshness/access: partial snapshots retain timestamps and coverage warnings.
   Source names render safely; unauthenticated page/API access is refused by
   the configured access layer. A shared Telegram URL does not grant access.
10. List/Export: of 30 cohort wallets, 12 have verified open positions, 15 are
    confirmed flat and 3 are unknown. List shows only the 12 in cohort-rank order;
    it does not backfill from rank 31 onward. Export before List is refused;
    exporting from page 2 includes all 12 and the same snapshot's coin summary.
11. Ratios: BTC has 9 Long and 3 Short wallets among 30 selected, with $9M Long
    and $1M Short. The workbook shows 75%/25% trader share, 40% participation,
    90%/10% capital share and +80% net capital bias. If observed total cohort
    gross exposure is $20M in the same currency, BTC concentration is 50%.
12. XLSX integrity: the workbook reopens successfully, Coin Summary is the first
    sheet, all required columns exist, numeric percentages use the correct
    formats, and wallet links resolve to the exact Hypurrscan address. A name
    beginning with `=` remains text. Exported totals reconcile to Open Positions.
13. Frozen report: after List, a wallet closes a position or the daily cohort
    changes. Export still describes the listed report with its original as-of;
    only Refresh list creates a new report. Expired/foreign report IDs cannot
    be exported. A valid empty report retains summary headers and coverage.
14. Trends: an unchanged 1 BTC Long position whose mark rises 10% is not
    `Increasing Long`. A wallet that closes all positions remains part of the
    fixed-cohort change calculation even though it is absent from the active list.
15. Delivery: a successful document send is reported only after Telegram accepts
    it. Failures retry the same report; no duplicate callback launches another
    concurrent export. Web URL availability does not gate List or Export.

## 12. References and review decisions

- [Leaderboard](https://app.hyperliquid.xyz/leaderboard)
- [Info endpoint: fills, roles, portfolio and account modes](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint)
- [Perpetual position APIs](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/info-endpoint/perpetuals)
- [WebSocket subscriptions](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/websocket/subscriptions)
- [Rate limits and 10-user streaming limit](https://hyperliquid.gitbook.io/hyperliquid-docs/for-developers/api/rate-limits-and-user-limits)
- [Historical datasets and archive sources](https://hyperliquid.gitbook.io/hyperliquid-docs/historical-data)

Review before implementation: approve the selection/schedule defaults and decide
whether to obtain a supported full 30-wallet stream or accept the explicitly
degraded public-API fallback. Provider capability/cost and archive depth remain
unverified; do not treat this plan as evidence that those services are available
at any particular price or completeness level.
