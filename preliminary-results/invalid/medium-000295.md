# [000295] Vesting period validation bypass
  
  **Summary:** The vesting period validation logic contains a  flaw that allows users to bypass the intended vesting period divisor requirement by setting vesting period to 1, which may enable premature token withdrawal.

**Root Cause** The fundamental root cause is a flawed conditional statement that creates an unintended exception in the validation logic:

In `placeBid` function, the vesting period validation contains a logical error: `require (vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, ...);` Logical `OR` operator misuse the `||` operator meaning if EITHER condition is true, the requirement passes. The edge case exploitation: `vestingPeriod < 2` means periods of 0 or 1 automatically pass validation.

The assumption violation is that the code assumes all vesting periods should follow the divisor rule, but creates an exception without proper justification.

**Vulnerability Details:** :In `placeBid` and function, the vesting period validation contains a logical error:
```javascript
// In placeBid
require (vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, ...);
 
```
The condition `vestingPeriod < 2` allows vesting period of 1 to bypass the divisor check entirely. This creates an unintended shortcut in the vesting mechanism.

**Impact:**
1. Users can set vesting period to 1 to avoid intended long-term locking
2. Defeats the purpose of vesting schedules designed for token price stability
3. May allow rapid token dumping after distribution
4. Undermines project tokenomics and investor confidence

**Tool Used:** Manual review

**Affected Line of Code:** https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L724
**POC:**

**Recommended Mitigation** The function can be written in this manner:
```javascript
function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
    // ... other checks ...
    
    // Fix: Remove the bypass for vestingPeriod = 1
    require(vestingPeriod % vestingPeriodDivisor == 0, "Invalid vesting period");
    require(vestingPeriod >= minimumVestingPeriod, "Vesting period too short");
    
    // ... rest of function ...
}
```

  