# [000083] Inverted token eligibility check excludes eligible NFTs from dividend distribution
  
  ### Summary

The inverted equality check in `getUnclaimedAmounts()` will cause NFTs holding the dividend distributor's target token to be excluded from dividend calculations, as the function returns 0 for matching tokens instead of non-matching tokens, preventing legitimate TVS holders from receiving their dividends.

### Root Cause

In [A26ZDividendDistributor.sol:141](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141), the `getUnclaimedAmounts` function checks if the NFT's token matches the dividend distributor's token using `==` (equality) instead of `!=` (inequality)

```javescript
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```
This logic is inverted. The function should return 0 for NFTs that DON'T match the target token, but instead it returns 0 for NFTs that DO match.

### Internal Pre-conditions

1. Owner needs to call `setToken()` to set the dividend distributor's target token to match the vesting contract's token type
2. Owner needs to call `setUpTheDividends()` to calculate and distribute dividends

### External Pre-conditions

None

### Attack Path

1. Owner deploys `A26ZDividendDistributor` with a specific token
2. Owner calls `setUpTheDividends()` which internally calls `getTotalUnclaimedAmounts()`
3. `getTotalUnclaimedAmounts()` iterates through all NFTs and calls `getUnclaimedAmounts(nftId)` for each
4. For NFTs holding the target token, `getUnclaimedAmounts()` returns 0 due to the inverted check at line 141
5. `_setDividends()` calculates dividends based on the incorrect `totalUnclaimedAmounts`, excluding all eligible NFT holders

### Impact

All NFT holders with the target token type suffer a 100% loss of their entitled dividends. The stablecoin dividends intended for distribution remain locked in the contract, as totalUnclaimedAmounts will be 0 or only include ineligible NFTs.

- Correct token holder: loss 100% of entitled dividends
- Other toke hold: recieve 100 % of the dividend pool 

_refer to the PoC section below wht contain a concrete example_

### PoC

Concrete Example
Assume:

- 100,000 USDT to distribute
- NFT 1: holds 1,000,000 A26Z tokens (target token)
- NFT 2: holds 500,000 ProjectX tokens (wrong token)

What happens:

1. `getUnclaimedAmounts(1)` → returns 0 (A26Z token excluded)
2. `getUnclaimedAmounts(2)` → returns 500,000 (ProjectX token included)
3. `totalUnclaimedAmounts = 500,000` (should be 1,000,000)
4. NFT 1 owner receives: (0 * 100,000) / 500,000 = 0 USDT
5. NFT 2 owner receives: (500,000 * 100,000) / 500,000 = 100,000 USDT 

### Mitigation

Change the equality check in A26ZDividendDistributor.sol:141:

```diff
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {  
    // Fixed: use != instead of ==  
    if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;  
    // ... rest of function  
}
```
  