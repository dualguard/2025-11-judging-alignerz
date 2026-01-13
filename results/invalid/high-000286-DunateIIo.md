# [000286] Cross-Project Allocation Forgery via mergeTVS leading to steal projects TVS
  
  ### Summary

_(This is AI written, with a full review by me to finish by deadline)_
A critical vulnerability exists in the `mergeTVS` function. The function fails to validate that a provided nftId belongs to the user-supplied `projectId`. This allows an attacker to access and merge allocation data from an unauthorized project, provided the underlying token is identical.

### Root Cause

mergeTVS() and splitTVS functions trust the user passed projectID to be the one that the TVS tokens associated with

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. **Prerequisite**: Attacker owns nftId_A in Project A (ID 0) and nftId_B in Project B (ID 1). Both projects must vest the same IERC20 token.
2. **Execution**: Attacker calls mergeTVS(projectId: 0, mergedNftId: nftId_A, projectIds: [1], nftIds: [nftId_B]).
3. **Flaw**: The internal _merge function trusts the user-supplied projectId: 1 for nftId: nftId_B. It reads the allocation from Project B and copies it into the mergedTVS allocation for Project A.
4. **Theft**: The attacker's nftId_A now has a fraudulent, inflated claim in Project A. The attacker calls claimTokens to drain Project A's token balance, stealing funds from legitimate participants.

### Impact

This flaw enables an attacker to artificially inflate their TVS allocation in a target project. This manipulation breaks the contract's internal accounting and permits the attacker to withdraw funds in excess of their legitimate entitlement, resulting in a direct loss of assets for other project participants.

### PoC

_No response_

### Mitigation

The contract must store which `projectId` an nftId belongs to at the moment of minting (e.g., in new mapping(uint256 => uint256) public projectOfNFT;) and then use that stored ID to access data, rather than trusting the user-supplied projectId.
  