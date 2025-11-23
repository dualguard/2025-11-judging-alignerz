# [000229] Infinite Loop in `getUnclaimedAmounts` Leads to Permanent Denial of Service for Dividend Distribution
  
  ### Summary

The `getUnclaimedAmounts` function, which is critical for calculating user dividends, contains a flaw that leads to an infinite loop. The function iterates through the vesting flows of an NFT but fails to increment its loop counter (i) before a `continue` statement is reached. If any vesting flow has either been fully claimed (`claimedFlows[i] == true`) or has not been claimed at all (`claimedSeconds[i] == 0`), the loop will get stuck, never terminating. This causes any transaction that calls this function to run out of gas and revert. Since this function is a core part of the dividend setup process, this bug creates a permanent Denial of Service (DoS), making it impossible for the owner to ever set up or distribute any dividends.

### Root Cause

https://github.com/dualguard/2025-11-alignerz-Ayomiposi233/blob/7b05b7b1bbb71e3e6957270e83365a936945ea5d/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161

### Internal Pre-conditions





### External Pre-conditions

_No response_

### Attack Path

..

### Impact

..

### PoC

_No response_

### Mitigation

_No response_
  