# [000569] Missing Zero Amount Validation in TVS Split Allows Permanent Claim DoS Attack
  
  
## Summary

The lack of validation preventing zero-amount flows during TVS splits will cause a permanent denial of service for NFT holders as an attacker will create a TVS with a zero-amount flow through splitting, merge it with a healthy TVS, transfer it to a victim, and cause all claim attempts to revert. 

**Note** that this can also happen naturally affecting the user splitting their TVS. However, this bug creates an attack vector that can be leverage by an attacker to grief other users. And we wrote the report in that context, because it's more interesting and reveals more details to the sponsors/devs. Additionally, we wrote it in a griefing context to avoid escalation arguments like it's a self harm to split one's TVS into this state.

&nbsp;

## Root Cause
```js
    function _computeSplitArrays(...)
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        // ...
        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            // ...
        }
    }

```
In [`AlignerzVesting.sol:1133`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1132), the split calculation `alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT` can produce zero when `(baseAmounts[j] * percentage) < BASIS_POINT` due to integer division rounding down. In [`AlignerzVesting.sol:995`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L994), `getClaimableAmountAndSeconds` requires `claimableAmount > 0` for each flow. When `claimTokens` (line 959) iterates through flows and calls `getClaimableAmountAndSeconds`, any flow with zero amount causes the entire transaction to revert, blocking claims even when other flows have claimable amounts.

```js
    function getClaimableAmountAndSeconds(...) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
        // ...
        uint256 amount = allocation.amounts[flowIndex];
        // ...
        claimableAmount = (amount * claimableSeconds) / vestingPeriod;
        require(claimableAmount > 0, No_Claimable_Tokens()); // <--@audit
        return (claimableAmount, claimableSeconds);
    }
```

**Note** that this a different issue from my other report *"Integer Division Rounding in `getClaimableAmountAndSeconds` Causes Permanent Claim Blocking for NFTs with Small Remaining Amounts"*. The root cause here is amount rounding down to zero, while the other report is about crafting a small flow amount value, but greater than zero.

&nbsp;

## Internal Pre-conditions

- Attacker must own a TVS NFT with at least one flow
- Attacker must own a healthy TVS NFT to merge with
- The split percentage must be small enough that `(baseAmounts[j] * percentage) < BASIS_POINT` for at least one flow, resulting in a zero amount after division

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. Attacker calls `splitTVS` with percentages that cause at least one flow to round to zero (e.g., a very small percentage on a small base amount)
2. Attacker receives a TVS NFT containing a flow with zero amount
3. Attacker calls `mergeTVS` to merge the zero-amount TVS with a healthy TVS, creating a merged NFT with at least one zero-amount flow
4. Attacker transfers the merged NFT to a victim (e.g., via NFT marketplace or direct transfer)
5. Victim calls `claimTokens` to claim vested tokens
6. `claimTokens` iterates through flows and calls `getClaimableAmountAndSeconds` for each unclaimed flow
7. When processing the zero-amount flow, `getClaimableAmountAndSeconds` calculates `claimableAmount = 0` and reverts at line 995 with `No_Claimable_Tokens()`
8. The entire claim transaction reverts, preventing the victim from claiming any tokens from any flow

&nbsp;

## Impact

NFT holders cannot claim any vested tokens from their NFTs result in a permanent asset loss.

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Add validation in `_computeSplitArrays` or `splitTVS` to prevent zero-amount flows. After calculating `alloc.amounts[j]` at line 1133, add:

```solidity
require(alloc.amounts[j] > 0, "Split amount cannot be zero");
```

  