# [000377] Uninitialized Memory Arrays Causing Split and Merge Function Failures
  
  ## Summary

The Alignerz Protocol contains two critical vulnerabilities where memory arrays are accessed by index without being initialized with a proper length first. These vulnerabilities cause runtime reverts in core functionality, preventing users from splitting or merging TVS. The affected functions attempt to write to array indices that don't exist because the arrays are created with zero length by default.

**Key Findings:**

- **AlignerzVesting.sol:** [_computeSplitArrays()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1121) function accesses uninitialized arrays, causing `splitTVS()` to revert
- **FeesManager.sol:** [calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C164-L174) function accesses uninitialized arrays, causing both `splitTVS() and `mergeTVS()` to revert
- Both functions create memory arrays but never initialize them with the required length before accessing elements
- This completely breaks the split and merge functionality for all users

**Affected Locations:**

- AlignerzVesting.sol:[_computeSplitArrays()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141)
- FeesManager.sol: [calculateFeeAndNewAmountForOneTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C164-L174)

## Root Cause

### Memory Array Default Initialization

In Solidity, when you declare a memory array without initializing it, it defaults to an empty array with length zero:

```solidity
uint256[] memory newAmounts;  // Length = 0, empty array
newAmounts[0] = value;         // Reverts: index out of bounds
```

### Vulnerable Code Patterns

**Issue 1: AlignerzVesting.sol - `_computeSplitArrays()`**

```solidity
function _computeSplitArrays(...) returns (Allocation memory alloc) {
    // alloc is created with default values
    // alloc.amounts has length 0
    // alloc.vestingPeriods has length 0
    // etc.
    
    for (uint256 j; j < nbOfFlows;) {
        alloc.amounts[j] = ...;           // Reverts: accessing index 0 of empty array
        alloc.vestingPeriods[j] = ...;   // Reverts: accessing index 0 of empty array
        alloc.vestingStartTimes[j] = ...; // Reverts: accessing index 0 of empty array
        alloc.claimedSeconds[j] = ...;    // Reverts: accessing index 0 of empty array
        alloc.claimedFlows[j] = ...;      // Reverts: accessing index 0 of empty array
    }
}
```

**Issue 2: FeesManager.sol - `calculateFeeAndNewAmountForOneTVS()`**

```solidity
function calculateFeeAndNewAmountForOneTVS(...)
    returns (uint256 feeAmount, uint256[] memory newAmounts) {
    // newAmounts is declared but never initialized
    // newAmounts has length 0 by default
    
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // Reverts: accessing index i of empty array
    }
}
```

### Technical Explanation

When a struct or array is declared in memory:
- Arrays default to empty arrays (length = 0)
- Accessing any index (even index 0) of an empty array causes an out-of-bounds error
- Solidity reverts the transaction when array bounds are exceeded
- The arrays must be explicitly initialized with the required length before accessing elements

## Impact

### Functional Impact

- Both `splitTVS()` and `mergeTVS()` functions are completely non-functional

### Financial Impact
- Users pay gas fees for failed transactions
- No functionality is provided despite gas consumption
- Users cannot manage their vesting positions as intended

### Protocol-Wide Impact

- Split functionality: 100% broken (always reverts)
- Merge functionality: 100% broken (always reverts)
- Affects all users attempting these operations
- No workaround available within the contract

## Attack Path

### Scenario 1: User Attempts to Split TVS Position

1. User has a TVS NFT with 10,000 tokens vesting over 90 days
2. User wants to split it into two 50/50 positions
3. User calls `splitTVS(projectId, [5000, 5000], nftId)`
4. Contract executes `splitTVS()` function
5. Function calls `calculateFeeAndNewAmountForOneTVS()` - succeeds (uses storage arrays)
6. Function calls `_computeSplitArrays()` for each split percentage
7. `_computeSplitArrays()` creates `Allocation memory alloc`
8. Arrays in `alloc` are empty (length 0)
9. Loop tries to access `alloc.amounts[0]`
10. Solidity detects out-of-bounds access
11. Transaction reverts with array bounds error
12. User's transaction fails, gas is consumed
13. User cannot split their position

### Scenario 2: User Attempts to Merge Multiple TVS Positions

1. User has 3 TVS NFTs they want to merge into one
2. User calls `mergeTVS(projectId, mergedNftId, projectIds, nftIds)`
3. Contract executes `mergeTVS()` function
4. Function calls `calculateFeeAndNewAmountForOneTVS()` for the merged NFT
5. `calculateFeeAndNewAmountForOneTVS()` declares `uint256[] memory newAmounts`
6. Array `newAmounts` is empty (length 0) by default
7. Loop tries to access `newAmounts[0]` for first flow
8. Solidity detects out-of-bounds access
9. Transaction reverts with array bounds error
10. User's transaction fails, gas is consumed
11. User cannot merge their positions

## Mitigation
Initialize Arrays with Required Length


  