# [000441] Splitting TVS will result in loss of funds for the user
  
  ### Summary



Users can split a single TVS into multiple TVSs by calling `AlignerzVesting::splitTVS` using the percentages they provide. However, due to a flaw in the `splitTVS` function, the splitting process results in a loss of tokens for the user.


### Root Cause


The `AlignerzVesting::splitTVS` function takes `uint256 projectId`, `uint256[] calldata percentages`, and `uint256 splitNftId`.

Consider the following example:

* `splitNftId = 10`
* `percentages = [5000, 5000]` (i.e., 50% and 50%)
* `NftId 10` has an allocated amount of `1,000,000`

In the expected behavior (assuming the fee is 0), the result should be two NFTs:

1. The original `splitNftId`
2. A newly minted NFT

Each of these NFTs should then have an allocation of `500,000`.
However, due to a flaw in the implementation, this is **not** what happens.

The problematic line appears in the following snippet:

```solidity
(Allocation storage allocation, IERC20 token) = isBiddingProject ?
    (biddingProjects[projectId].allocations[splitNftId], biddingProjects[projectId].token) :
    (rewardProjects[projectId].allocations[splitNftId], rewardProjects[projectId].token);
```

Here, `allocation` is a **storage reference** to the existing allocation of the `splitNftId`.
Because it is a storage pointer, any mutation performed during the split operation affects the original allocation directly.

Later, the function computes the split using:

```solidity
Allocation memory alloc = _computeSplitArrays(allocation, percentage, nbOfFlows);
```

Inside `_computeSplitArrays`, values are read from the `allocation` reference.
But since the original storage allocation is being modified during the splitting loop—due to writes happening via `_assignAllocation`—the `allocation` pointer no longer reflects the original base values.
Instead, it now points to already updated (partially split) values.






### Internal Pre-conditions

User needs to call `splitTVS()`

### External Pre-conditions

None

### Attack Path



The issue occurs when an NFT with a large allocation is split using multiple percentages. For example, consider `NftId 10` with an allocation amount of `1,000,000`, and a split request using percentages `[5000, 5000]`. The contract uses the following line:

```solidity
Allocation storage allocation = ... allocations[splitNftId];
```

This creates a **storage reference**, meaning `allocation` always points directly to the live storage values of `NftId 10`.

During the first split, the contract calculates the 50% split correctly:

```
1,000,000 * 5000 / 10000 = 500,000
```

The original NFT is then updated:

```
NftId 10.amount = 500,000
```

Because `allocation` is a storage reference, it now also reflects:

```
allocation.amount = 500,000
```

Instead of keeping access to the original `1,000,000`, it now points to the updated value `500,000`. This is the root of the problem.

When the second percentage split is processed, the contract incorrectly uses this updated value as the base. It computes the next 50% split as:

```
500,000 * 5000 / 10000 = 250,000
```

So the newly minted NFT receives:

```
newNft.amount = 250,000   // incorrect – should have been 500,000
```

The final allocations become:

* Original NFT: `500,000`
* New NFT: `250,000`

Total:

```
500,000 + 250,000 = 750,000   // 250,000 missing
```




### Impact


The user will lose tokens corresponding to the percentages provided to the `splitTVS()` function.


### PoC


To run the PoC without any errors, implement the following changes:

1.

Implement the function below in `AlignerzVesting.sol` to easily retrieve the total allocation of a given `nftId`:

```solidity
function getTotalAllocation(
    uint256 projectId,
    uint256 nftId
) external view returns (uint256) {
    BiddingProject storage biddingProject = biddingProjects[projectId];
    Allocation storage allocation = biddingProject.allocations[nftId];
    uint256 totalAllocation;
    for (uint256 i; i < allocation.amounts.length; i++) {
        totalAllocation += allocation.amounts[i];
    }
    return totalAllocation;
}
```

2.

Replace the `_computeSplitArrays` function with the version below to avoid `array out of bounds` errors:

