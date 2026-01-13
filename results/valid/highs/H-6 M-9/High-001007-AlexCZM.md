# [001007] Unbounded loop in `getTotalUnclaimedAmounts()` may DoS setting up the dividends
  
  ### Summary

`getTotalUnclaimedAmounts()` loops over all minted NFTs and then over all flows from all NFTs. 
`getTotalUnclaimedAmounts()` gas cost execution may exceed bloc gas limit, resulting in the inability to set the dividends details. 

### Root Cause

`getTotalUnclaimedAmounts()` loops first over [all NFTs](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L128-L129)
```solidity
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);// @audit call it for all NFTs
//...
```
then [getUnclaimedAmounts(nftId)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L147) loops over all flows a NFT has : 
```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
//...
        uint256 len = vesting.allocationOf(nftId).amounts.length; 
        for (uint i; i < len;) { // @audit loop over all flows
//...
````

The number of NFTs and the number of flows each NFT has can't be controlled by project admin. 
Due to double unbounded for loops, the gas cost to successfully execute `getTotalUnclaimedAmounts()` may exceed the block gas limit, resulting in 
dividend distribution DoS. 


### Internal Pre-conditions

None

### External Pre-conditions

Users must merge and split the NFTs such that the total number of flows to be high enough. 

### Attack Path

1. Admin calls `finalizeBids()`, `updateProjectAllocations()` to set up the roots. 
2. Thousands of NFTs are minted.
3. NFT holders merge and split the NFTs to fit their needs. Number of flows grows exponentially - see *example below.  
4. Admin send USD dividends to `A26ZDividendDistributor` contract and calls `setUpTheDividends()`. The gas required to execute the `setUpTheDividends()` exceed the block gas limit. 

#### *NFT flows growth example 
The number of flows grows exponentially with each split and merge.
Let's consider Alice has 2 TVS, both with 2 flows only:

Alice splits both TVS in 3 other TVSs: now each resulting TVSs (2 *3 = 6 in total) has 2 flows each.
After a while Alice merge all her 6 TVS into one: the resulting TVS has 6 * 2 = 12 flows.
Starting from really low numbers (2 TVS with 2 flows each), only after one split& merge cycle Alice got a TVS with 12 flows.

### Impact

Admin can't setup the dividend distribution when total number of flows is high. 

### PoC

_No response_

### Mitigation

- consider limiting the number of flows a NFT can have
- consider changing the functions that iterates over all NFTs/ all flows such that the caller (admin) can specify a processing range via start and end indices. 
  