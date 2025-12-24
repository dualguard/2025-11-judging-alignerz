# [000726] `calculateFeeAndNewAmountForOneTVS` incorrectly deducts the cumulative fee from each individual flow amount
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol` incorrectly accumulates fees across all flows but deducts the cumulative fee from each individual flow amount. This causes later flows in a multi-flow TVS to have exponentially larger deductions than intended, resulting in users receiving significantly fewer tokens than entitled when splitting or merging vesting NFTs with multiple token flows.

### Root Cause

In the `calculateFeeAndNewAmountForOneTVS` function, the fee calculation logic accumulates the total `feeAmount` across all iterations but incorrectly deducts this cumulative total from each individual flow amount instead of deducting only the individual flow's fee.

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
->        newAmounts[i] = amounts[i] - feeAmount;  // @audit deducts cumulative fee instead of individual fee
    }
}
```

### Internal Pre-conditions

1. User needs to have a vesting NFT with multiple token flows (multiple vesting periods)
2. Split or merge fee rate needs to be set to a non-zero value (e.g., 200 basis points = 2%)
3. User calls either `splitTVS` or `mergeTVS` on their multi-flow NFT

### External Pre-conditions

_No response_

### Attack Path

1. User has a vesting NFT with 3 separate token flows, each with 1000 tokens (3000 total)
2. User calls `splitTVS` with 2% fee rate to split their NFT
3. Fee calculation in `calculateFeeAndNewAmountForOneTVS`:
   - Flow 0: fee = 20 tokens, newAmount = 1000 - 20 = 980 (correct)
   - Flow 1: fee = 20 tokens (cumulative = 40), newAmount = 1000 - 40 = 960 (should be 980)
   - Flow 2: fee = 20 tokens (cumulative = 60), newAmount = 1000 - 60 = 940 (should be 980)
4. User's allocation is reduced to: 980 + 960 + 940 = 2880 tokens
5. Treasury receives: 60 tokens in fees (correct total fee)
6. User loses: 120 tokens instead of 60 tokens (100% excess loss on flows 1 and 2)


### Impact

Users with multi-flow vesting NFTs suffer exponentially increasing losses on each subsequent flow when performing split or merge operations.


### PoC

_No response_

### Mitigation

Fix the `calculateFeeAndNewAmountForOneTVS` function to deduct only the individual fee from each flow amount:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        uint256 individualFee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += individualFee;
        newAmounts[i] = amounts[i] - individualFee;  // Deduct only individual fee
        unchecked { ++i; }
    }
}
```

  