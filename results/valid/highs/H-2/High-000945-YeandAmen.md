# [000945] Uninitialized Array Causes All Merge and Split Operations to Revert
  
  ### Summary

The `calculateFeeAndNewAmountForOneTVS` function declares a return array newAmounts but never initializes it with a length. When the function attempts to assign values to this array, it triggers an array out of bounds panic, causing all merge and split operations to revert immediately.

### Root Cause

The function is called by both mergeTVS and splitTVS operations to calculate fees and adjusted token amounts:
```solidity
File: FeesManager.sol

function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate, 
    uint256[] memory amounts, 
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; //@audit uninitialized array causes panic
    }
}
```
In Solidity, declaring uint256[] memory newAmounts as a return parameter creates an empty array with length 0. The function never calls new uint256[](length) to allocate space. When it attempts newAmounts[i] =... it tries to write to index 0 of a zero length array, triggering panic code 0x32 (array out of bounds access).

> [https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169](url)

### Internal Pre-conditions

none

### External Pre-conditions

none

### Attack Path

none

### Impact

Complete denial of service for core protocol features:
All merge operations fail regardless of user input or fee settings
All split operations fail regardless of user input or fee settings
Users cannot consolidate multiple vesting positions into one NFT
Users cannot divide a vesting position into multiple NFTs
No workaround exists .the panic happens before any state changes

### PoC

```text
 (Walkthrough)
Scenario: Bob wants to merge two NFTs to consolidate his vesting positions.
Step 1: Bob calls mergeTVS with his two NFT IDs
Step 2: The merge function calls calculateFeeAndNewAmountForOneTVS(100, [5000, 1000], 2)
Step 3: Function enters loop with i = 0
Step 4: Attempts to execute newAmounts[0] = 5000 - 50
Step 5: Array newAmounts has length 0, so accessing newAmounts[0] is out of bounds
Step 6: Transaction reverts with panic code 0x32
Result: Bob cannot merge his NFTs. The entire merge feature is non functional.
The same issue affects splitTVS operations, making that feature equally broken
```

### Mitigation

Initialize the array before use.
  