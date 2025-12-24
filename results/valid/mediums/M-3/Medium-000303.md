# [000303] Stale `allocationOf` in `AlignerzVesting` after `claimTokens` execution Enables Systematic Dividend Theft
  
  Active token claimers can steal disproportionate dividend allocations from passive holders

# Summary

The failure to update the `allocationOf` state mapping after claiming tokens will cause unfair dividend distribution for all vesting participants as active claimers will appear to have more unclaimed tokens than reality, systematically receiving inflated dividend shares at the expense of passive holders who wait for full vesting.

# Root Cause

In [`AlignerzVesting.sol`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860), the `claimTokens` function updates the primary allocation storage (`biddingProjects[projectId].allocations[nftId]` or `rewardProjects[projectId].allocations[nftId]`) to track claimed progress via `claimedSeconds[]`, `claimedFlows[]`, and `isClaimed` fields, but fails to synchronize these changes to the `allocationOf[nftId]` state mapping, leaving it with stale pre-claim values.

```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    ...
    (Allocation storage allocation, IERC20 token) = isBiddingProject
        ? (biddingProjects[projectId].allocations[nftId],
           biddingProjects[projectId].token)
        : (rewardProjects[projectId].allocations[nftId],
           rewardProjects[projectId].token);

    for (uint256 i; i < nbOfFlows; i++) {
        ...
        allocation.claimedSeconds[i] += claimableSeconds;
        if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
            allocation.claimedFlows[i] = true;
        }
    }

    if (flowsClaimed == nbOfFlows) {
        nftContract.burn(nftId);
        allocation.isClaimed = true;
    }

    //@audit-issue MISSING: allocationOf[nftId] = allocation;

    token.safeTransfer(msg.sender, claimableAmounts);
}
```

The `A26ZDividendDistributor` contract exclusively reads from the stale state in [`A26ZDividendDistributor.sol`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140):

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    //@audit Reads stale claimedSeconds from state
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;

    for (uint i; i < len; ) {
        uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
        uint256 unclaimedAmount = amounts[i] - claimedAmount;
        amount += unclaimedAmount;  // Overcounts due to stale claimedSeconds
    }
}
```

# Internal Pre-conditions

1. User needs to call `claimNFT()` or equivalent to create an NFT with vesting allocation
2. Vesting period needs to progress to allow partial claiming (example: 6 months of a 12-month vesting)
3. User needs to call `claimTokens()` to claim partially vested tokens, triggering the cache desync
4. Owner needs to call `A26ZDividendDistributor.setUpTheDividends()` to calculate dividends using the stale cache
5. Dividend pool needs to have available USDC to distribute

# External Pre-conditions

NA

# Attack Path

1. Alice calls `claimNFT()` with 10,000 token allocation and 12-month vesting period
2. Time passes: 6 months elapse (50% of vesting period complete)
3. Alice calls `claimTokens(projectId: 0, nftId: 1)`:
   - Function transfers 5,000 tokens to Alice (50% of allocation)
   - Updates `biddingProjects[0].allocations[1].claimedSeconds` to `[15,552,000]` (6 months)
   - **Fails to update** `allocationOf[1].claimedSeconds`, which remains `[0]`
4. Bob (passive holder) has similar allocation but doesn't claim, waiting for full vesting
5. Owner deposits 20,000 USDC into `A26ZDividendDistributor` contract
6. Owner calls `setUpTheDividends()`:
   - Calls `getTotalUnclaimedAmounts()` which iterates all NFTs
   - Alice's NFT # 1: reads `allocationOf[1].claimedSeconds = [0]` 
   - Calculates Alice's unclaimed as 10,000 tokens (wrong - should be 5,000)
   - For Bob's NFT # 2: reads `allocationOf[2].claimedSeconds = [0]` (correct)
   - Calculates Bob's unclaimed as 10,000 tokens (correct)
   - Total: 20,000 tokens
7. Dividend allocation:
   - Alice: `(10,000 / 20,000) × 20,000 = 10,000 USDC`
   - Bob: `(10,000 / 20,000) × 20,000 = 10,000 USDC`
8. Alice claims her dividend, receiving 10,000 USDC
9. Bob claims his dividend, receiving 10,000 USDC

**Result:** Alice received 10,000 USDC but should have received 6,667 USDC (3,333 USDC excess). Bob received 10,000 USDC but should have received 13,333 USDC (3,333 USDC deficit).

# Impact

Passive token holders (those waiting for full vesting) suffer dividend losses of 25-33% per distribution cycle. Active claimers gain this difference, receiving 50% more dividends than deserved. In a scenario with 10 users and a 1,000,000 USDC dividend pool, 5 passive holders collectively lose 166,665 USDC (33,333 USDC each or 25% underpayment), which is unfairly redistributed to 5 active claimers who each gain 33,333 USDC (50% overpayment).

# Mitigation

Update the `allocationOf` state mapping immediately after modifying the primary allocation storage in `claimTokens`:

```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    (Allocation storage allocation, IERC20 token) = isBiddingProject
        ? (biddingProjects[projectId].allocations[nftId],
           biddingProjects[projectId].token)
        : (rewardProjects[projectId].allocations[nftId],
           rewardProjects[projectId].token);

    uint256 nbOfFlows = allocation.vestingPeriods.length;

    require(nbOfFlows > 0, "NFT not in this project");
    require(address(allocation.token) == address(token), "Token mismatch");

    uint256 claimableAmounts;
    uint256[] memory amountsClaimed = new uint256[](nbOfFlows);
    uint256[] memory allClaimableSeconds = new uint256[](nbOfFlows);
    uint256 flowsClaimed;

    for (uint256 i; i < nbOfFlows; i++) {
        if (allocation.claimedFlows[i]) {
            flowsClaimed++;
            continue;
        }

        (uint256 claimableAmount, uint256 claimableSeconds) =
            getClaimableAmountAndSeconds(allocation, i);

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

    //@note FIX: Sync state mapping after all modifications
    allocationOf[nftId] = allocation;

    token.safeTransfer(msg.sender, claimableAmounts);
    emit TokensClaimed(
        projectId,
        isBiddingProject,
        allocation.assignedPoolId,
        allocation.isClaimed,
        nftId,
        allClaimableSeconds,
        block.timestamp,
        msg.sender,
        amountsClaimed
    );
}
```
  