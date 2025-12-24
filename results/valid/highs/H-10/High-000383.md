# [000383] Split/Merge Operations Send Excessive Fees to Treasury, Creating Protocol Deficit
  
  ## **Summary**

The vesting contract charges split/merge fees on the **full allocation amount** instead of calculating fees only on the **remaining unvested portion**. When users have already claimed vested tokens, this causes the protocol to send **excess fees to treasury**, creating a token shortfall that leads to insolvency.

---

### **Root Cause**

Split/merge operations calculate fees on the **full allocation amount** instead of only the **remaining unvested portion**. 

When `splitTVS()` or `mergeTVS()` is called, fee calculation uses the unreduced `allocation.amounts[]`, charging fees on already-claimed tokens and sending **excess fees to treasury**:

```solidity
calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);  // ❌ Uses full 1000 instead of remaining 500
```

The excess fee comes from the protocol's token pool, causing the total distributed (user claims + treasury fees) to exceed the allocated amount, creating a **shortfall that prevents future claims**.

---

### **Attack Path**

1. **Setup:** User receives 1,000 tokens vesting over 864,000 seconds (10 days), split fee = 1%

   - `allocation.amounts[0]` = 1,000
   - `allocation.claimedSeconds[0]` = 0

2. **At 432,000s (Day 5):** User claims vested tokens

   - Calculation: (1,000 × 432,000) / 864,000 = 500 tokens
   - User receives 500 tokens
   - `allocation.claimedSeconds[0]` = 432,000
   - **Bug:** `allocation.amounts[0]` = 1,000 (unchanged)

3. **At 432,000s:** User splits TVS in 50-50 ratio

   - Fee calculated on: 1,000 tokens
   - Fee charged: 1,000 × 1% = 10 tokens
   - `allocation.amounts[0]` = 990
   - **Bug:** Fee charged on 500 tokens user already withdrew

4. **At 864,000s (Day 10):** User claims 
   - Each token amount = 495 tokens (990 splits in 2 part of 50-50 ratio)
   - Claimable per NFT: (495 × 432,000) / 864,000 = 247.5 tokens
   - Total claimed: 247.5 × 2 = 495 tokens
   - User receives 495 tokens

5. **Result:**
   - User received: 500 + 495 = **995 tokens** (User doesn't lose money directly)
   - Treasury received: 10 tokens (should be 5 tokens)
   - **Total distributed: 1,005 tokens from 1,000 token allocation** ❌
   - **Protocol overpaid by 5 tokens**
   - **This causes insolvency - contract will be short of tokens**

---

### **Impact**

**Protocol Impact:**

1. **Insolvency:** Contract distributes more tokens than allocated, causing shortfall
   - Per affected user: Overpays by `(Already Claimed Amount) × (Fee Rate)`
   - Examples with 1% fee:
     - Claimed 50% before split → Protocol short 0.5% per user
     - Claimed 80% before split → Protocol short 0.8% per user
     - Multiple affected users compound the problem
2. **Broken Accounting:** Total distributions exceed total allocations

   - Example: 1,005 tokens distributed from 1,000 allocation
   - Creates systematic shortfall across all projects

3. **Failed Claims:** Later users cannot claim due to insufficient contract balance

   - Contract will revert when attempting to transfer tokens
   - Users with valid allocations unable to withdraw their vested tokens

4. **Excess Treasury Fees:** Treasury receives more fees than expected
   - Fee charged on tokens already withdrawn by users

**User Impact:**

- **Individual users don't lose funds directly** - they receive expected amounts based on vesting
- **BUT users cannot claim** when contract runs out of tokens due to insolvency
- **Later claimants are blocked** - their transactions revert despite valid allocations
- Creates uncertainty about which users will be able to claim

---

### **Mitigation**

- Use `(vestingPeriod- claimedSecond) * amount / vestingPeriod` to get unclaimed amount and charge fee on this only.
  