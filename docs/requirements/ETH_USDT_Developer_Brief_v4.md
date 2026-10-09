# DEVELOPER BRIEF
## ETH/USDT Strategy Sourcing & Batch Backtesting Protocol (v4)

**TO:** Lead Quantitative Developer
**TARGET ASSET:** ETH/USDT Perpetual Futures (KuCoin)
**FROM:** Project Lead (Luke Bowen)
**PRIMARY STACK:** Python / Freqtrade / QuantConnect / CCXT / Pine Script v5

---

## PRIMARY OBJECTIVE

Shift immediately from manual strategy builds to an automated **Batch Strategy Sourcing, Screening & Backtesting Pipeline** capable of evaluating 15+ open-source strategies per day on KuCoin futures order book feeds.

---

## 1. Process Shift & Sourcing Workflow

Manual coding of unverified retail concepts or custom indicators is a major development bottleneck. Most unvetted strategies fail live due to lookahead bias, extreme curve-fitting, or sensitivity to exchange taker fees.

**Strategic Mandate (Single-Pair Focus):** We are strictly standardizing the pipeline on ETH/USDT KuCoin Futures first. Sourcing and backtesting across ETH provides deep order book liquidity, tight spreads, and consistent volatility regimes. Expanding to secondary pairs (BTC, SOL, AVAX) will occur only after a candidate strategy achieves our performance benchmarks on ETH.

### The 4-Step Sourcing & Testing Pipeline

- **Step 1 — Source:** Extract open-source, production-tested Python or Pine Script strategies directly from established repositories (Freqtrade, QuantConnect, TradingView Public Library, arXiv papers).
- **Step 2 — Screen (5-Minute Junk Filter):** Pass candidate logic through a strict 4-point disqualification check prior to writing code or initiating data ingestion.
- **Step 3 — Batch Backtest:** Execute automated CLI backtests across 3 years of 5m/15m/1h/4h ETH/USDT futures data using KuCoin execution models via CCXT.
- **Step 4 — Hyperoptimize:** Run parameter optimization (e.g., Freqtrade hyperopt) on top candidates to fine-tune ROI, stop-loss, and signal sensitivity.

---

## 2. The 5-Minute "Junk Filter" Protocol

Run every candidate strategy through this pre-screening checklist before writing code. Disqualify immediately if it fails any single rule:

| # | Filter Criterion | Disqualification Threshold | Reason / Core Logic |
|---|---|---|---|
| 1 | Lookahead & Repainting | Disqualify — uses future bars, unclosed candles, or non-causal indicators (ZigZag, center pivots) | Signals must trigger strictly on closed candles (`barmerge.lookahead_off` or `[1]` offset) |
| 2 | Over-Fitting | Disqualify — requires > 4–5 tuned indicators or variables | Excessive parameter counts cause immediate out-of-sample breakdown and failure in live regimes |
| 3 | Static Risk Bounds | Disqualify — uses static % or fixed dollar stop-losses without volatility adjustment | Crypto perpetuals require dynamic volatility stops (e.g., Average True Range / ATR multipliers) |
| 4 | High Fee Drag | Disqualify — sub-15m micro-scalping targeting < 0.3% price move per trade | Exchange taker fees (0.04%–0.06%) and slippage completely destroy expected value |

---

## 3. Plug-and-Play Multi-Source Ecosystem

| Source / Repository | Environment / Format | Integration & Porting Workflow |
|---|---|---|
| Freqtrade Repositories | Python Native | Clone `freqtrade-strategies`. Audit & run production scripts: BinHV2, EMAPill, Nostradamus, ClucMay720. |
| KuCoin CCXT Engine | Python / CCXT | Pull historical candles across KuCoin futures order books to verify liquidity, funding rate variance, and spread depth. |
| TradingView Library | Pine Script v5 | Filter by `strategy()` wrapper from verified quant authors. Confirm zero-repainting in Deep Backtest mode, then convert logic into Python. |
| QuantConnect / SSRN | Python / C# / Papers | Port institutional anomaly models: Time-Series Momentum (TSMOM), ATR Volatility Breakouts, and Funding Rate Arbitrage. |

---

## 4. KuCoin Batch Execution Commands

Execute these standard CLI routines to pull 3 years of KuCoin futures historical data and execute strategy batch runs:

```bash
# 1. Download 3 Years Data from KUCOIN Futures (5m, 15m, 1h, 4h)
freqtrade download-data \
--exchange kucoin \
--pairs ETH/USDT:USDT \
--timeframes 5m 15m 1h 4h \
--trading-mode futures \
--days 1095

# 2. Strategy Ingestion (Clone strategy repos)
cd user_data/strategies
git clone https://github.com/freqtrade/freqtrade-strategies.git repo_strategies
cp repo_strategies/user_data/strategies/BinHV2.py .
cp repo_strategies/user_data/strategies/EMAPill.py .
cp repo_strategies/user_data/strategies/Nostradamus.py .

# 3. Batch Backtest across Candidate Strategies
freqtrade backtesting \
--config user_data/config-futures.json \
--strategy-list BinHV2 EMAPill Nostradamus \
--timerange 20230101-20260801 \
--timeframe 15m \
--export trades \
--export-filename user_data/backtest_results/batch_test_01.json

# 4. Hyperoptimize Top Performer
freqtrade hyperopt \
--config user_data/config-futures.json \
--strategy BinHV2 \
--hyperopt-loss SharpeHyperOptLoss \
--epochs 100 \
--spaces buy sell roi stoploss
```

---

## 5. KuCoin Futures Config Template (`config-futures.json`)

```json
{
  "trading_mode": "futures",
  "margin_mode": "isolated",
  "stake_currency": "USDT",
  "stake_amount": "unlimited",
  "max_open_trades": 3,
  "dry_run": true,
  "fee": 0.0006,
  "margin_detail": true,
  "exchange": {
    "name": "kucoin",
    "key": "",
    "secret": "",
    "ccxt_config": {
      "enableRateLimit": true
    },
    "pair_whitelist": [
      "ETH/USDT:USDT"
    ]
  },
  "entry_pricing": {
    "price_side": "same",
    "use_order_book": true
  },
  "exit_pricing": {
    "price_side": "same",
    "use_order_book": true
  }
}
```

---

## 6. Acceptance Performance Benchmarks

All candidate strategies must meet or exceed these thresholds across both In-Sample (70%) and Out-of-Sample (30%) datasets before proceeding to live paper trading:

| Metric | Threshold |
|---|---|
| Sharpe Ratio | > 1.8 |
| Profit Factor | > 1.5 |
| Maximum Drawdown | < 15% |
| Minimum Trade Sample Size | > 600 completed trades |
| Leverage Boundary | 2x – 5x Isolated Margin |
| Out-of-Sample Performance Decay | < 20% degradation |

---

*ETH/USDT Strategy Backtesting Protocol (v4) • Confidential*
