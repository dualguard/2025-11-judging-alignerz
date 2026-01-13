# [000345] Incorrect return logic in `A26ZDividendDistributor::getUnclaimedAmounts#140` for token mismatch
  
  ### Summary

If the NFT’s token address matches the contract's token, the function returns `0`, even though valid vesting data may exist, this causes an incorrect unclaimed amounts.

### Root Cause

This condition in [`A26ZDividendDistributor::getUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141):

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    // ...
}
```

returns 0 for matching token, which is opposite of expected behavior.

### Internal Pre-conditions

1. Vesting token matches the contract token.
2. Function returns prematurely.

### External Pre-conditions

_No response_

### Attack Path

1. User calls function for a normal vesting allocation.
2. Function returns 0 incorrectly.
3. Unclaimed calculations become wrong.

### Impact

Users see an incorrect vesting data, affecting withdrawals and off-chain accounting.
This makes claiming unclaimed tokens impossible

### PoC

_No response_

### Mitigation

Remove or reverse the logic.

```diff
-     if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+     if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
```
  