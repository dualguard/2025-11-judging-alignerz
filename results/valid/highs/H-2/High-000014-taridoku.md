# [000014] Uninitialized array in `calculateFeeAndNewAmountForOneTVS` bricks merge/split
  
  ### Summary

`calculateFeeAndNewAmountForOneTVS` never allocates the `newAmounts` array before writing to it. Any call with a non-zero length will revert due to an out-of-bounds memory write. Because this helper is used by TVS merge/split functionality, those user facing features are effectively unusable.

### Root Cause

In [`FeesManager.calculateFeeAndNewAmountForOneTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), `newAmounts` is declared but never initialized with `new uint256[](length)`. This means `newAmounts` points to memory location `0x00`. Thus, any write to `newAmounts[i] for length > 0` is an out-of-bounds memory write and causes a panic revert.

### Internal Pre-conditions

1. `calculateFeeAndNewAmountForOneTVS` is called with `length > 0`
2. The calling path passes a non-empty amounts array

### External Pre-conditions

Nil

### Attack Path

1. User prepares an array with at least one flow:

### Impact

Impact level: High

All functionality that relies on this helper ie TVS merge/split flows is completely disabled. This is a severe disruption of a protocol feature.

Likelihood: High
The bug triggers deterministically whenever merge/split is used with at least one flow. There is no special setup.

### PoC

Add this test to `test/AlignerzVestingProtocolTest.t.sol`
```solidity
function test_mergeTVS_reverts_dueTo_uninitializedFeeHelper() public {
        // Use one bidder from the existing array
        address bidder = bidders[0];

        // === 1. Project + pool setup ===
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // launch bidding project with no whitelist (simpler)
        uint256 projectId = 0;
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 10 days,
            bytes32(0),
            false // whitelistStatus
        );

        // single pool
        vesting.createPool(
            projectId,
            1_000_000 ether,
            1e18,   // dummy price
            false   // hasExtraRefund
        );
        vm.stopPrank();

        // === 2. Bidder places a bid that will be accepted ===
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        uint256 vestingPeriodBid = 30 days;
        vesting.placeBid(projectId, BIDDER_USD, vestingPeriodBid);
        vm.stopPrank();

        // === 3. Finalize with a merkle root that accepts this bidder ===
        // We cheat: root == leaf, proof == empty array
        bytes32 leaf = getLeaf(bidder, BIDDER_USD, projectId, 0);
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = leaf;

        vm.prank(projectCreator);
        vesting.finalizeBids(projectId, bytes32(0), poolRoots, 7 days);

        // === 4. Bidder claims NFT; this creates a non empty allocation (one flow) ===
        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(projectId, 0, BIDDER_USD, new bytes32[](0));
        assertEq(nft.ownerOf(nftId), bidder, "pre: bidder should own merge target NFT");

        // === 5. Call mergeTVS and hit the bug in calculateFeeAndNewAmountForOneTVS ===
        // We can pass empty arrays for projectIds/nftIds because the helper is called
        // BEFORE the require(nbOfNFTs > 0) check.
        uint256[] memory projectIds = new uint256[](0);
        uint256[] memory nftIds = new uint256[](0);

        vm.startPrank(bidder);
        vm.expectRevert(); // out-of-bounds write to uninitialized newAmounts
        vesting.mergeTVS(projectId, nftId, projectIds, nftIds);
        vm.stopPrank();
    }
```
This panic reverts with out of bounds 

### Mitigation

Allocate the array before use:
```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
)
    public
    pure
    returns (uint256 feeAmount, uint256[] memory newAmounts)
{
    newAmounts = new uint256[](length);

    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount; // see Finding 1b for logic fix
        unchecked { ++i; }
    }
}
```
  