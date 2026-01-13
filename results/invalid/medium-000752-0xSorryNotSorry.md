# [000752] `placeBid` allows 1 second vesting periods, breaking the intended minimumvesting invariant
  
  
### Summary

The bidding flow allows users to choose a vesting period of exactly 1 second, which bypasses the intended “multiples of `vestingPeriodDivisor` (1 month)” constraint and lets bidders effectively obtain instantly vesting TVS, breaking the vesting design assumptions.

### Root Cause

In `placeBid`, the vesting period constraint is written so that `1` is always allowed, regardless of `vestingPeriodDivisor`:

```solidity
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    ...
>>  require(vestingPeriod > 0, Zero_Value()); 
    require(
>>      vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0,
        Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
    );
    ...
}
```

With `vestingPeriodDivisor` set to one month (2_592_000 seconds), this logic means:
- `vestingPeriod == 1` passes because `vestingPeriod > 0` and `vestingPeriod < 2` is true,  
- `vestingPeriod == N * vestingPeriodDivisor` also passes,  
but any other small value, like 10 minutes or 1 day, is rejected. So the only “short” vesting period allowed is 1 second, which is effectively instant vesting once claims are open.

### Impact

The whole tokenomics intent of “vesting periods are multiples of a base value (1 month)” is broken for bidding projects, because a bidder can choose 1 second and get a TVS that becomes fully claimable almost immediately, while others are bound to much longer schedules.

### Mitigation

Change the check so that all non-zero vesting periods must be at least one full divisor multiple:

```solidity
require(vestingPeriod > 0, Zero_Value());
require(
    vestingPeriod % vestingPeriodDivisor == 0,
    Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
);
```


  