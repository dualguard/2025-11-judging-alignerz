# [000660] HIGH Missing loop increment on `continue` branches in `getUnclaimedAmounts` causes infinite loop and out-of-gas revert, blocking dividend calculations
  
  ### Summary

Missing loop counter increments on `continue` branches in `A26ZDividendDistributor.getUnclaimedAmounts` will cause an out-of-gas revert for the owner or any caller as the loop will never progress past the first index where `claimedFlows[i] == true` or `claimedSeconds[i] == 0` (the normal initial state), making dividend setup impossible.[

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147C8-L159C10


In `A26ZDividendDistributor.sol` at `getUnclaimedAmounts(uint256 nftId)`, the `for` loop over allocation indices only increments `i` in the `unchecked` block at the bottom of the loop body, but there are two `continue` statements that skip this increment: one when `claimedFlows[i] == true` and another when `claimedSeconds[i] == 0`.

```solidity
uint256 len = vesting.allocationOf(nftId).amounts.length;
for (uint i; i < len;) {
    if (claimedFlows[i]) continue;  // BUG: never increments i
    if (claimedSeconds[i] == 0) {
        amount += amounts[i];
        continue;  // BUG: never increments i
    }
    uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
    uint256 unclaimedAmount = amounts[i] - claimedAmount;
    amount += unclaimedAmount;
    unchecked {
        ++i;  // Only reached if neither continue executed
    }
}
```

When either `continue` branch is hit, execution jumps back to the loop condition `i < len` without incrementing `i`, so the same index is evaluated infinitely until the transaction runs out of gas.

### Internal Pre-conditions

1. At least one NFT needs to be minted with a non-empty allocation where `vesting.allocationOf(nftId).amounts.length > 0`.

2. For at least one index `i` in the allocation arrays, either `claimedFlows[i]` needs to be `true` OR `claimedSeconds[i]` needs to be `0` (the latter is the default initial state after minting an NFT).

3. The distributor's `token` address needs to be different from `vesting.allocationOf(nftId).token` so that the early `return 0` guard is bypassed and the loop is entered (this would occur if the owner misconfigures the token via `setToken`, or if there's a secondary bug like the inverted check).

4. Owner or any caller needs to invoke `getUnclaimedAmounts(nftId)` directly, or indirectly via `getTotalUnclaimedAmounts()`, `setAmounts()`, or `setUpTheDividends()`.



### External Pre-conditions

none

### Attack Path

1. Vesting project runs normally, bids are finalized, and at least one NFT is minted with a valid allocation containing `amounts.length > 0`.

2. At the time of minting, the allocation's `claimedSeconds` array is initialized with all elements set to `0` (normal initial state), and `claimedFlows` array has at least one element set to `false`.

3. Owner deploys or reconfigures `A26ZDividendDistributor` such that `address(token) != address(vesting.allocationOf(nftId).token)` for at least one NFT (this could be accidental misconfiguration or deliberate, and also occurs due to the inverted token check bug).

4. Owner calls `setUpTheDividends()` or `setAmounts()`, which triggers `_setAmounts()` and then `getTotalUnclaimedAmounts()`.

5. `getTotalUnclaimedAmounts()` iterates over all minted NFTs and calls `getUnclaimedAmounts(i)` for each owned NFT.

6. Inside `getUnclaimedAmounts(nftId)`, the token check does not trigger an early return (because of the misconfiguration or inverted check), so execution proceeds to the `for` loop.

7. On the first iteration where `claimedFlows[i] == false` and `claimedSeconds[i] == 0`, the code enters the `if (claimedSeconds[i] == 0)` branch, adds `amounts[i]` to `amount`, and executes `continue`.

8. The `continue` statement jumps back to the loop condition `i < len` without ever executing `++i`, so `i` remains at the same value.

9. The loop re-evaluates the same index `i` with the same conditions, entering the same `continue` branch indefinitely, consuming gas with no forward progress.

10. The transaction reverts with an out-of-gas error, preventing `getUnclaimedAmounts` and any dependent function (including `setUpTheDividends`) from completing.



### Impact

The owner cannot set up dividends or compute unclaimed amounts for any NFT that has the default initial state (`claimedSeconds[i] == 0`) or has fully claimed flows (`claimedFlows[i] == true`), as the function will always revert due to an out-of-gas error. This effectively bricks the dividend distribution mechanism for the majority of legitimate use cases (since NFTs are typically in their initial state when dividends are first configured), locking stablecoin rewards in the contract and denying TVS holders their dividends.

### PoC

_No response_

### Mitigation

_No response_
  