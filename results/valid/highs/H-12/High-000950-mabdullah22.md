# [000950] Incorrect Token Check in A26ZDividendDistributor
  
  ### Summary

The `getUnclaimedAmounts` function in `A26ZDividendDistributor.sol` contains inverted logic when checking if an NFT's token matches the distributor's token. The function returns 0 (no dividends) when the tokens **match**, and proceeds to calculate dividends when they **don't match**. This is the opposite of the intended behavior, causing users holding the correct TVS tokens to receive zero dividends.

### Root Cause

The root cause is in [`A26ZDividendDistributor.sol`] 


```solidity
/// @notice USD value in 1e18 of all the unclaimed tokens of a TVS
/// @param nftId NFT Id
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // @audit seems inverted logic should be !=
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;  // BUG: Returns 0 when tokens MATCH
    
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    
    for (uint i; i < len;) {
        if (claimedFlows[i]) continue;
        if (claimedSeconds[i] == 0) {
            amount += amounts[i];
            continue;
        }
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
        unchecked {
            ++i;
        }
    }
    unclaimedAmountsIn[nftId] = amount;
}
```

**The Logic Error**:
- Current: `if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;`
- Meaning: "If the NFT's token matches our distributor's token, return 0 dividends"
- **This is backwards!**

**Correct Logic Should Be**:
- `if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;`
- Meaning: "If the NFT's token doesn't match our distributor's token, return 0 dividends"


### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

**Scenario: Legitimate users get 0 dividends**

1. Distributor is deployed for Token A
2. User holds NFT with TVS allocation in Token A (correct token)
3. Owner calls `setUpTheDividends()`
4. `getUnclaimedAmounts(nftId)` is called for user's NFT
5. Check: `address(tokenA) == address(tokenA)` → **true**
6. Function returns 0 (no dividends calculated)
7. User receives 0 dividends despite holding correct token

### Impact

1. **Dividend distribution failure**: Users holding the correct TVS tokens receive 0 dividends

### PoC

_No response_

### Mitigation

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    // FIX: Return 0 if tokens DON'T match (not if they DO match)
    if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
    
    // Rest of the function calculates dividends for matching tokens
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    // ... rest of implementation ...
}
```
  