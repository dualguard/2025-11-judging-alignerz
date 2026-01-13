# [000013] Fee overcharge in `calculateFeeAndNewAmountForOneTVS`
  
  ### Summary

Cumulative fee accumulation in `calculateFeeAndNewAmountForOneTVS` miscalculates per flow amounts and can underflow. The fee helper `calculateFeeAndNewAmountForOneTVS` incorrectly subtracts the cumulative fee from each flow instead of the per-flow fee. This leads to: 
- Over-charging later flows (e.g 20% instead of 10%, etc)
- Potential underflow and revert for realistic fee rates and flow patterns.
This breaks the correctness of the fee model for TVS merges/splits and can still cause partial DoS of those operations.

### Root Cause

`[calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174)` iterates along `newAmounts` charging on each amount to be merged. 
(note `newAmounts` was not initialized, an issue detailed in #8 )
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);// <@cumulative
            newAmounts[i] = amounts[i] - feeAmount; // <@  // subtract cumulative fee
        }
    }
```
The expected flow is that for each flow `i`:
- `fee_i = calculateFeeAmount(feeRate, amounts[i]);`
- `newAmount_i = amounts[i] - fee_i;`
- `totalFee += fee_i`
But what is happening is:
- For `i = 0`: fee is correct (subtracts fee_0)
- For `i = 1`: subtracts fee_0 + fee_1
- For `i = 2`: subtracts fee_0 + fee_1 + fee_2

This over taxes later flows and can make `newAmounts[i]` negative (underflow) once cumulative fees exceed that flow’s amount.

### Internal Pre-conditions

1. `calculateFeeAndNewAmountForOneTVS` is called with `length > 1` (multiple flows).
2. Fee rate and amounts such that:
- second or later flows are non-trivial, and
- cumulative fees can approach or exceed individual flow amounts.
These are normal configurations for multi flow TVS positions with non zero fees.

### External Pre-conditions

None

### Attack Path

Nil

### Impact

High impact: 
There are 2 distinct failure modes:
1. Silent economic miscalculation: Each flow after the first is charged more than `feeRate` on its principal. For example, with amounts [100, 100] and a 10% fee:
- Expected: (90, 90) and total fee 20.
- Actual: (90, 80) and total fee still 20, but second flow is charged 20% effectively.
2. Underflow/DoS: For certain combinations (larger feeRate, multiple flows, smaller amounts), cumulative fees exceed the flow’s amount and underflow on subtraction. This reverts the operation and breaks the merge/split feature in those configurations.

Likelihood: High
Any use of multi-flow TVSs with non trivial fee rates will be affected

### PoC

For the test, we use a local harness that mirrors the logic because in the main suite, `newAmounts` is not allocated as detailed in #8 
Place this test in `test/AlignerzVestingProtocolTest.t.sol`

```solidity

    function test_BadFeeHelper_overchargesLaterFlows_dueToCumulativeFee() public {
        BadFeeHelperHarness feeHarness = new BadFeeHelperHarness();
        
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 100 ether;
        amounts[1] = 100 ether;
        uint256 feeRate = 1_000; // 10%

        (uint256 totalFee, uint256[] memory newAmounts) =
            feeHarness.badCalculateFeeAndNewAmountForOneTVS(feeRate, amounts);

        assertEq(totalFee, 20 ether);      // 10 + 10
        assertEq(newAmounts[0], 90 ether); // charged 10%
        assertEq(newAmounts[1], 80 ether); // effectively charged 20% instead of 10%
    }

    function test_BadFeeHelper_underflowsOnLaterFlows() public {
        BadFeeHelperHarness feeHarness = new BadFeeHelperHarness();

        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 10 ether;
        amounts[1] = 10 ether;
        amounts[2] = 10 ether;

        uint256 feeRate = 5_000; // 50%

        // i=0: feeAmount=5,  new[0]=5
        // i=1: feeAmount=10, new[1]=0
        // i=2: feeAmount=15, new[2]=10-15 => underflow → revert

        vm.expectRevert();
        feeHarness.badCalculateFeeAndNewAmountForOneTVS(feeRate, amounts);
    }


}

contract BadFeeHelperHarness {
    uint256 public constant BASIS_POINT = 10_000;

    function badCalculateFeeAndNewAmountForOneTVS(
        uint256 feeRate,
        uint256[] memory amounts
    )
        external
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        uint256 length = amounts.length;
        newAmounts = new uint256[](length);

        for (uint256 i; i < length;) {
            // BUG: cumulative fee used for subtraction
            feeAmount += (amounts[i] * feeRate) / BASIS_POINT;
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked { ++i; }
        }
    }
}
```

### Mitigation

1. Use per-flow fee instead of cumulative when subtracting:
```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    newAmounts = new uint256[](length);

    for (uint256 i; i < length;) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]); // per-flow fee
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;
        unchecked { ++i; }
    }
}
```
  