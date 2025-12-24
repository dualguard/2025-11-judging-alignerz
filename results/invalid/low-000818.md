# [000818] `setTVSAllocation` does not validate vesting period - will lock tokens on period of zero (rather than instant unlock)
  
  `setTVSAllocation()` allows setting the parameters for a reward pool. The issue is that it allows a vesting period of zero.

TVSes of a zero vesting period will revert when claming, due to division by zero, rather than an instant unlock without vesting

```solidity
claimableAmount = (amount * claimableSeconds) / vestingPeriod;
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L993

This will lock funds in the contract.

It is also worth noting that `setTVSAllocation()` can only be called once.

  