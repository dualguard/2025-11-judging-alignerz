# [000771] Stale `allocationOf[nftId]` after `claimToken`s causes overcounted unclaimed TVS in dividends
  
  
### Summary

Not updating `allocationOf[nftId]` after `claimTokens` will cause overcounted dividends for already‑claimed TVS flows, as the dividend distributor keeps reading stale `claimedSeconds` / `claimedFlows` from `allocationOf` while the real vesting state lives only in the per‑project `allocations` mappings.


### Root Cause

In `AlignerzVesting`, `claimTokens` mutates only the per‑project `allocations[projectId].allocations[nftId]` struct and never syncs `allocationOf[nftId]`:

```solidity
// File: AlignerzVesting.sol

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

>>      allocation.claimedSeconds[i] += claimableSeconds;
        if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
            flowsClaimed++;
>>          allocation.claimedFlows[i] = true;
        }

        allClaimableSeconds[i] = claimableSeconds;
        amountsClaimed[i] = claimableAmount;
        claimableAmounts += claimableAmount;
    }

    if (flowsClaimed == nbOfFlows) {
>>      nftContract.burn(nftId);
>>      allocation.isClaimed = true;
    }
    token.safeTransfer(msg.sender, claimableAmounts);
    emit TokensClaimed(...);
}
```

By design, `allocationOf` is only written on mint/split paths:

```solidity
// File: AlignerzVesting.sol

function _claimRewardTVS(uint256 rewardProjectId, address kol) internal {
    ...
    uint256 nftId = nftContract.mint(kol);
    rewardProject.allocations[nftId].amounts.push(amount);
    ...
>>  allocationOf[nftId] = rewardProject.allocations[nftId];
    ...
}

function claimNFT(...) external returns (uint256) {
    ...
    uint256 nftId = nftContract.mint(msg.sender);
    biddingProject.allocations[nftId].amounts.push(amount);
    ...
    biddingProject.allocations[nftId].token = biddingProject.token;
    NFTBelongsToBiddingProject[nftId] = true;
>>  allocationOf[nftId] = biddingProject.allocations[nftId];
    ...
}

function splitTVS(...) external returns (uint256, uint256[] memory) {
    ...
    uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
    ...
    Allocation storage newAlloc =
        isBiddingProject ? biddingProjects[projectId].allocations[nftId] : rewardProjects[projectId].allocations[nftId];
    _assignAllocation(newAlloc, alloc);
>>  allocationOf[nftId] = newAlloc;
    ...
}
```

The dividend distributor uses `allocationOf`’s `claimedSeconds` / `claimedFlows` to compute unclaimed TVS:

```solidity
// File: A26ZDividendDistributor.sol

function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId);

    if (address(token) != address(alloc.token)) {
        return 0;
    }

    uint256 len = alloc.amounts.length;
    for (uint256 i; i < len; ) {
>>      if (alloc.claimedFlows[i]) {
            unchecked { ++i; }
            continue;
        }

        if (alloc.claimedSeconds[i] == 0) {
>>          amount += alloc.amounts[i];
        } else {
>>          uint256 claimedAmount = alloc.claimedSeconds[i] * alloc.amounts[i] / alloc.vestingPeriods[i];
            uint256 unclaimedAmount = alloc.amounts[i] - claimedAmount;
            amount += unclaimedAmount;
        }

        unchecked { ++i; }
    }

    unclaimedAmountsIn[nftId] = amount;
}
```

Because `claimTokens` never writes `allocationOf[nftId] = allocation;`, `allocationOf`’s `claimedSeconds`, `claimedFlows`, and `isClaimed` remain at their **pre‑claim** values, while the real vesting state is only reflected in the internal `allocations[...]` mapping.



### Internal Pre-conditions

1. A TVS NFT exists (`nftId`) with some active flows and `allocationOf[nftId]` correctly populated on mint / split.  
2. The holder calls `claimTokens(projectId, nftId)` at least once, so that `allocation.claimedSeconds[i]`, `allocation.claimedFlows[i]`, and possibly `allocation.isClaimed` are updated in the per‑project `allocations[projectId].allocations[nftId]`, but `allocationOf[nftId]` is not updated.



### External Pre-conditions

1. The dividend distributor uses `vesting.allocationOf(nftId)` to determine unclaimed TVS, rather than reading the internal per‑project `allocations[...]` mapping.  
2. Admin calls `setAmounts()` / `setDividends()` / `setUpTheDividends()` after users have called `claimTokens`, so that dividends are calculated based on the outdated `allocationOf` view.



### Attack / Failure Path

1. Admin deploys `AlignerzVesting` and `A26ZDividendDistributor`, and configures normal TVS allocations so that `allocationOf[nftId]` and the per‑project `allocations[...]` are initially in sync.  
2. A user holds an NFT (`nftId`) with one or more vesting flows. Time passes; the user calls `claimTokens(projectId, nftId)` enough times that, internally, `allocation.claimedSeconds[i]` reaches `vestingPeriods[i]` and `allocation.claimedFlows[i]` are set to `true` for those flows, and potentially `allocation.isClaimed = true` with the NFT burned.  
3. Because `claimTokens` never refreshes `allocationOf[nftId]`, the `allocationOf` snapshot still shows `claimedSeconds[i] == 0` and `claimedFlows[i] == false` for those flows (i.e., as if nothing was claimed).  
4. Later, when admin sets up or recomputes dividends, `getUnclaimedAmounts(nftId)` uses `vesting.allocationOf(nftId)` and, seeing `claimedSeconds == 0` and `claimedFlows == false`, treats the entire flow amount as unclaimed, even though those tokens were already fully claimed via `claimTokens`.  
5. As a result, that NFT’s “unclaimed amount” in the dividend module is **too high**, so it continues to receive dividends for TVS that have already vested and been withdrawn, skewing the dividend pool allocation.

### Impact

Holders of NFTs that have already exercised `claimTokens` continue to earn dividends as if their claimed flows were still fully unclaimed, because `allocationOf[nftId]` never reflects updated `claimedSeconds` / `claimedFlows` / `isClaimed`. This leads to systematic overpayment of dividends to those NFTs, at the expense of other TVS holders and/or leaving the system with inconsistent accounting between vesting state and dividend state and can be abused to have higher amounts of dividends easily.


### PoC

Because of bug - the `allocationOf` ABI/interface mismatch- , `A26ZDividendDistributor.getUnclaimedAmounts` currently reverts* when trying to read `vesting.allocationOf(nftId)`, so we can’t write a clean POC for this bug without first fixing it.

### Mitigation

After `claimTokens` mutates the per‑project allocation for `nftId` (updating `claimedSeconds`, `claimedFlows`, and `isClaimed`), the contract should also refresh the public `allocationOf[nftId]` snapshot, for example:

```diff
function claimTokens(uint256 projectId, uint256 nftId) external {
    ...
    (Allocation storage allocation, IERC20 token) = ...;

    // existing claim loop & state updates on `allocation`...

    if (flowsClaimed == nbOfFlows) {
        nftContract.burn(nftId);
        allocation.isClaimed = true;
    }

+  allocationOf[nftId] = allocation;

    token.safeTransfer(msg.sender, claimableAmounts);
    emit TokensClaimed(...);
}
```

Alternatively, if `allocationOf` is intended purely as a “view” for external modules, the protocol should **stop using it for any stateful logic like dividends** and instead make the dividend module read the same per‑project `allocations[...]` mapping that `claimTokens` uses.
  