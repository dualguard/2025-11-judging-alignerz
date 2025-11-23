# [000166] Small amount of dust may be left after an NFT split
  
  ### Summary

During the execution of the [`Vesting::splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054) function, the `_computeSplitArrays` function is used to compute the new allocation for the corresponding NFT. The amounts of the allocations depend on the following formula:
```solidity
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
```
This division will result in small dust amount for possibly every allocation amount resulting small loss for the user when and NFT is splitted

### Root Cause

The root cause of the issue is the absence of dust amount handling

### Internal Pre-conditions

User wants to split his NFT

### External Pre-conditions

None

### Attack Path

1. User wants to split his NFT
2. The amount get rounded down resulting in loss for the user

### Impact

Small loss for every user who wants to split his NFT

### PoC

-

### Mitigation

Track the sum of the percentages and when they reach 100%, subtract the distributed amount from the full amount and assign the difference to the allocation
  