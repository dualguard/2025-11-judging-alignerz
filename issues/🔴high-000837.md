# [000837] Infinite loop due to missing increment of loop variable
  
  ### Summary

The function [`FeesManager::calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C4-L174C6) attempts to compute fee-adjusted token amounts but doesn't increment the loop variable `i` and the function reverts due to out of gas.

### Root Cause

`FeesManager::calculateFeeAndNewAmountForOneTVS` function uses a for-loop to compute the the `feeAmount` and to fill the the `newAmounts` array:

```solidity

function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}

```

The problem is that the loop variable `i` is not incremented and because of that the function reverts. 


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

The issue occurs each time when a user calls `AlignerzVesting::splitTVS` or `AlignerzVesting::mergeTVS`, because they both call the `calculateFeeAndNewAmountForOneTVS` function and will revert due to the revert in this call.

### Impact

The function `FeesManager::calculateFeeAndNewAmountForOneTVS` is called in `AlignerzVesting::mergeTVS` and `AlignerzVesting::splitTVS`. These functions allow the user to merge and split TVS. The issue in the `FeesManager::calculateFeeAndNewAmountForOneTVS` function, means that the users can't use the functionality of the `mergeTVS` and `splitTVS`, because they will revert also when the `calculateFeeAndNewAmountForOneTVS` reverts. Therefore, the Impact is High: "Severe disruption of protocol functionality or availability" and the Likelihood is High, because it will happen every time when the function is called. Therefore, the Severity is also High.

### PoC

Let's assume that the `newAmounts` array in the `FeesManager::calculateFeeAndNewAmountForOneTVS` function is initialized correctly. The following test shows that the function reverts becuase of out of gas issue:

```solidity

//For this test we assume that the newAmounts array is properly initialized in the function FeesManager::calculateFeeAndNewAmountForOneTVS and the feeAmount is correctly assigned.
function testCalculateFeeAndNewAmountForOneTVS() public {
    uint256[] memory  amounts = new uint256[](3);
    amounts[0] = 1000;
    amounts[1] = 2000;
    amounts[2] = 3000;

    // This will revert due to OutOfGas, because the function FeesManager::calculateFeeAndNewAmountForOneTVS doesn't increment the loop variable 'i'.
    vm.expectRevert();  
    vesting.calculateFeeAndNewAmountForOneTVS(50, amounts, amounts.length);
    
}

```

The result:

```Traces:
  [4920132] AlignerzVestingTest::setUp()
    ├─ [4782687] → new AlignerzVesting@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← [Return] 23885 bytes of code
    ├─ [94101] AlignerzVesting::initialize(0x0000000000000000000000000000000000000222)
    │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: AlignerzVestingTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit Initialized(version: 1)
    │   └─ ← [Stop] 
    └─ ← [Return] 

  [1056946245] AlignerzVestingTest::testCalculateFeeAndNewAmountForOneTVS()
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return] 
    ├─ [1056935073] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(50, [1000, 2000, 3000], 3) [staticcall]
    │   └─ ← [OutOfGas] EvmError: OutOfGas
    └─ ← [Return] 


```

### Mitigation

Increment the loop variable `i`:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // Let's assume that this issue is fixed
    newAmounts = new uint256[](length);    
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
++  unchecked {
++      ++i;
++  }
}

```
  