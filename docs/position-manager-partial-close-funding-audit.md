# Security Finding: Partial Position Reduction Can Fully Close a Profitable Position Due to Inconsistent Funding Accounting

## Finding Information

**Severity:** High  
**Status:** Confirmed through verified source code and on-chain execution trace  
**Affected contract:** `PositionManager`  
**Contract address:** `0xCfd351b9C57e4C14E7F41EfFc7FF000401ac5cD9`  
**Network:** Flare  
**Affected functions:**

- `PositionManager._decreasePosition()`
- `PositionManager.getPnL()`

## Executive Summary

When a trader partially reduces a position, the contract calculates price PnL using only the portion being closed. However, it calculates and realizes the funding fee using the size of the entire position.

This inconsistent accounting can cause a small partial-close order to produce a large negative PnL even when the full position is substantially profitable.

If the resulting PnL is less than or equal to the negative value of the position's entire margin, `_decreasePosition()` converts the partial close into a full position closure, sets the remaining position size and margin to zero, and credits the entire margin to the protocol as trader loss.

This behavior occurred on-chain in transaction:

`0x74332079598c7c660d3c10258146065e4949bed4142347646f276575e99e382d`

The affected long position had an entry price of approximately `$1,737.19` and was reduced when ETH was approximately `$2,384.18`. Although the full position had a positive net PnL of approximately `1,005,424.86 WFLR` after funding, a reduce-only order for only `10,000 WFLR` caused the contract to record a loss equal to the full position margin of `288,353.08 WFLR`.

## Technical Details

### 1. Price PnL Is Calculated Using the Partial Close Size

In `getPnL()`, price PnL is calculated using `_executedOrderSize`:

```solidity
if (_isLong) {
    pnl =
        (int256(_executedOrderSize) *
            (int256(_price) - int256(_positionPrice))) /
        int256(_positionPrice);
} else {
    pnl =
        (int256(_executedOrderSize) *
            (int256(_positionPrice) - int256(_price))) /
        int256(_positionPrice);
}
```

For a partial close, `_executedOrderSize` is only the amount requested by the close order:

```solidity
uint256 executedOrderSize =
    position.size > order.size ? order.size : position.size;
```

### 2. Funding Is Calculated Using the Entire Position Size

The same `getPnL()` call receives the full `position.size` separately:

```solidity
(int256 pnl, int256 fundingFee) = getPnL(
    order.asset,
    order.market,
    position.isLong,
    price,
    position.price,
    executedOrderSize,
    position.size,
    position.fundingTracker,
    position.accumulatedFundingFee
);
```

Funding is then calculated using `_size`, which represents the entire position:

```solidity
fundingFee =
    (int256(_size) *
        (currentFundingTracker - _fundingTracker)) /
    (int256(BPS_DIVIDER) * int256(UNIT));

if (_accumulatedFundingFee != 0) {
    fundingFee += _accumulatedFundingFee;
}
```

For a long position, the full funding fee is subtracted from the price PnL of only the partially closed amount:

```solidity
if (_isLong) {
    pnl -= fundingFee;
}
```

Therefore, the calculation is effectively:

```text
Partial-close PnL =
    Price PnL on the partial size
    - Funding fee on the full position size
```

This creates a material mismatch between the two components.

## On-Chain Proof of Impact

### Position State Before Execution

The execution trace shows the following position:

```text
Direction:          Long
Position size:      3,564,000 WFLR
Position margin:    288,353.079249034218196573 WFLR
Average entry:      1,737.1861797
Execution price:    2,384.1849015
Accumulated period: approximately 70 days
```

The raw position state reports:

```text
position.isLong = true
```

The transaction also emitted:

```text
DecrementOI.isLong = true
```

This confirms that the underlying position was long.

### Reduce-Only Order

Order `#4490` was submitted by the affected wallet through transaction:

`0x92b9e44f8fd31e2244f98f16b1a74e3fd391eaf1aa85a37009b8090abf2b9485`

Its parameters included:

```text
size:          10,000 WFLR
isLong:        false
isReduceOnly:  true
margin:        0
```

The order direction was short because it was reducing an existing long position. It did not represent a separate short position.

### PnL Calculated for Order #4490

The price return was approximately:

```text
(2,384.1849015 - 1,737.1861797) / 1,737.1861797
= 37.24406338%
```

Price PnL on the requested `10,000 WFLR` reduction:

```text
10,000 x 37.24406338%
= +3,724.406338 WFLR
```

