# [000015] Wrong `projectId` in `claimTokens burns NFT leading to value loss
  
  ### Summary

If a usercalls `claimTokens` with a wrong `projectId` but a valid `nftId`, the function will:
- Load an empty allocation from the wrong project
- Treat this empty allocation as “fully claimed” (`flowsClaimed == nbOfFlows == 0`)
- Burn the user’s NFT
- Transfer 0 tokens to the user
The user then permanently loses practical access to their real claim under the correct project, because the NFT is the handle required for claiming. 

### Root Cause

Allocation lookup is keyed by (projectId, nftId) without any consistency check in [`claimTokens`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L944-L947):
```solidity
bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
(Allocation storage allocation, IERC20 token) = isBiddingProject
    ? (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token)
    : (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
```
The function blindly trusts the externally supplied projectId. There is no check that:
- `nftId` was actually minted under projectId
- The allocation has any vesting flows at all

For the retrieved `allocation`, the function does the following:
```solidity
uint256 nbOfFlows = allocation.vestingPeriods.length;
uint256 flowsClaimed;
// for loop guarded by nbOfFlows

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
    // ... accumulate claimableAmounts etc.
}

if (flowsClaimed == nbOfFlows) {
    nftContract.burn(nftId);
    allocation.isClaimed = true;
}

token.safeTransfer(msg.sender, claimableAmounts);
```
Hence, for a wrong `projectId`, `biddingProjects[projectId].allocations[nftId]` is an uninitialized struct: `vestingPeriods.length == 0`, `claimedFlows.length == 0`. So, the loop never executes with `nbOfFlows == 0`, so `flowsClaimed` stays 0. The condition `flowsClaimed == nbOfFlows` becomes `0 == 0`, so the NFT is burned and `allocation.isClaimed = true` is set on the empty allocation. Also, `claimableAmounts` is 0, so the user receives no tokens
The real vesting data is stored in `biddingProjects[correctProjectId].allocations[nftId]`. Calling `claimTokens(wrongProjectId, nftId)` never touches it, but the NFT (the only handle) is destroyed. Because `claimTokens` requires `msg.sender == nftContract.extOwnerOf(nftId)`, the user can no longer call it for the correct project once the NFT is burned.

Therefore, a single wrong `projectId` causes permanent economic loss for the user.

### Internal Pre-conditions

1. 2 projects are launched 
2. The user successfully claims their NFT via `nftId = vesting.claimNFT(0, 0, BIDDER_USD, proofForProject0);` so that `nft.ownerOf(nftId) == bidder`
3. `NFTBelongsToBiddingProject[nftId] == true`, so the bidding branch is taken in claimTokens.

### External Pre-conditions

1. The caller is the NFT owner
2. The caller passes a wrong but existing `projectId`

### Attack Path

1. User wins allocation in Project 0 
2. A separate Project 1 exists, but the user has no allocation there. Thus `biddingProjects[1].allocations[nftId]` is an empty struct
3. User  mistakenly calls `claimTokens(1, nftId)` even though the NFT belongs to Project 0
4. claimTokens fetches the allocation from `projectId = 1`. This allocation has:
- vestingPeriods.length == 0
- claimedFlows.length == 0
5. The function interprets 0 flows to claim as “all flows fully claimed”
because `flowsClaimed == nbOfFlows` is `0 == 0`
6. It burns the NFT, marks the empty allocation as claimed, and transfers 0 tokens.
7. Because the NFT is the only access path to claim tokens under Project 0, the user’s actual allocation becomes permanently inaccessible.

### Impact

High impact: 
1. The user permanently loses the ability to claim their vested tokens.
No workaround exists on-chain because `claimTokens` requires ownership of the NFT, and the NFT is burned.
2. A single incorrect parameter destroys the user’s entitlement irreversibly.
3. The valid allocation (in the correct project) becomes stranded 

Medium likelihood:
1. The issue requires no privileges. Any NFT holder can accidentally call `claimTokens` with a wrong `projectId` but this is a specific condition, albeit very possible.

### PoC

Add this funtion to `test/AlignerzVestingProtocolTest.t.sol`:
```solidity
    function test_ClaimTokens_WrongProjectId_BurnsNFT_AndLocksAllocation() public {
        address bidder = bidders[0];

        // === 1. Owner config ===
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        uint256 startTime0 = block.timestamp;
        uint256 endTime0 = block.timestamp + 10 days;
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            startTime0,
            endTime0,
            bytes32(0),
            false
        );

        uint256 totalAllocation0 = 1_000_000 ether;
        vesting.createPool(0, totalAllocation0, 1e18, false);

        // dummy project 1
        uint256 startTime1 = block.timestamp;
        uint256 endTime1 = block.timestamp + 20 days;
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            startTime1,
            endTime1,
            bytes32(0),
            false
        );
        vm.stopPrank();

        // === 2. Bid in project 0 and get a valid NFT ===
        vm.warp(startTime0 + 1);

        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        uint256 vestingPeriodBid = 30 days;
        vesting.placeBid(0, BIDDER_USD, vestingPeriodBid);
        vm.stopPrank();

        bytes32 leaf = getLeaf(bidder, BIDDER_USD, 0, 0);
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = leaf;

        vm.prank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRoots, 60 days);

        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(0, 0, BIDDER_USD, new bytes32[](0));

        // Pre: bidder owns NFT
        assertEq(nft.ownerOf(nftId), bidder, "pre: bidder should own NFT");

        // === 3. Attack: wrong projectId ===
        uint256 balanceBefore = token.balanceOf(bidder);

        vm.prank(bidder);
        vesting.claimTokens(1, nftId); // WRONG projectId

        // === 4. Post-conditions ===

        // NFT burned
        vm.expectRevert();
        nft.ownerOf(nftId);

        // No tokens paid
        uint256 balanceAfter = token.balanceOf(bidder);
        assertEq(balanceAfter, balanceBefore, "no tokens should be paid on wrong-project claim");
    }
```

### Mitigation

Require non-empty vesting flows before burning

```solidity
uint256 nbOfFlows = allocation.vestingPeriods.length;
require(nbOfFlows > 0, Invalid_NFT_For_Project());
```
Now, for the wrong project `nbOfFlows == 0` reverts. For the correct project, `nbOfFlows > 0` as expected.
  