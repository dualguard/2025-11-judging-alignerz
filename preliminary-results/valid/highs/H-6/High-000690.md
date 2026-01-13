# [000690] Unbounded loops in dividend calculation will cause a permanent DoS for the protocol preventing dividend distribution
  
  ### Summary

The use of unbounded loops iterating over the total NFT supply in `getTotalUnclaimedAmounts` and `_setDividends` will cause a permanent DoS for the protocol and TVS holders as the Owner will be unable to execute `setUpTheDividends()` once the number of NFTs grows sufficiently large, causing transactions to exceed the block gas limit.

### Root Cause

In `A26ZDividendDistributor.sol`, the functions `getTotalUnclaimedAmounts()` and `_setDividends()` utilize `for` loops that iterate from `0` to `nft.getTotalMinted()`.
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214
Specifically:
1.  Inside `getTotalUnclaimedAmounts`: `for (uint i; i < len;)`
2.  Inside `_setDividends`: `for (uint i; i < len;)`
```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

Furthermore, inside the `getTotalUnclaimedAmounts` loop, there are multiple external calls to `vesting.allocationOf(nftId)`, which is a gas-heavy operation. As the number of minted NFTs increases, the gas required to execute these functions grows linearly. Eventually, the gas cost will exceed the blockchain's block gas limit, making the functions impossible to execute.

### Internal Pre-conditions

1.  The `AlignerzNFT` contract needs to have minted a sufficient number of NFTs (e.g., several thousand) such that the computational cost of iterating through all of them exceeds the block gas limit.
2.  The Owner calls `setUpTheDividends()`, `setAmounts()`, or `setDividends()`.

### External Pre-conditions

None.

### Attack Path

1.  Legitimate users participate in the vesting/bidding projects, causing the `AlignerzNFT` contract to mint new TVS NFTs.
2.  Over time, the total number of minted NFTs (`len`) grows large.
3.  The protocol owner attempts to distribute dividends by calling `setUpTheDividends()`.
4.  The transaction reverts due to "Out of Gas" because the loop tries to process every single NFT in one transaction.
5.  The dividend distribution mechanism becomes permanently stuck and unusable.

### Impact

The protocol suffers a complete loss of functionality regarding dividend distribution. The `stablecoinAmountToDistribute` intended for users cannot be distributed via the logic defined in the contract. TVS holders suffer a loss of their expected dividend yield.

### PoC

_No response_

### Mitigation

Refactor the dividend distribution logic to avoid O(N) loops over the entire collection in a single transaction. 
  