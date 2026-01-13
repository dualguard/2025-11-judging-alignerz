# [000047] Misleading Event Name `userBlacklisted`
  
  


#### summary

The `userBlacklisted` event is emitted when a user is removed from a project’s whitelist. While the event name suggests that the user is explicitly prohibited from interacting with the protocol (i.e., “blacklisted”), in reality, the contract only removes the user from the whitelist. 

This creates a misleading signal for external observers, auditors, and frontend interfaces: anyone monitoring this event might assume the user can no longer participate

```solidity
    /// @notice Removes a user from the whitelist (internal logic)
    /// @param user address of the user to remove from the whitelist
    /// @param projectId ID of the project
    function _removeUserFromWhitelist(address user, uint256 projectId) internal {
        require(user != address(0), "Invalid address");
        require(isWhitelisted[user][projectId], "address is not whitelisted");
        isWhitelisted[user][projectId] = false;
        emit userBlacklisted(projectId, user);                <<<@
    }
```
https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L144
#### Impact

- **Confusion:** Developers, auditors, or front-end applications  misinterpret the user’s status.
    
- **Audit Risk:** Misleading event naming can cause misunderstanding during security reviews or compliance checks.
    

#### Recommendation

1. **Rename the event** to clearly reflect the action performed:
    
    ```solidity
    event userRemovedFromWhitelist(uint256 projectId, address user);
    ```
    
  