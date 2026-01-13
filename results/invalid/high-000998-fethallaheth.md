# [000998] DOS in Distribution Functions due to Unbounded Loops
  
  ### Summary

The unbounded loops in `distributeRemainingStablecoinAllocation` and `distributeRemainingRewardTVS` will cause a Denial of Service for the protocol. The functions iterate over all remaining unclaimed users to distribute tokens. If the number of users is large, the transaction will exceed the block gas limit, making the function unexecutable and forcing the owner to use emergency withdrawal methods.

### Root Cause

In `AlignerzVesting.sol:734` and `AlignerzVesting.sol:698`, the `for` loops are designed to run until the array of addresses is empty (`length > 0`).
```solidity
for (uint256 i = len - 1; rewardProject.kolStablecoinAddresses.length > 0;)
```
There is no mechanism to process this list in batches (e.g., using a `limit` or `offset`). The transaction must process all remaining users in a single block.



### Internal Pre-conditions

1. A large number of users (e.g., > 1000) have been allocated tokens but have not claimed them by the deadline.


### External Pre-conditions

None

### Attack Path

1. The claim deadline passes.
2. The owner attempts to call `distributeRemainingStablecoinAllocation` to push tokens to users.
3. The transaction reverts due to `Out of Gas`.
4. The intended distribution mechanism is broken.

### Impact

distribution dos, broken functionallity 

### PoC

_No response_

### Mitigation

Refactor the functions to allow batch processing. Instead of looping until `length > 0`, allow the caller to specify a number of users to process in one transaction.
  