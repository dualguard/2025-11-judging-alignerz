# [000555] Off-by-one error in NFT token ID iteration causes last minted NFT holder to be permanently excluded from dividend distribution
  
  

## Summary

An off-by-one error in `A26ZDividendDistributor.sol` will cause a complete loss of dividend eligibility for the holder of the last minted NFT as the owner calls `setUpTheDividends()` or `setDividends()`, because the loops in `getTotalUnclaimedAmounts()` and `_setDividends()` iterate from token ID `0` to `len - 1` while ERC721A token IDs start at 1, causing the highest token ID to be skipped.

Looking at the ERC721A implementation:

```js
function _startTokenId() internal pure virtual returns (uint256) {
    return 1;
}
```

```js
function _totalMinted() internal view returns (uint256) {
    // Counter underflow is impossible as _currentIndex does not decrement,
    // and it is initialized to _startTokenId()
    unchecked {
        return _currentIndex - _startTokenId();
    }
}
```

## Root Cause

In [`A26ZDividendDistributor.sol:129`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L129) and `A26ZDividendDistributor.sol:216`, the loops iterate from `i = 0` to `i < len`, but the NFT contract uses ERC721A where `_startTokenId()` returns `1` (as seen in [`ERC721A.sol:105-107`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/nft/ERC721A.sol#L105-L107)). This means valid token IDs are `1, 2, 3, ..., len`, but the loops check `0, 1, 2, ..., len - 1`, skipping the last minted token ID.

## Internal Pre-conditions

1. At least one NFT must be minted (`getTotalMinted() > 0`)
2. Owner must call `setUpTheDividends()` or `setDividends()` to trigger dividend distribution
3. The last minted NFT must still exist (not burned)

## External Pre-conditions

None required.

## Attack Path

1. Owner calls `setUpTheDividends()` which internally calls `_setAmounts()` and `_setDividends()`
2. `_setAmounts()` calls `getTotalUnclaimedAmounts()` at line 209
3. `getTotalUnclaimedAmounts()` at line 127-135 iterates from `i = 0` to `i < len`, checking token IDs `0, 1, 2, ..., len - 1`
4. Since token IDs start at 1, token ID `0` doesn't exist (handled by `safeOwnerOf`), and token ID `len` (the last minted NFT) is never checked
5. `_setDividends()` at line 214-222 performs the same iteration, excluding the last minted NFT from dividend allocation
6. The holder of token ID `len` never receives dividends and cannot claim them

## Impact

The holder of the last minted NFT suffers a complete loss of dividend eligibility. Their unclaimed amounts are excluded from `totalUnclaimedAmounts`, and they are never allocated dividends in `dividendsOf[owner].amount`. This affects at least one NFT holder every time dividends are set, and the impact increases if the last minted NFT has a large allocation.

## Proof of Concept

```solidity
// Scenario: 5 NFTs minted (token IDs: 1, 2, 3, 4, 5)
// getTotalMinted() returns: 5
// Loop iteration: i = 0, 1, 2, 3, 4
// 
// Checks performed:
// - safeOwnerOf(0) → doesn't exist (returns false)
// - safeOwnerOf(1) → exists ✅
// - safeOwnerOf(2) → exists ✅
// - safeOwnerOf(3) → exists ✅
// - safeOwnerOf(4) → exists ✅
// - safeOwnerOf(5) → NEVER CHECKED ❌
//
// Result: Token ID 5 holder receives zero dividends
```

## Mitigation

Change the loops in both functions to iterate from token ID 1 to `len` (inclusive):

```diff
// In getTotalUnclaimedAmounts() at line 127-135:
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
+    for (uint i = 1; i <= len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
        unchecked {
            ++i;
        }
    }
}
```
  