# [000409] Batch Whitelist Operations Waste Gas by Reverting Entire Transaction on Single Invalid Entry
  
  ### Summary

Four internal whitelist management functions in `WhitelistManager.sol` use `require` statements that cause complete transaction reversion when encountering a single duplicate or invalid entry during batch processing. Instead of skipping invalid entries and continuing with the remaining valid ones, the functions abort the entire operation - wasting all gas spent processing previous entries. For example, adding 200 addresses where only the 200th entry is already whitelisted causes the entire batch to revert, losing gas from processing the first 199 entries. Affected functions:
- [_enableWhitelist](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L55-L59)
- [_disableWhitelist](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L81-L85)
- [_addUserToWhitelist](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L110-L115)
- [_removeUserFromWhitelist](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L140-L145)

The issue impacts their respective batch wrapper functions: `enableWhitelists`, `disableWhitelists`, `addUsersToWhitelist`, `removeUsersFromWhitelist`.


### Root Cause

The internal helper functions use `require()` statements (e.g., `require(!isWhitelisted[user][projectId], "Already whitelisted")`) instead of conditional checks with early returns. When a require condition fails during batch processing, the entire transaction reverts rather than gracefully skipping the invalid entry and continuing with remaining items.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

- **Operator batches 200 user addresses** for whitelisting via `addUsersToWhitelist([user1, user2, ..., user200])`
- Contract processes entries 1-199 successfully (consuming gas for each SSTORE operation)
- Entry 200 (`user200`) is already whitelisted
- `_addUserToWhitelist` hits `require(!isWhitelisted[user][projectId], "Already whitelisted")` and **reverts entire transaction**
- **All gas spent processing the first 199 entries is wasted** - no state changes saved
- Operator must now manually identify `user200`, remove it from the array, and resubmit with fresh gas


### Impact

- **Massive gas waste**: Processing 199/200 entries successfully then reverting on the last entry wastes all gas from the first 199 SSTORE operations (~20k gas each = ~4M gas lost)
- **Exponential retry costs**: Each failed attempt requires full re-execution with filtered list; multiple failures compound gas waste


### PoC

_No response_

### Mitigation

## Recommended code changes
Replace `require` with conditional checks; skip entries that are already in the target state and only emit events for actual state changes. This allows graceful degradation and idempotent batch operations.

### Diff: Make _enableWhitelist skip-on-duplicate
```diff
@@ function _enableWhitelist(uint256 projectId) internal {
-    require(!isWhitelistEnabled[projectId], "Whitelisting is already enabled for this project");
-    isWhitelistEnabled[projectId] = true;
-    emit whitelistEnabled(projectId);
+    if (isWhitelistEnabled[projectId]) {
+        return; // Already enabled, skip silently
+    }
+    isWhitelistEnabled[projectId] = true;
+    emit whitelistEnabled(projectId);
}
```

### Diff: Make _disableWhitelist skip-on-duplicate
```diff
@@ function _disableWhitelist(uint256 projectId) internal {
-    require(isWhitelistEnabled[projectId], "Whitelisting is already disabled for this project");
-    isWhitelistEnabled[projectId] = false;
-    emit whitelistDisabled(projectId);
+    if (!isWhitelistEnabled[projectId]) {
+        return; // Already disabled, skip silently
+    }
+    isWhitelistEnabled[projectId] = false;
+    emit whitelistDisabled(projectId);
}
```

### Diff: Make _addUserToWhitelist skip-on-duplicate
```diff
@@ function _addUserToWhitelist(address user, uint256 projectId) internal {
-    require(user != address(0), "Invalid address");
-    require(!isWhitelisted[user][projectId], "Already whitelisted");
-    isWhitelisted[user][projectId] = true;
-    emit userWhitelisted(projectId, user);
+    if (user == address(0)) {
+        return; // Invalid address, skip silently (or revert if zero addresses should never be in batch)
+    }
+    if (isWhitelisted[user][projectId]) {
+        return; // Already whitelisted, skip silently
+    }
+    isWhitelisted[user][projectId] = true;
+    emit userWhitelisted(projectId, user);
}
```

### Diff: Make _removeUserFromWhitelist skip-on-duplicate
```diff
@@ function _removeUserFromWhitelist(address user, uint256 projectId) internal {
-    require(user != address(0), "Invalid address");
-    require(isWhitelisted[user][projectId], "address is not whitelisted");
-    isWhitelisted[user][projectId] = false;
-    emit userBlacklisted(projectId, user);
+    if (user == address(0)) {
+        return; // Invalid address, skip silently
+    }
+    if (!isWhitelisted[user][projectId]) {
+        return; // Already not whitelisted, skip silently
+    }
+    isWhitelisted[user][projectId] = false;
+    emit userBlacklisted(projectId, user);
}
```

### Optional: Add skip event for transparency
If you want to maintain visibility into skipped entries (for off-chain monitoring), emit a dedicated event:
```diff
+    event WhitelistOperationSkipped(uint256 indexed projectId, address indexed user, string reason);
```
Then emit this event before `return` statements in the modified functions.

  