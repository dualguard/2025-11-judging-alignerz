# [000224] Any user calling splitTVS() or mergeTVS() will always trigger an out-of-gas revert
  
  ### Summary

The missing loop increment in [`FeesManager.calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) will cause an out-of-gas revert for any call into this function, which in turn will cause a denial-of-service for all users of `AlignerzVesting.splitTVS()` and `AlignerzVesting.mergeTVS()` as any user will trigger an infinite loop and the transaction will always run out of gas.

### Root Cause

In [`FeesManager.sol::calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174) the for loop does not increment the loop variable `i`, causing an infinite loop and out-of-gas:

```js
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) 
public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            //@audit - no increment
        }
}
```

This function is called from [`AlignerzVesting.splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069) and [`AlignerzVesting.mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013), so any call to those functions inevitably hits this infinite loop and reverts with out-of-gas.

### Internal Pre-conditions

1. The protocol is deployed with the current `FeesManager.calculateFeeAndNewAmountForOneTVS()` implementation (no increment of i in the loop).
2. A project exists and has at least one TVS allocation with amounts.length > 0 (normal, expected state once vesting is used).
3. `splitTVS()` or `mergeTVS()` is exposed and callable by the TVS owner (as per current design).

### External Pre-conditions

None.
This bug is purely internal to the contracts and is chain-agnostic (reproduces on Polygon, Arbitrum, MegaETH, etc.).

### Attack Path

1. Any user who owns a TVS NFT calls `AlignerzVesting.splitTVS(projectId, percentages, splitNftId)` with valid parameters.
2. `splitTVS()` loads the `allocation.amounts` into amounts and `nbOfFlows = allocation.amounts.length`.
3. `splitTVS()` calls:
```js
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
```
4. Inside `calculateFeeAndNewAmountForOneTVS()`, the loop never increments i, so i stays 0 and i < length is always true.
5. The EVM repeatedly executes the loop body until gas is exhausted, then the transaction reverts with out-of-gas.
6. The same happens for `mergeTVS()`, which also calls `calculateFeeAndNewAmountForOneTVS()` with `mergeFeeRate`.

### Impact

The users cannot execute `splitTVS()` or `mergeTVS()` - TVS splitting and merging — core features promised by the protocol — are unusable.

### PoC

None.

### Mitigation

Increment `i`:

```js
unchecked { ++i; }
```
  