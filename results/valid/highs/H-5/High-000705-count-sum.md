# [000705] Incorrect Cumulative Fee Deduction Causes Token Loss and DoS in mergeTVS/splitTVS
  
  ### Summary

In `FeesManager.sol:172` the cumulative `feeAmount` is deducted from each array element instead of the individual fee, causing a loss of funds for users as any user calling `mergeTVS()` or `splitTVS()` will have their vesting allocations incorrectly reduced, with losses scaling quadratically with the number of vesting flows.

### Root Cause

In [FeesManager.sol:172](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L172) the fee calculation deducts the cumulative `feeAmount` from each element instead of the individual fee for that element:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // @audit BUG: deducts cumulative fee, not individual
    }
}
```

The bug causes:
- Element 0: `amounts[0] - fee0` (correct)
- Element 1: `amounts[1] - (fee0 + fee1)` (should be `amounts[1] - fee1`)
- Element 2: `amounts[2] - (fee0 + fee1 + fee2)` (should be `amounts[2] - fee2`)
- And so on...

This function is called by:
- `mergeTVS()` at [AlignerzVesting.sol:1013](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013)
- `splitTVS()` at [AlignerzVesting.sol:1069](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069)


### Internal Pre-conditions

1. Owner needs to call `setFees()` to set `splitFeeRate` or `mergeFeeRate` to be greater than 0 (up to 200 basis points / 2%)
2. Owner needs to call `setTreasury()` to set treasury to a non-zero address
3. User needs to have an NFT with multiple vesting flows (amounts array length > 1)


### External Pre-conditions

None

### Attack Path

This is a vulnerability path (not an attack), triggered by normal protocol usage:

1. User owns an NFT representing a TVS (Token Vesting Schedule) with multiple vesting flows
2. User calls `mergeTVS()` or `splitTVS()` to manage their vesting position
3. The function calls `calculateFeeAndNewAmountForOneTVS()` to calculate fees and new amounts
4. Due to the cumulative fee bug, each subsequent flow in the user's allocation is reduced by more than the intended fee
5. The user's vesting allocation is permanently reduced beyond the intended fee amount

### Impact

The users suffer a direct loss of vested tokens proportional to the number of vesting flows:

| Flows | Fee Rate | Expected Loss | Actual Loss | Extra Loss |
|-------|----------|---------------|-------------|------------|
| 3     | 1%       | 1%            | 2%          | 1%         |
| 5     | 1%       | 1%            | 3%          | 2%         |
| 10    | 1%       | 1%            | 5.5%        | 4.5%       |
| 50    | 2%       | 2%            | 51%         | 49% (last flow zeroed) |
| 51+   | 2%       | 2%            | DoS         | Transaction reverts |

Loss formulas:
- Extra loss (tokens) = `fee_per_element * n * (n-1) / 2`
- Extra loss (% of total) = `feeRate * (n-1) / 2`

For a user with 10 vesting flows totaling 1,000,000 tokens at 1% fee:
- Expected fee: 10,000 tokens (1%)
- Actual loss: 55,000 tokens (5.5%)
- Extra loss: 45,000 tokens (4.5% beyond intended fee)

With 51+ flows at 2% fee rate, the cumulative fee exceeds individual element amounts causing integer underflow and transaction revert (DoS).

### PoC

Save this POC in `protocol/test/C02_IncorrectCumulativeFeeDeduction.t.sol`

Run with:
```bash
cd protocol && forge test --match-contract C02_IncorrectCumulativeFeeDeductionTest -vvv
```

### POC Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

contract BuggyFeesCalculator {
    uint256 constant public BASIS_POINT = 10_000;

    function calculateFeeAmount(uint256 feeRate, uint256 amount) public pure returns(uint256 feeAmount) {
        feeAmount = amount * feeRate / BASIS_POINT;
    }

    function calculateFeeAndNewAmountForOneTVS_BUGGY(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked { ++i; }
        }
    }

    function calculateFeeAndNewAmountForOneTVS_CORRECT(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += fee;
            newAmounts[i] = amounts[i] - fee;
            unchecked { ++i; }
        }
    }
}

contract C02_IncorrectCumulativeFeeDeductionTest is Test {
    BuggyFeesCalculator public calculator;

    uint256 constant BASIS_POINT = 10_000;
    uint256 constant ONE_PERCENT_FEE = 100;
    uint256 constant TWO_PERCENT_FEE = 200;

    function setUp() public {
        calculator = new BuggyFeesCalculator();
    }

    function test_C02_ProofOfBug_CumulativeVsIndividualFee() public view {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000e18;
        amounts[1] = 1000e18;
        amounts[2] = 1000e18;

        (, uint256[] memory buggyNewAmounts) =
            calculator.calculateFeeAndNewAmountForOneTVS_BUGGY(ONE_PERCENT_FEE, amounts, 3);

        (, uint256[] memory correctNewAmounts) =
            calculator.calculateFeeAndNewAmountForOneTVS_CORRECT(ONE_PERCENT_FEE, amounts, 3);

        assertEq(correctNewAmounts[0], 990e18);
        assertEq(correctNewAmounts[1], 990e18);
        assertEq(correctNewAmounts[2], 990e18);

        assertEq(buggyNewAmounts[0], 990e18);
        assertEq(buggyNewAmounts[1], 980e18);
        assertEq(buggyNewAmounts[2], 970e18);

        uint256 userLoss = (correctNewAmounts[0] + correctNewAmounts[1] + correctNewAmounts[2]) -
                          (buggyNewAmounts[0] + buggyNewAmounts[1] + buggyNewAmounts[2]);
        assertEq(userLoss, 30e18);
    }

    function test_C02_MaximumImpact_LargeMultiFlowTVS() public view {
        uint256[] memory amounts = new uint256[](10);
        for (uint256 i = 0; i < 10; i++) {
            amounts[i] = 100_000e18;
        }

        (, uint256[] memory buggyNewAmounts) =
            calculator.calculateFeeAndNewAmountForOneTVS_BUGGY(ONE_PERCENT_FEE, amounts, 10);

        (, uint256[] memory correctNewAmounts) =
            calculator.calculateFeeAndNewAmountForOneTVS_CORRECT(ONE_PERCENT_FEE, amounts, 10);

        uint256 correctTotal = 0;
        uint256 buggyTotal = 0;
        for (uint256 i = 0; i < 10; i++) {
            correctTotal += correctNewAmounts[i];
            buggyTotal += buggyNewAmounts[i];
        }

        uint256 userLoss = correctTotal - buggyTotal;
        assertEq(userLoss, 45_000e18);

        uint256 lossPercent = (userLoss * 10000) / 1_000_000e18;
        assertEq(lossPercent, 450);
    }

    function test_C02_ExtremeLoss_50Flows_LastFlowZeroed() public view {
        uint256[] memory amounts = new uint256[](50);
        for (uint256 i = 0; i < 50; i++) {
            amounts[i] = 1000e18;
        }

        (, uint256[] memory buggyNewAmounts) =
            calculator.calculateFeeAndNewAmountForOneTVS_BUGGY(TWO_PERCENT_FEE, amounts, 50);

        (, uint256[] memory correctNewAmounts) =
            calculator.calculateFeeAndNewAmountForOneTVS_CORRECT(TWO_PERCENT_FEE, amounts, 50);

        assertEq(buggyNewAmounts[49], 0);
        assertEq(correctNewAmounts[49], 980e18);

        uint256 correctTotal = 0;
        uint256 buggyTotal = 0;
        for (uint256 i = 0; i < 50; i++) {
            correctTotal += correctNewAmounts[i];
            buggyTotal += buggyNewAmounts[i];
        }

        uint256 userLoss = correctTotal - buggyTotal;
        assertEq(userLoss, 24_500e18);
    }

    function test_C02_DoS_UnderflowReverts_51Flows() public {
        uint256[] memory amounts = new uint256[](51);
        for (uint256 i = 0; i < 51; i++) {
            amounts[i] = 1000e18;
        }

        vm.expectRevert();
        calculator.calculateFeeAndNewAmountForOneTVS_BUGGY(TWO_PERCENT_FEE, amounts, 51);
    }
}

```

### Mitigation

Calculate and deduct individual fees instead of cumulative:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);
    for (uint256 i; i < length;) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;  // Deduct individual fee, not cumulative
        unchecked { ++i; }
    }
}
```
  