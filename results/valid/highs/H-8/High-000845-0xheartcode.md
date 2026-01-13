# [000845] Infinite Loop DoS in Dividend Distribution Due to Continue Statements Bypassing Loop Increment
  
  ## Summary

The `getUnclaimedAmounts` function in `A26ZDividendDistributor.sol` contains a critical loop structure flaw where `continue` statements bypass the loop increment, causing infinite loops during normal dividend calculations when flows are claimed or in initial state.

## Vulnerability Detail

The vulnerability exists in the loop structure of the dividend calculation function:

**Location**: [`A26ZDividendDistributor.sol:147-158`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147-L158)

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // ... variable declarations ...
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;      // ❌ CRITICAL: Bypasses increment!
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;                       // ❌ CRITICAL: Bypasses increment!
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked {
            ++i;  // ❌ Only reached if no continue statements execute
        }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

**Root Cause**: The loop increment `unchecked { ++i; }` is placed after conditional `continue` statements. When `continue` executes, the increment is bypassed, leaving the loop variable `i` unchanged and creating an infinite loop.

### Attack Mechanism

**Normal Protocol Operation** triggers infinite loops:

1. **User Claims Dividend Flow**: User claims one flow of their TVS, setting `claimedFlows[0] = true`
2. **Dividend Recalculation**: Protocol calls `getUnclaimedAmounts` to recalculate remaining dividends
3. **Infinite Loop Trigger**: 
   ```solidity
   i = 0, len = 3 (3 flows total)
   Loop iteration 1: claimedFlows[0] == true → continue
   Loop iteration 2: i is still 0 → claimedFlows[0] == true → continue
   Loop iteration 3: i is still 0 → claimedFlows[0] == true → continue
   // Infinite loop - transaction runs out of gas
   ```
4. **Gas Exhaustion**: Transaction fails with out-of-gas error

**Code References**:
- Called by [`A26ZDividendDistributor.sol:131`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L131) - `getTotalUnclaimedAmounts`
- Used in dividend distribution calculations throughout the protocol

## Impact

**High Severity**: Complete breakdown of dividend distribution system

- **Protocol DoS**: Dividend calculations become impossible once any flow is claimed
- **User Fund Lock**: Users cannot claim remaining dividends due to calculation failures
- **Economic Loss**: Users lose gas on every failed dividend calculation attempt
- **Business Logic Failure**: Core 5% profit sharing mechanism becomes inoperable
- **Systematic Impact**: Affects all users with multi-flow TVS positions

**Real-World Scenarios**:
```solidity
// Scenario 1: User claims first flow
user.claimFlow(nftId, 0);  // Sets claimedFlows[0] = true
// All subsequent dividend calculations enter infinite loop

// Scenario 2: New TVS with zero claimed seconds
// claimedSeconds[i] == 0 for new allocations
// Infinite loop on first dividend calculation
```

**Critical Function Dependencies**:
- `getTotalUnclaimedAmounts()` - Used for total dividend pool calculation
- `_setDividends()` - Uses cached results from this broken function
- All dividend distribution operations become unusable

## Code Snippet

**Current broken implementation**:
```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;  // ❌ Skips increment - infinite loop
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;  // ❌ Skips increment - infinite loop
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked {
            ++i;  // ❌ Only reached for partial claims
        }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

**Context showing normal operation triggers the bug**:
```solidity
// getTotalUnclaimedAmounts calls the broken function
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i); // ❌ Infinite loop here
        unchecked { ++i; }
    }
}
```


## Recommendation
Use standard for loop with built-in increment.
  