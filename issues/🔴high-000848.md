# [000848] Array Out-of-Bounds Access in Distribution Functions Causes Complete DoS
  
  ## Summary

The `distributeRewardTVS` and `distributeStablecoinAllocation` functions in `AlignerzVesting.sol` contain critical array bounds vulnerabilities that cause transaction reverts when KOLs claim during the claim window, making distribution functions completely unusable under normal protocol conditions.

## Vulnerability Detail

Both distribution functions use the length of an internal array for loop iteration but access elements from a user-provided array, creating inevitable out-of-bounds access when partial claims occur:

**Location**: [`AlignerzVesting.sol:525-535`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535)

```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length; // ❌ BUG: Uses internal array length
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]); // ❌ BUG: Accesses user array with internal length
        unchecked {
            ++i;
        }
    }
}
```

**Root Cause**: When KOLs claim during the window, they are removed from the internal `kolTVSAddresses` array via swap-and-pop mechanism ([`AlignerzVesting.sol:615-621`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L615-L621)), but the function still uses the internal array length to iterate over the user-provided array.

**Claim Removal Mechanism**:
```solidity
function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
    // ... claiming logic ...
    uint256 index = rewardProject.kolTVSIndexOf[kol]; // Line 615
    uint256 arrayLength = rewardProject.kolTVSAddresses.length; // Line 616
    // Swap-and-pop removal (lines 617-621)
    rewardProject.kolTVSAddresses[index] = rewardProject.kolTVSAddresses[arrayLength - 1];
    rewardProject.kolTVSAddresses.pop(); // Internal array shrinks
}
```

**Same vulnerability exists in**: [`AlignerzVesting.sol:540-550`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L550) - `distributeStablecoinAllocation` function

### Attack Mechanism

1. **Project Setup**: Reward project created with 5 KOLs eligible for TVS rewards
2. **Partial Claims**: 2 KOLs claim their rewards during claim window (removed from internal array)
3. **Array State Mismatch**: Internal array now has length 3, but owner provides array with 2 addresses  
4. **Distribution Attempt**: Owner calls `distributeRewardTVS(projectId, [kol3, kol4])`
5. **Out-of-Bounds Access**: Loop tries to access `kol[2]` which doesn't exist
6. **Transaction Revert**: Panic error "array index out of bounds" (0x32)

**Step-by-Step Execution**:
```solidity
// Initial state: kolTVSAddresses = [KOL1, KOL2, KOL3, KOL4, KOL5] (length=5)
// After KOL1 and KOL2 claim: kolTVSAddresses = [KOL5, KOL4, KOL3] (length=3)

distributeRewardTVS(projectId, [KOL3, KOL4]); // User array has length=2
uint256 len = rewardProject.kolTVSAddresses.length; // len = 3
for (uint256 i; i < 3;) { // Loop runs 3 times
    // i=0: _claimRewardTVS(projectId, kol[0]); ✅ Works (KOL3)
    // i=1: _claimRewardTVS(projectId, kol[1]); ✅ Works (KOL4)  
    // i=2: _claimRewardTVS(projectId, kol[2]); ❌ PANIC! Array out of bounds
}
```

## Impact

**High Severity**: Core distribution functionality becomes completely unusable under normal protocol conditions

- **Complete DoS**: Distribution functions fail with panic errors whenever partial claims occur
- **Funds Permanently Locked**: Remaining TVS/stablecoin allocations become inaccessible via these functions
- **Broken Protocol Flow**: Normal user behavior (claiming during window) makes admin functions unusable
- **No Workaround**: Only alternative is using `distributeRemainingRewardTVS()` which doesn't allow selective distribution

**Real-world Impact Scenarios**:
```solidity
// Scenario 1: 5 KOLs, 2 claim early, 3 remain
distributeRewardTVS(projectId, [kol3, kol4, kol5]); // ❌ PANIC: array out of bounds

// Scenario 2: Admin only wants to distribute to 2 specific KOLs  
distributeRewardTVS(projectId, [specificKol1, specificKol2]); // ❌ PANIC: array out of bounds

// Result: Distribution becomes impossible when it's most needed
```

**Economic Impact**:
- KOLs lose access to their allocations through intended distribution mechanism
- Protocol loses ability to handle partial distribution scenarios
- Gas costs wasted on guaranteed-to-fail transactions

## Code Snippet

**Current vulnerable implementation**:
```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length; // Uses internal array length
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]); // Accesses user array - MISMATCH!
        unchecked {
            ++i;
        }
    }
}
```

**Identical pattern in stablecoin distribution**:
```solidity
function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
    // ... same vulnerable pattern ...
    uint256 len = rewardProject.kolStablecoinAddresses.length; // Internal length
    for (uint256 i; i < len;) {
        _claimStablecoinAllocation(rewardProjectId, kol[i]); // User array access
    }
}
```

**Working alternative for comparison**:
```solidity
// This function works because it only uses internal array
function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner {
    for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
        address kol = rewardProject.kolTVSAddresses[i]; // ✅ Consistent: uses internal array
        _claimRewardTVS(rewardProjectId, kol);
        rewardProject.kolTVSAddresses.pop();
    }
}
```

## Recommendation

**Add Length Validation**
  