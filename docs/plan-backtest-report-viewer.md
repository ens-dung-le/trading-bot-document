# Backtest Report Viewer implementation plan

Goal: build a basic internal web view that reads stored candles, accepts the bot's report files, and
shows trades, market-regime periods, report metrics, and data-quality warnings together without
changing strategy execution.

Out of scope: live order entry, paper or live position control, exchange API access, report editing,
strategy parameter changes, and using the viewer as an execution signal.

## 1. Product boundary

The viewer is a read-only analysis tool. It must remain operationally and structurally separate from
the bot, backtest worker, CLARITY workers, and every strategy module.

```text
browser report files                         operational PostgreSQL
  backtest-result.txt                          candles + trading_symbols
  trades.csv                                           |
  regime-periods.csv                                   v
          |                                    read-only candle API
          v                                             |
client-side validation and parsing                      v
          |--------------------------------------> chart workspace
                                                        |
                                                        v
                                              local derived statistics
```

Rules:

- Uploaded files stay in browser memory for MVP. They are not written to disk or PostgreSQL.
- The server opens only the market-data database connection needed to read symbols and closed
  candles.
- The server must not import bot modules from root `src/**` at runtime.
- The viewer must never receive exchange, Telegram, or live/paper trading credentials.
- A chart annotation is descriptive only. It cannot call a bot command or create an order.
- The source report remains authoritative for stored metrics. Client-derived metrics are labeled
  `calculated from uploaded trades` and never overwrite report values.

## 2. Isolated source ownership

All implementation files live under one new application directory:

```text
apps/backtest-report-viewer/
  package.json
  package-lock.json
  tsconfig.json
  next.config.ts
  .env.example
  README.md
  app/
    api/
      health/route.ts
      market-data/symbols/route.ts
      market-data/candles/route.ts
    layout.tsx
    page.tsx
    globals.css
  components/
    chart/
    import/
    metrics/
    workspace/
  domain/
    candles/
    reports/
    statistics/
  server/
    database/
    market-data/
  tests/
    fixtures/
  Dockerfile
  compose.yml
```

The feature branch owns only:

```text
apps/backtest-report-viewer/**
docs/plan-backtest-report-viewer.md
```

The feature branch must not edit:

```text
src/**
scripts/**
migrations/**
package.json
package-lock.json
docker-compose.yml
.gitignore
docs/market-regime-analysis-spec.md
docs/plan-clarity-act-telegram-alert.md
```

This arrangement gives the viewer its own dependency graph and lockfile. Root builds, bot images,
backtest workers, CLARITY work, and strategy branches do not see its frontend dependencies.

No database migration is required for MVP. Adding the viewer to the root Compose stack or creating a
dedicated PostgreSQL login is a later integration-owner change, not part of the viewer feature branch.

## 3. Git worktree setup

Create the worktree from an explicit reviewed base commit so later changes to `local-test` cannot
silently change the viewer's starting point:

```bash
VIEWER_BASE_SHA=$(git rev-parse local-test)
mkdir -p /Users/MAC/ai-consensus-trading-bot-worktrees
git worktree add \
  /Users/MAC/ai-consensus-trading-bot-worktrees/backtest-report-viewer \
  -b feature/backtest-report-viewer \
  "$VIEWER_BASE_SHA"
git worktree list
```

Recommended commit sequence:

```text
backtest-viewer: scaffold isolated web application
backtest-viewer: expose read-only candle range API
backtest-viewer: parse bot report bundle
backtest-viewer: overlay trades and regimes on candles
backtest-viewer: add report and regime statistics
backtest-viewer: harden import and responsive states
```

## 4. Existing data contracts

### Candle source

Candles are read from the operational database, not the backtest database:

```text
trading_symbols
  id
  exchange_name
  unified_symbol
  exchange_symbol

candles
  symbol_id
  timeframe
  open_timestamp
  open_price
  high_price
  low_price
  close_price
  base_volume
  is_closed
  source
```

Only `is_closed = true` candles may be returned. The API orders by `open_timestamp ASC` and returns
numbers as JSON numbers after checking they are finite.

