# [000312] DoS on token claims for merged NFTs when any flow has zero claimable amount
  
  ## Finding Description
In `AlignerzVesting.sol` , the [`claimTokens()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975) function allows users to claim vested tokens for a given NFT that can contain one or more vesting “flows” (e.g. after `mergeTVS()` / `splitTVS()`).

The core logic in `claimTokens()` is:

- It determines whether the NFT belongs to a bidding or reward project using `NFTBelongsToBiddingProject[nftId]`.
- It loads the corresponding `Allocation` (which contains arrays `amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`).
- It iterates all flows `i` in the allocation (`nbOfFlows = allocation.vestingPeriods.length`).
- For each flow that is not fully claimed, it calls `getClaimableAmountAndSeconds(allocation, i)` and then accumulates the returned `claimableAmount` into the total `claimableAmounts`.

The relevant snippet is:

```solidity
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
    token.safeTransfer(msg.sender, claimableAmounts);
    emit TokensClaimed(...);
}
```

The helper [`getClaimableAmountAndSeconds()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996) is implemented as:

```solidity 
function getClaimableAmountAndSeconds(
    Allocation memory allocation,
    uint256 flowIndex
) public view returns (uint256 claimableAmount, uint256 claimableSeconds) {
    uint256 secondsPassed;
    uint256 claimedSeconds = allocation.claimedSeconds[flowIndex];
    uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
    uint256 vestingStartTime = allocation.vestingStartTimes[flowIndex];
    uint256 amount = allocation.amounts[flowIndex];

    if (block.timestamp > vestingPeriod + vestingStartTime) {
        secondsPassed = vestingPeriod;
    } else {
        secondsPassed = block.timestamp - vestingStartTime;
    }

    claimableSeconds = secondsPassed - claimedSeconds;
    claimableAmount = (amount * claimableSeconds) / vestingPeriod;
    require(claimableAmount > 0, No_Claimable_Tokens());
    return (claimableAmount, claimableSeconds);
}
```

Root cause:

- `getClaimableAmountAndSeconds()` enforces `require(claimableAmount > 0, No_Claimable_Tokens());` **per flow**.
- `claimTokens()` does **not** guard against this requirement; it calls `getClaimableAmountAndSeconds()` for each not-fully-claimed flow in a loop without skipping flows that have zero claimable amounts.
- As a result, if any active flow `i` has `claimableAmount == 0` (for example:
  - because the flow has not effectively started yet for this call (e.g. heterogeneous `vestingStartTimes` after `mergeTVS()`), or
  - because `(amount * claimableSeconds) / vestingPeriod` rounds down to 0 due to a very small `amount` or large `vestingPeriod`),
  then `getClaimableAmountAndSeconds()` reverts and **aborts the entire claim across all flows**.

This is particularly impactful for **multi-flow NFTs** created via `mergeTVS()`:

- `mergeTVS()` appends flows from multiple NFTs into a single `Allocation` without normalizing start times or amounts:

```solidity 
mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
```

- This means a user can (and is expected to) obtain a single NFT with multiple flows having **different `amounts`, `vestingPeriods` and `vestingStartTimes`**.

Highest-impact scenario will occur like:

1. A user holds two NFTs:
   - NFT A: large allocation, shorter vesting configuration such that at time `t` it has a **positive** `claimableAmount`.
   - NFT B: tiny allocation and/or longer vesting such that at the same time `t`, `(amountB * (t - startB)) / vestingPeriodB == 0`.
2. The user merges NFT A and NFT B into a single multi-flow NFT using `mergeTVS()`.
3. At time `t`, the user calls `claimTokens()` on the merged NFT.
4. In the `for` loop of `claimTokens()`, when the flow corresponding to NFT B is processed:
   - `getClaimableAmountAndSeconds()` computes `claimableAmount == 0` and reverts with `No_Claimable_Tokens()`.
5. The entire transaction fails, and the user cannot claim **any** tokens from the merged NFT, including the fully/partially vested flow from NFT A.

