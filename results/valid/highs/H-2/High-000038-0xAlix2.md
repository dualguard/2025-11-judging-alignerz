# [000038] `newAmounts` is not initialized in `calculateFeeAndNewAmountForOneTVS` blocking merging and splitting TVS
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS` is called when either merging or splitting TVS positions, which is used to cut the fees from the passed amounts. However, `newAmounts` is not initialized, and all calls to it will revert with `array out-of-bounds access`:
```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length; ) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

### Root Cause

`newAmounts` is not initialized in `calculateFeeAndNewAmountForOneTVS`, https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

N/A

### Impact

Merging and splitting TVS can never be done.

### PoC

```solidity
function test_UninitializedNewAmountsArray() public {
    uint256[] memory amounts = new uint256[](1);
    amounts[0] = 1000;

    vm.expectRevert(); // array out-of-bounds access
    vesting.calculateFeeAndNewAmountForOneTVS(200, amounts, 1);
}
```

### Mitigation

Consider initializing the new amounts array at the very beginning:
```solidity
newAmounts = new uint256[](length);
```
  