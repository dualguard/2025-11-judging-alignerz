# [000850] Immediate Protocol DoS on Launch - Fresh NFT Infinite Loop in Dividend Distribution
  
  ## Summary

The `getUnclaimedAmounts` function contains a critical infinite loop bug that **immediately breaks all dividend distribution functionality on protocol launch**. Fresh NFTs have `claimedSeconds[i] == 0`, triggering infinite loops that make dividend distribution unusable from day 1 of deployment.

## Vulnerability Detail

The `getUnclaimedAmounts` function has a fatal loop structure where the `continue` statement prevents the loop counter from incrementing when `claimedSeconds[i] == 0`, which is the **default state for all fresh NFTs**.

**Location**: [`A26ZDividendDistributor.sol:147-158`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L147-L158)

```solidity
for (uint i; i < len;) {
    if (claimedFlows[i]) continue;
    if (claimedSeconds[i] == 0) { // ← BUG: Fresh NFTs have claimedSeconds == 0
        amount += amounts[i];
        continue; // ← INFINITE LOOP: i never increments
    }
    // ... other logic ...
    unchecked {
        ++i; // ← NEVER REACHED when claimedSeconds[i] == 0
    }
}
```

### Root Cause Analysis

**Critical Issue**: All NFTs start with `claimedSeconds[i] == 0` until users make their first claim:

**Fresh Allocation State** (default values):
```solidity
struct Allocation {
    uint256[] amounts;
    uint256[] vestingPeriods; 
    uint256[] vestingStartTimes;
    uint256[] claimedSeconds; // ← Defaults to [0, 0, 0, ...] for fresh NFTs
    bool[] claimedFlows;      // ← Defaults to [false, false, false, ...]
}
```

**Execution Flow**:
1. Fresh NFT: `claimedSeconds[0] == 0` ✓
2. Line 149: `if (claimedSeconds[i] == 0)` → **true**
3. Line 151: `continue` → **skips increment at line 157**
4. Loop condition `i < len` → **still true** (i=0, len>0)
5. **Infinite loop** → gas exhaustion → transaction failure

### Critical Function Dependencies

**Primary Entry Point**: [`getTotalUnclaimedAmounts()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L127-L135)

```solidity
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i); // ← FAILS HERE
    }
}
```

**Admin Functions Affected**:
- [`setUpTheDividends()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L113) → calls `_setAmounts()` → calls `getTotalUnclaimedAmounts()`
- [`setAmounts()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L117) → calls `getTotalUnclaimedAmounts()`

## Impact

Complete protocol dysfunction from deployment

Dividend system unusable from deployment day 1
Unlike eventual DoS from fully claimed flows, this triggers immediately
Every fresh NFT causes infinite loop
Protocol cannot distribute dividends until fixed

### Timeline Analysis

1. **Day 1**: Protocol launches with fresh NFTs → all have `claimedSeconds[i] == 0`
2. **First Dividend Attempt**: Admin calls `setUpTheDividends()` or `setAmounts()`
3. **Immediate Failure**: First NFT with `claimedSeconds[0] == 0` triggers infinite loop
4. **Protocol DOA**: Dividend functionality dead on arrival

### Attack Mechanism


1. **Protocol Deployment**: Fresh NFTs minted with default `claimedSeconds[i] == 0`
2. **Admin Dividend Setup**: Admin attempts first dividend distribution  
3. **Immediate DoS**: `getUnclaimedAmounts()` hits infinite loop on first fresh NFT
4. **Complete Failure**: All dividend functionality permanently broken


## Code Snippet

```solidity
// A26ZDividendDistributor.sol:127-134 - Will fail on first dividend attempt
function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (, bool isOwned) = safeOwnerOf(i);
        if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i); // ← FAILS HERE
    }
}

// A26ZDividendDistributor.sol:147-158 - The immediate infinite loop
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // ... setup code ...
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) { // ← Fresh NFT: i=0, claimedSeconds[0]=0
            amount += amounts[i];
            continue; // ← INFINITE LOOP: continue without ++i  
        }
        // ... calculation logic ...
        unchecked { ++i; } // ← NEVER REACHED when claimedSeconds[i]=0
    }
}
```

  