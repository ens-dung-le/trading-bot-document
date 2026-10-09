# KuCoin lead traders and position history

Status: scope agreed; specification ready for review. Implementation requires
separate user approval.
Date: 2026-09-17.

Follow-up specification (2026-09-22):
[KuCoin copy-trading history storage and backtest](kucoin-copy-trading-backtest-spec.md)
supersedes this document's session-only storage and no-persistence scope for the
next implementation. It defines durable history, separate synchronization and
database-only replay. The original browsing/source contracts below remain the
baseline except where explicitly superseded; the follow-up is not yet implemented.

## 1. Goal and scope

Goal: add a `KuCoin Lead Traders` button to the existing Telegram main menu,
list KuCoin lead traders, and load all publicly available closed-position
history when the user selects a trader profile.

"All history" means every page exposed by KuCoin for that trader during the
request. It does not mean the trader's entire lifetime trading record, every
individual fill, or private records KuCoin does not disclose.

In scope:

- One main-menu entry, a paginated trader list, and a profile/history screen.
- Only records whose source exchange is `KU`; exclude `BN` and unknown sources.
- Basic profile information, 30-day ROI/PNL, and all available history pages.
- History details: opening/closing times, direction, leverage, margin mode,
  quantities, average prices, reported margin, PNL and position return.
- On-demand loading, explicit access/error states, and retry/navigation.

Out of scope: current positions, trade copying, signals, alerts, watchlists,
scheduled collection, Binance trader history, charts, custom ranking/filter
controls, CSV/Excel exports, account login, and database persistence.

No indicator or strategy folder is involved. This task creates only this
specification; it does not authorize implementation.

## 2. Telegram flow

Keep user-facing messages in English, matching `src/control/menu-view.ts`.

### Main menu

Add one button labeled `KuCoin Lead Traders`. It is available independently
of the selected strategy, symbol, or engine running state. Selecting it does
not alter a trading draft, active strategy, or paper/live positions.

### Trader list

1. Acknowledge the callback immediately and show a loading state.
2. Fetch the complete source leaderboard, then filter `exchange === "KU"`.
3. Sort by numeric 30-day PNL descending, with `leadConfigId` as a stable
   tie-breaker. Missing PNL sorts last. Do not introduce a custom score.
4. Display ten traders per Telegram page. For each trader show the name,
   30-day ROI and 30-day PNL in its reported currency, plus a selection button.
5. Show the number of KuCoin traders, page count, and snapshot retrieval time.
6. Provide `Previous`, `Next`, `Refresh`, and `Back` as applicable.

Filter after retrieving all source pages: a source page can mix KuCoin and
Binance traders. Do not assume the first ten source records are the full list.
Keep traders with restricted history in the list; explain access on selection.
Use stable IDs for callbacks, never names or a live remote page index.

### Profile and history

Selecting a trader automatically loads its profile and, if public, every
history page. No additional confirmation or `Load history` button is required.

Profile header:

- Name and lead trader ID.
- 30-day ROI and PNL from the selected leaderboard snapshot, labeled with its
  retrieval time; do not silently present old metrics as freshly retrieved.
- Days as a lead trader and current copier count when supplied.
- History access and loading/completion status.
- A KuCoin profile link in the message body:
  `https://www.kucoin.com/copytrading/trader-profile/{leadConfigId}`.

While loading, show the number of records/pages fetched and keep `Back`
available. Throttle progress edits to at most one every two seconds.

After loading, show:

- `All available history loaded: N records` only for a validated full sweep.
- The earliest and latest closing times returned, not an inferred lifetime
  coverage guarantee.
- Snapshot retrieval time and `Page X / Y`.
- Three complete position records per Telegram page, newest close first.
  Reduce the number on a page if needed to stay within Telegram message limits;
  all records and required fields must remain reachable.
- `Previous`, `Next`, `Refresh`, and `Back to traders` as applicable.

Preserve source order for equal closing timestamps. Returning to the trader
list restores its page. Local page navigation uses the loaded snapshot and
does not request the same remote pages again. Refresh starts a new sweep.

Use UTC explicitly in timestamps, matching existing bot time formatting.
Do not send the entire history as a burst of chat messages.

### Access and failure states

| Condition | User-facing behavior |
| --- | --- |
| `positionVisibility = 1` | Fetch public history. |
| `positionVisibility = 0` | `This trader has hidden their position history.` |
| `positionVisibility = 2` | `History is available only to eligible copy traders on KuCoin.` |
| Missing/unknown visibility | `History access could not be verified.` Do not infer public access. |
| Public access and validated zero records | `No closed-position history is currently available from KuCoin.` |
| Failed/inconsistent sweep after some pages | `History incomplete: N records loaded.` Preserve accessible partial results and offer `Retry`. |
| Network/service/schema failure with no records | `Could not load KuCoin history. Please retry.` |
| Session expired or bot restarted | Reopen the menu and fetch again; no durable-history promise. |

