# [001050] User can be blocked to claim his tokens due to revert/underflow
  
  ### Summary

In `AlignerzVesting::claimTokens`  claiming can be blocked when any single allocation flow either has a vestingStart in the future or has amount == 0. Two cooperating causes are present in the code: an unsafe subtraction that can underflow when the vesting start is in the future, and an unconditional require(claimableAmount > 0) in `getClaimableAmountAndSeconds` that reverts when the claimable amount is zero. As a result, a single problematic flow (including a zero-sized flow produced by split/merge) can prevent the whole NFT from being claimed.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L958

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996

### Root Cause

If we look at `getClaimableAmountAndSeconds`:
```solidity
    function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex)
        public
        view
        returns (uint256 claimableAmount, uint256 claimableSeconds)
    {
        uint256 secondsPassed;
        uint256 claimedSeconds = allocation.claimedSeconds[flowIndex];
        uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
        uint256 vestingStartTime = allocation.vestingStartTimes[flowIndex];
        uint256 amount = allocation.amounts[flowIndex];
        if (block.timestamp > vestingPeriod + vestingStartTime) {
            secondsPassed = vestingPeriod;
        } else {
            //@audit underflow will revert resulting in claiming denial
>>          secondsPassed = block.timestamp - vestingStartTime;
        }

        claimableSeconds = secondsPassed - claimedSeconds;
        claimableAmount = (amount * claimableSeconds) / vestingPeriod;
        //@audit claimableAmount = 0 will block claiming of other flows
>>      require(claimableAmount > 0, No_Claimable_Tokens());
        return (claimableAmount, claimableSeconds);
    }
``` 
If vestingStartTime > block.timestamp (flow starts in the future) the else branch executes and block.timestamp - vestingStartTime underflows and revert.
Moreover, after computing `claimableAmount`, the function does require(claimableAmount > 0, No_Claimable_Tokens());
Even when no underflow occurs, if claimableAmount == 0 (flow amount is zero, or no new seconds to claim), this require will revert and abort the whole claimTokens call.
claimTokens processes all flows in a single loop. A revert on any flow prevents other flows (which may be eligible) from being claimed and the NFT from being burned.

This could be judged as separate issue, but I will make it in 1 report due to explanation and POC convenience.

Zero-sized flows are produced by split/merge
splitTVS and mergeTVS can create flows with amount = 0 (e.g. split percentage = 0 or fee equals amount). Those zero flows then trigger the require above, making them a denial-of-service vector for valid flows within the same NFT.

### Internal Pre-conditions

1. Reward project is launched with startTime in the future > block.timestamp
2. TVS is splitted or merged resulting in flows with zero amount

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

A holder cannot claim vested tokens for an NFT if any flow in that NFT is either not yet vested or has zero claimable amount - the whole claim reverts.
Tokens are effectively stuck until flows are fixed.

### PoC

In order to show the vulnerability, some functions must be modified which contain separate issues that will be in another report.
In `FeesManager` add helper function:
```solidity
    function calculateFeeAndNewAmountForOneTVSSafe(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        require(length <= amounts.length, "Invalid length");
        newAmounts = new uint256[](length);
        for (uint256 i = 0; i < length;) {
            uint256 perFee = calculateFeeAmount(feeRate, amounts[i]);
            // Subtract per-flow fee, avoid underflow by clamping to zero
            if (amounts[i] >= perFee) {
                newAmounts[i] = amounts[i] - perFee;
            } else {
                newAmounts[i] = 0;
            }
            feeAmount += perFee;
            unchecked {
                ++i;
            }
        }
        return (feeAmount, newAmounts);
    }
```
In `AlignerzVesting` add helper function:
```solidity
    function _computeSplitArraysSafe(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;

        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;

        // allocate memory arrays with the correct length before writing
        alloc.amounts = new uint256[](nbOfFlows);
        alloc.vestingPeriods = new uint256[](nbOfFlows);
        alloc.vestingStartTimes = new uint256[](nbOfFlows);
        alloc.claimedSeconds = new uint256[](nbOfFlows);
        alloc.claimedFlows = new bool[](nbOfFlows);

        for (uint256 j; j < nbOfFlows;) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```
And use them in correspondent functions:
in `splitTVS`

```solidity
    function splitTVS(uint256 projectId, uint256[] calldata percentages, uint256 splitNftId)
        external
        returns (uint256, uint256[] memory)
    {
        // ...
        uint256 nbOfFlows = allocation.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) =
>>          calculateFeeAndNewAmountForOneTVSSafe(splitFeeRate, amounts, nbOfFlows);
        allocation.amounts = newAmounts;
        token.safeTransfer(treasury, feeAmount);

        // ...
>>          Allocation memory alloc = _computeSplitArraysSafe(allocation, percentage, nbOfFlows);
            NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
        // ...
    }
```
in `mergeTVS`
```solidity
    function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds)
        external
        returns (uint256)
    {
        // ...
        uint256 nbOfFlows = mergedTVS.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) =
            calculateFeeAndNewAmountForOneTVSSafe(mergeFeeRate, amounts, nbOfFlows);
        mergedTVS.amounts = newAmounts;
        // ....
    }
```

