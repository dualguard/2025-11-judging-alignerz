# [000027] Incorrect token-matching condition causes all valid TVS NFTs to be skipped in dividend calculations
  
  ### Summary

The function `getUnclaimedAmounts()` contains an inverted token-matching check:

```solidity
if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
```

This logic returns `0` **whenever the NFT's token matches the expected TVS token**—which is the *only case* where the NFT should be included in dividend calculations. As written, all valid NFTs are incorrectly discarded, while irrelevant NFTs (from other tokens or projects) are incorrectly included in the dividend computation.

This completely breaks the distribution of dividends and results in permanent, large-scale misallocation of stablecoin rewards across unrelated or invalid recipients.

### Root Cause

Token validation logic is inverted, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Legitimate TVS holders receive **no dividends**, while holders of unrelated NFTs may receive **all or most of the dividend pool**, depending on supply size.

### PoC

_No response_

### Mitigation

Consider fixing the condition so that only NFTs whose token matches the distributor’s TVS token are included:

```diff
-   if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+   if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  