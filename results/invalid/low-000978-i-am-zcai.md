# [000978] NFT traders will misallocate dividends from original holders through frontrunning [Low]
  
  
### Summary

The decoupled dividend snapshot mechanism will cause misallocation of dividends for original TVS holders as NFT traders will transfer high-weight NFTs between the `setAmounts()` and `setDividends()` calls to capture dividends computed from previous holders' unclaimed exposure.

### Root cause

In `A26ZDividendDistributor.sol:207-211` and `A26ZDividendDistributor.sol:214-218` the dividend distribution mechanism decouples snapshot data collection from beneficiary resolution, creating a frontrunning window where NFT ownership can change between weight calculation and dividend crediting.

### Internal pre-conditions

1. Admin needs to call `setAmounts()` to set dividend weights based on current TVS unclaimed amounts without immediately calling `setDividends()`
2. NFTs with high unclaimed amounts to exist in the system during the snapshot period

### External pre-conditions

None required.

### Attack path

1. Admin calls `setAmounts()` which stores `stablecoinAmountToDistribute`, `totalUnclaimedAmounts`, and populates `unclaimedAmountsIn[nftId]` based on current state
2. Trader monitors the blockchain and identifies NFTs with high `unclaimedAmountsIn` values from the snapshot
3. Trader purchases targeted high-weight NFTs from current holders before the admin calls `setDividends()`
4. Admin calls `setDividends()` which uses `safeOwnerOf(i)` to determine current ownership and credits dividends to the new NFT owners using the previously stored weights

### Impact

The original TVS holders suffer an approximate loss of their entitled dividend portion based on their historical unclaimed exposure. The attacking traders gain these misallocated dividends without bearing the underlying vesting risk during the measured period.

### Mitigation

Consider modifying the dividend distribution mechanism to ensure atomicity between snapshot capture and dividend crediting. One approach could be to store both weights and beneficiary addresses during the `_setAmounts()` phase, eliminating the dependency on current ownership during `_setDividends()`. Alternatively, consider encouraging exclusive use of the atomic `setUpTheDividends()` function and potentially deprecating the separate entry points to prevent operational mistakes that could expose this vulnerability window.

  