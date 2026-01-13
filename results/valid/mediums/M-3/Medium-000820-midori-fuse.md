# [000820] `allocationOf` mapping stores stale copies, causing unfair dividend distribution
  
  ### Summary

The `allocationOf[nftId]` mapping stores a copy of allocations at mint time but is never updated when users claim tokens. The dividend distributor reads stale data from `allocationOf` and calculate dividend unfairly for longer holded user


### Root Cause


In Solidity, assigning a storage struct to another storage variable like `allocationOf[nftId] = allocation` creates an independent copy, not a reference. `allocationOf` is assigned when the a new NFT is minted in [\_claimRewardTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L601-L623), [claimNFT()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860-L891), [splitTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107) and [distributeRemainingRewardTVS()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L554-L577) like this:

```solidity
function distributeRemainingRewardTVS(...) external onlyOwner {
    //...
    allocationOf[nftId] = rewardProject.allocations[nftId];
    //...
}
```

However, when users claim tokens via [claimTokens()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975), only the original allocation (`biddingProjects[projectId].allocations[nftId]`/`rewardProjects[projectId].allocations[nftId]`) is updated:

```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    //...
    (Allocation storage allocation, IERC20 token) = isBiddingProject
        ? (biddingProjects[projectId].allocations[nftId], ...)
        : (rewardProjects[projectId].allocations[nftId], ...);
    //...
    for (uint256 i; i < nbOfFlows; i++) {
        // ...
        allocation.claimedSeconds[i] += claimableSeconds;
        if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
            flowsClaimed++;
            allocation.claimedFlows[i] = true;
        }
        allClaimableSeconds[i] = claimableSeconds;
        //...
    }
    //...
}
```

And the dividend distributor reads from `allocationOf` in [getUnclaimedAmount()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161):

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    //...
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    //...
    for (uint i; i < len; ) {
        //..
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;
    }
    unclaimedAmountsIn[nftId] = amount;
}
```


### Internal Pre-conditions

1. User has called `claimTokens()` to make `allocationOf` stale
2. Owner subsequently calls `setUpTheDividends()`

### External Pre-conditions

None

### Attack Path


1. User receives NFT with allocation (e.g., 1000 tokens over 30 days)
2. After 15 days, user calls `claimTokens()` and receives 500 token
3. The original allocation (`biddingProjects[projectId].allocations[nftId]`) is updated:
   - `claimedSeconds[0] = 15 days`
   - Remaining unclaimed = 500 tokens
4. However, `allocationOf[nftId]` still shows:
   - `claimedSeconds[0] = 0`
   - Unclaimed = 1000 tokens (stale)
5. Owner deploys dividend distributor and calls `setUpTheDividends()`
6. `getUnclaimedAmounts(nftId)` reads from `allocationOf` and returns 1000 tokens instead of 500

### Impact

- Users who claimed early get more dividends than they should; users who haven't claimed yet get less
- The `allocationOf` mapping becomes increasingly inaccurate over time as more claims occur


### PoC

_No response_

### Mitigation

Add a single line to sync `allocationOf` in `claimTokens()`:

```diff
function claimTokens(uint256 projectId, uint256 nftId) external {
    //...
    allocationOf[nftId] = allocation;
    //...
}
```

  