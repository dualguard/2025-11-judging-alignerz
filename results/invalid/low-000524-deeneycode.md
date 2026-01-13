# [000524] Event Naming Semantics error when removeUserFromWhitelist is called.
  
  ### Summary

You emit userBlacklisted when `removeUserFromWhitelist` is called.

### Root Cause

https://github.com/dualguard/2025-11-alignerz-deeneycode/blob/ebed8b27817fac595438b4150ffb591102369952/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L140
```javascript
    function _removeUserFromWhitelist(address user, uint256 projectId) internal {
        require(user != address(0), "Invalid address");
        require(isWhitelisted[user][projectId], "address is not whitelisted");
        isWhitelisted[user][projectId] = false;
@>>        emit userBlacklisted(projectId, user);
    }
```


### Internal Pre-conditions

No response

### External Pre-conditions

No response

### Attack Path

No response

### Impact

No response

### PoC

No response

### Mitigation

Rename the event to userRemovedFromWhitelist to be accurate to the logic.
  