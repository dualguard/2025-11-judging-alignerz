# [000825] Wrong vesting period check on `placeBid` and `updateBid`, a vesting period of 1 will bypass any `vestingPeriodDivisor` requirements
  
  ### Summary

Wrong vesting period check on `placeBid` and `updateBid`, a vesting period of 1 will bypass any `vestingPeriodDivisor` requirements


### Root Cause


When placing a bid on bid projects, there is a requirement that is the vesting period must be divisible by `vestingPeriodDivisor`. This check is enforced on `placeBid()` and `updateBid()`, and enforced at bid creation.

However, the check is enforced wrongly. Consider the check on `placeBid()`:

```solidity
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external { 
    // ...
    require (vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value());
}
```

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L724

Note that a `vestingPeriod` of 1 will always evaluate the first clause to true, and bypass the check entirely, regardless of what the `vestingPeriodDivisor` is (by default it is set to 1 month).

The check on `updateBid()` has the same issue:

```solidity
function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
    // ...
    if (newVestingPeriod > 1) { // @audit newVestingPeriod = 1 will always pass even though it's not supposed to
        require(
            newVestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
        );
    }
    // ...
}
```

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

Any bids of vesting period = 1 can be placed, regardless of what `vestingPeriodDivisor` is set to


### Impact

The `vestingPeriodDivisor` requirement can be bypassed

### PoC

_No response_

### Mitigation


On `placeBid`, the check should be 

```solidity
require (vestingPeriodDivisor < 2 || vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value());
```

On `updateBid`, the check should be

```solidity
if (vestingPeriodDivisor > 1) {
     // ...
}
```

  