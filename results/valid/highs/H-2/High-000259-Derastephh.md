# [000259] `FeesManager::calculateFeeAndNewAmountForOneTVS` Always Reverts Due to Unallocated `newAmounts` Array
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069

The `FeesManager::calculateFeeAndNewAmountForOneTVS` function writes to `newAmounts[i]` without ever allocating the array. Since `newAmounts` has a default length of 0, any attempt to access index 0 causes a `Panic(0x32)` array out of bounds access revert. This makes both `AlignerzVesting::mergeTVS` and `AlignerzVesting::splitTVS` unusable, as they always revert when calling this function.

### Root Cause

No memory was allocated for the `newAmounts` array before writing to it. In Solidity, memory dynamic arrays declared as return variables start with:
- length = 0
- no allocated slots

Thus, accessing:
```solidity
newAmounts[i]
```
without an initial allocation is an out of bounds revert.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

The affected function:
```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // always reverts
    }
}
```
- `newAmounts` length = 0
- first loop iteration tries `newAmounts[0]`
- Panic(0x32) array out of bounds reverts

The revert happens in:
- `AlignerzVesting::mergeTVS`:
```solidity
(uint256 feeAmount, uint256[] memory newAmounts) =
    calculateFeeAndNewAmountForOneTVS(...); // always reverts
```
- `AlignerzVesting::splitTVS`:
```solidity
(uint256 feeAmount, uint256[] memory newAmounts) =
    calculateFeeAndNewAmountForOneTVS(...); // always reverts
```
Both are user facing operations, and they are completely unuseable as they will always revert.

### Impact

- Users cannot merge Tokenized Vesting Schedules
- Users cannot split Tokenized Vesting Schedules
- Treasury fee logic cannot run

### PoC

Paste the below test in `protocol/test/AlignerzVestingPotocolTest.t.sol` and run test with `forge test --match-test test_FeesManager_calculateFeeAndNewAmountForOneTVS_broken -vv`.

```solidity
function test_FeesManager_calculateFeeAndNewAmountForOneTVS_broken() public {
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 1 ether;
        amounts[1] = 2 ether;

       
        vm.expectRevert();
        vesting.calculateFeeAndNewAmountForOneTVS(1, amounts, 2);
    }
```

### Mitigation

- Allocate the array before writing to it:
```solidity
newAmounts = new uint256[](length);
```
  