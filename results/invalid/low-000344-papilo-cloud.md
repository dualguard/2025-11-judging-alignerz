# [000344] No Validation of Percentages Before Loop in `AlignerzVesting.sol::splitTVS` logic
  
  ### Summary

Validating that `sumOfPercentages == BASIS_POINT` after performing state mutations like (fee transfer, allocation modification), will cause the function to expend significant gas and perform external actions before discovering invalid input.

### Root Cause

In [`AlignerzVesting.sol::splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054),
The code applies the fee, updates `allocation.amounts = newAmounts`, transfers `feeAmount` to treasury, mints new NFTs and writes allocation storage, and only after the loop checks `require(sumOfPercentages == BASIS_POINT)`. The validation is performed too late, after costly state changes.

### Internal Pre-conditions

1. Caller passes `percentages[]` whose sum is not `BASIS_POINT`.
2. The function proceeds to compute fee and new allocations before validating percentages.
3. Fee transfer, NFT mints, and storage writes are executed prior to the final require.

### External Pre-conditions

Any caller can supply bad `percentages` to trigger this.

### Attack Path

1. Attacker (or any caller) calls `splitTVS` with percentages that do not add up to `BASIS_POINT`.
2. The contract: calculates `feeAmount`, writes `allocation.amounts = newAmounts`, transfers `feeAmount` to `treasury`, mints NFTs and performs storage writes per split iteration. All of these cost gas and may involve external calls.
3. At function end `require(sumOfPercentages == BASIS_POINT)` fails, causing revert.
4. Revert will rollback state changes (on-chain state) but the caller still paid gas for all work.
5. Repeated calls with bad percentages can be used to grief and waste the caller’s gas

### Impact

1. Owner/caller suffers gas wastage; poor UX; possible griefing by maliciously submitting bad percentages to force expensive rollback.
2. Risk of DoS by spamming expensive revert calls.

### PoC

_No response_

### Mitigation

Validate inputs early before any state change or expensive work, validate:

```solidity
    function splitTVS(...) external returns (uint256, uint256[] memory) {
        // Check percentages sum to 100% BEFORE doing anything
        uint256 sumOfPercentages;
        for (uint256 i; i < percentages.length; i++) {
            sumOfPercentages += percentages[i];
        }
    require(sumOfPercentages == BASIS_POINT, Percentages_Do_Not_Add_Up_To_One_Hundred());
    // ... The remaining code goes here ...
}
```

  