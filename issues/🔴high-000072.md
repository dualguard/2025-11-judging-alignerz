# [000072] Uninitialized Array in calculateFeeAndNewAmountForOneTVS() Causes Complete DoS
  
  ## Summary

The [`calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169) function never initializes its `newAmounts` return array, causing all calls to revert with array out-of-bounds errors (Panic 0x32). This makes `splitTVS()` and `mergeTVS()` completely non-functional, preventing users from managing their vesting positions.

## Vulnerability Details

### Code

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // bug - newAmounts never initialized (length = 0)
    }
}
```

### Root Cause

The function declares `uint256[] memory newAmounts` as a return value but never initializes it. In Solidity, memory arrays must be explicitly sized before use:

```solidity
newAmounts = new uint256[](length);  // missing - This line is required
```

Without initialization:

- `newAmounts.length == 0`
- Any access to `newAmounts[i]` where `i >= 0` reverts with `Panic(0x32)` (array out-of-bounds access)
- The function always reverts on the first iteration when `i = 0`

### Where It's Called

This broken function is called by two critical user-facing functions:

1. `mergeTVS()` at [line 1013](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013):

```solidity
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external {
    // ...
    (uint256 feeAmount, uint256[] memory newAmounts) =
        calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);  // reverts here
    // ...
}
```

2. `splitTVS()` at [line 1069](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069):

```solidity
function splitTVS(uint256 projectId, uint256[] calldata percentages, uint256 splitNftId) external {
    // ...
    (uint256 feeAmount, uint256[] memory newAmounts) =
        calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows); // reverts here
    // ...
}
```

## Proof of Concept

Add this test to `AlignerzVestingProtocolTest.t.sol`:

```solidity
    function test_ArrayNotInitialized_Reverts() public {
        // Setup: Create test data with 3 flows
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000 ether;
        amounts[1] = 2000 ether;
        amounts[2] = 3000 ether;
        
        uint256 feeRate = 100; // 1%
        
        // This will revert with panic code 0x32 (array out-of-bounds access)
        // because newAmounts is never initialized in the function
        vm.expectRevert(stdError.indexOOBError);
        vesting.calculateFeeAndNewAmountForOneTVS(feeRate, amounts, 3);
    }
```

Run the test:

```bash
forge test --match-test "test_ArrayNotInitialized_Reverts" -vv
```

## Impact

### Complete DoS of Core Functionality

1. `splitTVS()` is completely non-functional

   - Users cannot split their vesting positions
   - Cannot divide TVS for partial transfers
   - Cannot separate flows for different beneficiaries
   - 100% of split attempts revert

2. `mergeTVS()` is completely non-functional
   - Users cannot consolidate multiple TVS NFTs
   - Cannot combine vesting positions
   - Cannot simplify portfolio management
   - 100% of merge attempts revert

## Recommended Mitigation

Initialize the `newAmounts` array before use:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {

    // FIX: Initialize the array with the correct length
    newAmounts = new uint256[](length);

    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;

        unchecked { ++i; }
    }
}
```

  