Do not convert HTTP 200 plus an empty array into a claim that the trader has
never traded. Recheck visibility before declaring an empty or completed result.
If visibility becomes restricted, remove the history from the active screen
and discard that session's history snapshot.

## 3. Verified data source

Use ordinary HTTP JSON requests to the website, without API keys or cookies.
These are undocumented website endpoints, not a supported public API contract.
Use a dedicated read-only client; do not add website calls to CCXT execution.

Base URL: `https://www.kucoin.com/_api/ct-copy-trade`.

| Data | Request |
| --- | --- |
| Leaderboard | `POST /v1/copyTrading/leaderboard/query` |
| Profile and access | `GET /v1/copyTrading/leadShow/summary?leadConfigId={id}` |
| Closed-position history | `GET /v2/copyTrading/leadHomePage/positions/history?leadConfigId={id}&currentPage={page}&pageSize=100` |

Leaderboard request:

```json
{
  "criteria": [],
  "hideFull": false,
  "sort": { "field": "thirty_day_pnl", "direction": "DESC" },
  "currentPage": 1,
  "pageSize": 100
}
```

Increment `currentPage` for the full sweep. Keep `hideFull: false` so a trader
being full does not remove their profile/history from browsing.

Validate HTTP status, JSON content, `success`, application `code`, and the data
shape. Paginated responses contain `data.items`, `currentPage`, `pageSize`,
`totalNum`, and `totalPage`. Unknown extra fields are allowed; malformed
required fields must not become a successful empty response.

Research evidence from 2026-09-17, not production constants or live-test
expectations:

- Leaderboard: 142 distinct IDs across two pages of size 100; 138 `KU`, 4 `BN`.
- Happyman (`1007041`): public; 127 history records across two pages, with
  closing dates from 2026-01-16 through 2026-09-11.
- Serekesh (`1007040`): public; API reported 26 history records.
- iramo ila (`1007662`): hidden; history returned success with zero records.
- Low Risk Swing King (`1006980`): copier-only; anonymous history returned zero.
- A direct Node.js `fetch` request to a public profile succeeded.

Do not use `leadShow/positionHistory` for this feature: that endpoint returned
holding-duration/PNL distribution data, not the detailed history table.

## 4. Position fields and meaning

Preserve decimal strings and use the repository's `decimal.js` for calculations.
Absent values render as `N/A`, not zero. Validate finite timestamps and numeric
values; keep any invalid row as an unavailable record rather than silently
dropping it and calling the history complete.

| Display | Source / calculation | Meaning |
| --- | --- | --- |
| Contract | `symbol` | Keep the source contract symbol. |
| Direction | `positionDirection` | Long/Short; `positionSide = BOTH` alone is not a direction. |
| Margin mode | `marginMode` | Cross/Isolated as reported. |
| Reported leverage | `leverage` | Not a guarantee of the original configured leverage. |
| Opened / closed | `startTime` / `endTime` | Epoch milliseconds; position record times, not every fill. |
| Held for | `endTime - startTime` | Only if both timestamps are valid and ordered. |
| Closed contracts | `closeQty` | Source contract count. |
| Base quantity | `closeQty * multiplier` | Show only when the multiplier is valid. |
| Average entry / exit | `avgEntryPrice` / `avgClosePrice` | Aggregate prices supplied by KuCoin. |
| Entry value of closed quantity | `abs(closeQty * multiplier * avgEntryPrice)` | Derived notional, not original cash deposited or every add-on. |
| Reported margin | `posMargin` | Do not label it initial margin committed at entry. |
| Position PNL | `pnl`, `currency` | Preserve the source value; do not invent a fee breakdown. |
| Position return | `pnlRatio * 100` | Percentage, separate from profile ROI. |

Profile ROI uses `thirtyDayPnlRatio * 100`; profile PNL uses `thirtyDayPnl`.
Do not derive either by summing this history: the coverage and definitions can
differ. Never use `orderSize` as position quantity or account equity.

If showing account allocation alongside these details, display
`Account allocation at entry: N/A (not provided by KuCoin)`.
Do not compute it from current `leadPrincipal`, copier AUM, the bot's balance,
or an `accountEquity` placeholder. Historical account equity is unavailable.

SL/TP, liquidation price and per-fill fees are not required in this history
screen: the verified history response does not provide those details.

## 5. Full-history loading contract

1. Fetch a fresh profile; validate the requested ID, `exchange === "KU"`, and
   public visibility before requesting history.
2. Fetch page 1 with size 100; record total counts, page metadata, and a page
   content fingerprint. Fetch every remaining page sequentially without a
   date, symbol, or "last N trades" filter.
3. Retry a failed page in place; append its records only after one successful
   validated response. Never append retries twice.
4. Validate advancing page numbers, consistent metadata, and the final record
   count against `totalNum`. A repeated page, unexpected empty intermediate
   page, malformed record, or changing total makes the sweep incomplete.
