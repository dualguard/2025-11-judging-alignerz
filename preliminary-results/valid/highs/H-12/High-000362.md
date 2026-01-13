# [000362] H14 Inverted Logic in Distributor
  
  ## Summary

In `A26ZDividendDistributor.sol`, the function `getUnclaimedAmounts` contains a logic error that effectively disables dividend distribution. It checks if the project token matches the NFT token, but returns `0` if they match, instead of if they don't match.

## Root Cause

In `getUnclaimedAmounts`:

```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        // ...
    }
```

The check `address(token) == address(vesting.allocationOf(nftId).token)` evaluates to `true` for valid NFTs of the project. The function then returns `0`. It only proceeds if the tokens *do not* match (or if the NFT holds a different token).

## Internal Pre-Conditions

None.

## External Pre-Conditions

None.

## Impact

**Critical**. The dividend distribution mechanism is completely broken. Valid users receive 0 dividends regardless of their holdings.

## PoC

Trivial.

## Mitigation

Invert the check to ensure the tokens DO match, or return 0 if they DO NOT match.

```diff
- if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+ if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```

  