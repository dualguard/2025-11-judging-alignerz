# [000250] Duplicate Dividend Allocation Leads To Contract Insolvency
  
  ### Summary

The `A26ZDividendDistributor::_setDividends()` function uses the `+=` operator to allocate dividends to users. This causes allocations to accumulate instead of being replaced when the function is called multiple times. If `::setDividends()` or `::setUpTheDividends()` is called more than once, users will be allocated multiples of their intended dividend amounts, making the contract insolvent. The contract will owe more stablecoins than it actually holds, causing later claimants to receive nothing.

### Root Cause

- https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224

```solidity
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) {
                dividendsOf[owner].amount += ((unclaimedAmountsIn[i] * stablecoinAmountToDistribute)
                        / totalUnclaimedAmounts);
            }
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

The `+=` operator adds to the existing allocation instead of setting a new value. Each time the function runs, it adds the same allocation again.

### Internal Pre-conditions

Contract must have NFT holders who are entitled to dividends.

### External Pre-conditions

`A26zDividendDistributor::_setDividends()` is called more than once.

### Attack Path

**Setup:**
- Alice owns NFT-0 (has 1000 unclaimed tokens in vesting contract)
- Bob owns NFT-1 (has 1000 unclaimed tokens in vesting contract)  
- Charlie owns NFT-2 (has 1000 unclaimed tokens in vesting contract)
- Total unclaimed tokens across all NFTs: 3000 tokens

**Steps:**

1. **Owner (protocol admin) deposits 3000 USDC** into the DividendDistributor contract
   - Contract balance: 3000 USDC
   - This is profit-sharing for TVS holders

2. **Owner calls `setUpTheDividends()`** (first time)
   - `_setAmounts()`: Reads contract balance = 3000 USDC, calculates total unclaimed = 3000 tokens
   - `_setDividends()`: Allocates dividends proportionally
     - Alice: 0 + (1000 * 3000) / 3000 = 1000 USDC
     - Bob: 0 + (1000 * 3000) / 3000 = 1000 USDC
     - Charlie: 0 + (1000 * 3000) / 3000 = 1000 USDC
   - Total allocated: 3000 USDC ✓ Solvent

3. **Owner deposits additional 2000 USDC** (next quarter's profits)
   - Contract balance: 5000 USDC total

4. **Owner calls `setUpTheDividends()`** again (thinking it will distribute the new 2000 USDC)
   - `_setAmounts()`: Reads contract balance = 5000 USDC, total unclaimed = 3000 tokens (unchanged)
   - `_setDividends()`: Uses `+=` on existing allocations
     - Alice: 1000 + (1000 * 5000) / 3000 = 1000 + 1666.67 = 2666.67 USDC
     - Bob: 1000 + 1666.67 = 2666.67 USDC
     - Charlie: 1000 + 1666.67 = 2666.67 USDC
   - Total allocated: 8000 USDC ✗ **Insolvent** (has 5000, owes 8000)

5. **NFT holders claim over vesting period**
   - First ~62.5% of claims succeed (5000 / 8000)
   - Last ~37.5% of claims **revert** due to insufficient balance
   - Permanent loss for whoever claims last

### Impact

- Total allocations exceed contract balance
- First claimers get paid, last claimers get nothing

### PoC

_No response_

### Mitigation

_No response_
  