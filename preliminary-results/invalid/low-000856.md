# [000856] Duplicate KOL addresses in reward allocation create accounting mismatches
  
  ## Summary
The `setTVSAllocation` and `setStablecoinAllocation` functions do not prevent duplicate KOL addresses, causing allocation overwrites and accounting inconsistencies when admin accidentally provides the same address multiple times.

## Vulnerability Detail
The reward allocation functions allow admins to set KOL allocations but lack duplicate address validation. When the same KOL address appears multiple times due to admin error, the following operational issues occur:

```solidity
// setTVSAllocation function (Lines 464-477)
for (uint256 i = 0; i < length; i++) {
    address kol = kolTVS[i];
    rewardProject.kolTVSAddresses.push(kol);     // Always pushes - no duplicate check
    uint256 amount = TVSamounts[i];
    rewardProject.kolTVSRewards[kol] = amount;   // Overwrites previous allocation
    rewardProject.kolTVSIndexOf[kol] = i;        // Overwrites index mapping
    totalAmount += amount;                       // Counts all amounts including duplicates
}
```

### Configuration Error Flow:
1. Admin accidentally provides duplicate KOL addresses during reward setup
2. Array stores all entries including duplicates
3. Mapping overwrites previous allocations with latest value
4. Total token calculation includes all amounts, but only latest allocation is claimable
5. Difference creates accounting mismatch between transferred and allocated amounts

**Example Scenario:**
```solidity
// Admin input:
kolTVS = [Alice, Bob, Alice, Charlie]  // Alice appears twice
TVSamounts = [100, 200, 300, 400]     // Total = 1000 tokens
totalTVSAllocation = 1000

// Result:
kolTVSAddresses = [Alice, Bob, Alice, Charlie]  // 4 entries
kolTVSRewards[Alice] = 300  // Latest value, 100 is lost
kolTVSRewards[Bob] = 200
kolTVSRewards[Charlie] = 400
// Claimable total: 900 tokens
// Unallocated tokens: 100 tokens (remain in contract)
```

## Impact
This configuration issue creates operational inefficiencies and accounting complications:

- Tokens become unallocated and remain in contract balance
- Creates accounting mismatch between intended and actual distributions
- No feedback when duplicate addresses are provided

**Example:**
- Admin intends Alice to receive: 400 tokens total (100 + 300)
- Actual allocation after duplicate: 300 tokens
- Unallocated amount: 100 tokens (remains in contract, not destroyed)

## Code Snippet

**Vulnerable Functions:**
```solidity
// setTVSAllocation (Lines 458-477)
function setTVSAllocation(uint256 rewardProjectId, uint256 totalTVSAllocation, uint256 vestingPeriod, address[] calldata kolTVS, uint256[] calldata TVSamounts) external onlyOwner {
    // ... setup code
    for (uint256 i = 0; i < length; i++) {
        address kol = kolTVS[i];
        rewardProject.kolTVSAddresses.push(kol);     // BUG: No duplicate check
        uint256 amount = TVSamounts[i];
        rewardProject.kolTVSRewards[kol] = amount;   // BUG: Overwrites duplicates
        rewardProject.kolTVSIndexOf[kol] = i;        // BUG: Index corruption
        totalAmount += amount;                       // BUG: Double-counts amounts
    }
    // Total validation passes but effective allocation is less
    require(totalTVSAllocation == totalAmount, Amounts_Do_Not_Add_Up_To_Total_Allocation());
}

// setStablecoinAllocation has identical issue (Lines 484-502)
```

**Index Mapping Corruption:**
```solidity
// With duplicates, index mapping becomes inconsistent
kolTVSIndexOf[Alice] = 2  // Points to second occurrence, not first
// This breaks array manipulation logic in _claimRewardTVS
```

## Tool used
Manual Review

## Recommendation

Add duplicate address validation
  