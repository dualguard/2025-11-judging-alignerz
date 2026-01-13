# [000940] Batch Whitelist Operations Fail if a single address fails validation
  
  ### Summary

The `WhitelistManager` has three batch operations (`addUsersToWhitelist`, `removeUsersFromWhitelist`, and `disableWhitelists`), they all revert entirely if any single address/project in the batch fails validation. Project owners must manually verify status before batching or waste gas on failed transactions as the functions has no check.

### Root Cause

All three batch functions follow the same pattern:

**1. `addUsersToWhitelist`** 
- Calls `_addUserToWhitelist` which requires `!isWhitelisted[user][projectId]` 
- If any user is already whitelisted, entire batch reverts
[_addUsersToWhitelist](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L110C2-L115C6)

**2. `removeUsersFromWhitelist`** 
- Calls `_removeUserFromWhitelist` which requires `isWhitelisted[user][projectId]` 
- If any user is not whitelisted, entire batch reverts
[_removeUserFromWhitelist](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L139C1-L145C6)

**3. `disableWhitelists`** 
- Calls `_disableWhitelist` which requires `isWhitelistEnabled[projectId]` 
- If any project already has whitelisting disabled, entire batch reverts
[_disableWhitelist](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/whitelistManager/WhitelistManager.sol#L81-L85)
They all lack of conditional logic to skip already-processed addresses/projects instead of reverting.



### Impact

 It causes the transaction to revert each time an already whitelisted address is added to the adresses to be whitelisted or  reverrt if any project that has whitelisting disabled is called in disableWhitelist and  if a non whitelisted address is listed in removeUserFromWhitelist , no way to check for a whitelisted address and skip it and move on to the addresses that are not whitelisted 

### PoC

```solidity
function test_AllBatchOperationsFail() public {  
    // Setup  
    address[] memory users = new address[](3);  
    users[0] = address(0x1);  
    users[1] = address(0x2);  
    users[2] = address(0x3);  
      
    uint256[] memory projects = new uint256[](3);  
    projects[0] = 1;  
    projects[1] = 2;  
    projects[2] = 3;  
      
    vm.startPrank(projectCreator);  
      
    // === Test 1: addUsersToWhitelist ===  
    vesting.addUsersToWhitelist(users, PROJECT_ID);  
      
    address[] memory overlappingUsers = new address[](3);  
    overlappingUsers[0] = address(0x2); // Already whitelisted  
    overlappingUsers[1] = address(0x4); // New user  
    overlappingUsers[2] = address(0x5); // New user  
      
    vm.expectRevert("Already whitelisted");  
    vesting.addUsersToWhitelist(overlappingUsers, PROJECT_ID);  
      
    // 0x4 and 0x5 NOT whitelisted despite being valid  
    assertFalse(vesting.isWhitelisted(address(0x4), PROJECT_ID));  
    assertFalse(vesting.isWhitelisted(address(0x5), PROJECT_ID));  
      
    // === Test 2: removeUsersFromWhitelist ===  
    address[] memory removalList = new address[](3);  
    removalList[0] = address(0x1); // Whitelisted  
    removalList[1] = address(0x6); // Never whitelisted  
    removalList[2] = address(0x2); // Whitelisted  
      
    vm.expectRevert("address is not whitelisted");  
    vesting.removeUsersFromWhitelist(removalList, PROJECT_ID);  
      
    // 0x1 and 0x2 still whitelisted despite being valid removal targets  
    assertTrue(vesting.isWhitelisted(address(0x1), PROJECT_ID));  
    assertTrue(vesting.isWhitelisted(address(0x2), PROJECT_ID));  
      
    // === Test 3: disableWhitelists ===  
    vesting.enableWhitelists(projects);  
    vesting.disableWhitelist(2); // Disable project 2  
      
    uint256[] memory disableList = new uint256[](3);  
    disableList[0] = 2; // Already disabled  
    disableList[1] = 1; // Enabled  
    disableList[2] = 3; // Enabled  
      
    vm.expectRevert("Whitelisting is already disabled for this project");  
    vesting.disableWhitelists(disableList);  
      
    // Projects 1 and 3 still enabled despite being valid targets  
    assertTrue(vesting.isWhitelistEnabled(1));  
    assertTrue(vesting.isWhitelistEnabled(3));  
      
    vm.stopPrank();  
}
```

### Mitigation

Modify all three batch functions to skip invalid entries instead of reverting:


  