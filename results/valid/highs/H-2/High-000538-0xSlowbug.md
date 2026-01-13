# [000538] Missing Memory Allocation for `newAmounts`
  
  ### Summary

The missing allocation of `newAmounts` in [`calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L175) will cause a full revert for all callers as the function will attempt to write to uninitialized memory when processing vesting flows.

### Root Cause

In `calculateFeeAndNewAmountForOneTVS()` the array `newAmounts` is declared but never allocated:

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

Attempting to write to an uninitialized memory array will always revert.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. User calls `mergeTVS()` or `splitTVS()` with any valid project & NFT IDs.
2. The function loads `mergedTVS.amounts` into memory and calls:

   ```solidity
   calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows)
   ```
3. Execution reaches the loop and attempts:

   ```solidity
   newAmounts[i] = ...
   ```
4. Because `newAmounts` was never allocated, the function reverts immediately.
5. `mergeTVS()` or `splitTVS()` reverts entirely.

### Impact

The users cannot merge their TVSs.
All calls to `mergeTVS()` or `splitTVS()` with at least one vesting flow will **always revert**, making the merge and split feature non-functional.

### PoC

_No response_

### Mitigation

```solidity
function calculateFeeAndNewAmountForOneTVS(
  uint256 feeRate,
  uint256[] memory amounts,
  uint256 length
)
  public
  pure
  returns (uint256 feeAmount, uint256[] memory newAmounts)
{
  // Allocate the memory array to avoid writing to uninitialized memory
  newAmounts = new uint256[](length);

  for (uint256 i; i < length;) {
    feeAmount += calculateFeeAmount(feeRate, amounts[i]);
    newAmounts[i] = amounts[i] - feeAmount;

  }
}
```
  