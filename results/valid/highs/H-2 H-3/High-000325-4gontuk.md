# [000325] TVS splitting and merging always revert, making these features unusable
  
  ## Finding Description
The vesting system exposes two core user features to manage Token Vesting Schedules (TVSs): [`mergeTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1026) and [`splitTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054-L1107), implemented in `AlignerzVesting`. Both rely on helper logic in `FeesManager` and [`_computeSplitArrays()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141) to charge fees and recompute per-flow amounts. Due to incorrect handling of dynamic memory arrays and a missing loop increment, these helpers are fundamentally broken, causing any realistic call to `mergeTVS()` or `splitTVS()` on a non-empty TVS to revert. This results in a permanent DoS of the splitting/merging functionality.

The first root cause is in [`FeesManager.calculateFeeAndNewAmountForOneTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174):

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

Issues in this function:

- `newAmounts` is never allocated with `new uint256[](length)`. In memory, a dynamic array default value is a zero-length array; writing `newAmounts[i]` when `length > 0` is an out-of-bounds access, which reverts with a panic.
- The loop uses `for (uint256 i; i < length;)` without incrementing `i`. Even if `newAmounts` were allocated correctly, the loop would never terminate because `i` remains zero.

This helper is used in both `mergeTVS()` and `splitTVS()`:

```solidity
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
    ...
    uint256[] memory amounts = mergedTVS.amounts;
    uint256 nbOfFlows = mergedTVS.amounts.length;
    (uint256 feeAmount, uint256[] memory newAmounts) =
        calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
    mergedTVS.amounts = newAmounts;
    ...
}
```

```solidity
function splitTVS(
    uint256 projectId,
    uint256[] calldata percentages,
    uint256 splitNftId
) external returns (uint256, uint256[] memory) {
    ...
    uint256[] memory amounts = allocation.amounts;
    uint256 nbOfFlows = allocation.amounts.length;
    (uint256 feeAmount, uint256[] memory newAmounts) =
        calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows);
    allocation.amounts = newAmounts;
    token.safeTransfer(treasury, feeAmount);
    ...
}
```

For any realistic TVS, `allocation.amounts.length` / `mergedTVS.amounts.length` is at least 1 (there is at least one vesting flow). Thus, whenever a user calls `mergeTVS()` or `splitTVS()` on a valid TVS, the call reaches `calculateFeeAndNewAmountForOneTVS()` with `length > 0` and reverts at the first out-of-bounds write to `newAmounts[0]`.

The second independent root cause lies in [`AlignerzVesting._computeSplitArrays()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141):

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
)
    internal
    view
    returns (
        Allocation memory alloc
    )
{
    uint256[] memory baseAmounts = allocation.amounts;
    uint256[] memory baseVestings = allocation.vestingPeriods;
    uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
    uint256[] memory baseClaimed = allocation.claimedSeconds;
    bool[] memory baseClaimedFlows = allocation.claimedFlows;
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;
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

Here, `alloc` is an `Allocation memory` struct whose dynamic arrays (`alloc.amounts`, `alloc.vestingPeriods`, `alloc.vestingStartTimes`, `alloc.claimedSeconds`, `alloc.claimedFlows`) are never allocated before use. Each of these is initially a zero-length memory array. When `nbOfFlows > 0`, the first iteration attempts `alloc.amounts[0] = ...`, which again is an out-of-bounds write and reverts.

This helper is used in `splitTVS()` when redistributing the updated base allocation across multiple new NFTs:

```solidity
Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
NFTBelongsToBiddingProject[nftId] = isBiddingProject ? true : false;
Allocation storage newAlloc = isBiddingProject
    ? biddingProjects[projectId].allocations[nftId]
    : rewardProjects[projectId].allocations[nftId];
_assignAllocation(newAlloc, alloc);
allocationOf[nftId] = newAlloc;
```

Thus, even if `calculateFeeAndNewAmountForOneTVS()` were fixed, `splitTVS()` would still revert inside `_computeSplitArrays()` for any TVS that has at least one flow.

In summary, there are two independent root causes (unallocated dynamic arrays and missing loop increment), both contributing to the same user-visible effect: any attempt to split or merge a non-empty TVS reverts, making TVS splitting/merging unusable in practice.

## Attack Path
1. A user participates in a project (bidding or reward) and receives a TVS NFT. For example, in the reward path, the owner calls `launchRewardProject()` and `setTVSAllocation()`, then the KOL calls `claimRewardTVS()` to mint an NFT with a non-empty `Allocation` for that NFT ID.
2. Once the user owns this NFT, they attempt to manage their position by calling `splitTVS()` to split the TVS into multiple NFTs, or `mergeTVS()` to merge several TVSs into one.
3. Inside `splitTVS()`:
   - The contract computes `nbOfFlows = allocation.amounts.length`, which is at least 1 for a valid TVS.
   - It calls `calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows)`, which reverts due to writing into an unallocated `newAmounts` array (and would also never terminate due to the missing `i++`).
   - Even if this function were corrected, `_computeSplitArrays()` would later revert when writing into unallocated dynamic arrays in `Allocation memory alloc`.
4. Inside `mergeTVS()`:
   - The contract computes `nbOfFlows = mergedTVS.amounts.length` and calls `calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows)`, which also reverts due to the same unallocated `newAmounts` issue.
5. As a result, any user calling `splitTVS()` or `mergeTVS()` on a normal, non-empty TVS consistently triggers a revert, preventing them from using these features at all.

## Impact
Medium. Splitting and merging TVSs are fully DoSed for all users, which is a significant loss of advertised protocol functionality, even though user funds are not directly stolen or frozen.

## Likelihood
High. Any user holding a valid TVS NFT can trigger the bug by directly calling `splitTVS()` or `mergeTVS()` on-chain; there are no special preconditions beyond normal TVS creation.

## Proof of Concept
Add the PoC test to `protocol/test/AlignerzVestingProtocolTest.t.sol` as `test_splitTVS_reverts_due_to_broken_fee_logic()`.

```solidity
    function test_splitTVS_reverts_due_to_broken_fee_logic() public {
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 30 days);

        address[] memory kolTVS = new address[](1);
        kolTVS[0] = bidders[0];
        uint256[] memory TVSamounts = new uint256[](1);
        TVSamounts[0] = 1 ether;

        vesting.setTVSAllocation(0, 1 ether, 90 days, kolTVS, TVSamounts);
        vm.stopPrank();

        address kol = bidders[0];
        vm.prank(kol);
        vesting.claimRewardTVS(0);

        // ERC721A starts token IDs at 1, and getTotalMinted() returns the
        // number of minted tokens, so after the first mint the NFT ID is 1.
        uint256 nftId = nft.getTotalMinted();
        assertEq(nft.ownerOf(nftId), kol);

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;

        vm.startPrank(kol);
        vm.expectRevert();
        vesting.splitTVS(0, percentages, nftId);
        vm.stopPrank();
    }
