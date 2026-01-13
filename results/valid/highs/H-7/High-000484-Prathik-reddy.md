# [000484] No increment of counter in FeeManager::calculateFeeAndNewAmountForOneTVS() function
  
  ### Summary

The for loop inside the `calculateFeeAndNewAmountForOneTVS` function of the `FeeManager` contract does not increment the counter variable `i`. This will lead to an infinite loop, which results in an out of gas revert.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C4-L174C6

### Root Cause

Not incrementing the counter varibale `i`

```solidity

 function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }

```

### Internal Pre-conditions

NA

### External Pre-conditions

NA

### Attack Path

Some user calls mergeTVS() or splitTVS() function.

### Impact

The mergeTVS() & splitTVS() function rely on this function to calculate fees. If the function runs into an infinite loop, these functions will also fail to execute properly. This can lead to a denial of service for users trying to merge or split their TVS tokens.

### PoC

NA

### Mitigation

Increment the counter variable `i` at the end of each iteration of the for loop.
  