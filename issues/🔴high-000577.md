# [000577] Inclusion of Fully Claimed TVS Flows in Merge Fee Calculation Causes Token Shortfall and Loss for Remaining Users
  
  
&nbsp;

## Summary

The inclusion of fully claimed TVS flows in merge fee calculations will cause a token shortfall for remaining users in the protocol as the contract will attempt to transfer fees from tokens that have already been claimed and transferred to users.

&nbsp;

## Root Cause

In [`AlignerzVesting.sol:1014`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013) and `AlignerzVesting.sol:1070`, the `calculateFeeAndNewAmountForOneTVS` function is called with all TVS flow amounts without checking whether those flows have been fully claimed. The function in `FeesManager.sol:169-174` calculates fees on all provided amounts, including flows where `claimedFlows[i] == true`. When a TVS flow is fully claimed, the tokens have already been transferred to the user via `claimTokens()`, so the contract no longer holds those tokens. When merge operations calculate fees on claimed flows, the contract attempts to transfer fee amounts to the treasury that can exceed the supposed amount, creating a shortfall that affects subsequent withdrawals.

```js
    function mergeTVS(...) external returns(uint256) {
        // ...

        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        // ...
    }
```

&nbsp;

## Internal Pre-conditions

1. A user must have a TVS with at least one flow that has been fully claimed (`claimedFlows[i] == true`)
2. The same user (or another user who owns the TVS) must call `mergeTVS()` to trigger fee calculation
3. The `mergeFeeRate` must be greater than 0

&nbsp;

## External Pre-conditions

None

&nbsp;

## Attack Path

1. User A claims all tokens from a TVS flow, setting `claimedFlows[i] = true` and receiving the tokens
2. User A calls `mergeTVS()` to merge the TVS with another TVS
3. The `mergeTVS()` function calls `calculateFeeAndNewAmountForOneTVS()` at line 1014 with all `mergedTVS.amounts`, including amounts from fully claimed flows
4. `calculateFeeAndNewAmountForOneTVS()` calculates fees on all flows, including the claimed ones, without checking `claimedFlows[i]`
5. The function attempts to transfer the total `feeAmount` (including fees from claimed flows) to the treasury at line 1024
6. If the contract balance is insufficient, the transaction may revert, or if it succeeds, a shortfall is created
7. Subsequent users attempting to claim their tokens may find insufficient tokens in the contract, causing losses for the last withdrawers

&nbsp;

## Impact

The remaining users in the protocol suffer a loss proportional to the fees calculated on fully claimed flows. The loss accumulates as more merge or split operations are performed on TVSs containing claimed flows. The last users to withdraw their tokens bear the full impact of the shortfall, as the contract's token balance becomes insufficient to fulfill their legitimate claims.

&nbsp;

## Proof of Concept
N/A

&nbsp;

## Mitigation

Modify `mergeTVS` implementation to remove the amount for claimed flows before calling `calculateFeeAndNewAmountForOneTVS`.

  