# [000841] Dividend Distribution Manipulation Through Stale Cache Exploitation
  
  
## Summary

The dividend distribution system caches unclaimed amounts that become stale when users modify their allocations, allowing attackers to exploit timing windows where cached values no longer reflect actual token states for unfair dividend distribution.

## Vulnerability Detail

The `getUnclaimedAmounts` function caches values that become outdated when users perform TVS operations, creating exploitable inconsistencies in dividend calculations:

**Location**: [`A26ZDividendDistributor.sol:140-161`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161)

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // Calculates based on CURRENT allocation state
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    
    // ... calculation logic ...
    
    unclaimedAmountsIn[nftId] = amount; // 🚨 CACHED VALUE that becomes stale
}
```

**Critical Issue**: Dividend distribution relies on potentially stale cached values:

**Location**: [`A26ZDividendDistributor.sol:214-224`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224)

```solidity
function _setDividends() internal {
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) dividendsOf[owner].amount += 
            (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        //  ^^^^^^^^^^^^^^^^^^^^ May use STALE cached values
        unchecked { ++i; }
    }
}
```

### Root Cause Analysis

**Cache Invalidation Problem**: The cache is not automatically invalidated when underlying allocation state changes through:
- [`claimTokens()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L941) - modifies `claimedSeconds` and `claimedFlows`
- [`splitTVS()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1055) - modifies `amounts` and vesting arrays
- [`mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) - modifies allocation structure

**Timing Attack Window**: When admins use separate [`setAmounts()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L117) and [`setDividends()`](https://github.com/dualguard/2025-11-alignerz/tree/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L122) calls:

```solidity
// Phase 1: setAmounts() caches current state
function _setAmounts() internal {
    totalUnclaimedAmounts = getTotalUnclaimedAmounts(); // Calls getUnclaimedAmounts(i) for all NFTs
}

// Phase 2: User operations make cache stale
// User calls claimTokens() / splitTVS() / mergeTVS()

// Phase 3: setDividends() uses stale cache
function _setDividends() internal {
    // Uses potentially outdated unclaimedAmountsIn[i] values
}
```

### Attack Mechanism

**Pre-Claim Dividend Amplification**:
1. **Cache Setup**: Attacker calls `getUnclaimedAmounts(nftId)` → caches current unclaimed amount (e.g., 1000 tokens)
2. **Token Claiming**: Attacker calls `claimTokens()` → reduces actual allocation to 500 tokens
3. **Dividend Distribution**: Admin calls `setDividends()` → uses stale cache value (1000 tokens)
4. **Double Benefit**: Attacker receives dividends for 1000 tokens + already claimed 500 tokens

**Post-Split Cache Manipulation**:
1. **Initial Cache**: Cache shows 1000 tokens unclaimed
2. **TVS Split**: User calls `splitTVS()` → actual allocation becomes 500 tokens (50% split)  
3. **Stale Usage**: `setDividends()` uses stale 1000 token cache value
4. **Excess Dividends**: User receives 2x intended dividend share

**Cross-User Exploitation**:
1. **Natural Staleness**: Users perform normal TVS operations making their caches stale
2. **Selective Updates**: Attacker calls `getUnclaimedAmounts()` for specific NFTs to refresh select caches
3. **Mixed State**: Some caches fresh, others stale when dividends calculated
4. **Distribution Errors**: Mathematical inconsistency in dividend allocation

## Impact

**High Severity**: Economic manipulation through timing-based cache exploitation

- **Dividend Theft**: Users receive dividends for tokens they already claimed or split
- **Unfair Distribution**: Cache staleness creates unequal dividend treatment
- **Economic Arbitrage**: Attackers time operations to maximize dividend capture
- **Mathematical Inconsistency**: Sum of cached values ≠ actual total unclaimed

### Exploitation Scenarios

**Scenario 1: Pre-Distribution Token Claiming**
```solidity
// 1. Cache: 1000 tokens unclaimed
// 2. claimTokens(): actual becomes 500 tokens  
// 3. setDividends(): uses cached 1000 tokens
// Result: Dividends paid for 1000 + user already claimed 500 = double benefit
```

**Scenario 2: Strategic TVS Operations**  
```solidity
// 1. Cache: 2000 tokens in single TVS
// 2. splitTVS(): creates two 1000-token TVS
// 3. setDividends(): uses stale 2000 cache for one NFT
// Result: Receives dividends for 2000 + 1000 = 3000 instead of 2000
```

**Scenario 3: Cross-User Cache Pollution**
```solidity
// 1. Multiple users have stale caches after operations  
// 2. Attacker refreshes select caches via getUnclaimedAmounts()
// 3. Dividend calculation uses mixed fresh/stale values
// Result: Inconsistent dividend distribution across users
```

## Code Snippet

**Cache Creation Without Invalidation**:
```solidity
// A26ZDividendDistributor.sol:160 - Cache set without invalidation mechanism  
unclaimedAmountsIn[nftId] = amount;
```

**TVS Operations Don't Invalidate Cache**:
```solidity
// AlignerzVesting.sol:960-964 - claimTokens modifies state
allocation.claimedSeconds[i] += claimableSeconds;
if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
    allocation.claimedFlows[i] = true;
}
// 🚨 No cache invalidation in dividend distributor
```

**Split Operations Create Stale Cache**:
```solidity  
// AlignerzVesting.sol - splitTVS modifies amounts array
// 🚨 Original NFT's cached value becomes invalid but not updated
```

**Stale Cache Usage in Distribution**:
```solidity
// A26ZDividendDistributor.sol:218 - May use stale values
dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
```

## Tool used

Manual Review

## Recommendation

**1. Implement Cache Invalidation System**:

```solidity
// Add cache invalidation when allocations change
function _invalidateCache(uint256 nftId) internal {
    delete unclaimedAmountsIn[nftId];
    emit CacheInvalidated(nftId);
}