5. Re-read the profile and first history page at completion. If access changed,
   enforce the access state. If totals or first-page content changed, mark the
   sweep incomplete and offer a fresh retry; do not silently combine sweeps.
6. Mark complete only after these checks pass. This is a complete observed
   pagination sweep, not an atomic exchange snapshot: KuCoin provides no
   snapshot token, and older records may be revised without detection.

The verified history rows have no stable close ID. Preserve each returned row
with its page and row index in the snapshot. Do not deduplicate history by
symbol, timestamps, or a guessed composite key; distinct records could match.
Use page fingerprints to detect pagination failures, not to discard rows.

Use equivalent pagination validation for the leaderboard; deduplication by
its explicit `leadConfigId` is valid, but unexplained overlap/count mismatch
must not be presented as a complete trader list.

Apply request timeouts (10 seconds), at most two retries for transient network,
429, and 5xx failures, and backoff starting at one then two seconds. Respect
`Retry-After` within the operation deadline. Limit this feature to one outgoing
KuCoin request at a time per runtime, spaced at least one second apart.
These are conservative application defaults, not published KuCoin limits.

Use a two-minute operation deadline and a bounded response buffer (32 MiB per
session). A deadline or memory limit produces an explicit incomplete result,
never a truncated success. Retry starts a new sweep. Do not retry privacy or
schema errors as if they were transient outages.

## 6. Existing bot integration

Reuse the current reducer/effect/menu flow and its authorization checks:

| Existing code | Integration responsibility |
| --- | --- |
| `src/control/menu-view.ts` | Main-menu button and rendering of trader/profile/history screens. |
| `src/control/control-state.ts` | Screens, navigation, selection, loading/completion events and versioned callbacks. |
| `src/control/control-handler.service.ts` | Acknowledge, start reads, and apply guarded completion results. |
| `src/control/control-session.store.ts` | Existing per-user lifetime; retain bounded browsing snapshots with session cleanup. |
| `src/notification/telegram-control-runtime.ts` | Construct/inject the read-only client and cancel work during shutdown. |

Keep HTTP validation and pagination in a small dedicated client under
`src/control/`; keep the reducer and view free of network calls. The table
identifies integration points, not permission for unrelated refactoring.

The Telegram polling loop currently awaits each update handler. Therefore the
handler must acknowledge and save the loading state before starting the
multi-page operation, then return promptly. Run the request asynchronously
with caught errors and an abort signal; no new worker process is needed.

Tie results to the originating runtime, user, session, menu message, selected
trader and request generation. A late response must not overwrite a newer
selection, a refreshed menu, an expired session, or another user's screen.
Returning to the list cancels that profile load; reselecting the same profile
while loading must not launch another sweep.

Use existing callback versioning and duplicate-callback handling. Keep callback
payloads below 64 UTF-8 bytes and escape remote names/text as Telegram HTML.
Keep each rendered message below 4,096 characters. Existing trading commands,
including pause and stop, must remain responsive during a history load.

Keep only the list and the currently selected profile/history per active user
session. Discard history on selection change, expiry, eviction, or shutdown.
Runtime isolation follows existing Telegram account runtimes; no cross-user
selection state. Do not add tables, migrations, a queue service, dependencies,
environment switches, or a scheduled polling job for this feature.

## 7. Verification and acceptance

For this specification: diff review confirms only this document is added and
the scope matches the requested menu-to-profile-to-history flow.

For implementation after separate approval, use fixture-based tests rather
than depending on live trader names/counts:

1. The main menu exposes one new button without changing existing actions.
2. A mixed multi-page leaderboard lists every `KU` trader and excludes other
   sources; numeric sorting handles decimal strings and missing values.
3. Selecting a public profile automatically loads a 100 + 27 record fixture;
   all 127 records are browsable without further remote calls for local pages.
4. Hidden/copier-only/unknown access never becomes "no history". Public zero
   records is distinct. A visibility change during loading discards history.
5. A page failure, repeated page, changed total, invalid row, resource limit or
   deadline cannot produce a complete-history label; retry does not duplicate
   rows. Equal-looking rows are preserved, not guessed to be duplicates.
6. Multipliers, average prices, leverage, millisecond timestamps and percentage
   conversion render correctly; allocation at entry remains unavailable.
7. Back, refresh, selection changes, session expiry and late completions do not
   corrupt menus or leak results across users/account runtimes. Existing
   authorization, callback limits, HTML escaping and message limits still hold.
8. A pending history fetch does not block the command loop; shutdown aborts it.
9. No execution, strategy, balance, order or persistent trading-state mutation
   occurs from browsing. No database migration or background polling is added.

Run the relevant client/control/runtime tests and TypeScript checks, then a
manual Telegram check: open the button, select a public profile, browse the
oldest history page, check a restricted profile, and return to the main menu.
Live counts may differ from the research examples. Obtain user approval of
this specification before starting implementation.
