# Trailing Geometric Grid v1 decisions

The public Botcamp page describes the intent and lifecycle, but not the exact
formulas. These decisions make the first implementation deterministic and are
kept separate so they can be replaced if the original source becomes
available.

Percent parameters use percentage points: `1` means one percent.

## TG-C01 - Grid increase

**Status:** Accepted

`upper = reference * (1 + step_interval / 100)`.

## TG-C02 - Grid decrease

**Status:** Accepted

`lower = reference / (1 + step_interval / 100)`. This is logarithmically
symmetric with TG-C01 and never produces a non-positive price.

## TG-C03 - Initial anchor

**Status:** Accepted

Use the exchange mid-price `(best_bid + best_ask) / 2`. Fall back to the last
price only when either side of the book is unavailable. Wait without placing an
order while no positive market price is available.

## TG-C04 - Anchor update

**Status:** Accepted

The anchor advances exactly one grid boundary after a completed entry: to the
lower boundary after a limit bid fill and to the upper boundary after a market
catch-up fill. Price movement alone never changes the anchor.

## TG-C05 - Step entry price

**Status:** Accepted

Use the geometric midpoint of the downward step between the lower boundary and
the current reference:
`entry = sqrt(lower_boundary * reference_price)`. This preserves equal
percentage distance between the resting bid and both ends of that downward
step. The separately calculated upper boundary remains the market catch-up
trigger; it is not an input to the resting bid price.

## TG-C06 - Market catch-up

**Status:** Accepted

Catch-up is a market buy when market price reaches the upper boundary of the
pending step. An existing limit bid is cancelled first. The same `step_id` is
reused, and a completed/cumulative fill always wins over a concurrent cancel or
catch-up attempt.

A partially filled entry is not cancelled for catch-up. It remains pending
until the same order completes; a partial order that is cancelled or becomes
ambiguous fails closed for reconciliation.

## TG-C07 - Maximum active steps

**Status:** Accepted

`max_steps = max(1, floor(ln(1 / (1 - stop_loss / 100)) / ln(1 + step_interval / 100)))`.
`stop_loss` must be below 100. Planned and open entry orders do not count until
they contain a completed base fill.

## TG-C08 - Uppermost step

**Status:** Accepted

Select the active step with the highest average entry price. Ties are resolved
by the smallest `step_index`, then `step_id`, so selection is deterministic.

## TG-C09 - Take-profit lifecycle

**Status:** Accepted

After a buy completes, place one resting limit sell for the filled base amount.
Target price is `average_fill_price * (1 + take_profit / 100)`, rounded upward.
The target is gross; fees are included in realized PnL, not added to the target.
A cancelled or missing TP is restored on a later tick. Partial entry fills do
not create a TP until the entry is complete. A partially filled exit is allowed
to complete on the same order, but is never cancelled and replaced
automatically because that would require cross-order cumulative fill accounting.

## TG-C10 - Stop-loss and rotation

**Status:** Accepted

`stop_loss` defines geometric capacity through TG-C07. When capacity is full,
the strategy waits until the next planned entry becomes executable. It then
cancels the selected uppermost step's TP and closes exactly that step with one
market sell before submitting the new entry. Exit work has priority over entry
work, and one exchange side effect is performed per tick.

## TG-C11 - Market type

**Status:** Accepted

The Telegram v1 runtime is long-only paper trading over the account's selected
futures market source. The selected unified symbol supplies the effective pair,
connector, contract multiplier and precision rules. The `trading_pair` and
`connector_name` stored in the shared template are legacy defaults and do not
override BTC, ETH, SOL or AVAX selected from Telegram.

## TG-C12 - Fees

**Status:** Accepted

Take-profit remains a gross percentage target. Entry and exit fees are converted
to quote value by the adapter and subtracted from realized PnL.

Telegram Paper selects the fee profile from the account's paper exchange:
Binance USD-M uses maker `0.02%` / taker `0.05%`; KuCoin Futures uses maker
`0.02%` / taker `0.06%`. Limit entry and take-profit fills use maker fees;
market catch-up, rotation and manual closes use taker fees. These standard-tier
assumptions do not include VIP discounts, funding, spread or slippage.

## TG-C13 - Precision and minimum order

**Status:** Accepted

Limit buy prices round down, sell prices round up, and base quantity always
rounds down. An order below minimum quantity or minimum notional is rejected
before submission. Dust remains visible in state and is never rounded into a
larger exit.

## TG-C14 - Orphan orders after restart

**Status:** Accepted

Orders carrying this strategy instance prefix but absent from persisted state
are cancelled. Orders outside the prefix are ignored. Persisted non-terminal
orders are queried before any retry or new order.

## TG-C15 - Tick scheduling

**Status:** Accepted

The host chooses the tick interval. Event handlers only update/persist state and
enqueue work. A tick performs at most one create/cancel exchange side effect;
read-only reconciliation may inspect multiple tracked orders.

## TG-C16 - Fixed contract sizing

**Status:** Accepted

Every new grid step uses `initialOrderContractCount`. The shared sizing rule
accepts integer or fractional contract amounts, validates the value against the
selected exchange symbol's contract step and minimum, then converts it once with
`base_quantity = contract_count * contract_multiplier`. The resulting contract
count and base quantity are frozen in the step ledger, so later price changes
alter entry notional but never alter that step's quantity or its TP quantity.

Fractional contract amounts are required on markets such as Binance BTC
(`0.001` contract step). KuCoin contracts retain their whole-number quantities.
Startup validates minimum quantity and minimum notional before the first entry;
it never rounds a configured amount upward silently.

`order_amount_in_quote` remains readable only for ledgers created before this
rule. A legacy pending step keeps its original quote-based size; all newly
planned steps use fixed contracts. The contract count is excluded from the grid
geometry fingerprint so an existing ledger can resume after this compatible
rollout. Interactive configuration changes are still refused while any filled
step remains open.
