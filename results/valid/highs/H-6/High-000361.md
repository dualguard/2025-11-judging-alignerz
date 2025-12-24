# [000361] M01 DoS in DividendDistributor
  
  # DoS in A26ZDividendDistributor due to Unbounded Loops

## Description
In `A26ZDividendDistributor.sol`, the functions `_setDividends` and `getTotalUnclaimedAmounts` iterate from 0 to `nft.getTotalMinted()`.
```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            // ... complex logic ...
        }
    }
```
As the number of NFTs grows, the gas cost of these functions will grow linearly. Eventually, it will exceed the block gas limit, making it impossible to set or distribute dividends.
Since `_setDividends` is required to enable claiming, the entire dividend mechanism will become permanently stuck.

## Impact
**Medium/High**. Permanent Denial of Service for dividend distribution once the user base grows sufficiently large.

## Recommendation
Implement a pull-based mechanism or process dividends in batches. Avoid iterating over all NFTs in a single transaction.

  