### Uploaded report bundle

The current bot generates these files:

| File | Required | Viewer use |
|---|---:|---|
| `backtest-result.txt` | yes | Run metadata, stored summary, parameters, benchmark, acceptance, and diagnostic text |
| `trades.csv` | yes | Entry and exit markers, trade detail, P&L, costs, reasons, and entry/exit regime context |
| `regime-periods.csv` | optional | Contiguous regime background bands and regime-level statistics |

The import accepts files by schema, not only by filename, so a Telegram or browser rename does not
break a valid report. Unknown columns are preserved in raw row data and ignored by the current view.
Missing required columns fail the import with a field-level error.

Required `trades.csv` columns:

```text
side,entry_timestamp,entry_price,exit_timestamp,exit_price,quantity,
net_profit_and_loss,entry_reason,exit_reason
```

Supported regime trade columns:

```text
entry_regime,entry_volatility,entry_regime_confidence,
entry_regime_candle_timestamp,exit_regime,exit_volatility,
exit_regime_confidence,exit_regime_candle_timestamp,
regime_transition_count,regime_model_version
```

Required `regime-periods.csv` columns:

```text
sequence,start_timestamp,end_timestamp,direction,volatility,candle_count,
minimum_confidence,average_confidence,entry_count,asset_return_rate,
mark_to_market_profit_and_loss,worst_drawdown_amount,worst_drawdown_rate
```

The text parser reads at least `Run label`, `Strategy`, `Symbol`, `Trigger timeframe`, `Period`,
`Starting balance`, `Ending balance`, and `Acceptance` when present. Unrecognized report sections are
kept as plain text so a newer bot report remains inspectable even before the parser is upgraded.

### Import consistency checks

Before requesting candles, the browser validates:

- all timestamps are valid ISO-8601 values and use one canonical UTC representation;
- trade exits are not earlier than entries;
- prices and quantities are positive finite numbers;
- regime periods are ordered and do not overlap;
- imported trade/regime timestamps fit the report period, with explicit warnings for exceptions;
- `entry_count` by regime reconciles with trade entry timestamps where both files are present;
- one regime model version is used, or a mixed-version warning is shown;
- the report symbol and timeframe resolve to exactly one stored market-data series.

If symbol resolution is ambiguous, the UI asks the user to choose an exchange, symbol, and timeframe.
It must not guess silently.

## 5. Read-only candle API

### Endpoints

```text
GET /api/health
GET /api/market-data/symbols
GET /api/market-data/candles?exchange=<name>&symbol=<unified>&timeframe=<tf>
    &from=<iso>&to=<iso>&cursor=<iso>&limit=<n>
```

`GET /api/market-data/symbols` returns only the identities and timeframe coverage needed by the
picker. It must not return strategy, account, position, order, or credential data.

`GET /api/market-data/candles` uses keyset pagination. The default page is 5,000 candles and the hard
maximum is 10,000. The response shape is:

```ts
interface CandlePage {
  readonly series: {
    readonly exchange: string;
    readonly symbol: string;
    readonly timeframe: string;
  };
  readonly candles: readonly {
    readonly time: number;
    readonly open: number;
    readonly high: number;
    readonly low: number;
    readonly close: number;
    readonly volume: number;
  }[];
  readonly nextCursor: string | null;
  readonly coverage: {
    readonly requestedFrom: string;
    readonly requestedTo: string;
    readonly returnedFrom: string | null;
    readonly returnedTo: string | null;
    readonly missingSlotCount: number;
  };
}
```

Server requirements:

- validate all query fields before SQL;
- execute database sessions as read-only and expose no mutation route;
- select explicit columns instead of `SELECT *`;
- cap date span and result size to prevent an accidental multi-million-candle response;
- abort a query when the client disconnects or the server timeout expires;
- return `404` for an unknown series and `422` for invalid ranges;
- never log uploaded report content or a database URL.

## 6. Viewer screen

The screen follows the reference's trading-terminal hierarchy without copying MEXC or TradingView
branding.

