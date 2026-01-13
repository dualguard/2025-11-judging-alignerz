# [000713] Splitting and merging TVSs always revert, disabling core features
  
  

## summary
Calls to splitTVS and mergeTVS deterministically revert for any non-empty TVS due to two independent bugs: writing to unallocated dynamic arrays and a non-terminating loop. This fully DoSes portfolio management of Token Vesting Schedules.

## Finding Description
The vesting system exposes user-facing TVS management via mergeTVS and splitTVS in AlignerzVesting. Both paths immediately hit broken helper logic.

1) Fee helper is fundamentally broken
- In [`calculateFeeAndNewAmountForOneTVS()` (`protocol/src/contracts/vesting/feesManager/FeesManager.sol:169`)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), the function returns (feeAmount, newAmounts) but never allocates the newAmounts array before writing indexed entries. Attempting newAmounts[i] = ...for any length > 0` performs an out-of-bounds memory write and reverts at the first iteration.
- The loop uses for (uint256 i; i < length;) { ... } without incrementing `i`. Even if newAmounts were correctly allocated, `i` remains `0` and the loop never terminates.

- Root cause 1: Unallocated memory writes + non-terminating loop.
  - In[ `calculateFeeAndNewAmountForOneTVS()` (`protocol/src/contracts/vesting/feesManager/FeesManager.sol:169`)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), newAmounts is returned but never allocated, then written via newAmounts[i], which is out-of-bounds in memory and reverts. The for loop uses for (uint256 i; i < length;) and does not increment `i`, so even if the array were allocated the loop never terminates.

[affected function code](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174)
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

This helper is called unconditionally by both TVS operations:
- mergeTVS pre-processing (`protocol/src/contracts/vesting/AlignerzVesting.sol:1002`):
  calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows)` → immediate revert on non-empty TVS.
- `splitTVS()` pre-processing (`protocol/src/contracts/vesting/AlignerzVesting.sol:1054`):
  `calculateFeeAndNewAmountForOneTVS(splitFeeRate, amounts, nbOfFlows)` → same failure.

2) Split array construction writes into unallocated dynamic arrays
- In[ `_computeSplitArrays()` (`protocol/src/contracts/vesting/AlignerzVesting.sol:1113`)](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141), the returned Allocation memory alloc contains dynamic arrays (amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows) that are never allocated before indexed writes. The first assignment alloc.amounts[0] = ... for any nbOfFlows > 0 is an out-of-bounds memory write and reverts.

[affected function code](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141)
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
        unchecked { ++j; }
    }
}
```

Highest-impact scenario
- Any user with a valid TVS NFT attempts to manage their position using splitTVS or mergeTVS. Because TVSs contain at least one flow, both paths hit calculateFeeAndNewAmountForOneTVS and revert immediately. Even after fixing that helper, splitTVS still reverts in _computeSplitArrays due to unallocated dynamic arrays. As a result, splitting and merging are fully unusable across the protocol.  
- Any TVS NFT minted via claimNFT creates at least one flow (`protocol/src/contracts/vesting/AlignerzVesting.sol:878–886`). When the holder calls splitTVS or mergeTVS, execution enters the broken helpers and reverts immediately. This fully disables both features across all projects.

- Confirmed use sites:
  - mergeTVS entry path: protocol/src/contracts/vesting/AlignerzVesting.sol:1002–1026
  - splitTVS entry path: protocol/src/contracts/vesting/AlignerzVesting.sol:1054–1107

**Attack Path**

- Preconditions: A user owns a valid TVS NFT with at least one flow (normal case after claimNFT()).
- Steps:
  - Call splitTVS(projectId, [5000, 5000], nftId). Reverts during fee pre-processing because newAmounts[i] writes to unallocated memory and the loop does not increment.
  - Call mergeTVS(projectId, mergedNftId, [projectId], [nftId]). Reverts during fee pre-processing for the same reason.
- Highest-impact scenario:
  - All TVS splits and merges are unusable protocol-wide; users cannot manage or reshape vesting schedules, blocking dependent flows and operational flexibility


## Impact
High. The protocol’s advertised TVS splitting and merging are completely disabled under normal conditions, preventing position management and blocking downstream flows that depend on re-shaping TVSs.
## poc 


Commands Run

- Build:
  - forge clean
  - forge build
- Targeted test:
  - forge test --match-test test_PoC_SplitAndMergeRevert_Simple -vvv

```solidity
    function test_PoC_SplitAndMergeRevert() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUserToWhitelist(bidders[0], PROJECT_ID);
        vesting.addUserToWhitelist(bidders[1], PROJECT_ID);
        vm.stopPrank();

        vm.startPrank(bidders[0]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidders[1]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        BidInfo[] memory bids = new BidInfo[](2);
        bids[0] = BidInfo({ bidder: bidders[0], amount: BIDDER_USD, vestingPeriod: 90 days, poolId: 0, accepted: true });
        bids[1] = BidInfo({ bidder: bidders[1], amount: BIDDER_USD, vestingPeriod: 90 days, poolId: 0, accepted: true });
        bytes32 root = generateMerkleProofs(bids, 0);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        vm.prank(bidders[0]);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[0]]);

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;

        vm.prank(bidders[0]);
        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentages, nftId);

        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftId;

        vm.prank(bidders[0]);
        vm.expectRevert();
        vesting.mergeTVS(PROJECT_ID, nftId, projectIds, nftIds);
    }
}

contract AlignerzVestingPoCTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address[] public bidders;

    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    function setUp() public {
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1));

        address bidder0 = makeAddr("poc_bidder0");
        address bidder1 = makeAddr("poc_bidder1");
        bidders.push(bidder0);
        bidders.push(bidder1);
        vm.deal(bidder0, 50 ether);
        vm.deal(bidder1, 50 ether);
        usdt.mint(bidder0, BIDDER_USD);
        usdt.mint(bidder1, BIDDER_USD);

        vm.prank(address(this));
        token.approve(address(vesting), 3_000_000 ether);

        vm.prank(address(this));
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        vm.prank(address(this));
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        vm.prank(address(this));
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);

        vm.startPrank(bidder0);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidder1);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();
    }

    function test_PoC_SplitAndMergeRevert_Simple() public {
        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(bidders[0], BIDDER_USD, PROJECT_ID, uint256(0)));
        leaves[1] = keccak256(abi.encodePacked(bidders[1], BIDDER_USD, PROJECT_ID, uint256(0)));
        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        bytes32[] memory proof0 = m.getProof(leaves, 0);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;
        vm.prank(address(this));
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        vm.prank(bidders[0]);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, proof0);

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;
        vm.prank(bidders[0]);
        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentages, nftId);

        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftId;
        vm.prank(bidders[0]);
        vm.expectRevert();
        vesting.mergeTVS(PROJECT_ID, nftId, projectIds, nftIds);
    }
```
## Recommendation
Allocate dynamic arrays and fix loop progression in auxiliary logic; confirm fee semantics (per-flow vs cumulative) and ensure helpers terminate and return correctly.
  