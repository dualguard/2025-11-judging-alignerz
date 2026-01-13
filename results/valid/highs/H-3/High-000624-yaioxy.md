# [000624] Uninitialized Memory Array in `_computeSplitArrays` Function Will Cause Array Out-of-Bounds Error
  
  ### Summary

Uninitialized memory array `alloc` in `_computeSplitArrays` function will cause an array out-of-bounds access error for any user attempting to split NFTs, as the function tries to write to an array with zero length.

### Root Cause

In [AlignerzVesting::_computeSplitArrays](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1121) the `alloc` array is declared but never initialized with a length:
```solidity
function _computeSplitArrays(
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc // <- declared but never initialized
        )
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId; //alloc never initialized
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

### Internal Pre-conditions

user calls splitTVS function

### External Pre-conditions

_No response_

### Attack Path

1. user calls `splitTVS` function
2. function calls `_computeSplitArrays`
3. Inside function: `alloc` has length 0
4. Solidity checks: 0 < 0.length → 0 < 0 → panic: array out-of-bounds (0x32)

### Impact

User cannot split NFTs, breaking the core protocol feature

### PoC

In order to get this right, we need first to fix the bug in `calculateFeeAndNewAmountForOneTVS` function (reported in another report)
Edit [calculateFeeAndNewAmountForOneTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169C5-L174C6) like this :
```diff
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
+       newAmounts = new uint256[](length);
+       for (uint256 i; i < length;i++) {
-        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
Note: fixed two issues in `calculateFeeAndNewAmountForOneTVS`: array out of bounds error for `newAmounts` and the loop iteration, so the test won't revert here and prove the array out of bounds is in `_computeSplitArrays` function

Next, pass this function in `AlignerzVestingProtocolTest.t.sol` file:
```solidity
function test_SplitTVS() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vesting.launchBiddingProject(
            address(token), 
            address(usdt), 
            block.timestamp, 
            block.timestamp + 1_000_000, 
            "0x0", 
            true
        );

        // 2. Create pool
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, false);

        // 3. Whitelist bidders
        address[] memory selectedBidders = new address[](3);
        selectedBidders[0] = bidders[0];
        selectedBidders[1] = bidders[1];
        selectedBidders[2] = bidders[2];
        vesting.addUsersToWhitelist(selectedBidders, PROJECT_ID);
        vm.stopPrank();

        // 4. Three users place bids
        for (uint256 i = 0; i < 3; i++) {
            vm.startPrank(bidders[i]);
            usdt.approve(address(vesting), BIDDER_USD);
            vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
            vm.stopPrank();
        }

        // 5. Prepare allocations - all 3 bidders accepted
        BidInfo[] memory allBids = new BidInfo[](3);
        for (uint256 i = 0; i < 3; i++) {
            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD,
                vestingPeriod: 90 days,
                poolId: 0,
                accepted: true
            });
        }

        // 6. Generate merkle roots
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        // 7. Finalize project
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // 8. All three users claim NFTs
        uint256[] memory nftIds = new uint256[](3);
        for (uint256 i = 0; i < 3; i++) {
            vm.prank(bidders[i]);
            nftIds[i] = vesting.claimNFT(
                PROJECT_ID, 
                0, 
                BIDDER_USD, 
                bidderProofs[bidders[i]]
            );
            
            // Verify NFT ownership
            assertEq(nft.ownerOf(nftIds[i]), bidders[i]);
        }
        uint256[] memory percentages = new uint256[](3);
        percentages[0] = 1000;
        percentages[1] = 2000;
        percentages[2] = 7000;

        vm.prank(bidders[0]);
        vesting.splitTVS(PROJECT_ID, percentages, nftIds[0]);
}
```
Run
```
forge test --mt test_SplitTVS
```
the output:
```
[FAIL: panic: array out-of-bounds access (0x32)] test_SplitTVS()
```



### Mitigation

Initialize the `alloc` array
  