Design direction:

```text
DESIGN_VARIANCE: 4
MOTION_INTENSITY: 2
VISUAL_DENSITY: 9
```

Use one dark neutral theme for MVP, one semantic accent per state, compact spacing, and monospaced
numeric values. Motion is limited to loading and selection feedback. The screen must remain keyboard
navigable and readable at 200 percent zoom.

Desktop layout:

```text
+----------------------------------------------------------------------------------+
| Report / symbol / timeframe / UTC range / file health / import action            |
+---------------------------------------------------------------+------------------+
|                                                               | Run metrics      |
| Candlesticks                                                  | Acceptance       |
| - entry and exit markers                                      | Cost summary     |
| - regime background bands                                     | Regime breakdown |
| - selected trade connector                                    | Selected trade   |
|                                                               | Data warnings    |
+---------------------------------------------------------------+------------------+
| Volume                                                        |                  |
+---------------------------------------------------------------+------------------+
| Trades | Regime periods | Full report text                                       |
+----------------------------------------------------------------------------------+
```

Below 1,024 px, the metric rail moves below the chart. Below 768 px, the chart keeps a minimum usable
height, filters wrap into two rows, and detail tables use horizontal scrolling. No control depends on
hover alone.

### Chart layers

1. Candlestick series from stored closed candles.
2. Volume histogram aligned to the candle time scale.
3. Entry markers: green `B` for buy and red `S` for sell.
4. Exit markers with a distinct outline so entry and exit cannot be confused.
5. A connector or highlighted time span for the selected trade only.
6. Regime background bands from `regime-periods.csv`:
   - uptrend uses a restrained positive tint;
   - downtrend uses a restrained negative tint;
   - sideways uses a neutral tint;
   - volatility is expressed by opacity or a small band label, not another competing color system.
7. Crosshair tooltip with OHLCV, regime, and any trade event at the selected candle.

Multiple markers on one candle are grouped and expanded in the selected-candle detail. They must not
be silently collapsed into one trade.

### Right-side statistics

Run metrics from the report:

- strategy and version;
- symbol, timeframe, and period;
- starting and ending balance;
- net return and total trades;
- win rate and profit factor;
- maximum drawdown and Sharpe ratio;
- fees, funding, slippage, and liquidation count when available;
- acceptance verdict and failed criteria.

Regime effectiveness calculated from uploaded data:

- trade count, win rate, net P&L, and profit factor by `uptrend`, `downtrend`, and `sideways`;
- the same breakdown by volatility level;
- average regime confidence at entry;
- regime transition count while positions are open;
- best and worst regime period by mark-to-market P&L;
- report/file reconciliation status.

The initial version reports observations. It does not recommend enabling or disabling a strategy in
any regime.

### Detail tabs

- `Trades`: sortable rows; selecting a row focuses its entry/exit range on the chart.
- `Regime periods`: period timestamps, direction, volatility, confidence, return, P&L, and drawdown;
  selecting a row focuses that period.
- `Full report`: safe plain-text rendering of every section in `backtest-result.txt`; no HTML from an
  uploaded file is executed.

## 7. Implementation slices

### Isolated application shell

Work:

- create the standalone Next.js TypeScript application and local scripts;
- define visual tokens, responsive shell, loading, empty, and error states;
- add a local `.env.example` containing only the viewer's read-only database settings;
- add app-local Docker and Compose definitions without changing root Compose.

Acceptance:

- root `npm run build` has the same inputs as before;
- viewer install/build/test runs only inside `apps/backtest-report-viewer`;
- starting the viewer does not start or import the bot.

### Candle read model

Work:

- implement symbol/coverage lookup and paginated closed-candle queries;
- validate ranges, enforce limits, and calculate missing timeframe slots;
- add API contract and database integration tests using a disposable database.

Acceptance:

- API ordering is stable across pages with no duplicate or skipped timestamp;
- open candles are excluded;
- an unavailable or gapped range is visible to the user instead of appearing complete.

### Report bundle import

Work:

