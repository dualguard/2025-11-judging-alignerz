# [000967] Array Length Mismatch Leading to DoS in Distribution Functions
  
  ### Summary

The `distributeRewardTVS` and `distributeStablecoinAllocation` functions use the length of one array to iterate over another, which leads to a DoS when the lengths do not match. When `kol.length < rewardProject.kolTVSAddresses.length`, a non-existent index is accessed and the transaction is reverted.

### Root Cause

The functions use the length of the internal array, but the iteration is performed on the kol array passed as a parameter:
```solidity 
  function distributeRewardTVS(
        uint256 rewardProjectId,
>>>        address[] calldata kol
    ) external {
>>>        uint256 len = rewardProject.kolTVSAddresses.length; 
>>>        for (uint256 i; i < len; ) {
            _claimRewardTVS(rewardProjectId, kol[i]); 
            unchecked {
                ++i; 
            }
        }
    }
```

A similar problem in `distributeStablecoinAllocation`

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. You need to call this function and pass the parameter `kol < kolTVSAddresses`

### Impact

DoS when kol.length < rewardProject.kolTVSAddresses.length: accessing kol[i] outside the bounds causes a revert, blocking reward distribution.

### PoC

_No response_

### Mitigation

Change the loop

```solidity 
 for (uint256 i; i < kol.length; )
```
  