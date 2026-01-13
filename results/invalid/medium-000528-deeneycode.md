# [000528] [Medium] Insufficient Access Control in Distribution Functions Allows Unauthorized Token Distribution
  
  ### Summary

The NatSpec documentation states: /// @notice Allows the owner to distribute, 
but the `distributeRewardTVS` and `distributeStablecoinAllocation` functions lack proper access control, allowing any user to trigger token distribution after the claim deadline, which could potentially disrupt the intended distribution schedule.

### Root Cause

https://github.com/dualguard/2025-11-alignerz-deeneycode/blob/ebed8b27817fac595438b4150ffb591102369952/protocol/src/contracts/vesting/AlignerzVesting.sol#L540
https://github.com/dualguard/2025-11-alignerz-deeneycode/blob/ebed8b27817fac595438b4150ffb591102369952/protocol/src/contracts/vesting/AlignerzVesting.sol#L525
Missing `onlyOwner` modifier on critical distribution functions that should be restricted to the contract owner only.

### Internal Pre-conditions

- Reward project must exist and have passed its claim deadline
- KOL addresses must have unclaimed allocations in the project
- The contract must hold sufficient token balances

### External Pre-conditions

- Any address can call these functions
- The project's claim deadline must have elapsed

### Attack Path

No response

### Impact

Unauthorized users can force token distribution at unexpected times, potentially:
- Disrupting project's planned distribution schedule
- Bypassing owner's intended distribution timing strategy

### PoC

No response

### Mitigation

Add `onlyOwner` modifier to both distribution functions.

  