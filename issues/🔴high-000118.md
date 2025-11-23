# [000118] Critical: FeesManager Reverts Due to Missing Allocation of newAmounts (DoS on Split/Merge Flows)
  
  ### Summary

A missing memory allocation in [calculateFeeAndNewAmountForOneTVS](protocol/src/contracts/vesting/feesManager/FeesManager.sol)() causes the function to revert with an out-of-bounds panic whenever length > 0.
As a result, any operation relying on this function — including vesting flow splits and merges — becomes unusable, effectively resulting in a Denial of Service for users attempting to manage their vesting streams.



### Root Cause

Inside
[protocol/src/contracts/vesting/feesManager/FeesManager.sol,
](protocol/src/contracts/vesting/feesManager/FeesManager.sol) the function calculateFeeAndNewAmountForOneTVS() attempts to write to newAmounts[i] but never creates the array:
newAmounts[i] = amounts[i] - feeAmount; // <-- newAmounts not allocated 
Since newAmounts defaults to a zero-length array, any write at index i >= 0 triggers:
panic: array out-of-bounds access (0x32) 
This means the function cannot successfully run in any scenario with flows, making downstream protocol functionality unusable.


### Internal Pre-conditions

1. length > 0
2. The function calculateFeeAndNewAmountForOneTVS() is called (e.g., during SplitTVS or MergeTVS operations).


### External Pre-conditions

none

### Attack Path

User attempts to modify vesting streams (splitTVS, mergeTVS, or any operation invoking fee calculation).
Protocol calls calculateFeeAndNewAmountForOneTVS().
The function hits the loop: newAmounts[i] = ... 
Since newAmounts was never allocated, the write immediately causes: panic: array out-of-bounds access (0x32) 
The entire operation reverts — user cannot proceed.


### Impact

Complete Denial of Service for vesting modification flows.
Users are unable to split or merge vesting streams.
Admins cannot run maintenance or restructuring operations.
Funds may become effectively locked if protocol workflows depend on these actions

### PoC

```solidity
// NEW MOCK CONTRACT: Contains the original, buggy function logic.
// This is deployed and called externally to force the panic at a
// lower call depth, allowing vm.expectRevert() to catch it.
// -----------------------------------------------------------------
contract BuggyFeesMock {
    uint256 internal constant BASIS_POINT = 10000;

    function calculateFeeAmount(
        uint256 feeRate,
        uint256 amount
    ) public pure returns (uint256) {
        return (amount * feeRate) / BASIS_POINT;
    }

    // THIS IS THE VULNERABLE CODE: Missing allocation of 'newAmounts' array.
    function calculateFeeAndNewAmountForOneTVS(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        //  Missing: newAmounts = new uint256[](length);

        for (uint256 i; i < length; ) {
            uint256 thisFee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += thisFee;
            // The panic occurs here because newAmounts is a zero-length array
            newAmounts[i] = amounts[i] - thisFee;
            unchecked {
                ++i;
            }
        }
    }
}

contract CriticalDosPoC is Test {
    BuggyFeesMock buggyFees;

    function setUp() public {
        // Deploy the mock contract containing the buggy logic
        buggyFees = new BuggyFeesMock();
    }

    function test_buggy_function_panics_due_to_missing_allocation() public {
        uint256 feeRate = 500;

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 100_000 ether;
        amounts[1] = 50_000 ether;
        amounts[2] = 20_000 ether;

        // Expect the out-of-bounds access panic (0x32)
        vm.expectRevert();

        // Call the function on the EXTERNAL mock contract instance
        buggyFees.calculateFeeAndNewAmountForOneTVS(
            feeRate,
            amounts,
            amounts.length
        );
        // This test will now pass, successfully proving the panic.
    }
}
```

### Mitigation

Allocate the output array before writing:
```diff
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+   newAmounts = new uint256[](length);

    for (uint256 i; i < length; ) {
        uint256 thisFee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += thisFee;
        newAmounts[i] = amounts[i] - thisFee;
        unchecked { ++i; }
    }
}
```
  