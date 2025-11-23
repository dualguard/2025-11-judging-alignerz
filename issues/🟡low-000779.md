# [000779] Missing Duplicate Address Validation in Reward Allocation Functions
  
  ### Summary

Missing validation checks in `setTVSAllocation` and `setStablecoinAllocation` will cause state inconsistency and incorrect reward tracking for the protocol as the owner can inadvertently or intentionally provide duplicate addresses that overwrite and duplicate entries in address tracking arrays.

### Root Cause

In AlignerzVesting.sol, the `setTVSAllocation` function and `setStablecoinAllocation` function lack validation to prevent duplicate addresses in the `kolTVS` and `kolStablecoin` input arrays. When duplicate addresses are provided, the mapping `kolTVSRewards[kol]` and `kolStablecoinRewards[kol]` will be overwritten with the latest amount, while the address tracking arrays `kolTVSAddresses` and `kolStablecoinAddresses` will contain duplicate entries, causing a mismatch between the arrays and the reward mappings.

[push of duplicate address](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L466)

[overwriting existing memory](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L468)


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Owner calls `setTVSAllocation` with duplicate addresses in the `kolTVS` array (e.g., [0xABC, 0xDEF, 0xABC])
2. The loop processes each address sequentially, and for the duplicate address 0xABC, it first sets `kolTVSRewards[0xABC]` to the first amount, then overwrites it with the second amount
3. Both instances of 0xABC are appended to `kolTVSAddresses`, creating a duplicate entry in the array
4. The `kolTVSIndexOf` mapping is updated to the latest index (i=2), pointing to the second occurrence
5. Similar behavior occurs in `setStablecoinAllocation`


### Impact

The protocol suffers state inconsistency where the `kolTVSAddresses` and `kolStablecoinAddresses` arrays contain duplicate entries that do not correspond to actual unique reward allocations. This causes operational issues when iterating over these arrays or relying on their length for accounting. The duplicate mapping will store only the last provided amount for each duplicate address, potentially losing or overwriting intended reward data.

### PoC

_No response_

### Mitigation

Add duplicate address validation before processing allocations.
  