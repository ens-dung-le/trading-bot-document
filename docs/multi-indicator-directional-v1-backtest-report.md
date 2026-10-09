# Multi-Indicator Directional v1 - Six-month research and backtest

Timeframe review (2026-09-16): see the [strategy portability specification](plan-multi-indicator-directional-strategy.md#timeframe-portability-review-2026-09-16).
The hourly training horizon, recorded results and decisions below are historical
evidence; they do not validate another timeframe or authorize model transfer.

## September 7 follow-up: per-symbol chronological training

The recommended second pass was completed without enabling paper or live mode.
It used every active KuCoin Futures symbol returned by the database rather than
a symbol constant in the strategy or research runner.

- Evaluation window: 2026-03-07 through 2026-09-07 UTC.
- Split: first 70% chronological training, last 30% untouched out of sample.
- Split timestamp: 2026-07-13 19:00 UTC for all four complete 1h series.
- Training target: next-1h log return.
- Model: ridge-fitted intercept plus separate short/long z-score bucket
  coefficients for each symbol.
- Level 2: disabled by unreachable thresholds while partial-reduce accounting
  parity remains unimplemented.
- Costs: KuCoin taker fee, half-spread, one tick slippage and recorded funding.
- Funding coverage: 553 events per symbol from 2026-03-07 00:00 UTC through
  2026-09-07 00:00 UTC.

| Symbol | Train correlation | Train direction | Forecast p95 | Forecast max | Full-period trades | OOS trades | OOS return | Gate |
|---|---:|---:|---:|---:|---:|---:|---:|---|
| BTC | 0.0333 | 53.5% | 2.95 bp | 5.35 bp | 0 | 0 | 0.00% | Fail |
| ETH | 0.0317 | 50.3% | 3.53 bp | 6.12 bp | 0 | 0 | 0.00% | Fail |
| SOL | 0.0421 | 51.5% | 5.43 bp | 11.98 bp | 0 | 0 | 0.00% | Fail |
| AVAX | 0.0558 | 49.4% | 8.44 bp | 17.56 bp | 1 | 0 | 0.00% | Fail |

The baseline round-trip friction is about 14 basis points before symbol tick
effects, and the configured entry gate requires a further 1.5 basis points of
net edge. BTC, ETH and SOL never forecast enough return to cover that floor.
AVAX crosses it once in the fitted period but never out of sample. Funding is
loaded and applied by the runner; OOS funding PnL is zero because no OOS
position was opened.

This is a stronger rejection than the shared-model loss. Per-symbol fitting
does not reveal an economically tradable next-hour edge. Building partial
reduction would reduce turnover only after a position exists; it cannot solve
the absence of qualifying forecasts. The lean stop condition is therefore
met: do not build partial-reduce infrastructure for this strategy yet, and do
not enable paper or live mode.

Stored OOS backtest run IDs: BTC 28, ETH 29, SOL 30 and AVAX 31. The complete
machine-readable artifact is
`output/multi-indicator-directional/research-2026-09-07.json`.

## Scope

- Exchange: KuCoin Futures.
- Period evaluated: 2026-03-07 through 2026-09-07 UTC.
- Warm-up data: from 2026-02-20.
- Timeframe: 1h.
- Starting balance: 10,000 USDT independently for each symbol.
- Leverage: 2x isolated.
- Active KuCoin symbols tested: BTC, ETH, SOL and AVAX perpetual futures.
- Model scope: one experimental shared normalized model.
- Botcamp private coefficients were unavailable; these results do not claim
  parity with the reference implementation.

## Cost assumptions

- Taker fee: 0.06% per fill, the standard public KuCoin Futures tier configured
  by the repository.
- Maker fee: 0.02%, retained in the cost model but not used by this market-order
  prototype.
- Half-spread: 0.01% per fill.
- Slippage: one exchange price tick per fill.
- Funding: the initial shared-model runs below predated the funding sync and did
  not include it. The follow-up above includes the recorded exchange history.

## Results

| Symbol | Run ID | Trades | Net PnL | Return | Profit factor | Sharpe | Max drawdown | Fees | Time in market |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| BTC | 20 | 152 | -379.77 USDT | -3.80% | 0.76 | -1.35 | -6.31% | 273.20 USDT | 61.83% |
| ETH | 19 | 162 | -342.34 USDT | -3.42% | 0.85 | -0.70 | -10.13% | 307.59 USDT | 60.49% |
| SOL | 17 | 167 | +56.49 USDT | +0.56% | 1.03 | 0.10 | -7.08% | 315.59 USDT | 62.30% |
| AVAX | 18 | 166 | -1,166.05 USDT | -11.66% | 0.57 | -2.63 | -14.73% | 308.90 USDT | 50.76% |

Equal-weight interpretation across four isolated 10,000 USDT allocations:

- combined net PnL: -1,831.67 USDT;
- combined return on 40,000 USDT: -4.58%;
- total trading fees: 1,205.28 USDT;
- profitable symbols: one of four, and SOL's 1.03 profit factor is too close to
  break-even to be evidence of an edge.

SOL was rerun as backtest run 21 with 200 random-entry simulations and the
repository's doubled-cost stress test:

- percentile against random entries: 100th percentile;
- doubled-cost return: -1.45%;
- conclusion: the signal contains some information in this sample, but the net
  result is too small and too cost-sensitive to deploy.

## Risk judgement

### Critical

1. The shared model does not transfer safely across symbols. Three of four
   symbols lose money and AVAX breaches the 12% drawdown gate.
2. The model has no demonstrated net edge after costs. BTC, ETH and AVAX are
   already gross-negative before explicit fees; this is not only a fee problem.
3. The coefficients are provisional because the reference model values are
   private. Enabling this version would trade an unvalidated substitute model.

### High

1. Trading friction dominates the result. The four runs pay 1,205.28 USDT in
   fees, while target reductions implemented as close-and-reopen add avoidable
   turnover compared with true partial reduction.
2. The only profitable symbol turns negative when trading costs are doubled.
   The apparent SOL edge therefore has no adequate cost-safety margin.
3. Funding history is absent. Long holding periods average roughly 13 to 18
   hours, so ignoring funding can materially overstate or understate PnL.
4. Time in market is 51% to 62%, leaving sustained exposure to regime changes
   despite weak risk-adjusted returns.

### Medium

1. The six-month sample has only 152 to 167 completed trades per symbol. It is
   useful for rejecting this parameter set but not for establishing a stable
   edge.
2. One fixed notional and one coefficient set are used across assets with very
   different volatility and microstructure.
3. The catastrophe stop prevents liquidation in these runs, but its wide 3% to
   12% envelope permits long drawdown recovery periods.

## Decision

Do not enable `multi-indicator-directional` version 1 for paper or live account
selection. Keep the migration template disabled and non-default.

Before another enablement review:

1. change or add forecast features using a new predeclared hypothesis; do not
   keep tuning these coefficients against the same six-month sample;
2. reserve a new untouched OOS window before evaluating that hypothesis;
3. require forecast magnitude to exceed real round-trip friction before
   considering execution work;
4. implement true partial reduction across backtest, paper, live and
   persistence only after a model produces enough qualifying OOS trades;
5. rerun random-entry, buy-and-hold, doubled-cost and monthly-regime comparisons
   after basic net expectancy becomes positive.
