# [000791] Incorrect token comparison prohibits token distribution
  
  ### Summary

A wrong equality check in `getUnclaimedAmounts` will cause a complete inability to claim dividends for all NFT holders as any caller will always trigger a forced `return 0`, preventing the function from executing the vesting logic.


### Root Cause

In `A26ZDividendDistributor.sol` the condition:

```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
````

is implemented incorrectly.
The check should instead verify inequality (`!=`). As written, it guarantees that whenever the reward token matches the allocation token—which is the normal and expected case—the function immediately returns `0`, skipping all logic and effectively disabling dividend claims.

[Incorrect token check](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141)

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Any user calls `getUnclaimedAmounts(nftId)` to retrieve the unclaimed dividends.
2. The function evaluates `if (address(token) == address(vesting.allocationOf(nftId).token))`.
3. Since the dividend token is intended to match the allocation token, this condition is always true.
4. Execution returns `0` prematurely, preventing all further computation and halting the claim logic.
5. As a result, `unclaimedAmountsIn[nftId]` is never updated and dividends cannot be claimed.


### Impact

The NFT holders cannot retrieve any unclaimed dividends, resulting in a **complete loss of access to vested rewards**. All dividend distribution functionality is effectively bricked.

### PoC

_No response_

### Mitigation

Update the comparison to check whether the tokens are different and return 0.

  