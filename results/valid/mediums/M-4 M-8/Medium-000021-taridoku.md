# [000021] High : Off-by-one enumeration in `getTotalUnclaimedAmounts`
  
  ### Summary

Off-by-one enumeration of `nftId` leads to underaccounting of unclaimed allocations and stuck undistributed stablecoin for TVS holders because `getTotalUnclaimedAmounts` skips the actual last minted NFT in multiple NFT cases and all NFTs in a single NFT case, when aggregating `totalUnclaimedAmounts`.

### Root Cause

After TVS has been distributed as rewards or a bidding process is completed, users are allowed to claim them as NFTs. The claim process involves calling `AlignerzNFT.mint()`, this eventually gets to `[ERC721A._mint()](https://github.com/dualguard/2025-11-alignerz-taridoku/blob/bca8c6f4be40b274a1c40cd2016a545e622b045c/protocol/src/contracts/nft/ERC721A.sol#L357-L394)`. It is important to note that minting never touches 0 because on first mint: 
- `startTokenId = _currentIndex = 1`
- It emits `Transfer(0x0, to, 1)`
- Sets `_ownerships[1].addr = to`
- Sets `_currentIndex = 2`
```solidity
function _mint(address to, uint256 quantity, bytes memory _data, bool safe) internal {
    uint256 startTokenId = _currentIndex;
    ...
    _ownerships[startTokenId].addr = to;
    ...
    uint256 updatedIndex = startTokenId;
    uint256 end = updatedIndex + quantity;
    ...
    do {
        emit Transfer(address(0), to, updatedIndex++);
    } while (updatedIndex != end);

    _currentIndex = updatedIndex;
}
```
There is never a write to `_ownerships[0]` because token 0 is simply not part of the minted range.
It is also important to note that [ownership lookup ](https://github.com/dualguard/2025-11-alignerz-taridoku/blob/bca8c6f4be40b274a1c40cd2016a545e622b045c/protocol/src/contracts/nft/ERC721A.sol#L180-L205)explicitly excludes 0
For tokenId = 0:
- `_startTokenId() = 1`
- Condition `1 <= 0 && 0 < _currentIndex is false`
- It jumps straight to revert `OwnerQueryForNonexistentToken()`

So any call to `extOwnerOf(0) -> _ownershipOf(0)` will always revert, by design, `safeOwnerOf()` just wraps that revert ensuring the whole transaction isnt affected.

In `A26ZDividendDistributor.setUpTheDividends()`, when `setAmounts()` is called, the function calls `getTotalUnclaimedAmounts` to aggregate the total amount from the unclaimed TVS minted by calling `vesting.allocationOf(nftId).token)` through `getUnclaimedAmounts()`. Before aggregating with `getUnclaimedAmounts()`, the function first checks if the nft is owned.  This is where the issue lies. 

[`getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz-taridoku/blob/bca8c6f4be40b274a1c40cd2016a545e622b045c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L136) loops through the total nfts minted but it loops with `i < len` where len is the total NFTs minted not including 0. This function assumes that there is an `nftId == 0`. This results in the last NFT that was minted not being included in the calculation. 
```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked {
            ++i;
        }
    }
}
```

With `getTotalMinted == 1`, this means that that there will be no dividends on that amount. Also, `_setDividends`, which calculates dividends results in a revert with `OwnerQueryForNonexistentToken()` which is caught by `safeOwnerOf()`, thereby masking the issue. This also leads to the last holder never receiving dividends.

### Internal Pre-conditions

1. `AlignerzNFT.sol` uses a non zero starting token ID 
2. The dividend contract calculates dividends using `len == N-1`

### External Pre-conditions

Nil

### Attack Path

1. `AlignerzNFT` mints NFTs with IDs starting at 1 and `_currentIndex` increasing. `getTotalMinted()` returns the count of minted NFTs
2. `A26ZDividendDistributor._setDividends()` does:
```solidity
uint256 len = nft.getTotalMinted(); // N
for (uint i; i < len;) {            // i = 0..N-1
    (owner, bool isOwned) = safeOwnerOf(i);
    if (isOwned) {
        dividendsOf[owner].amount += (
            unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts
        );
    }
    unchecked { ++i; }
}
```
3. `safeOwnerOf(0)` calls `extOwnerOf(0) -> `_ownershipOf(0)` reverts with `OwnerQueryForNonexistentToken`, which is caught and returned as (address(0), false)
4. The loop never visits i = N, even though the last real NFT holder has tokenId = N. Their `unclaimedAmountsIn[N]` is never included in `totalUnclaimedAmounts` or in `dividendsOf[owner].amount`
5. The last NFT holder receives zero dividends, while earlier holders (IDs < N) share 100% of `stablecoinAmountToDistribute` among themselves

### Impact

Impact: High
- Causes direct loss of dividends for at least one legitimate holder. From the affected user’s perspective, this is a permanent loss of owed funds.

Likelihood: High:
- It happens under normal operation whenever: At least one NFT is minted and `setAmounts()` or `_setDividends()` is called.

### PoC

_No response_

### Mitigation

Iteration should be 

```solidity
 uint256 len = nft.getTotalMinted() + 1; <@ corrected here 
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
```
  