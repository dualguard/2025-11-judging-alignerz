# [000949] Off-by-One Error in Dividend Distributor Loops
  
  ### Summary

The `getTotalUnclaimedAmounts()` and `_setDividends()` functions in `A26ZDividendDistributor.sol` contain an off-by-one error in their loops. The loops start at index 0 and use `<` comparison, but NFT IDs start at 1. This causes the loop to waste one iteration checking non-existent NFT ID 0, while completely missing the last minted NFT. As a result, the holder of the last minted NFT receives zero dividends in every dividend distribution round.


### Root Cause

`A26ZDividendDistributor.sol`
[`getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz-mabdullah22/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127)
[`_setDividends()`](https://github.com/dualguard/2025-11-alignerz-mabdullah22/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127)

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();  // Returns count of minted NFTs
    for (uint i; i < len;) {             // BUG: Starts at 0, uses <
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked {
            ++i;
        }
    }
}

function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {             // BUG: Same issue
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        unchecked {
            ++i;
        }
    }
    emit dividendsSet();
}
```

**The Off-by-One Error**: NFT IDs start from **1**, not 0

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

In the case of multiple mints , the last NFT holder will be excluded in the calculations.

### Impact

1. **Complete loss for last holder**: The holder of the last minted NFT receives **zero dividends** (100% loss)
2. **Systematic exclusion**: This happens in **every single** dividend distribution round
3. **Unfair enrichment**: Earlier NFT holders receive more than their fair share

### PoC

_No response_

### Mitigation

_No response_
  