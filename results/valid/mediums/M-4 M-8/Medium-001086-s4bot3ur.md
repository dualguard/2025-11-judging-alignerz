# [001086] Last NFT Owner Cannot Claim Dividends Due to Off-By-One Error in Indexing
  
  ### Summary

In ERC721A, NFT IDs start at 1, not 0.
If `_currentIndex = 20` and `_startTokenId = 1`, then:
- `totalMinted = 20 - 1 = 19`
- `Meaning the valid NFT IDs are: 1, 2, 3, …, 19`
However, both `getTotalUnclaimedAmounts()` and `_setDividends()` iterate using:
```solidity
uint256 len = nft.getTotalMinted();
for (uint i; i < len; i++)
```
This iterates over `[0 … 18]`, which is incorrect because:
- **0** is not a valid NFT ID
- **19** (the last NFT) is completely skipped
If the last minted NFT belongs to the current dividend project, its owner will never be included in the dividend calculations. As a result, the last NFT holder cannot claim dividends

### Root Cause

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L129

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L216

The above both lines assume that NFT IDs start at 0 and iterate until len - 1.
However, in ERC721A, NFT IDs actually start at 1, and the loop should iterate from 1 to len (inclusive).
Because the loop incorrectly starts at 0 and ends at len - 1, the last minted NFT is skipped, and index 0 (which is invalid) is wrongly checked.

### Internal Pre-conditions

1. The owner must transfer the dividend amount to the deployed `A26ZDividendDistributor` contract.
2. Owner must to call `A26ZDividendDistributor::setUpTheDividends`

### External Pre-conditions

1. AlignerzVesting must be deployed, and at least one project must be created.
2. Users must submit bids for the project, the project owner must finalize all bids, and atleast one user must claim their NFTs.
3. The project must announce dividends and deploy an instance of `A26ZDividendDistributor` with the correct parameters for that specific project.
4. The last Alignerz NFT must be minted to user from this project

### Attack Path

1. AlignerzVesting contract is deployed and a project **Project A** has been created.
2. Users submit bids for the **Project A** and project owner have finalized the bids. Most of the user have claimed their NFTs represting TVS.
3. The project **Project A** have announced the dividend and created instance of `A26ZDividendDistributor`, then transferred USD fund to `A26ZDividendDistributor` contract.
4. UserA claims NFT or splits their NFT.
5. Right after the above step the project owner calls `A26ZDividendDistributor::setUpTheDividends`.
6. The above step will skip including the `UserA's` NFT in dividned. 

### Impact

The user holding the highest-ID NFT (the last minted one) is permanently excluded from dividend allocation:
- Their unclaimed amount is never calculated
- Their proportion of the dividend pool is never added
- They lose 100% of their dividend entitlement for that distribution

### PoC

_No response_

### Mitigation

```diff
contract A26ZDividendDistributor{

    .
    .
    .
    .
    .


    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
-       for (uint i; i < len;) {
+       for (uint i=1; i <= len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }
    .
    .
    .
    .
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
-       for (uint i; i < len;) {
+       for (uint i=1; i <= len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
}


```
  