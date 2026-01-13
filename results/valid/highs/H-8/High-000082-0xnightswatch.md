# [000082] Missing loop increment causes infinite loop in getUnclaimedAmounts
  
  ### Summary

The missing loop increment in `getUnclaimedAmounts()` at lines 148-151 will cause an infinite loop and permanent DoS of dividend distribution for the protocol as NFT holders with unclaimed allocations (`claimedSeconds[i] == 0`) will trigger the bug when the owner calls `setUpTheDividends()`. 

### Root Cause

In [A26ZDividendDistributor.sol:148-151](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L148-L151), the loop continues without incrementing i when `claimedSeconds[i] == 0`, causing an infinite loop. 

The code has three paths through the loop:

1. `if (claimedFlows[i]) continue;`  - missing increment (line 148)
2. `if (claimedSeconds[i] == 0) { amount += amounts[i]; continue; }`; missing increment (lines 149-151)
3. Normal path with `unchecked { ++i; }` 

Only path 3 increments i, while paths 1 and 2 execute continue without incrementing, causing the loop to check the same index forever.

### Internal Pre-conditions

1. At least one NFT needs to exist with `claimedSeconds[i] == 0` (unclaimed allocation)
2. Owner needs to call `setUpTheDividends()` to trigger dividend distribution

### External Pre-conditions

None

### Attack Path

1. Protocol operates normally with users minting NFTs through `claimNFT()` - these NFTs start with `claimedSeconds[0] == 0` 
2. Owner calls `setUpTheDividends()` to distribute dividends
3. `_setAmounts()` internally calls `getTotalUnclaimedAmounts()` 
4. `getTotalUnclaimedAmounts()` iterates through all NFTs and calls `getUnclaimedAmounts(nftId)` for each
5. When `getUnclaimedAmounts()` encounters an NFT with `claimedSeconds[i] == 0`, it enters the infinite loop at line 149-151
6. Transaction runs out of gas and reverts, making dividend distribution permanently impossible

### Impact

The protocol cannot distribute dividends to any TVS holders. The owner is permanently unable to call `setUpTheDividends()` as the transaction will always run out of gas when encountering NFTs with unclaimed allocations.
Since NFTs typically start with `claimedSeconds[0] == 0` (no claims yet), this bug manifests immediately upon the first dividend distribution attempt, rendering the entire dividend distribution system completely unusable and locking all stablecoin dividends in the contract permanently.

### PoC

N/A

### Mitigation

Add the missing loop increment in both early-exit paths:

```diff
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {  
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;  
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;  
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;  
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;  
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;  
    uint256 len = vesting.allocationOf(nftId).amounts.length;  
    for (uint i; i < len;) {  
        if (claimedFlows[i]) {  
+            unchecked { ++i; }  // Add increment here  
            continue;  
        }  
        if (claimedSeconds[i] == 0) {  
            amount += amounts[i];  
+            unchecked { ++i; }  // Add increment here  
            continue;  
        }  
        uint256 claimedAmount = (claimedSeconds[i] * amounts[i]) / vestingPeriods[i];  
        uint256 unclaimedAmount = amounts[i] - claimedAmount;  
        amount += unclaimedAmount;  
        unchecked { ++i; }  
    }  
    unclaimedAmountsIn[nftId] = amount;  
}
```
  