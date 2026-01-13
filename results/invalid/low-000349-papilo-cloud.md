# [000349] [L-1] Batch whitelist removal in `WhitelistManager.sol::removeUsersFromWhitelist#120` performs redundant validation causing full revert and wasted gas
  
  ### Summary

A redundant validation pattern inside `WhitelistManager.sol::removeUsersFromWhitelist#120` causes the transaction to reverts and therefore causing gas wastage for project administrators, as the owner will perform expensive `require` checks inside a `for` loop and the entire operation reverts when any user in the batch is invalid or not whitelisted.

### Root Cause

In [`WhitelistManager.sol::removeUsersFromWhitelist#120`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L123), the function calls the internal [`WhitelistManager.sol::_removeUserFromWhitelist`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L140) function inside a loop, where `WhitelistManager.sol::_removeUserFromWhitelist` performs strict `require` validation:

```solidity
    function _removeUserFromWhitelist(address user, uint256 projectId) internal {
        require(user != address(0), "Invalid address");
        require(isWhitelisted[user][projectId], "address is not whitelisted");
        // ...
    }
```
A single invalid user causes the entire batch removal to revert.

### Internal Pre-conditions

1. If he owner calls `WhitelistManager.sol::removeUsersFromWhitelist` with a batch of user addresses.
2. And at least one user in the array is either: `address(0)`, or not whitelisted for the given `projectId`.
3. Prior users in the list passed the validation and were checked successfully.

### External Pre-conditions

_No response_

### Attack Path

1. Let say the owner wants an to remove 50 users (array of users).
2. 40 users are valid and whitelisted, but user #41 is not whitelisted.
3. The function loops: for users 1–40, `_WhitelistManager.sol::removeUserFromWhitelist` passes validation.
4. At user #41, validation fails (`require(isWhitelisted[user][projectId])`).
5. The transaction reverts, rolling back all previous successful checks.
6. But the owner has to pays for gas of 40 expensive checks and there is no state change in return.

### Impact

The project owner suffers a direct financial loss equal to the gas wasted on all successful iterations before the revert.
No attacker gains funds here.

### PoC

_No response_

### Mitigation

```diff
    function removeUsersFromWhitelist(address[] calldata users, uint256 projectId) external onlyOwner {
        uint256 length = users.length;
       for (uint256 i = 0; i < length;) {
+            address user = users[i];

+           if (user == address(0)) {
+                emit RemovalFailed(user, "Invalid address");
+                unchecked { ++i; }
+                continue;
+           }

+            if (!isWhitelisted[user][projectId]) {
+                emit RemovalFailed(user, "Not whitelisted");
+                unchecked { ++i; }
+                continue;
+            }
               
+            isWhitelisted[user][projectId] = false;
+            emit userBlacklisted(projectId, user);

-           _removeUserFromWhitelist(users[i], projectId);
            unchecked {
                ++i;
            }
        }
    }

//We can also allow `_removeUserFromWhitelist` to silently skip invalid users.
    function _removeUserFromWhitelist(address user, uint256 projectId) internal {
+        if (user == address(0)) return;
+        if (!isWhitelisted[user][projectId]) return;
-         require(user != address(0), "Invalid address");
-         require(isWhitelisted[user][projectId], "address is not whitelisted");
    
        isWhitelisted[user][projectId] = false;
        emit userBlacklisted(projectId, user);
    }
```

Either one of the codes will do
  