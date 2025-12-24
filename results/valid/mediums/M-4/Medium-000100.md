# [000100] `A26ZDividendDistributor` tracks NFTs wrong, which will never allow the owner of the last NFT to get dividends
  
  ### Summary

Function [`getTotalUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127) which is used inside [`_setAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207) to calculate the total unclaimed amounts of each allocation of each NFT existent that is not burnt, goes in the for loop between `0 -> len - 1`. This does not take into account the fact that NFTs in `AlignerzNFT` are indexed from 1 and not from 0. This will not take the owner of the last NFT into account, since the for loop should go between `1 -> len`.
```solidity
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
@>        for (uint256 i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
```
Same goes for function [`_setDividends`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214) which tracks each allocation:
```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
@>        for (uint256 i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) {
                dividendsOf[owner].amount +=
                    (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            }
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

### Root Cause

In function `getTotalUnclaimedAmounts` and `_setDividends`, for loop goes from 0 to len - 1, but it should go from 1 to len.

### Internal Pre-conditions

1. NFT with the last id is not burnt

### External Pre-conditions

None

### Attack Path

1. Dividends are distributed.
2. Owner of the NFT with the last index does not get accounted for.

### Impact

A user loses rewards, because the owner of the NFT with the last id will not get accounted for in the `A26ZDividendDistributor` and will not be able to claim any dividends when he should be able to.

### PoC

None

### Mitigation

Make the for loop go from 1 to len, not 0 to len - 1.
  