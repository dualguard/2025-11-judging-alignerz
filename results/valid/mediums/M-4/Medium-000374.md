# [000374] H04 _setDividends in A26ZDividentDistributor incorrectly loops from NFT ID 0
  
  ## Summary

Within ```A26ZDividentDistributor::_setDividends``` the function is looping through existing, owned NFTs to distribute dividends, however an oversight causes the last NFT to not get an allocation.

## Root Cause

Considering the function ```A26ZDividentDistributor::_setDividends``` here:

```solidity
function _setDividends() internal {
@>      uint256 len = nft.getTotalMinted();
@>      for (uint i; i < len; ) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned)
                dividendsOf[owner].amount += ((unclaimedAmountsIn[i] *
                    stablecoinAmountToDistribute) / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

we can see the ```for loop``` attempting to iterate over owned NFTs, however the ```uint256 i``` is initialized to 0 and the loop stops at ```len - 1```. Since the NFTs are minted beginning with an ID of 1 this will cause the last minted NFT to be skipped and not receive dividends.

## Internal Pre-Conditions

Nothing special

## External Pre-Conditions

Nothing special

## Attack Path

Just a bug, not an attack

## Impact

This will cause the owner of the last NFT ID to be entirely skipped regarding the dividends, a High severity seems accurate for financial loss and broken core functionality.

## PoC

Trivial

## Mitigation

```diff
function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
-       for (uint i; i < len; ) {
+       for (uint i = 1; i <= len; ) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned)
                dividendsOf[owner].amount += ((unclaimedAmountsIn[i] *
                    stablecoinAmountToDistribute) / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```
  