```

Conceptually, the PoC:

1. Uses the existing test setup to deploy `AlignerzVesting`, `Aligners26`, `AlignerzNFT`, and `MockUSD`.
2. As `projectCreator`, it calls:
   - `launchRewardProject(address(token), address(usdt), block.timestamp, 30 days)`
   - `setTVSAllocation(0, 1 ether, 90 days, [bidders[0]], [1 ether])` to allocate a 1-ether TVS to `bidders[0]`.
3. As `bidders[0]`, it calls `claimRewardTVS(0)` to mint an NFT that owns a TVS with exactly one vesting flow.
4. It retrieves the NFT ID using `nft.getTotalMinted()` (which is 1 after the first mint, given `ERC721A` starts at 1), and asserts that `nft.ownerOf(nftId)` is `bidders[0]`.
5. It constructs `percentages = [5000, 5000]` (a 50/50 split) and then:

   - Starts a prank as the KOL (the NFT owner),
   - Calls `vm.expectRevert()`,
   - Calls `vesting.splitTVS(0, percentages, nftId)`,
   - Ends the prank.

Because `splitTVS()` hits the broken `calculateFeeAndNewAmountForOneTVS()` (and would later hit `_computeSplitArrays()`), the call reverts, making the test pass under `expectRevert()` and proving that `splitTVS()` is unusable on a real TVS.

To run the PoC:

1. From the `protocol` directory, ensure a full build:

   ```bash
   forge clean
   forge build
   ```

2. Then run the targeted test:

   ```bash
   forge test --mt test_splitTVS_reverts_due_to_broken_fee_logic
   ```

The test should pass by expecting the revert, demonstrating that calling `splitTVS()` on a valid, non-empty TVS always reverts.

A similar PoC could be constructed for `mergeTVS()` by creating two TVSs for the same token and attempting to merge them; the call would revert at the same helper function.

## Recommendation
There are two core fixes required:

1. **Fix `calculateFeeAndNewAmountForOneTVS()`**:
   - Allocate `newAmounts` with the correct length before writing into it.
   - Increment the loop index `i` each iteration.
   - Consider applying the fee per flow (or cumulatively) in a way that matches the intended economics; the example below keeps the cumulative fee logic but at least makes the function correct and terminating.

2. **Fix `_computeSplitArrays()`**:
   - Allocate all dynamic arrays (`amounts`, `vestingPeriods`, `vestingStartTimes`, `claimedSeconds`, `claimedFlows`) with length `nbOfFlows` before writing into them.


  