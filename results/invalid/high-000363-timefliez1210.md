# [000363] H13 Cross Project Merge Dividend Theft
  
  # Cross-Project Merge Allows Dividend Theft

## Description
The `mergeTVS` function in `AlignerzVesting.sol` allows merging NFTs from different projects as long as they share the same underlying token.
```solidity
        require(address(token) == address(tokenToMerge), Different_Tokens());
```
However, it does not verify that the source NFT belongs to the same project as the destination NFT.
If Project A (with dividends) and Project B (without dividends) share the same token (e.g. different rounds of the same token), a user can merge a large allocation from Project B into an NFT from Project A.

The `A26ZDividendDistributor` calculates dividends based on the total unclaimed amount in the NFT.
```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        // ...
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        // ...
    }
```
By merging Project B tokens into Project A NFT, the user artificially inflates their "unclaimed amount" in Project A, thus claiming a larger share of the dividends than they are entitled to.

## Impact
**High**. Users can drain dividend pools meant for specific project participants by merging external allocations.

## Recommendation
In `mergeTVS`, ensure that the source NFT belongs to the same project as the destination NFT (or at least the same project ID).

```diff
    function _merge(...) internal ... {
        // ...
+       require(projectId == mergedTVSProjectId, "Cannot merge across projects");
        // ...
    }
```


  