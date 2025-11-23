# [000562] Misleading Event Name Emits `userBlacklisted` When Removing Users From Whitelist Causing Semantic Confusion for Off-Chain Systems
  
  
&nbsp;

## Summary

The use of the `userBlacklisted` event when removing users from the whitelist will cause incorrect state tracking and user status misrepresentation for off-chain monitoring systems and UIs as these systems will interpret whitelist removal as active blacklisting, leading to false blocking status and incorrect user access decisions.

&nbsp;

## Root Cause

In [`WhitelistManager.sol:144`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L144), the `_removeUserFromWhitelist()` function emits the `userBlacklisted` event when removing a user from the whitelist. The event declaration at line 30 is named `userBlacklisted` with a comment stating "Emitted when a user is blacklisted for a project", but it is emitted during whitelist removal (line 144), not during an active blacklisting action.

```js
    function _removeUserFromWhitelist(...) internal {
        // ...
        isWhitelisted[user][projectId] = false;
        emit userBlacklisted(projectId, user);
    }

    /// @notice Emitted when a user is blacklisted for a project
    /// @param projectId Id of the project
    /// @param user address to whitelist
    event userBlacklisted(uint256 projectId, address user);
```

The choice to use the same event name for both removing from whitelist and blacklisting is a mistake because these are semantically different operations:
- Removing from whitelist: returns the user to the default state (not whitelisted, but not actively blocked)
- Blacklisting: actively blocks or prevents access, which is a stronger restriction

This semantic confusion causes off-chain systems to misinterpret the user's actual status.

&nbsp;

## Internal Pre-conditions

1. The protocol owner needs to call `removeUserFromWhitelist()` or `removeUsersFromWhitelist()` to remove one or more users from the whitelist
2. Off-chain monitoring systems, UIs, or integration systems need to be listening to and processing the `userBlacklisted` event

&nbsp;

## External Pre-conditions

None required. The issue manifests whenever a user is removed from the whitelist and off-chain systems process the event.

&nbsp;

## Attack Path

1. Protocol owner calls `removeUserFromWhitelist(user, projectId)` to remove a user from the whitelist for a specific project
2. The function `_removeUserFromWhitelist()` sets `isWhitelisted[user][projectId] = false` (line 143)
3. The function emits `userBlacklisted(projectId, user)` event (line 144)
4. Off-chain monitoring systems, UIs, or integration services detect the `userBlacklisted` event
5. These systems interpret the event as an active blacklisting action rather than a simple whitelist removal
6. Systems may incorrectly mark the user as "blacklisted" or "blocked" in their databases or UIs
7. UIs may display incorrect user status, showing the user as actively blocked when they are simply not whitelisted
8. Integration systems may incorrectly prevent the user from accessing services or features based on the misleading event name
9. The user's actual state (not whitelisted but not actively blocked) is misrepresented, causing confusion and potential access issues

&nbsp;

## Impact

Off-chain monitoring systems and UIs cannot accurately track user status. These systems incorrectly interpret whitelist removal as active blacklisting, leading to false blocking status in databases, incorrect user status displays in UIs, and potential incorrect access control decisions in integration systems. This causes user confusion and may prevent legitimate users from accessing services they should be able to use.

&nbsp;

## Proof of Concept
N/A


&nbsp;

## Mitigation

Rename the event to accurately reflect the action being performed:
1. Change the event declaration from `userBlacklisted` to `userRemovedFromWhitelist`:
2. Update the emit statement in `_removeUserFromWhitelist()`:
3. If actual blacklisting functionality is needed in the future, create a separate `userBlacklisted` event and implement a distinct blacklisting mechanism with its own state variable and logic.


  