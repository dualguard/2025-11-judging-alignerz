# [000508] Malicious Users Can Inflate Dividend Allocations by Delaying Token Claims After Vesting Completion
  
  ### Summary

Users who complete their vesting period can intentionally delay calling `claimTokens()` to artificially inflate their unclaimed token balance in the dividend distributor contract. This allows them to receive disproportionately larger dividend allocations at the expense of legitimate users who claim promptly.


### Root Cause

The `claimTokens()` function in `AlignerzVesting.sol` can only be called by the NFT owner:
```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    // ...
}
```

There is no mechanism for the owner/protocol to force-claim tokens on behalf of users after the vesting period ends. Meanwhile, the dividend distributor's `getUnclaimedAmounts()` function counts all unclaimed tokens (including fully vested ones) when calculating dividend shares, even though these tokens are already claimable and just waiting for user action.

Code snippet-
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975


### Internal Pre-conditions

1. User's vesting period must be complete (`block.timestamp > vestingPeriod + vestingStartTime`).
2. User has not called `claimTokens()` to burn their NFT and claim tokens.
3. Dividend distribution occurs via `setUpTheDividends()` while user still holds the NFT.


### External Pre-conditions

1. Owner calls `setUpTheDividends()` in the dividend distributor contract.
2. Multiple users have vested tokens at different stages of claiming.


### Attack Path

Taking this for better understanding purpose.
**Setup:**
- User A: 100 tokens, vesting complete 3 months ago, **claims immediately**
- User B: 100 tokens, vesting complete 3 months ago, **delays claiming intentionally**

**Round 1 Dividend Distribution (Day 1 after completion of vesting period of UserA & UserB. Both's `vestingPeriod` and `vestingStartTime` are same ):**
1. Owner calls `setUpTheDividends()` with 1000 USDC to distribute
2. `getTotalUnclaimedAmounts()` runs:
   - User A: 0 tokens (already claimed and NFT burned before owner calling  `setUpTheDividends()`)
   - User B: 100 tokens (still unclaimed, NFT exists)
   - Total: 100 tokens
3. Dividend allocation:
   - User A: 0 USDC (no unclaimed tokens)
   - User B: 1000 USDC (100/100 = 100%)

**Round 2 Dividend Distribution (Day 30):**
1. Owner calls `setUpTheDividends()` with another 1000 USDC
2. User B still hasn't claimed:
   - User B: 100 tokens (still unclaimed)
   - Total: 100 tokens
3. Dividend allocation:
   - User A: 0 USDC
   - User B: += 1000 USDC (total 2000 USDC)

**Result:** User B receives 2000 USDC for the same 100 vested tokens, while User A (who had identical vesting) receives 0 USDC simply because they claimed promptly.

### Impact

Unfair dividend distribution mechanism. Current users getting less amount because of already vested users(nft) in `AlignerzVesting.sol`. Malicious/strategic users can game the system to receive multiple dividend rounds. Honest users who claim promptly are penalized with reduced dividends. Protocol cannot enforce fair distribution as it has no control over when users claim.

### PoC

N/A

### Mitigation

Add an owner-controlled force-claim function in `AlignerzVesting.sol`
  