- implement CSV parsing with quoted field support and explicit schemas;
- parse known text fields and preserve unknown report sections;
- normalize values into one in-memory `ImportedBacktestReport` model;
- show per-file success, warning, and failure messages.

Acceptance:

- current, legacy nullable-regime, malformed, renamed, and mixed-file fixtures are covered;
- parsing never sends the file to the server;
- one malformed row reports its row and field without crashing the whole page.

### Chart replay

Work:

- render candle and volume series using a maintained financial chart library;
- map trade timestamps to candle slots without changing original event timestamps;
- add entry/exit markers, selected-trade focus, regime bands, crosshair detail, and legend controls;
- virtualize or aggregate annotations when the visible range is dense.

Acceptance:

- marker price/time matches the uploaded row exactly;
- timezone stays UTC across header, chart, tooltip, and tables;
- zooming or panning never changes report statistics;
- 10,000 candles and 2,000 trades remain interactive on a normal development laptop.

### Statistics and audit panel

Work:

- render stored report metrics separately from locally calculated metrics;
- calculate trade/regime breakdowns using deterministic pure functions;
- implement selected trade and selected regime details;
- surface coverage, schema, reconciliation, and model-version warnings.

Acceptance:

- aggregate trade count and net P&L reconcile with the imported rows within documented decimal
  tolerance;
- zero-trade, open-trade, missing-regime, and missing-candle states have explicit output;
- no displayed recommendation can be interpreted as an automated trading instruction.

### Hardening and handoff

Work:

- add keyboard navigation, focus visibility, contrast, reduced motion, and 200 percent zoom checks;
- test desktop, tablet, and mobile layouts;
- document local startup, report import, database requirements, and known limits;
- verify dependency licenses and any required financial-chart attribution before release.

Acceptance:

- unit, API integration, browser component, and one end-to-end import scenario pass;
- production build contains no exchange or Telegram environment variable;
- source diff stays inside the ownership boundary in section 2.

## 8. Recommended dependencies

Dependencies belong only to `apps/backtest-report-viewer/package.json`:

| Need | Candidate |
|---|---|
| Full-stack React application | Next.js with React and TypeScript |
| Financial candlestick rendering | `lightweight-charts` |
| CSV parsing | `papaparse` plus its TypeScript types |
| Runtime validation | `zod` |
| PostgreSQL reads | `postgres` |
| Unit/component tests | Vitest and Testing Library |
| Browser flow | Playwright |

Do not add a global state library for MVP. The imported report and active selection fit one workspace
reducer plus memoized selectors. Do not add a data-grid package until the simple virtualized detail
table proves insufficient.

## 9. Security and failure behavior

- Bind the development server to `127.0.0.1` by default.
- Deploy behind existing authentication if exposed beyond the local machine. The viewer does not
  invent a second account system in MVP.
- Set a configurable per-file size limit and reject binary or unsupported inputs.
- Treat every uploaded string as untrusted text. Never use `dangerouslySetInnerHTML`.
- Do not persist recent filenames, report content, or trading results in analytics or server logs.
- On candle API failure, keep the parsed report available and explain that chart data is unavailable.
- On report parse failure, do not make a candle request using partially trusted metadata.
- On candle gaps, draw explicit gap boundaries and show the missing-slot count.

## 10. Verification commands

Run from the viewer directory:

```bash
npm ci
npm run typecheck
npm test
npm run build
npm run test:e2e
```

Then verify repository isolation from the original integration worktree:

```bash
git diff --name-only local-test...feature/backtest-report-viewer
```

Every listed file must be inside `apps/backtest-report-viewer/**` or be this plan document.

## 11. Definition of done

The feature is complete when a user can:

1. open the viewer without starting the trading bot;
2. import the three current report files in one action;
3. resolve the report to a stored candle series;
4. see candle and volume history with every entry, exit, and regime period aligned in UTC;
5. select a marker or table row and inspect the corresponding trade;
6. compare the report summary with strategy performance by detected regime;
7. identify missing files, malformed rows, candle gaps, or reconciliation mismatches;
8. complete all of this without writing to either database or exposing execution credentials.
