# Trading Bot Document

A documentation repository for trading bot design, strategy specifications, and research reports. It covers crypto trading strategies, backtesting, paper trading, risk management, market data, and Telegram-based bot controls.

> **This repository contains documentation, not executable bot source code.** Source paths, migrations, and commands referenced in the documents belong to the corresponding implementation project and cannot be run from this repository alone.

## Getting Started

| What you want to explore | Start here |
| --- | --- |
| Initial ETH/USDT requirements | [ETH/USDT Developer Brief](docs/requirements/ETH_USDT_Developer_Brief_v4.md) |
| Telegram-based bot controls | [Telegram Control](docs/plan-telegram-control.md) |
| Market data and backtest workflows | [Telegram Market Data & Backtest Flow](docs/plan-telegram-market-data-backtest-flow.md) |
| Exported backtest runs | [Report guide](report/README.md) and [run manifest](report/backtest-manifest.tsv) |
| Market regime analysis | [Market Regime Analysis](docs/market-regime-analysis-spec.md) |
| Multi-timeframe support | [Strategy Timeframe Support](docs/strategy-timeframe-support.md) and [Backtest Timeframe Implementation](docs/backtest-timeframe-implementation.md) |

## Repository Structure

```text
.
├── README.md
├── docs/
│   ├── requirements/                  # Business requirements and developer briefs
│   ├── assets/                        # Documentation screenshots and illustrations
│   ├── freqtrade-strategy-explainers/  # Strategy reference PDFs
│   └── *.md                           # Plans, specifications, decisions, and audits
└── report/
    ├── README.md                      # Backtest scope and limitations
    ├── backtest-manifest.tsv           # Backtest run index
    ├── <strategy>-backtest/
    │   └── <symbol>/                  # Results by strategy and trading symbol
    └── <research-name>-<date>/         # Research reports and related snapshots
```

## Documentation Guide

### Trading Strategies

Strategy documents describe signals, entry and exit conditions, parameters, risk constraints, and testing requirements. Selected starting points:

- **London Open / Liquidity Macro:** [original](docs/london-open-liquidity-macro-strategy.md), [V2](docs/london-open-liquidity-macro-strategy-v2.md), [V3](docs/london-open-liquidity-macro-strategy-v3.md).
- **Box & Scalping:** [Box Strategy](docs/plan-box-strategy.md), [Scalping Lower-Wick Reclaim](docs/scalping-lower-wick-reclaim-v1.md).
- **Moving Average & Directional:** [Multi-MA Quantum](docs/multi-ma-quantum-v1-strategy-report.md), [Multi-Indicator Directional](docs/plan-multi-indicator-directional-strategy.md), [Triple SMA](docs/plan-triple-sma-directional-strategy.md), [Five-Gate Directional](docs/plan-five-gate-directional-strategy.md).
- **Market Structure:** [Order Blocks](docs/plan-order-blocks-strategy.md), [Breaker Blocks](docs/plan-breaker-blocks-strategy.md), [Fair Value Gaps](docs/plan-fair-value-gaps-strategy.md), [ICT Suspension Blocks](docs/plan-ict-suspension-blocks-strategy.md).
- **VWAP & Fibonacci:** [Anchored VWAP](docs/plan-anchored-vwap-suite-strategy.md), [VWAP Divergence](docs/plan-vwap-divergence-strategy.md), [Fibonacci Swing](docs/plan-fibonacci-sequence-swing-strategy.md), [Fibonacci Swing V2](docs/plan-fibonacci-sequence-swing-v2.md).
- **Grid & DCA:** [Trailing Geometric Grid](docs/plan-trailing-geometric-grid-strategy.md), [Grid design decisions](docs/trailing-geometric-grid-decisions.md), [Grid Amount Scaler](docs/plan-grid-amount-scaler-strategy.md), [Adaptive DCA V2](docs/plan-adaptive-dca-v2.md).