Funding charged on the full `3,564,000 WFLR` position:

```text
-321,953.562149 WFLR
```

Result returned by `getPnL()`:

```text
+3,724.406338
-321,953.562149
----------------
-318,229.155811 WFLR
```

This result was independently confirmed by calling `getPnL()` at the historical execution block using the position and order parameters from the raw trace.

## Forced Full Closure

The calculated loss exceeded the entire position margin:

```text
Calculated PnL:  -318,229.155811 WFLR
Position margin:  288,353.079249 WFLR
```

The following branch was therefore executed:

```solidity
if (pnl <= -1 * int256(uint256(position.margin))) {
    pnl = -1 * int256(uint256(position.margin));
    executedPositionMargin = position.margin;
    executedOrderSize = position.size;
    position.size = 0;
    position.margin = 0;
}
```

Although the user requested a reduction of only `10,000 WFLR`, the contract changed `executedOrderSize` to the entire `3,564,000 WFLR` position and set:

```text
position.size   = 0
position.margin = 0
```

The capped loss was then credited to the protocol:

```solidity
store.creditTraderLoss(
    order.user,
    order.asset,
    order.market,
    absPnl
);
```

The resulting events confirm:

```text
PositionDecreased.pnl:            -288,353.079249 WFLR
PositionDecreased.positionMargin:  0
PositionDecreased.positionSize:    0
PoolPayIn.amount:                  288,353.079249 WFLR
RemovePosition:                    emitted
```

No collateral remained to return to the user.

## Full-Close Comparison

If price PnL were calculated using the full position size, the outcome at the same execution price would have been:

```text
Full-position price PnL:
3,564,000 x 37.24406338%
= +1,327,378.418871 WFLR

Less full funding:
1,327,378.418871 - 321,953.562149
= +1,005,424.856722 WFLR
```

Therefore, the full position was economically profitable after funding.

The loss occurred because the contract combined:

```text
Price PnL from only 10,000 WFLR
```

with:

```text
Funding from the full 3,564,000 WFLR position
```

## Non-Monotonic Partial-Close Behavior

This issue produces highly counterintuitive behavior where reducing the requested close size can cause a more severe result.

For the affected position, approximate behavior at the same price was:

| Requested close size | Price PnL | Funding charged | Expected contract behavior |
|---:|---:|---:|---|
| `10,000` | `+3,724` | `-321,953` | Full position forcibly closed |
| `356,400` | `+132,738` | `-321,953` | Reverts with `!not-enough-margin-for-pnl` |
| `3,564,000` | `+1,327,378` | `-321,953` | Approximately `+1,005,425 WFLR` net profit |

The approximate size ranges for this specific position were:

```text
Requested size <= 90,217 WFLR:
    PnL becomes lower than the negative full margin.
    The entire position is closed.

Requested size between 90,217 and 710,169 WFLR:
    The partial close generally reverts because the negative PnL
    exceeds the proportional margin assigned to the partial close.

Requested size above approximately 710,169 WFLR:
    The partial close may succeed.

Requested size above approximately 864,443 WFLR:
    Price PnL becomes sufficient to offset the full funding fee.
```

These thresholds exclude minor differences caused by fees and integer rounding.

## Previous Failed Partial-Close Attempt

Immediately before order `#4490`, the same wallet submitted order `#4489` through transaction:

`0xa5bf05279ec0855a846860f690dc24bd41174c7f7efd2500d61cefe236e8878f`

Order `#4489` attempted to reduce:

```text
356,400 WFLR
```

The keeper execution was cancelled with:

```text
!not-enough-margin-for-pnl
```

The cancellation occurred in transaction:

`0xc259e7191b4247585731a5b1256656aef7f154184a037010e401158efa9c088b`

The wallet subsequently submitted the smaller `10,000 WFLR` order. Instead of reducing the required loss, the smaller order caused the calculated PnL to cross the full-margin threshold and triggered complete position removal.

## Impact

The issue can cause:

- Complete loss of position collateral during a normal reduce-only operation.
- Forced closure of a profitable position.
- Partial-close transactions reverting even though the position has sufficient full-position unrealized profit.
- Smaller close orders producing worse outcomes than larger close orders.
- Users being unable to safely reduce exposure incrementally.
- Material differences between frontend PnL expectations and on-chain settlement.
- Unexpected pool gains at the expense of profitable traders.

The issue does not require adverse market movement. It occurs due to inconsistent accounting bases inside `getPnL()`.

