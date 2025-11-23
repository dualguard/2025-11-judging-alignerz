# [000424] Arbitrary caller can seize partial control of the `AlignerzVesting` contract’s owner privileges
  
  ### Summary

In the `AlignerzVesting` contract, the `distributeRewardTVS` and `distributeStablecoinAllocation` functions allow any arbitrary user to trigger `_claimRewardTVS` and `_claimStablecoinAllocation` for any reward project and any KOL.

However, based on the comments stating that these functions are intended to be called by the owner, it appears these functions should have access control restrictions (i.e. `onlyOwner`). Because these functions are publicly callable, an attacker can arbitrarily modify the expected flow of reward distribution. Specifically:

- A malicious user can manipulate the order in which a KOL receives their nftIds.

- An attacker can distribute rewards to KOL for any `RewardProject` in any sequence they choose, effectively allowing unauthorized control over reward allocation.

### Root Cause

In `AlignerzVesting.sol:525`, `AlignerzVesting.sol:540`, there is a lack of access control on a function that should be only called by the owner.

### Internal Pre-conditions

_No response_

### External Pre-conditions

Arbirary caller only needs gas fee to call the function.

### Attack Path

Attackes can just call the functions `distributeRewardTVS` and `distributeStablecoinAllocation` whenever they want.

For example, the owner may want to prevent the sale of a certain amount of tokens for a period of time by setting the `claimWindow` to 0 so that, after the `claimDeadline` has passed, the owner can choose the exact timing to distribute tokens to KOLs. However, a KOL who wants to sell immediately—or any arbitrary user who is holding a short position on the token—can call `distributeRewardTVS` themselves, forcing the distribution to the KOL earlier than intended.

### Impact

The scenario above enables them to induce the KOL to sell and profit from the price impact.

### PoC

_No response_

### Mitigation

The functions `distributeRewardTVS` and `distributeStablecoinAllocation` should be restricted with the `onlyOwner` modifier.
  