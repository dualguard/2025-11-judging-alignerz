# [000722] Incorrect token check in `getUnclaimedAmounts` prevents dividend calculation for legitimate token holders
  
  ### Summary

The `getUnclaimedAmounts` function uses an incorrect equality check (`==`) instead of an inequality check (`!=`), causing it to return zero for NFTs holding the correct dividend-eligible token while attempting to calculate dividends for wrong tokens.


### Root Cause

In the `getUnclaimedAmounts` function in `A26ZDividendDistributor.sol`, the token validation check is inverted. The function checks `if (address(token) == address(vesting.allocationOf(nftId).token))` and returns 0 when tokens match, which is the opposite of the intended logic.

The purpose of this check is to skip NFTs that don't hold the dividend-eligible token. However, the current implementation does the reverse:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
->    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    ...
}
```


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Protocol deploys `A26ZDividendDistributor` with `token` set to `USDC` (the dividend-eligible token)
2. Three users hold vesting NFTs:
   - Alice: NFT 1 with 1000 USDC vesting tokens (correct token)
   - Bob: NFT 2 with 2000 USDC vesting tokens (correct token)  
   - Charlie: NFT 3 with 500 DAI vesting tokens (wrong token)
3. Owner calls `getTotalUnclaimedAmounts()` to calculate total:
   - For Alice's NFT 1: `token (USDC) == allocation.token (USDC)` → returns 0
   - For Bob's NFT 2: `token (USDC) == allocation.token (USDC)` → returns 0
   - For Charlie's NFT 3: `token (USDC) != allocation.token (DAI)` → calculates 500 (wrong)
6. When Alice and Bob try to claim dividends:
   - Their `getUnclaimedAmounts` returns 0
   - They receive no dividend share despite being legitimate holders
   - Charlie (who holds wrong token) gets counted in distribution calculation

### Impact

All NFT holders with the correct dividend-eligible token receive zero dividend allocation, while the function incorrectly attempts to calculate dividends for NFTs holding different tokens. This completely breaks the dividend distribution mechanism for its intended beneficiaries.

### PoC

_No response_

### Mitigation


Change the equality check to an inequality check:

```diff
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
-   if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+   if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    // ... rest of calculation ...
}
```
  