No evidence was found that an unrelated third party created order `#4490`. The transaction was submitted by the affected wallet. However, a frontend or integration that recommends progressively smaller close sizes after a revert may unintentionally trigger the full-loss branch.

## Secondary Observation: Misleading Position Direction Event

`PositionDecreased` emits:

```solidity
order.isLong
```

instead of the actual position direction:

```solidity
position.isLong
```

For a reduce-only order closing a long position, `order.isLong` is false. As a result, the emitted event appears to describe a short position even though the underlying position was long.

This does not directly cause the loss but makes incident investigation and off-chain accounting more difficult.

The event should separately expose:

```text
positionIsLong
orderIsLong
requestedSize
executedSize
forcedFullClose
```

## Recommended Remediation

### Option A: Realize Funding Proportionally

For a partial close, calculate the funding attributable to the closed portion:

```solidity
int256 totalFundingFee = ...;

int256 realizedFundingFee =
    (totalFundingFee * int256(executedOrderSize)) /
    int256(position.size);
```

The remaining unpaid funding should remain associated with the remaining position:

```solidity
position.accumulatedFundingFee =
    totalFundingFee - realizedFundingFee;

position.fundingTracker =
    fundingTracker.getFundingTracker(order.asset, order.market);
```

Price PnL and funding would then use the same proportional basis.

### Option B: Settle Full Funding Separately

If the intended protocol design is to settle all accrued funding whenever any reduction occurs, full-position funding should not be combined only with partial price PnL.

Instead:

1. Calculate full-position unrealized price PnL.
2. Evaluate the health of the full position using full unrealized PnL and full funding.
3. Settle funding against the full position margin or full unrealized equity.
4. Calculate the partial-close payout separately.
5. Force a full close only when the full position is genuinely insolvent.

### Required Safety Checks

The contract should not silently execute more size than the user requested unless the operation is an explicit liquidation.

A normal reduce-only order should enforce:

```solidity
require(
    executedOrderSize <= requestedOrderSize,
    "unexpected-full-close"
);
```

If the protocol intentionally supports forced closure during a reduce operation, it should require an explicit liquidation condition and emit a dedicated event.

## Recommended Tests

The following tests should be added:

1. A profitable long position with accrued funding performs a small partial close.
2. A profitable short position with accrued funding performs a small partial close.
3. Closing `1%`, `10%`, `50%`, and `100%` at the same price produces economically consistent results.
4. Sequential partial closes produce approximately the same aggregate result as one full close, excluding time-dependent funding and fees.
5. Reducing the requested close size must not unexpectedly change a normal reduction into a complete position closure.
6. A partial close with insufficient proportional margin should not ignore the full position's unrealized profit.
7. Event data must distinguish the position direction from the closing-order direction.
8. The requested size, actual executed size, and forced-close reason must be emitted separately.

A useful accounting invariant is:

```text
PnL(close A) + PnL(close B)
approximately equals PnL(close A + B)
```

This assumes the execution price, funding tracker, and fees remain unchanged.

The current implementation violates this property because the full accrued funding fee is applied to the first successful partial realization while price PnL remains proportional to the requested close size.

## Evidence

### Affected Contract

https://flare-explorer.flare.network/address/0xCfd351b9C57e4C14E7F41EfFc7FF000401ac5cD9?tab=contract

### Position-Closing Transaction

https://flare-explorer.flare.network/tx/0x74332079598c7c660d3c10258146065e4949bed4142347646f276575e99e382d?tab=logs

### Order #4490 Submission

https://flare-explorer.flare.network/tx/0x92b9e44f8fd31e2244f98f16b1a74e3fd391eaf1aa85a37009b8090abf2b9485

### Previous Order #4489 Submission

https://flare-explorer.flare.network/tx/0xa5bf05279ec0855a846860f690dc24bd41174c7f7efd2500d61cefe236e8878f

### Order #4489 Cancellation

https://flare-explorer.flare.network/tx/0xc259e7191b4247585731a5b1256656aef7f154184a037010e401158efa9c088b

## Conclusion

The affected position was long and substantially profitable based on market price movement. The recorded loss did not result from the ETH price moving against the trader.

The loss resulted from applying the full-position funding fee to the price PnL of a very small partial reduction. This produced an artificial negative PnL below the position's full margin and caused `_decreasePosition()` to convert the partial reduce-only order into a complete position closure.

The issue should be treated as a high-severity accounting flaw because it can cause direct and irreversible loss of user collateral during an ordinary position-management operation.
