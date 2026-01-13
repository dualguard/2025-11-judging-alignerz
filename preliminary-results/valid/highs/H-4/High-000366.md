# [000366] H03 DoS in claimTokens for Multi-Flow Vesting
  
  ## Summary

The `AlignerzVesting::claimTokens` function reverts if *any* flow within the NFT's allocation has not yet started or has 0 claimable tokens. This creates a Denial of Service (DoS) for multi-flow NFTs (created via `mergeTVS`), where users cannot claim tokens from *active* flows if they are merged with *future* flows.

## Root Cause

In `AlignerzVesting::claimTokens`, the code iterates over all flows and calls `getClaimableAmountAndSeconds`:

```solidity
        for (uint256 i; i < nbOfFlows; i++) {
            // ...
            (
                uint256 claimableAmount,
                uint256 claimableSeconds
            ) = getClaimableAmountAndSeconds(allocation, i);
            // ...
        }
```

However, `getClaimableAmountAndSeconds` reverts in two scenarios:

1.  **Underflow (Not Started):**
    ```solidity
    } else {
        secondsPassed = block.timestamp - vestingStartTime; // <--- Reverts if now < vestingStartTime
    }
    ```
2.  **Zero Claimable:**
    ```solidity
    require(claimableAmount > 0, No_Claimable_Tokens()); // <--- Reverts if amount is 0
    ```

If an NFT has multiple flows (e.g., Flow A starts now, Flow B starts next year), `claimTokens` will fail for Flow A because Flow B triggers a revert.

## Internal Pre-Conditions

1.  An NFT has at least two flows.
2.  One flow is claimable (started, amount > 0).
3.  One flow is NOT claimable (not started OR amount == 0).

## External Pre-Conditions

1.  User merges two NFTs with different vesting schedules using `mergeTVS`.

## Attack Path

1.  User has NFT A (vesting now) and NFT B (vesting in 1 year).
2.  User calls `mergeTVS` to combine them into NFT A.
3.  User calls `claimTokens(NFT A)` to get tokens from the active flow.
4.  The transaction reverts because the loop encounters the future flow (NFT B's part) and fails the timestamp check or the `claimableAmount > 0` check.
5.  User's funds from NFT A are locked until NFT B starts vesting.

## Impact

**High**. Users are effectively locked out of their funds if they use the `mergeTVS` feature with disparate schedules. This breaks the composability of the vesting tokens.

## PoC

Trivial.

## Mitigation

Modify `getClaimableAmountAndSeconds` to return `(0, 0)` instead of reverting when tokens are not claimable.
Update `claimTokens` to handle the `0` return gracefully (e.g., `continue`).

```diff
    function getClaimableAmountAndSeconds(...) public view returns (...) {
        // ...
        if (block.timestamp < vestingStartTime) {
            return (0, 0);
        }
        // ...
        if (claimableAmount == 0) {
            return (0, 0);
        }
        return (claimableAmount, claimableSeconds);
    }
```

  