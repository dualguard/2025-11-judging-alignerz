# [000836] Incorrect fee subtraction in `FeesManager::calculateFeeAndNewAmountForOneTVS` function
  
  ### Summary

The function [`calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C4-L174C6) incorrectly subtracts the total accumulated fee (`feeAmount`) from each item in the amounts array.

### Root Cause

The `FeesManager::calculateFeeAndNewAmountForOneTVS` function computes the fee-adjusted token amounts and returns the `feeAmount` and the `newAmounts`:

```solidity

function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
@>      feeAmount += calculateFeeAmount(feeRate, amounts[i]);
@>      newAmounts[i] = amounts[i] - feeAmount;
    }
}

```

The problem is that the function subtracts the accumulated `feeAmount` from the `amounts[i]` instead of subtracting only the corresponding `fee` for the i-th `amount`. This leads to incorrect and lower values `newAmounts` array than the intended, only the first new amount will be correct. The function is used in the `AlignerzVesting::splitTVS` and `AlignerzVesting::mergeTVS` functions. That means the both function receives incorrect `newAmounts` array and this leads to loss of funds for the users.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

When the users want to split or merge TVS, they call the `AlignerzVesting::splitTVS` or `AlignerzVesting::mergeTVS`. These functions call the `FeesManager::calculateFeeAndNewAmountForOneTVS` function and expects to receive correct `feeAmount` value and `newAmounts` array. But the function returns incorect `newAmounts` array.

### Impact

Users loss funds when they want to split or merge TVS. Therefore, we have High Impact, because funds are directly lost (in risk not by attacker, but by the function logic) and the Likelihood is High, because this happens every time, when this function is called. Therefore, the Severity is High.


### PoC

Let's assume that the `newAmounts` array is properly initialized in the function `FeesManager::calculateFeeAndNewAmountForOneTVS` and the loop increments correctly, because otherwise the function will continue to revert. Also, the `amounts` array is populated with random values. The following function shows that the `FeesManager::calculateFeeAndNewAmountForOneTVS` function returns lower values for the `newAmounts` array than the expected, because of the incorrect subtraction of the accumulated `feeAmount` from the `amounts` array:

```solidity

//For this test we assume that the newAmounts array is properly initialized in the function FeesManager::calculateFeeAndNewAmountForOneTVS and the loop variable 'i' is correctly incremented
function testCalculateFeeAndNewAmountForOneTVS() public {
    uint256[] memory  amounts = new uint256[](3);
    amounts[0] = 1000;
    amounts[1] = 2000;
    amounts[2] = 3000;  
    vesting.calculateFeeAndNewAmountForOneTVS(40, amounts, amounts.length);
    
}

```
The result:

```solidity

Traces:
  [4949413] AlignerzVestingTest::setUp()
    ├─ [4811923] → new AlignerzVesting@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← [Return] 24031 bytes of code
    ├─ [94101] AlignerzVesting::initialize(0x0000000000000000000000000000000000000222)
    │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: AlignerzVestingTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit Initialized(version: 1)
    │   └─ ← [Stop] 
    └─ ← [Return] 

  [15969] AlignerzVestingTest::testCalculateFeeAndNewAmountForOneTVS()
    ├─ [9335] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(40, [1000, 2000, 3000], 3) [staticcall]
    │   └─ ← [Return] 24, [996, 1988, 2976]
    └─ ← [Return] 

```

The expected result (after the function is fixed):

```solidity
 [11255] AlignerzVestingTest::testCalculateFeeAndNewAmountForOneTVS()
    ├─ [4621] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(40, [1000, 2000, 3000], 3) [staticcall]
    │   └─ ← [Return] 24, [996, 1992, 2988]
    └─ ← [Return] 

```

It is obvious from the test results that the returned values of the `newAmounts` array after the first index are less than the values that the function should return.

### Mitigation

Create a new variable `fee` that holds the value of the fee for the current `amount` and subtract this fee from the `amount`. The `feeAmount` should be only a variable that holds the accumulated fee:

```diff

function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    //This is fixed.
    newAmounts = new uint256[](length);
+   uint256 fee;
    for (uint256 i; i < length;) {
+       fee = calculateFeeAmount(feeRate, amounts[i]);
+       feeAmount += fee;
+       newAmounts[i] = amounts[i] - fee;
-       feeAmount += calculateFeeAmount(feeRate, amounts[i]);
-       newAmounts[i] = amounts[i] - feeAmount;
        // This is fixed.
        ++i;
    }
}

```
  