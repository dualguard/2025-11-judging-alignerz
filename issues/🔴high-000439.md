# [000439] `calculateFeeAndNewAmountForOneTVS` is flawed
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS` will revert due to flawed implementation

### Root Cause

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }

```

Above function will revert cause `newAmounts` is never initialized. and we can find similar issue `_computeSplitArrays` also

### Internal Pre-conditions

Users have to call `splitTVS` and `mergeTVS` 

### External Pre-conditions

None

### Attack Path

Users will either call `splitTVS` or `mergeTVS` which will trigger `calculateFeeAndNewAmountForOneTVS` and due to flawed implementation the tx will revert.

### Impact

Merging and Splitting TVS is not possible

### PoC

`POC :`

```solidity
    function test_arrayOutOfBounds() public {
        uint256 fee = 100;
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 1000;
        amounts[1] = 2000;
        uint256 length = amounts.length;
        vm.expectRevert();
        vesting.calculateFeeAndNewAmountForOneTVS(fee, amounts, length);
    }
```

`Logs :`

```shell

Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[PASS] test_arrayOutOfBounds() (gas: 24071)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 3.45s (1.01ms CPU time)

Ran 1 test suite in 3.45s (3.45s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

Implement the following code 
```solidity
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length); // new line
        for (uint256 i; i < length; ) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }

```


```solidity
    function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    ) internal view returns (Allocation memory alloc) {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        // Allocate arrays before assigning values
        alloc.amounts = new uint256[](nbOfFlows); //@audit added line to allocate amounts array
        alloc.vestingPeriods = new uint256[](nbOfFlows); //@audit added line to allocate vestingPeriods array
        alloc.vestingStartTimes = new uint256[](nbOfFlows); //@audit added line to allocate vestingStartTimes array
        alloc.claimedSeconds = new uint256[](nbOfFlows); //@audit added line to allocate claimedSeconds array
        alloc.claimedFlows = new bool[](nbOfFlows); //@audit added line to allocate claimedFlows array
        for (uint256 j; j < nbOfFlows; ) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }

```
  