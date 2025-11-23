# [000382] Inconsistent Deadline Handling Prevents Claims at Exact Deadline
  
  ### **Summary**
In `AlignerzVesting:claimRewardTVS()` , `AlignerzVesting:claimStablecoinAllocation()`, `AlignerzVesting:claimRefund()`, `AlignerzVesting:claimNFT()`.

The contract uses inconsistent logic for deadline checks, preventing users from claiming at the exact `claimDeadline` timestamp while throwing misleading error messages. While claiming at exactly deadline, the error msg says `Deadline_Has_Passed()` , but in reality it has not passed.

---

### **Root Cause**

The deadline validation uses strict inequality (`<`) instead of inclusive inequality (`<=`), creating a gap where users cannot claim at the exact deadline timestamp. Additionally, the error message `Deadline_Has_Passed()` is misleading when the deadline has only been reached, not passed.

**Problematic Code:**


[claimRewardTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L508)
```solidity
require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed());
```

[claimStablecoinAllocation()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L517)
```solidity
require(block.timestamp < rewardProject.claimDeadline, Deadline_Has_Passed());
```
[claimRefund()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L837)
```solidity
require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());
```
[claimNFT()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L865)
```solidity
require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());
```

---

### **Attack Path**

1. **Setup:** User has valid allocation with `claimDeadline = 1000`
2. **At timestamp 999:** User can claim successfully (`999 < 1000` = true)
3. **At timestamp 1000:** User cannot claim (`1000 < 1000` = false)
   - Transaction reverts with `Deadline_Has_Passed()`
   - But deadline hasn't "passed" - it's been reached exactly
---

### **Impact**

**User Impact:**

- **Funds temporarily locked** during exact deadline timestamp
- **Mismatch** between documentation and actual deadline status
---

### **Mitigation**

**Recommended Fix:**

Update deadline checks to be inclusive of the deadline timestamp:

```solidity
 Allow claims until and including deadline
require(block.timestamp <= rewardProject.claimDeadline, Deadline_Has_Passed());
require(biddingProject.claimDeadline >= block.timestamp, Deadline_Has_Passed())
```
  