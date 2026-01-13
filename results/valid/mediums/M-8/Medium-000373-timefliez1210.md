# [000373] H05 getToalUnclaimedAmounts in A26ZDividendDistributor incorrectly loops from NFT ID 0
  
  ## Summary

Within ```A26ZDividentDistributor::getToalUnclaimedAmounts``` the function is looping through existing, owned NFTs to distribute dividends, however an oversight causes the last NFT to not be counted towards dividend calculations.

## Root Cause

Considering the function ```A26ZDividentDistributor::_setDividends``` here:

```solidity
function getTotalUnclaimedAmounts()
        public
        returns (uint256 _totalUnclaimedAmounts)
    {
@>      uint256 len = nft.getTotalMinted();
@>      for (uint i; i < len; ) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```

we can see the ```for loop``` attempting to iterate over owned NFTs, however the ```uint256 i``` is initialized to 0 and the loop stops at ```len - 1```. Since the NFTs are minted beginning with an ID of 1 this will cause the last minted NFT to be skipped and not counted towards the crucial calculations for the dividends.

## Internal Pre-Conditions

Nothing special

## External Pre-Conditions

Nothing special

## Attack Path

Just a bug, not an attack

## Impact

This will cause NFT to be entirely skipped during crucial calculations for dividend distribution, a high severity seems accurate given the importance of the dividends in Alignerz Business model.

## PoC

Trivial

## Mitigation

```diff
function getTotalUnclaimedAmounts()
        public
        returns (uint256 _totalUnclaimedAmounts)
    {
        uint256 len = nft.getTotalMinted();
+       for (uint i = 1; i <= len; ) {
-       for (uint i; i < len; ) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```
  