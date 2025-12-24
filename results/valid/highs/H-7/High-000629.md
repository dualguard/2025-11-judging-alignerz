# [000629] User will be unable to merge or split NFTs due to infinite loop in `calculateFeeAndNewAmountForOneTVS`
  
  ### Summary

Missing loop increment in [calculateFeeAndNewAmountForOneTVS ](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C5-L174C6) will cause an infinite loop for any user attempting to merge or split NFTs, as the function will never exit the for-loop and eventually revert with an out-of-gas error.

### Root Cause

in [calculateFeeAndNewAmountForOneTVS ](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C5-L174C6) the `i` variable never increases
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    internal
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    for (uint256 i; i < length;) {  //  Loop never increments i
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        // Missing i++;
    }
}
```
The loop will execute indefinitely with `i = 0` until the transaction runs out of gas.

### Internal Pre-conditions

1. User must own multiple NFTs (at least 2) from the same project
2. NFTs must have the same token type 


### External Pre-conditions

_No response_

### Attack Path

- User owns NFT #100 and NFT #200 from the same bidding/reward project
- User calls` mergeTVS(projectId, 100, [projectId], [200])`
- Function calls `calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, 1)` for NFT #100
- Loop starts with i = 0 and length = 1
- Loop executes: calculates fee, updates array
- Loop condition checks: i < length → 0 < 1 → true
- Loop repeats step 5 indefinitely (i never increments)
- Transaction runs out of gas and reverts
- User loses gas fees and cannot merge NFTs

### Impact

Users cannot merge or split NFTs at all, completely breaking the merge and split functionalities

### PoC

_No response_

### Mitigation

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    internal
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    for (uint256 i; i < length;) {
           feeAmount += calculateFeeAmount(feeRate, amounts[i]);
           newAmounts[i] = amounts[i] - feeAmount;
+         unchecked { ++i; }
    }
}
```
  