// Call from TVS operations
function claimTokens(uint256 projectId, uint256 nftId) external {
    // ... existing claim logic ...
    
    // Invalidate dividend cache after claim
    if (address(dividendDistributor) != address(0)) {
        dividendDistributor.invalidateCache(nftId);
    }
}
```

**2. Use Fresh Calculations Instead of Cache**:

```solidity
function _setDividends() internal {
    for (uint i; i < len;) {
        (address owner, bool isOwned) = safeOwnerOf(i);
        if (isOwned) {
            // Calculate fresh instead of using potentially stale cache
            uint256 freshAmount = getUnclaimedAmounts(i);
            dividendsOf[owner].amount += 
                (freshAmount * stablecoinAmountToDistribute / totalUnclaimedAmounts);
        }
        unchecked { ++i; }
    }
}
```

**3. Atomic Dividend Distribution Only**:

```solidity
// Force atomic operations to prevent stale cache usage
function setUpTheDividends() external onlyOwner {
    _setAmounts();    // Fresh calculation
    _setDividends();  // Immediate usage - no staleness window
}

// Deprecate separate functions to prevent timing attacks
// function setAmounts() public onlyOwner { revert("Use setUpTheDividends()"); }
// function setDividends() external onlyOwner { revert("Use setUpTheDividends()"); }
```

**4. Add Cache Timestamp Validation**:

```solidity
mapping(uint256 => uint256) public cacheTimestamp;
uint256 public constant CACHE_VALIDITY_PERIOD = 1 hours;

function getUnclaimedAmounts(uint256 nftId) public view returns (uint256 amount) {
    // Return cached value only if recent
    if (block.timestamp - cacheTimestamp[nftId] < CACHE_VALIDITY_PERIOD) {
        return unclaimedAmountsIn[nftId];
    }
    
    // Calculate fresh if cache is stale
    return _calculateFreshUnclaimedAmount(nftId);
}

function _updateCache(uint256 nftId) internal {
    unclaimedAmountsIn[nftId] = _calculateFreshUnclaimedAmount(nftId);
    cacheTimestamp[nftId] = block.timestamp;
}
```

**5. Add Cross-Contract Communication**:

```solidity
// Interface for cache invalidation
interface IDividendDistributor {
    function invalidateCache(uint256 nftId) external;
}

// Call from AlignerzVesting operations
modifier invalidatesDividendCache(uint256 nftId) {
    _;
    if (address(dividendDistributor) != address(0)) {
        try dividendDistributor.invalidateCache(nftId) {} catch {}
    }
}

function claimTokens(uint256 projectId, uint256 nftId) 
    external invalidatesDividendCache(nftId) {
    // ... existing logic ...
}
```

**Priority**: **High** - Cache staleness enables systematic manipulation of dividend distributions through timing attacks exploiting the gap between cache creation and usage.
  