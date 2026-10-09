# Documented strategy backtests

`backtest-manifest.tsv` indexes 69 completed runs: 23 registered strategies × BTC, ETH, and SOL. Each `report/<strategy>-backtest/<symbol>/` directory contains `backtest-result.txt`, `trades.csv`, and `regime-periods.csv` exported from that run. The pre-existing `london-open-market-strategy-backtest/` reference is separate.

- Window: 2025-10-07 00:00 UTC through 2026-10-07 00:00 UTC (end exclusive); sample `all`.
- Venue: `kucoinfutures`; symbol contract tick, lot, minimum quantity, and multiplier came from `trading_symbols` at run time. Strategy parameters, sizing, leverage, fees, and exits used their individual defaults; no common risk profile was imposed.
- Runner: `npm run backtest -- --strategy <name> --symbol <symbol> --from 2025-10-07 --to 2026-10-07 --sample all --benchmark false` plus the venue symbol and contract-specification flags. The reports retain the exact run labels and acceptance results.
- London macro V3 used a recorded-news file containing `[]`. Its London branch can be compared with V2, but the New York news-confirmation branch has **not** been validated: an empty snapshot is not proof that no macro releases occurred.
- All 69 runs completed and wrote reports; none passed the complete repository acceptance gate. A completed run is not an endorsement for live trading. The report for each market explains its failed criteria.

Documentation-only proposals without registered backtest implementations (`adaptive-dca-v2`, `opening-range-breakout`, `gaussian-filter-trend`) were not run. The registry's `fixture-*` strategies were excluded because they are test fixtures rather than documented trading strategies.
