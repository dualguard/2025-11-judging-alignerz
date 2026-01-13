# [000526] Uninitialized memory arrays in `_computeSplitArrays` causes Denial of Service in `splitTVS`
  
  ### Summary

The `AlignerzVesting::splitTVS` function is permanently broken due to improper memory management in the helper function `AlignerzVesting::_computeSplitArrays`. The function attempts to write to dynamic arrays within a memory struct without allocating memory for them, causing an immediate out-of-bounds revert on every call.

### Root Cause

In Solidity, declaring a struct containing dynamic arrays in memory (e.g., `Allocation memory alloc`) does not initialize the arrays; they default to a length of 0.
In `AlignerzVesting::_computeSplitArrays`, the code attempts to write to index `j` of these arrays (`alloc.amounts[j]`, `alloc.vestingPeriods[j]`, etc.) without first allocating their size using the `new` keyword.

### Internal Pre-conditions

1. A user must hold a valid Vesting NFT.
2. The `AlignerzVesting::splitTVS` function is called with valid parameters (projectid, percentages and NFT ID).

### External Pre-conditions

None.


### Attack Path

no response

### Impact

The splitting functionality is completely inoperable. Users are unable to split their Token Vesting Shares (TVS) as intended by the protocol design. This results in a broken core feature.

### PoC

1. Place the test in side the `AlignerzVestingProtocolTest.t.sol`
2. Run the code forge clean && forge test --mt test_splitTVS_Reverts_DueToMemoryAllocation -vvvv

```javascript
// New test that demonstrates the memory-allocation (out-of-bounds) bug in splitTVS
// It attempts to split a claimed TVS into two parts and expects the call to revert
// due to writing into uninitialized memory arrays in `_computeSplitArrays`.
contract DemonstrateSplitTVSBug is AlignerzVestingProtocolTest {
    function test_splitTVS_Reverts_DueToMemoryAllocation() public {
        // Setup a minimal project and a single accepted bidder
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        // disable whitelist for this minimal repro to avoid unrelated reverts
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", false);
        vesting.createPool(PROJECT_ID, 1_000 ether, 0.01 ether, true);
        vm.stopPrank();

        // Place bids for bidder[0] and bidder[1] so merkle tree has multiple leaves
        vm.startPrank(bidders[0]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidders[1]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // Prepare merkle roots and finalize with bidder[0] accepted
        BidInfo[] memory onlyBid = new BidInfo[](2);
        onlyBid[0] = BidInfo({ bidder: bidders[0], amount: BIDDER_USD, vestingPeriod: 90 days, poolId: 0, accepted: true });
        onlyBid[1] = BidInfo({ bidder: bidders[1], amount: BIDDER_USD, vestingPeriod: 90 days, poolId: 0, accepted: true });
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(onlyBid, 0);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // Claim NFT for bidder[0]
        vm.prank(bidders[0]);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[0]]);

        // Attempt to split into two TVSs (50% / 50%) and expect a revert due to out-of-bounds writes
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5_000; // 50%
        percentages[1] = 5_000; // 50%

        vm.prank(bidders[0]);
        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentages, nftId);
    }
```
```javascript
    │   ├─ [14199] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return]
```

### Mitigation

```diff
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
        
+       // Allocate memory for dynamic arrays
+       alloc.amounts = new uint256[](nbOfFlows);
+       alloc.vestingPeriods = new uint256[](nbOfFlows);
+       alloc.vestingStartTimes = new uint256[](nbOfFlows);
+       alloc.claimedSeconds = new uint256[](nbOfFlows);
+       alloc.claimedFlows = new bool[](nbOfFlows);

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
  