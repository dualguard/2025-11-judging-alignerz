# [000084] Unbounded loops in dividend distribution will cause permanent DoS as NFT count grows
  
  ### Summary

The unbounded loops in `getTotalUnclaimedAmounts()` and `_setDividends()` will cause permanent inability to distribute dividends to TVS holders as the protocol scales, since the admin will call `setUpTheDividends()` which iterates through all minted NFTs without gas limit consideration, eventually exceeding block gas limits and making the entire dividend distribution system permanently unusable.

### Root Cause

In [A26ZDividendDistributor.sol:128-135](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L128-L135) and [A26ZDividendDistributor.sol:215-222](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L215-L222), the `getTotalUnclaimedAmounts()` and `_setDividends()` iterate through every NFT ever minted using `nft.getTotalMinted()` without any pagination or gas limit checks. 

### Internal Pre-conditions

1. Protocol needs to mint a sufficient number of NFTs (estimated 5,000-10,000 depending on network) to exceed block gas limits during iteration
2. Admin needs to call `setUpTheDividends()` or `setDividends()` to trigger the unbounded loops

### External Pre-conditions

None

### Attack Path

This is not an attack but an inevitable operational failure:

1. Protocol launches and conducts multiple IWO rounds over time, minting NFTs for each participant
2. NFT count grows to 5,000+ as the protocol gains adoption across multiple projects
3. Admin attempts to call `setUpTheDividends()` to distribute quarterly dividends
4. Transaction reverts due to exceeding block gas limit during the loop iteration
5. All future dividend distributions become impossible - the function is permanently DoS'd
6. Stablecoin dividends accumulate in the contract but cannot be distributed to TVS holders

### Impact

The protocol suffers permenant loss of dividend distribution fucntionality once the NFT exceeds gas limits. 
The admin has no alternative method to allocate the dividends for each user

Even if the owner decide to deploy a new contract every 3 month to distribute dividends instead of updating the currant one, the NFT contract remain the same, `nft.getTotalMinted()` continue to increase, and each new dividend still quereis the same growing NFT count. 

The impact is high severity because:

- Core feature failure
- No recovery mechanism: no alternative function that has batch option
- Inevitable occurance: this is not theoratical issue; it will occur happen as the protocol scales through multiple IWO round

### PoC

The issue is evident from code inspection: both functions iterate through `nft.getTotalMinted()` without bounds, and gas consumption scales linearly with NFT count.

### Mitigation

There is 3 options for the solution:

Option 1: Implement a dividend distribution fucntion with batchs

```diff
function setDividendsInBatches(uint256 startIndex, uint256 endIndex) external onlyOwner {  
    require(endIndex <= nft.getTotalMinted(), "Invalid range");  
    require(startIndex < endIndex, "Invalid range");  
      
    for (uint256 i = startIndex; i < endIndex;) {  
        (address owner, bool isOwned) = safeOwnerOf(i);  
        if (isOwned) {  
            dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);  
        }  
        unchecked { ++i; }  
    }  
}
```

Option 2: Implement a pull-based claim sytem where users calculate and claim their own dividends on-demand, eliminating the need for admin to iterate through all the NFTs

Option 3: Maintain an active NFT holdre list that gets updated on teh transfer/burn, avoiding iteration through all minted NFTs
  