# [000849] Complete DoS of Split/Merge Operations Due to Missing Loop Increment
  
  ## Summary

The `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol` contains a missing loop increment that causes infinite loops, making all TVS split and merge operations impossible and causing complete protocol DoS.

## Vulnerability Detail

The fee calculation function is missing a critical loop increment, causing infinite loops when called:

**Location**: [`FeesManager.sol:169-174`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174)

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        // ❌ CRITICAL BUG: Missing loop increment - infinite loop!
    }
}
```

**Root Cause**: The loop variable `i` is never incremented, so `i < length` remains true forever, creating an infinite loop that will consume all available gas and cause transaction failure.

### Attack Mechanism

1. **User Action**: User calls `splitTVS` or `mergeTVS` to restructure their TVS position
2. **Function Call**: Protocol calls `calculateFeeAndNewAmountForOneTVS` to calculate fees
3. **Infinite Loop**: Loop starts with `i = 0` and never increments `i`
4. **Gas Exhaustion**: Transaction runs until gas limit is reached
5. **Transaction Failure**: All split/merge operations fail with out-of-gas error

**Critical Function Dependencies**:
- [`AlignerzVesting.sol:1013`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1013) - `mergeTVS` operation
- [`AlignerzVesting.sol:1069`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1069) - `splitTVS` operation

```solidity
// mergeTVS calls the broken function - will hang forever
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);

// splitTVS calls the broken function - will hang forever
(uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
```

## Impact

**High Severity**: Complete breakdown of core protocol functionality

- **Total System DoS**: All split and merge operations fail due to gas exhaustion
- **Core Features Unusable**: Users cannot restructure their TVS positions at all
- **Gas Drain Attack**: Malicious actors can cause users to waste gas on guaranteed-to-fail transactions
- **Protocol Reputation Damage**: 100% failure rate for major functionality creates user frustration
- **Economic Impact**: Users lose gas fees on every failed attempt

**Affected Operations**:
```solidity
splitTVS(nftId, [50, 50]);      // ❌ Always runs out of gas
mergeTVS([nftId1, nftId2]);     // ❌ Always runs out of gas
// 100% failure rate due to infinite loop
```

**Infinite Loop Pattern**:
```solidity
// What happens when function is called:
uint256 i = 0;
while (i < 2) {  // length = 2
    // Process amounts[0] forever
    // i never changes - infinite loop!
    // Eventually: Error: Transaction ran out of gas
}
```

## Code Snippet

**Current broken implementation**:
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        // ❌ MISSING: unchecked { ++i; }
    }
    // This return statement is never reached due to infinite loop
}
```

**Usage context showing impact**:
```solidity
// AlignerzVesting.sol - mergeTVS function
function mergeTVS(...) external {
    // Function execution stops here forever:
    (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
    // Everything below never executes
    mergedTVS.amounts = newAmounts; 
    // ...
}
```

## Tool used

Manual Review / Foundry Testing

## Recommendation

**Immediate Fix** - Add the missing loop increment:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        unchecked {
            ++i; //  Add missing loop increment
        }
    }
}
```
  