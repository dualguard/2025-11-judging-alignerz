# [000110] Unbounded loops over the KOL address array will cause a denial of service for the protocol owner
  
  ### Summary

Unbounded loops over the KOL address array in AlignerzVesting.sol will cause a denial of service for the protocol owner and KOLs as the distribution function will exceed gas limits.

### Root Cause

In src/contracts/vesting/AlignerzVesting.sol, the function distributeRemainingRewardTVS iterates over the entire kolTVSAddresses array in a single transaction.


### Internal Pre-conditions

1-A Reward Project is initialized with a large number of KOLs (e.g., > 1000).


### External Pre-conditions

Nonw

### Attack Path

1-Project reaches the claim deadline.
2-Owner calls distributeRemainingRewardTVS.
3-Transaction reverts due to Out of Gas.

### Impact

KOLs cannot receive their allocated tokens. The tokens remain locked in the contract.


### PoC

```solidity
// See test_PoC_8_Vesting_Distribution_DoS in AlignerzAdditionalPoC.t.sol
function test_PoC_8_Vesting_Distribution_DoS() public {
    // ... Add 50 KOLs ...
    vesting.distributeRemainingRewardTVS(projectId);
    // Gas usage analysis shows linear growth leading to DoS
}
```

### Mitigation

Implement a batching mechanism allowing the owner to distribute to a subset of users (e.g., start to end index) in multiple transactions.

  