In summary, `claimTokens()` behaves as “all flows must individually have strictly positive claimable amount for this call to succeed”, which is not documented and causes protocol-level DoS for users of legitimate merge/split functionality.

## Impact
High. A user’s vested tokens in a merged/split NFT can be unclaimable for extended periods whenever any flow has zero claimable tokens, causing a severe disruption of protocol functionality/availability for legitimate users.

## Likelihood
Medium. Multi-flow NFTs are an intended and exposed feature via `mergeTVS()`/`splitTVS()`, and it is realistic that users or integrators will create heterogeneous flows; however, exploiting the issue generally requires the user to perform particular merge/split patterns rather than an external attacker forcing it.

## Proof of Concept
Add the PoC to `protocol/test/AlignerzVestingProtocolTest.t.sol` :

```solidity
    function test_ClaimTokens_Reverts_When_One_Flow_Has_Zero_Claimable() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        uint256 startTime = block.timestamp;
        vesting.launchRewardProject(address(token), address(usdt), startTime, 30 days);

        address[] memory kolTVS = new address[](2);
        kolTVS[0] = makeAddr("kolBig");
        kolTVS[1] = makeAddr("kolSmall");

        uint256[] memory TVSamounts = new uint256[](2);
        TVSamounts[0] = 1 ether; // large flow
        TVSamounts[1] = 1;       // very small flow (1 wei)

        uint256 vestingPeriod = 1000;
        uint256 totalTVSAllocation = TVSamounts[0] + TVSamounts[1];

        vesting.setTVSAllocation(0, totalTVSAllocation, vestingPeriod, kolTVS, TVSamounts);
        vm.stopPrank();

        vm.prank(kolTVS[0]);
        vesting.claimRewardTVS(0);
        uint256 nftIdBig = nft.getTotalMinted();

        vm.prank(kolTVS[1]);
        vesting.claimRewardTVS(0);
        uint256 nftIdSmall = nft.getTotalMinted();

        address aggregator = kolTVS[0];

        vm.prank(kolTVS[1]);
        nft.transferFrom(kolTVS[1], aggregator, nftIdSmall);

        uint256 mergedNftId = nft.mint(aggregator);

        vm.startPrank(aggregator);

        uint256[] memory projectIds = new uint256[](2);
        projectIds[0] = 0;
        projectIds[1] = 0;

        uint256[] memory nftIdsToMerge = new uint256[](2);
        nftIdsToMerge[0] = nftIdBig;
        nftIdsToMerge[1] = nftIdSmall;

        vesting.mergeTVS(0, mergedNftId, projectIds, nftIdsToMerge);
        vm.stopPrank();

        vm.warp(startTime + 1);

        assertEq(nft.ownerOf(mergedNftId), aggregator);

        vm.startPrank(aggregator);
        vm.expectRevert(AlignerzVesting.No_Claimable_Tokens.selector);
        vesting.claimTokens(0, mergedNftId);
        vm.stopPrank();
    }
```

**How to run the PoC:**
From the `protocol` directory:

```bash 
forge clean
forge build
```

- Run only the PoC test:

```bash 
forge test --mt test_ClaimTokens_Reverts_When_One_Flow_Has_Zero_Claimable -vvv
```

The test should pass and clearly show `claimTokens()` reverting with `No_Claimable_Tokens` despite at least one flow having a positive claimable amount.

## Recommendation
The safest and simplest fix is to:

1. **Remove the per-flow strict check** `require(claimableAmount > 0, No_Claimable_Tokens());` in `getClaimableAmountAndSeconds()`.
2. Make `getClaimableAmountAndSeconds()` return `(0, 0)` when a flow has no claimable tokens (e.g. `secondsPassed <= claimedSeconds`).
3. In `claimTokens()`, **skip flows with `claimableAmount == 0`** (do not update `claimedSeconds` for them).
4. Optionally add a **single aggregate check** after the loop: `require(claimableAmounts > 0, No_Claimable_Tokens());` to preserve the “no-op claim” protection at the NFT level rather than per-flow level.
  