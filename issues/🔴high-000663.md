# [000663] Unbounded flow accumulation enables DoS of `getUnclaimedAmounts()`.
  
  ### Summary

The absence of limits on the number of flows per TVS combined with the ability to split into unlimited parts and merge unlimited TVS will cause users to create TVS with excessive flow counts. It makes any iteration over the flows of a TVS unusable. As `getUnclaimedAmount()` iterates over every flow of one TVS, the function can revert. 

### Root Cause


https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002
The protocol does not impose any limit on the number of vesting flows (arrays in Allocation) that a TVS can contain.

https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L146-L147
`getUnclaimedAmounts()` iterate over all flows.
With excessively large flow arrays, these iterations exceed the block gas limit, causing systemic reverts.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- Attacker splits an NFT into many sub-NFTs
- Attacker merges all NFTs back together
- The protocol calls `setUpTheDividends()`
    -> It reverts as `getUnclaimedAmounts()` is called

### Impact

- The protocol can't compute total unclaimed amounts.
- The users can't claim their dividends
- Users having a TVS with a large number of flow may not be able to call `claimTokens()`

### PoC

_No response_

### Mitigation

Implement a cap to the number of flows per NFT/TVS.

  