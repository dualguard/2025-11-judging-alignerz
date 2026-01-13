# [000384] Users won't be able to split or merge TVS because of Infinite Loop DoS in FeesManager:calculateFeeAndNewAmountForOneTVS()
  
  ### **Summary**
The `calculateFeeAndNewAmountForOneTVS` function contains an infinite loop because the loop counter `i` is never incremented, causing complete Denial of Service (DoS) for all TVS operations.

### **Impact**
- **Complete Protocol Failure:** All split/merge operations become unusable
- **Gas Exhaustion:** Function calls consume all available gas and revert
- **Permanent DoS:** No user can perform TVS operations until fixed
- **Protocol Halt:** Core vesting functionality completely broken

### **Attack Path**
1. User calls any function that uses `calculateFeeAndNewAmountForOneTVS` 
2. Function enters infinite loop
3. Transaction consumes all gas and reverts
4. **No attacker needed** - legitimate users trigger the bug

### **Root Cause**
Not incrementing `i` in FeesManager:calculateFeeAndNewAmountForOneTVS()
[VULNERABLE CODE (Lines 169-174)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174)
```solidity
for (uint256 i; i < length;) {
    feeAmount += calculateFeeAmount(feeRate, amounts[i]);
    newAmounts[i] = amounts[i] - feeAmount;
    // ❌ Missing: i++ or ++i or unchecked { ++i; }
}
```

### **Proof of Concept (PoC)**

**Scenario:** Any call with `splitTVS()` or `mergeTVS()`
```solidity
uint256[] memory amounts = [1000];
uint256 feeRate = 100;
uint256 length = 1;

// What happens:
// i = 0 (initial)
// Loop: i < 1? YES → Execute loop body
// i is still 0 (never incremented)  
// Loop: i < 1? YES → Execute loop body again
// ... infinite loop continues until gas runs out
```

**Test Code:**
```solidity
function testInfiniteLoopDoS() public {
    uint256[] memory amounts = new uint256[](1);
    amounts[0] = 1000;
    
    // This call will run out of gas due to infinite loop
    vm.expectRevert("out of gas");
    feesManager.calculateFeeAndNewAmountForOneTVS(100, amounts, 1);
}
```

### **Fix**
**Key Change:** Add the missing loop increment `unchecked { ++i; }` to prevent infinite loop.
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length)
    public pure returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    newAmounts = new uint256[](length);
    
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
        
        unchecked { ++i; }  // ✅ ADD MISSING INCREMENT
    }
}
```


  