# [000935] Users will be unable to claim any tokens from their NFT when merging NFTs from different projects with different vesting start times
  
  ### Summary

The lack of validation for future vesting start times in ` AlignerzVesting::getClaimableAmountAndSeconds` will cause denial of service for users with multi-flow NFTs as an attacker or normal user will merge NFTs from different projects (with the same token but different vesting schedules), causing arithmetic underflow when attempting to claim tokens from any flow. 

### Root Cause

In [AlignerzVesting.sol:989 ](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L989)
the calculation
```solidity
 secondsPassed = block.timestamp - vestingStartTime 
```
causes an arithmetic underflow when vestingStartTime is in the future. In [AlignerzVesting.sol:994](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L994) 
the requirement
```solidity
 require(claimableAmount > 0, No_Claimable_Tokens())
```
 forces every flow to have claimable tokens, contradicting the multi-flow design where flows can have different vesting schedules and claim states.
Scenerio: 
- User wins allocation in "Presale Round 1" (Project ID 0, vestingStartTime = block.timestamp - 30 days)
- User wins allocation in "Presale Round 2" (Project ID 1, vestingStartTime = block.timestamp + 30 days)
- Both projects vest the same token (passes the Different_Tokens() check ) [AlignerzVesting.sol:1035](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1035)
- User calls `mergeTVS(projectId, mergedNftId, [0, 1], [nftA, nftB])` to consolidate allocations
- The `_merge` preserves each flow's original `vestingStartTime` 
- User attempts to call `claimTokens` on the merged NFT to claim tokens from Presale Round 1's flow
- The `claimTokens` loop  iterates through all flows 
- When processing Presale Round 2's flow (future vesting), `getClaimableAmountAndSeconds` is called 
then the  calculation `block.timestamp - vestingStartTime underflows` (e.g., 100 - 130 = underflow) 
- Transaction reverts, user cannot claim any tokens from either flow

### Impact

Users cannot claim any tokens from their merged NFT when one flow has a future vesting start time

### PoC

_No response_

### Mitigation

_No response_
  