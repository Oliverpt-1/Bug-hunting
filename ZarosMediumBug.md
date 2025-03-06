# Faulty offchain orders cause revert for valid orders

## Summary

In `SettlementBranch::fillOfchainOrders`, there is a for loop which goes through each offchain order within a batch. It calculates bid, ask, and all necessary accounting to fill the order. However, there are checks which result in a revert in the case of a faulty order. Even if there is a batch order of size 50 and one user submits an order with size delta 0, all 50 orders would be rejected.

## Vulnerability Details

```solidity
// iterate through off-chain orders; intentionally not caching
// Length as reading from calldata is faster
for (uint256 i; i < offchainOrders.Length; i++) {
    ctx.offchainOrder = offchainOrders[i];

    // enforce size > 0
    if (ctx.offchainOrder.sizeDelta == 0) {
        revert Errors.ZeroInput("offchainOrder.sizeDelta");
    }
}
```

Within `SettlementBranch::fillOfchainOrders`, the for loop is used to sift through the off-chain orders and determine validity and to fill the order. However, the checks such as lines 7 through 9 above cause a function to REVERT in the case of a faulty order. This creates a direct denial-of-service to all other orders within the batch, in turn creating an opening for manipulation via a malicious actor.

## Impact

Malicious actor can cancel all off-chain orders within a batch.

## Text-POC

```markdown
1. 25 users create valid off-chain orders.
2. 1 malicious actor creates an invalid off-chain order with a `sizeDelta` of zero.
3. When the for loop reaches the malicious actor's order, the `if` statement which checks if `ctx.offchainOrder.sizeDelta == 0` will be triggered.
4. This will cause a revert, reverting all valid filled off-chain orders which took place and any valid off-chain orders ahead of the malicious one from being filled.
```

## Tools Used

```markdown
- Manual Review
```

## Recommendations

```markdown
Instead of using `revert`, the protocol could use `continue` to ensure that the order is skipped over but other orders are not invalidated.
```
