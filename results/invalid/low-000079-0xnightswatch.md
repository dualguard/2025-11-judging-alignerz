# [000079] Misleading userBlacklisted event emission creates confusion about whitelist removal
  
  ### Summary

The emission of `userBlacklisted` event in `_removeUserFromWhitelist` at line 144 will cause confusion for off-chain systems and users as the owner will remove users from the whitelist, making it appear as if users are being actively blacklisted when they are simply having their whitelist status revoked.

### Root Cause

In [WhitelistManager.sol:144](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L144), the function emits `userBlacklisted(projectId, user`) when removing a user from the whitelist, but the actual operation only sets `isWhitelisted[user][projectId] = false`.  There is no separate blacklist mechanism or `isBlacklisted` mapping in the contract. 

### Internal Pre-conditions

1. Owner needs to call `removeUserFromWhitelist()` or `removeUsersFromWhitelist()` to remove a user from the whitelist for a specific project

### External Pre-conditions

None

### Attack Path

1. Owner calls `removeUserFromWhitelist(user, projectId)` to revoke a user's whitelist status 
2. The internal function`_removeUserFromWhitelist()` sets`isWhitelisted[user][projectId] = false` 
3. The function emits `userBlacklisted(projectId, user)` event
4. Off-chain systems and users interpret this as the user being actively blacklisted, when in reality they are simply no longer whitelisted

### Impact

Off-chain systems monitoring events cannot distinguish between users who are removed from the whitelist (neutral status) versus users who are actively blacklisted (negative status). This creates semantic confusion and may lead to incorrect UI displays, analytics, or compliance reporting that treats whitelist removal as a punitive action.

### PoC

N/A

### Mitigation

Option 1: Rename the event to accurately reflect the operation:

```diff
event userRemovedFromWhitelist(uint256 projectId, address user);  
  
function _removeUserFromWhitelist(address user, uint256 projectId) internal {  
    require(user != address(0), "Invalid address");  
    require(isWhitelisted[user][projectId], "address is not whitelisted");  
    isWhitelisted[user][projectId] = false;  
    emit userRemovedFromWhitelist(projectId, user);  
}
```

Option 2: If the protocol intends to have actual blacklisting functionality, implement a separate `isBlacklisted` mapping and corresponding logic with proper access controls.
  