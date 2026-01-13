# [000493] Method AlignerzVesting::claimTokens() doesn't updates allocationOf mapping of NFT
  
  ### Summary

Method `AlignerzVesting::claimTokens()` only updates the `biddingProjects` and `rewardProjects` mappings. But it skips `allocationOf` mapping. This mapping is used by `A26ZDividendDistributor::getUnclaimedAmounts()`. So, even if user has claimed some of its tokens, He'll still get shares as dividend on total amount because **allocationOf** is not updated.

- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L937C1-L975C6
- https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141C1-L146


### Root Cause

Method `AlignerzVesting::claimTokens()` doesn't update `allocationOf` mapping. While it transfers unlocked tokens to the user. So, for `allocationOf` mapping, no tokens are claimed yet.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

As long as user's NFT is not burned, He'll continue to get dividend on 100% of it's amount. Even if he has claimed 99% of it's tokens.

### Impact

Users should be incentivised to keep their tokens locked. But instead, Here users can claim their tokens as per the vesting period and still get the dividend on the full amount.

### PoC

_No response_

### Mitigation

```diff
    // For both bidding and reward projects
    /// @notice Claims vested tokens
    /// @param projectId ID of the biddingProject
    /// @param nftId ID of the ownership NFT of the bid
    function claimTokens(uint256 projectId, uint256 nftId) external {
        address nftOwner = nftContract.extOwnerOf(nftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
        bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
        (Allocation storage allocation, IERC20 token) = isBiddingProject ? 
        (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) : 
        (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
        uint256 nbOfFlows = allocation.vestingPeriods.length;
        uint256 claimableAmounts;
        uint256[] memory amountsClaimed = new uint256[](nbOfFlows);
        uint256[] memory allClaimableSeconds = new uint256[](nbOfFlows);
        uint256 flowsClaimed;
        for (uint256 i; i < nbOfFlows; i++) {
            if (allocation.claimedFlows[i]) {
                flowsClaimed++;
                continue;
            }
            (uint256 claimableAmount, uint256 claimableSeconds) = getClaimableAmountAndSeconds(allocation, i);

            allocation.claimedSeconds[i] += claimableSeconds;
            if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
                flowsClaimed++;
                allocation.claimedFlows[i] = true;
            }
            allClaimableSeconds[i] = claimableSeconds;
            amountsClaimed[i] = claimableAmount;
            claimableAmounts += claimableAmount;
        }
        if (flowsClaimed == nbOfFlows) {
            nftContract.burn(nftId);
            allocation.isClaimed = true;
        }
++      allocationOf[nftId] = allocation;
        token.safeTransfer(msg.sender, claimableAmounts);
        emit TokensClaimed(projectId, isBiddingProject, allocation.assignedPoolId, allocation.isClaimed, nftId, allClaimableSeconds, block.timestamp, msg.sender, amountsClaimed);
    }
```
  