Then add the following tests in `AlignerzVestingProtocolTest` and run them separately:
`forge test --mt test_ClaimTokens_Blocked_By_ZeroSplit -vvvv` and
`forge test --mt test_ClaimTokens_Blocked_By_MergedFutureFlow -vvvv`

```solidity
    function test_ClaimTokens_Blocked_By_ZeroSplit() public {
        address kol = bidders[0];

        // projectCreator (owner) launches reward project
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 days);

        // allocate a normal TVS amount to kol
        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory tvsAmounts = new uint256[](1);
        tvsAmounts[0] = 1_000 ether; // normal allocation

        // total allocation = 1
        vesting.setTVSAllocation(0, tvsAmounts[0], 90 days, kols, tvsAmounts);
        vm.stopPrank();

        // kol claims his reward TVS -> receives an NFT with a single flow of amount == 1
        vm.prank(kol);
        vesting.claimRewardTVS(0);

        // nft id should be 1 (first minted) - get owner and find token id by reading extOwnerOf(1)
        uint256 nftId = 1;
        assertEq(nft.extOwnerOf(nftId), kol);

        // Split the NFT into two parts with 0/10000 percentages -> original NFT will have amount 0, new NFT will have the full amount
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 0; // original NFT becomes zero-amount
        percentages[1] = 10000; // new NFT receives the full amount

        vm.prank(kol);
        // capture the split result: (splitNftId (original), newNftIds)
        (uint256 splitNftId, uint256[] memory newNftIds) = vesting.splitTVS(0, percentages, nftId);

        // Expect ownership: original (splitNftId) and new minted NFT belong to kol
        assertEq(nft.extOwnerOf(splitNftId), kol);
        assertEq(nft.extOwnerOf(newNftIds[0]), kol);

        // Merge the zero-amount NFT (original splitNftId) into the full one (newNftIds[0])
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = 0;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = splitNftId;

        vm.prank(kol);
        vesting.mergeTVS(0, newNftIds[0], projectIds, nftIds);

        // After merging, the merged NFT contains a zero-valued flow and should block claimTokens
        vm.prank(kol);
        vm.expectRevert();
        vesting.claimTokens(0, newNftIds[0]);
    }

    function test_ClaimTokens_Blocked_By_MergedFutureFlow() public {
        address kol = bidders[3];

        // Project 0: vesting starts now (claimable)
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 2 days);
        address[] memory kolsA = new address[](1);
        kolsA[0] = kol;
        uint256[] memory amtsA = new uint256[](1);
        amtsA[0] = 1_000 ether;
        vesting.setTVSAllocation(0, amtsA[0], 90 days, kolsA, amtsA);
        vm.stopPrank();

        // Project 1: vesting starts in the future (will create a future-start flow)
        uint256 futureStart = block.timestamp + 1 days;
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), futureStart, 2 days);
        address[] memory kolsB = new address[](1);
        kolsB[0] = kol;
        uint256[] memory amtsB = new uint256[](1);
        amtsB[0] = 1_000 ether;
        vesting.setTVSAllocation(1, amtsB[0], 90 days, kolsB, amtsB);
        vm.stopPrank();

        // kol claims both reward TVS NFTs. Read the counter after each mint to get the token id.
        vm.prank(kol);
        vesting.claimRewardTVS(0);
        uint256 nft1 = nft.getTotalMinted();

        vm.prank(kol);
        vesting.claimRewardTVS(1);
        uint256 nft2 = nft.getTotalMinted();

        // Sanity: ensure kol owns both NFTs
        assertEq(nft.extOwnerOf(nft1), kol);
        assertEq(nft.extOwnerOf(nft2), kol);

        // Merge the future-start NFT (project 1) into the claimable one (project 0)
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = 1;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nft2;

        vm.prank(kol);
        vesting.mergeTVS(0, nft1, projectIds, nftIds);

        // Attempting to claim the merged NFT should revert because one flow's
        // vestingStart is in the future (causes underflow / revert in helper)
        vm.prank(kol);
        vm.expectRevert();
        vesting.claimTokens(0, nft1);
    }
```

### Mitigation

Make `getClaimableAmountAndSeconds` defensive:
If block.timestamp <= vestingStartTime return (0, 0) (do not underflow).
Compute secondsPassed using safe comparisons.
Compute claimableSeconds as max(0, secondsPassed - claimedSeconds).
Return (0, 0) instead of reverting when claimableSeconds == 0 OR claimableAmount == 0.
Update claimTokens to treat (0,0) as “nothing to claim” and continue the loop rather than allowing a revert.
  