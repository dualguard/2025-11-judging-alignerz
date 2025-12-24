# [000838] Missing Initialization of `newAmounts` in `FeesManager::calculateFeeAndNewAmountForOneTVS` function
  
  ### Summary

The function [`FeesManager::calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C4-L174C6) attempts to compute fee-adjusted token amounts but doesn't initialize the memory array `newAmounts` before writing into it. This causes a runtime revert due to out-of-bounds writes in memory, making the function unusable in its current state.

### Root Cause

The `FeesManager::calculateFeeAndNewAmountForOneTVS` function doesn't initialize the expected return array `newAmounts`:


```solidity

function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}

```

When the function is called, it reverts, because the line `newAmounts[i] = amounts[i] - feeAmount;` writes into memory that has not been allocated, causing a low-level panic: array out-of-bounds.

There are also other issues in this functions, but this report is only for the missing initializing of the `newAmounts` array, because each of the issues is a separated root cause.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

It is not attack path, it is the function flow call, when the issue occurs: each time when a user calls `AlignerzVesting::splitTVS` or `AlignerzVesting::mergeTVS`, because they both call the `calculateFeeAndNewAmountForOneTVS` function and will revert due to the revert in this call.

### Impact

The function `FeesManager::calculateFeeAndNewAmountForOneTVS` is called in `AlignerzVesting::mergeTVS` and `AlignerzVesting::splitTVS`. These functions allow the user to merge and split TVS. The issue in the `FeesManager::calculateFeeAndNewAmountForOneTVS` function, means that the users can't use the functionality of the `mergeTVS` and `splitTVS`, because they will revert also when the `calculateFeeAndNewAmountForOneTVS` reverts. Therefore, the Impact is High: "Severe disruption of protocol functionality or availability" and the Likelihood is High, because it will happen every time when the function is called. Therefore, the Severity is also High.

### PoC

The following test shows that when the function is called, it reverts, because of the missing initialization of the `newAmounts` array:

```solidity
contract AlignerzVestingTest is StdInvariant, Test {
    
    AlignerzVesting vesting;

    function setUp() public {
        vesting = new AlignerzVesting();
        vesting.initialize(address(0x222));
    }

    function testCalculateFeeAndNewAmountForOneTVS() public {
        uint256[] memory  amounts = new uint256[](3);
        amounts[0] = 1000;
        amounts[1] = 2000;
        amounts[2] = 3000;

        // This will revert due to array out-of-bounds access, because the function FeesManager::calculateFeeAndNewAmountForOneTVS doesn't initialize the uint256[] memory newAmounts
        vm.expectRevert();  
        vesting.calculateFeeAndNewAmountForOneTVS(20, amounts, amounts.length);
    }
}

```

The result from the test:

```
Traces:
  [4925350] AlignerzVestingTest::setUp()
    ├─ [4787896] → new AlignerzVesting@0x5615dEB798BB3E4dFa0139dFa1b3D433Cc23b72f
    │   └─ ← [Return] 23911 bytes of code
    ├─ [94101] AlignerzVesting::initialize(0x0000000000000000000000000000000000000222)
    │   ├─ emit OwnershipTransferred(previousOwner: 0x0000000000000000000000000000000000000000, newOwner: AlignerzVestingTest: [0x7FA9385bE102ac3EAc297483Dd6233D62b3e1496])
    │   ├─ emit Initialized(version: 1)
    │   └─ ← [Stop] 
    └─ ← [Return] 

  [13507] AlignerzVestingTest::testCalculateFeeAndNewAmountForOneTVS()
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return] 
    ├─ [2520] AlignerzVesting::calculateFeeAndNewAmountForOneTVS(20, [1000, 2000, 3000], 3) [staticcall]
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return] 

```

### Mitigation

Initialize the `newAmounts` array in the function properly:

```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+   newAmounts = new uint256[](length);    
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}

```
  