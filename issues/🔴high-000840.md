# [000840] Dividend Distribution Manipulation Through Public Cache Modification
  
  ## Summary

The `getUnclaimedAmounts` function is public and directly modifies the `unclaimedAmountsIn` mapping used for dividend calculations, allowing attackers to manipulate dividend shares by directly calling this function without access control.

## Vulnerability Detail

The `getUnclaimedAmounts` function lacks access control while modifying critical state used in dividend distribution calculations:

**Location**: [`A26ZDividendDistributor.sol:140-161`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161)

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = amounts.length;
    
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked { ++i; }
    }
    unclaimedAmountsIn[nftId] = amount; // 🚨 PUBLIC writes to critical dividend mapping
}
```

**Critical Issue**: This mapping is used in dividend distribution calculations:

**Location**: [`A26ZDividendDistributor.sol:214-224`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224)

```solidity
function _setDividends() internal {
    uint256 len = nft.getTotalMinted();
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += 
            (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        //  ^^^^^^^^^^^^^^^^^^^^ Manipulable through public function calls
        unchecked { ++i; }
    }
}
```

### Root Cause Analysis

**Design Flaw**: The function serves dual purposes without proper access control:
1. **Query Function**: Should calculate and return unclaimed amounts
2. **State Modifier**: Actually writes to critical dividend calculation mapping

**Attack Window**: If admins use separate [`setAmounts()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L117-L119) and [`setDividends()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L122-L124) calls instead of atomic [`setUpTheDividends()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L111-L114), attackers can manipulate cache between calls.

### Attack Mechanism

1. **Setup Phase**: Admin calls `setAmounts()` establishing baseline cache values
2. **Manipulation Phase**: Attacker calls `getUnclaimedAmounts(targetNftId)` to update specific cache entries
3. **Exploitation Phase**: Admin calls `setDividends()` using manipulated cache values
4. **Impact**: Dividend distribution uses inconsistent cache state

**Example Attack**:
```solidity
// After setAmounts(): [1000, 2000, 1500] cached, total = 4500
// Attacker calls getUnclaimedAmounts(1) after user claims tokens
// Cache becomes: [800, 2000, 1500], but total remains 4500
// setDividends() now uses: sum(individual)=4300 ≠ total=4500
```

## Impact

**High Severity**: Economic manipulation affecting dividend distribution integrity

- **Dividend Share Manipulation**: Attackers can inflate/deflate specific cache entries
- **Mathematical Inconsistency**: Creates invalid state where sum of parts ≠ whole
- **Economic Unfairness**: Legitimate users receive incorrect dividend allocations  
- **Protocol Trust**: Undermines dividend distribution mathematical reliability

### Financial Impact Scenarios

**Scenario 1: Direct Cache Manipulation**
- Attacker updates cache for higher unclaimed amounts before dividend calculation
- Receives larger dividend share than entitled

**Scenario 2: Cross-User Impact**  
- Attacker manipulates multiple users' cache entries
- Creates systematic distribution errors affecting entire user base

**Scenario 3: Timing-Based Exploitation**
- Attacker monitors for admin `setAmounts()` transactions
- Front-runs `setDividends()` with cache manipulation
- Exploits vulnerability window in admin workflow

## Code Snippet

**Vulnerable Public Access**:
```solidity
// A26ZDividendDistributor.sol:140 - NO ACCESS CONTROL
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // ... calculation logic ...
    unclaimedAmountsIn[nftId] = amount; // Any user can modify any NFT's cache
}
```

**Dependency in Critical Function**:
```solidity
// A26ZDividendDistributor.sol:218 - Uses manipulable values
dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
```

**Admin Functions Creating Vulnerability Window**:
```solidity
// Separate calls enable manipulation window
function setAmounts() public onlyOwner { _setAmounts(); }
function setDividends() external onlyOwner { _setDividends(); }

// vs Atomic operation (safe)
function setUpTheDividends() external onlyOwner {
    _setAmounts();
    _setDividends();
}
```
  