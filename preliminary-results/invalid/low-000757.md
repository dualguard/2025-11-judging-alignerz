# [000757] NFT minting flows lack reentrancy protection around `safeMint`,riskin via ERC721 receiver callbacks
  
  
### Summary

Several vesting functions that mint NFTs using `nftContract.mint` (which internally uses `safeMint`) do not use any reentrancy guard, so a malicious NFT receiver contract can reenter the vesting contract via `onERC721Received` while state is only partially updated.

### Root Cause

In `AlignerzVesting.sol` functions like `distributeRemainingRewardTVS`, `_claimRewardTVS`, `claimNFT`, and `splitTVS` call `nftContract.mint(...)` without `nonReentrant`, even though `AlignerzNFT.mint` uses `_safeMint`, which triggers `onERC721Received` on receiver contracts and allows arbitrary reentrant calls back into vesting while allocations and arrays are mid-update.

```solidity
    function distributeRemainingRewardTVS(uint256 rewardProjectId) external onlyOwner{ 
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        uint256 len = rewardProject.kolTVSAddresses.length;
        for (uint256 i = len - 1; rewardProject.kolTVSAddresses.length > 0;) {
            address kol = rewardProject.kolTVSAddresses[i];
            uint256 amount = rewardProject.kolTVSRewards[kol];
            rewardProject.kolTVSRewards[kol] = 0;
>>          uint256 nftId = nftContract.mint(kol); 
            rewardProject.allocations[nftId].amounts.push(amount);
            ...
            rewardProject.kolTVSAddresses.pop();
            allocationOf[nftId] = rewardProject.allocations[nftId];
            ...
        }
    }
```

Similar patterns exist in `_claimRewardTVS`, `claimNFT`, and `splitTVS`, all of which mint to user-controlled addresses that may be contracts with reentrant `onERC721Received` hooks.

### Impact

Under the current code and token/NFT choices there is no concrete exploit demonstrated, but these functions are reentrancy-open around sensitive state mutations, so a malicious KOL contract (or other receiver) could in the future exploit partial-state windows (e.g. during distributions, splits, or claims) if additional state-dependent logic is added; this is best treated as hardening.

### Mitigation

Wrap all user-facing functions that call `nftContract.mint` (`distributeRemainingRewardTVS`, `distributeRewardTVS`, `claimRewardTVS` via `_claimRewardTVS`, `claimNFT`, `splitTVS`, `mergeTVS` if it ever mints) with a `nonReentrant` modifier (from `ReentrancyGuard`) and keep the existing Checks-Effects-Interactions ordering, ensuring no external callback (including `onERC721Received`) can reenter vesting while allocations and arrays are mid-update.
  