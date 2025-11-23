# [000456] Unbounded arrays are prone to OOG
  
  Related code: https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L161

### Summary

In multiple instances in the code, unbounded loops can be inflated by users, causing transaction to cost so much gas that they may revert in an OOG (Out Of Gas) error. The worst place this can happen is in the `A26ZDividendDistributor` contract.

This contract is used to distribute the dividends to the users. The function `_setAmounts` is critically important because it is the only function that sets 2 core variables used in the amount of dividends to give to users. If this function is not run, then there are no dividends that can ever be sent to users.

`_setAmounts` calls `getTotalUnclaimedAmounts`. This function loops through the entire list of nft ever created, even those burned (though it executes less logic for burned NFTs, it still wastes gas). For all NFT owned (not burned), the function then calls `getUnclaimedAmounts`.

This nested function has another for loop, that goes through all allocations from every owned NFT. Since allocations can be splitted as desired by the NFT owner, this can result in an extremely long list. If those list consume too much gas, it will revert and prevent the owner to ever distribute dividends. 

Since these lists are each likely to naturally grow even bigger over time, this will likely cause a permanent DoS of every instances of the distributor contract (deploying a new contract won't fix the issue), and loss of funds for users.

### Root Cause

Unbounded lists in `getTotalUnclaimedAmounts` and nested in `getUnclaimedAmounts`

### Internal Pre-conditions

Enough NFT holders exists to cause a large amount of gas consumption

### Attack Path

1. Users mint NFTs, and eventually split them (this causes the issue to arise faster)
2. Owner wants to give dividends to NFT holders. They call `setAmounts` to generate the values to distribute in the variables. The function reverts due to OOG exception.

### Impact

DoS of dividend distribution + loss of rewards for users

### Mitigation

I would suggest to rethink the mechanisms of looping through all NFTs and allowing users to split them when handling the dividends. This process looks hard to optimize. Instead, having an array of nftIds to distribute would be better, as we won't need to loop through burned NFTs, and avoiding TVS splitting would prevent a nesting unbounded for loop that can drastically increase gas consumption.

If splitting TVS is a must, consider pre-calculating the owed amount so it's not computed in a nested loop.
  