```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    uint256[] memory baseAmounts = allocation.amounts;
    uint256[] memory baseVestings = allocation.vestingPeriods;
    uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
    uint256[] memory baseClaimed = allocation.claimedSeconds;
    bool[] memory baseClaimedFlows = allocation.claimedFlows;
    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;
    // Allocate arrays before assigning values
    alloc.amounts = new uint256[](nbOfFlows); //@audit added line to allocate amounts array
    alloc.vestingPeriods = new uint256[](nbOfFlows); //@audit added line to allocate vestingPeriods array
    alloc.vestingStartTimes = new uint256[](nbOfFlows); //@audit added line to allocate vestingStartTimes array
    alloc.claimedSeconds = new uint256[](nbOfFlows); //@audit added line to allocate claimedSeconds array
    alloc.claimedFlows = new bool[](nbOfFlows); //@audit added line to allocate claimedFlows array
    for (uint256 j; j < nbOfFlows; ) {
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

3.

Replace the `calculateFeeAndNewAmountForOneTVS` function in `FeesManager.sol` with the version below:

```solidity
function calculateFeeAndNewAmountForOneTVS(
    uint256 feeRate,
    uint256[] memory amounts,
    uint256 length
) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length); //@audit this is extra line
    for (uint256 i; i < length; i++) {
        //@audit here it also is extra line i++
        feeAmount += calculateFeeAmount(feeRate, amounts[i]); 
        newAmounts[i] = amounts[i] - feeAmount;
    }
}
```

4.

Create a file named `SplitTVSTest.t.sol` inside the `test` folder and paste the following code:

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

contract SplitTVSTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public projectCreator;
    address public bidder;

    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    mapping(address => bytes32[]) public bidderProofs;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        vm.deal(projectCreator, 100 ether);

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT(
            "AlignerzNFT",
            "AZNFT",
            "https://nft.alignerz.bid/"
        );

        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
            )
        );
        vesting = AlignerzVesting(proxy);

        vm.prank(owner);
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Single bidder
        bidder = makeAddr("bidder");
        vm.deal(bidder, 50 ether);
        usdt.mint(bidder, BIDDER_USD);

        // Fund project creator with tokens and transfer ownership of vesting
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    function getLeaf(
        address _bidder,
        uint256 amount,
        uint256 projectId,
        uint256 poolId
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(_bidder, amount, projectId, poolId));
    }

    function generateMerkleProofs(
        BidInfo[] memory bids,
        uint256 poolId
    ) internal returns (bytes32) {
        // First collect accepted leaves
        uint256 acceptedCount = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) acceptedCount++;
        }

        // Handle single-leaf edge case: CompleteMerkle requires >1 leaves
        bytes32[] memory leaves;
        if (acceptedCount == 1) {
            leaves = new bytes32;
        } else {
            leaves = new bytes32[](acceptedCount);
        }

        uint256 leafCount = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[leafCount] = getLeaf(
                    bids[i].bidder,
                    bids[i].amount,
                    PROJECT_ID,
                    poolId
                );
                // if only one accepted, duplicate to make length 2
                if (acceptedCount == 1) {
                    leaves[1] = leaves[leafCount];
                }
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                // if acceptedCount==1, our actual index is 0 (the duplicated leaf proof will work)
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }

    function test_SplitTVS_Succeeds() public {
        // projectCreator launches project and creates a pool
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.addUsersToWhitelist(toArray(bidder), PROJECT_ID);
        vm.stopPrank();

        // bidder places a bid
        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // prepare allocation off-chain -> accept the single bidder
        BidInfo;
        allBids[0] = BidInfo({
            bidder: bidder,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        // generate merkle root and finalize
        bytes32;
        poolRoots[0] = generateMerkleProofs(allBids, 0);
        bytes32 refundRoot = bytes32(0);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);

        // claim NFT
        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(
            PROJECT_ID,
            0,
            BIDDER_USD,
            bidderProofs[bidder]
        );

        // split the NFT into two equal parts (5000 + 5000 = 10000 BASIS_POINT)
        uint256;
        percentages[0] = 5_000;
        percentages[1] = 5_000;
        uint256 totalAllocationBeforesplit = vesting.getTotalAllocation(
            PROJECT_ID,
            nftId
        );
        console.log(
            "Total allocation before split:",
            totalAllocationBeforesplit
        );
        vm.prank(bidder);
        (uint256 returnedId, uint256[] memory newNftIds) = vesting.splitTVS(
            PROJECT_ID,
            percentages,
            nftId
        );
        uint256 totalAllocationAfterSplit1 = vesting.getTotalAllocation(
            PROJECT_ID,
            newNftIds[0]
        );
        uint256 totalAllocationAfterSplit2 = vesting.getTotalAllocation(
            PROJECT_ID,
            nftId
        );
        console.log(
            "Total allocation after split NFT 1:",
            totalAllocationAfterSplit1
        );
        console.log(
            "Total allocation after split NFT 2:",
            totalAllocationAfterSplit2
        );
        console.log(
            "Sum of allocations after split:",
            totalAllocationAfterSplit1 + totalAllocationAfterSplit2
        );

        // assertions
        assertEq(returnedId, nftId);
        assertEq(newNftIds.length, 1);
        // owner of newly minted splitted NFT should be the bidder
        assertEq(nft.ownerOf(newNftIds[0]), bidder);
        // original NFT should still exist and be owned by bidder
        assertEq(nft.ownerOf(nftId), bidder);

        console.log(
            "Difference in allocation after split:",
            totalAllocationBeforesplit -
                (totalAllocationAfterSplit1 + totalAllocationAfterSplit2)
        );
    }

    // small helper to convert a single address to array
    function toArray(address a) internal pure returns (address[] memory arr) {
        arr = new address;
        arr[0] = a;
    }
}
```

To run the test:

```shell
forge clean
forge test --match-test test_SplitTVS_Succeeds -vv
```

**Logs:**

```shell
Ran 1 test for test/SplitTVSTest.t.sol:SplitTVSTest
[PASS] test_SplitTVS_Succeeds() (gas: 2693315)
Logs:
  Total allocation before split: 1000000000000000000000
  Total allocation after split NFT 1: 250000000000000000000
  Total allocation after split NFT 2: 500000000000000000000
  Sum of allocations after split: 750000000000000000000
  Difference in allocation after split: 250000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.78s (1.87ms CPU time)

Ran 1 test suite in 2.78s (2.78s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


### Mitigation


To prevent token loss during TVS splitting, the contract must ensure that all percentage calculations are performed using the *original* allocation values, not values mutated during the split process. This can be achieved by avoiding the use of a **storage reference** when reading the base allocation.

Instead of:

```solidity
Allocation storage allocation = ... allocations[splitNftId];
```

the contract should create a **memory copy** of the allocation before applying any modifications:

```solidity
Allocation memory baseAllocation = ... allocations[splitNftId];
```

All split calculations should then use `baseAllocation` as the immutable source of truth. 

  