These are selected groups rather than a complete index. Browse [`docs/`](docs/) for all documents.

### Market Data, Execution, and Operations

- [Database Candle History](docs/plan-database-candle-history.md): historical candle storage planning.
- [Realtime Market Data Causality & Backpressure](docs/realtime-market-data-causality-and-backpressure.md): data causality and realtime workload handling.
- [Strategy Execution, Risk & Telegram Parameters](docs/plan-strategy-execution-risk-telegram-parameters.md): strategy, execution, and risk configuration.
- [Position Manager Audit](docs/position-manager-partial-close-funding-audit.md): partial-close and funding review.
- [Backtest Report Viewer](docs/plan-backtest-report-viewer.md): report viewer planning.

### Trader Monitoring and Alerts

- **KuCoin:** [Lead Traders History](docs/kucoin-lead-traders-history-spec.md), [Copy Trading Backtest](docs/kucoin-copy-trading-backtest-spec.md).
- **Hyperliquid:** [Trading History](docs/hyperliquid-trading-history-spec.md), [Top 30 Trader Monitor](docs/hyperliquid-top30-trader-monitor-plan.md).
- **CLARITY Act:** [Telegram Alert Plan](docs/plan-clarity-act-telegram-alert.md), [Alert SOP](docs/requirements/CLARITY_Act_Telegram_Alert_SOP.md).

## Backtest and Research Reports

According to [`report/README.md`](report/README.md), the manifest indexes **69 completed runs** across **23 strategies × BTC, ETH, and SOL**, covering **October 7, 2025, 00:00 UTC through October 7, 2026, 00:00 UTC**, with the end timestamp excluded.

**None of these 69 runs passed the complete acceptance gate.** `COMPLETED` means a run finished, not that a strategy was approved for live trading. Parameters, sizing, leverage, and costs used each strategy's individual defaults rather than a shared risk profile.

Common report files:

| File | Contents |
| --- | --- |
| `backtest-result.txt` | Run summary and evaluation results |
| `trades.csv` | Exported backtest trade records |
| `regime-periods.csv` | Market regime periods |
| `report.md` | Research analysis and limitations |
| `results.json` | Structured research results |

Selected standalone research reports:

- [Box 1h Optimization — October 8, 2026](report/box-1h-optimization-2026-10-08/report.md), with [paper-test notes](report/box-1h-optimization-2026-10-08/paper-test.md).
- [Scalping Wick — October 8, 2026](report/scalping-wick-2026-10-08/report.md).
- [Markowitz — October 8, 2026](report/markowitz-2026-10-08/report.md).
- [Multi-Indicator Directional V1](docs/multi-indicator-directional-v1-backtest-report.md).

## Reading Workflow

1. Read the strategy specification to understand its assumptions, signals, and execution rules.
2. Check each document's status and date. A design proposal does not imply an implemented feature.
3. Use the manifest and matching report to identify the symbol, timeframe, data window, and actual run configuration.
4. Review failed criteria, data limitations, and paper-test notes before interpreting results.

No dependency installation or build is required to read this repository. View Markdown on GitHub or in an editor, open CSV/TSV files in a spreadsheet tool, and use a PDF reader for reference documents.

## Contribution Guidelines

- Keep specifications and plans in `docs/`; keep run outputs and research reports in `report/`.
- State the version, date, status, and scope of changes. Distinguish proposals, implementation snapshots, and verified results.
- For new results, record the strategy, symbol, timeframe, data window, configuration, and limitations. Update the relevant index when adding runs to a manifest-managed backtest collection.
- Use relative links so documents remain navigable locally and on GitHub.
- Do not include API keys, Telegram tokens, credentials, or sensitive account information in documents or snapshots.

## Disclaimer

This repository supports technical documentation and research. It is not investment advice or a promise of profitability. Backtest and paper-trading results do not guarantee live performance. Implementation notes are snapshots of the recorded date, not confirmation of the system's current operational state.
