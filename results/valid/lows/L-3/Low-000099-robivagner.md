# [000099] `withdrawAllPostDeadlineProfits` goes through all existent bidding projects, which will cost a lot of gas
  
  ### Summary

Function [`withdrawAllPostDeadlineProfits`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L894) loops through all of the existent bidding projects to withdraw profits. The problem is that after a lot of bidding projects are launched, this will cost more and more gas.
```solidity
    function withdrawAllPostDeadlineProfits() external onlyOwner {
        uint256 _projectCount = biddingProjectCount;
@>        for (uint256 i; i < _projectCount; i++) {
            _withdrawPostDeadlineProfit(i);
        }
    }
```

### Root Cause

Function `withdrawAllPostDeadlineProfits` loops through the amount of ever existing bidding projects.

### Internal Pre-conditions

1. There have been a lot of bidding projects.

### External Pre-conditions

None

### Attack Path

None

### Impact

As time goes by, function `withdrawAllPostDeadlineProfits` will cost more and more gas to call. It will get to a point where the gas cost will be pretty significant.

### PoC

None

### Mitigation

Make a mapping of projects that have not been claimed yet.
  