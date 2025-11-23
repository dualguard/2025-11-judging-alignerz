# [000430] A malicious user can honeypot others by claiming the vested tokens before transferring the NFT.
  
  ### Summary

A malicious user can ``honeypot`` others by claiming the ``vested tokens`` before transferring the ``NFT``.

### Root Cause

As per the ``whitepaper``:
> we decided to tokenize vesting schedule so users can sell their vested tokens even before
maturity.
Investors can exit anytime by selling their TVS instead of dumping unlocked tokens.
At AlignerZ, vesting schedules are wrapped as NFTs (TVSs) that are claimable by the best
investors that secured an allocation.
This means they can be bought, sold, or traded on secondary markets (NFT marketplaces), even
before the vesting matures.
A long-term investor can buy vested tokens at a discount (TVS) similarly to Pendle Finance
LSTs but instead of a yield it’s a pre-paid discount.

https://drive.google.com/file/d/1xN5bYUPd_BkBMtoRruHEO1CBUx0vBiit/edit?pli=1

This suggests a few things:
1. ``TVS NFT`` is tradable on ``NFT marketplaces``.
2. Vesting doesn't need to ``mature`` and ``vestedTokens`` can be sold. 

But this is very easily susceptible to ``honeypot`` as ``vestedTokens`` can be claimed by the ``seller`` using ``claimTokens()``.
```solidity
./AlignerzVesting.sol

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
        token.safeTransfer(msg.sender, claimableAmounts);
        emit TokensClaimed(projectId, isBiddingProject, allocation.assignedPoolId, allocation.isClaimed, nftId, allClaimableSeconds, block.timestamp, msg.sender, amountsClaimed);
    }
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. Malicious seller ``Alice`` ``front-runs`` the ``NFT`` transfer after sale with ``claimTokens()`` to claim the ``vestedTokens``.
2. ``Bob`` the buyer who was supposed to get the ``vestedTokens`` + ``unvestedTokens`` at a discount will only get ``unvestedTokens``.
3. ``Alice`` steals ``vestedTokens`` from ``Bob``.

### Impact

Direct stealing of ``vestedTokens``.

### PoC

_No response_

### Mitigation

``vestedTokens`` shouldn't be up for sale. 
  