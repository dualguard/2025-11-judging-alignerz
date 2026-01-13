# [000602] Direct Access to Dynamic Array Inside Struct From an External Contract Causes Revert Due to Unsupported Access Pattern
  
  ### Summary

In [`A26ZDividendDistributor.sol` lines 140–161](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161), the code directly accesses a struct stored in a mapping that contains dynamic arrays such as `amounts`, `claimedSeconds`, `vestingPeriods`, and `claimedFlows`. However, as documented in this Solidity issue: [https://github.com/argotorg/solidity/issues/12792](https://github.com/argotorg/solidity/issues/12792), dynamic arrays inside a struct cannot be reliably accessed through this pattern. When accessed directly from an external contract, these dynamic arrays do not appear, causing the operation to fail or revert.


### Root Cause


The code attempts to directly access a struct containing dynamic arrays (`amounts`, `claimedSeconds`, `vestingPeriods`, `claimedFlows`) through a mapping (allocationOf variable) from an external contract. This access pattern is unsupported in Solidity, as documented in the GitHub issue: [https://github.com/argotorg/solidity/issues/12792](https://github.com/argotorg/solidity/issues/12792). Because dynamic arrays do not load correctly when accessed this way, the operation reverts.
Affected code: [`A26ZDividendDistributor.sol` lines 140–161](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161).

Snippet Of Root Cause Bug
```solidity
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
        uint256 len = vesting.allocationOf(nftId).amounts.length;
        for (uint i; i < len;) {
            if (claimedFlows[i]) continue;
            if (claimedSeconds[i] == 0) {
                amount += amounts[i];
                continue;
            }
            uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
            uint256 unclaimedAmount = amounts[i] - claimedAmount;
            amount += unclaimedAmount;
            unchecked {
                ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }
```

**Snippet of the `allocationOf` mapping:**
See [`AlignerzVesting.sol` lines 113–114](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L113-L114).

```solidity
/// @title AlignerzVesting - A vesting contract for token sales
/// @notice This contract manages token vesting schedules for multiple projects
/// @author 0xjarix | Alignerz
contract AlignerzVesting is Initializable, UUPSUpgradeable, OwnableUpgradeable, WhitelistManager, FeesManager {
    SNIP
    /// @notice Mapping to fetch the allocation of a TVS given the NFT Id
    mapping(uint256 => Allocation) public allocationOf;
    SNIP
}
```

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

1. The user attempts to set the unclaimable amount in the dividend distributor contract.
2. The transaction reverts because the direct access pattern described in the root cause fails, as the struct does not properly expose its dynamic arrays through this approach.


### Impact

The user may lose access to their dividend because the operation fails and the unclaimable amount cannot be set.

### PoC

Add a new test named `test_DynamicArrayInsideStructAccessBug` to `test/AlignerzVestingProtocolTest.t.sol`.
Then run the following command in the terminal:

```
forge test --mt test_DynamicArrayInsideStructAccessBug -vvv
```


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
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

contract AlignerzVestingProtocolTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public projectCreator;
    address[] public bidders;

    // Constants
    uint256 constant NUM_BIDDERS = 20;
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;

    // Project structure for organization
    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    // Track allocated bids and their proofs
    mapping(address => bytes32[]) public bidderProofs;
    mapping(address => uint256) public bidderPoolIds;
    mapping(address => uint256) public bidderNFTIds;
    bytes32 public refundRoot;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        vm.deal(projectCreator, 100 ether);

        // Deploy contracts
        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);

        // Set NFT minter to vesting contract
        vm.prank(owner);
        nft.addMinter(proxy);
        vesting.setTreasury(address(1));

        // Create bidders with ETH and USDT
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            address bidder = makeAddr(string.concat("bidder", vm.toString(i)));
            vm.deal(bidder, 50 ether);
            bidders.push(bidder);
            usdt.mint(bidder, BIDDER_USD);
        }

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    // Helper function to create a leaf node for the merkle tree
    function getLeaf(address bidder, uint256 amount, uint256 projectId, uint256 poolId)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(bidder, amount, projectId, poolId));
    }

    // Helper for generating merkle proofs
    function generateMerkleProofs(BidInfo[] memory bids, uint256 poolId) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);

                bidderPoolIds[bids[i].bidder] = poolId;
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }

    // Helper for generating refund proofs
    function generateRefundProofs(BidInfo[] memory bids) internal returns (bytes32) {
        bytes32[] memory leaves = new bytes32[](bids.length);
        uint256 leafCount = 0;
        uint256 poolId = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                leaves[leafCount] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);

                bidderPoolIds[bids[i].bidder] = poolId;
                leafCount++;
            }
        }

        CompleteMerkle m = new CompleteMerkle();
        bytes32 root = m.getRoot(leaves);
        uint256 indexTracker = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (!bids[i].accepted) {
                bytes32[] memory proof = m.getProof(leaves, indexTracker);
                bidderProofs[bids[i].bidder] = proof;
                indexTracker++;
            }
        }

        return root;
    }
    function test_DynamicArrayInsideStructAccessBug() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        // 3. Place bids from different whitelisted users
        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days;

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
            vm.stopPrank();
        }

        // 4. Update some bids
        vm.prank(bidders[0]);
        vesting.updateBid(PROJECT_ID, BIDDER_USD, 180 days);

        // 5. Prepare bid allocations (this would be done off-chain)
        BidInfo[] memory allBids = new BidInfo[](NUM_BIDDERS);

        // Simulate off-chain allocation process
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            // Assign bids to pools (in reality, this would be based on some algorithm)
            uint256 poolId = i % 3;
            bool accepted = i < 15; // First 15 bidders are accepted

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD, // For simplicity, all bids are the same amount
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days,
                poolId: poolId,
                accepted: accepted
            });
        }

        // 6. Generate merkle roots for each pool
        bytes32[] memory poolRoots = new bytes32[](3);
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
        }

        // 7. Generate refund proofs
        refundRoot = generateRefundProofs(allBids);

        // 8. Finalize project with merkle roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);
        uint256[] memory nftIds = new uint256[](15);
        // 9. Users claim NFTs with proofs
        for (uint256 i = 0; i < 15; i++) {
            // Only accepted bidders
            address bidder = bidders[i];
            uint256 poolId = bidderPoolIds[bidder];

            vm.prank(bidder);
            uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
            nftIds[i] = nftId;
            vm.prank(bidder);
            //(uint256 oldNft, uint256 newNft) = vesting.splitTVS(PROJECT_ID, nftId, 5000);
            //assertEq(oldNft, nftId);
            bidderNFTIds[bidder] = nftId;

            // Verify NFT ownership
            assertEq(nft.ownerOf(nftId), bidder);
        }

        // 10. Some users try to claim refunds
        for (uint256 i = 15; i < NUM_BIDDERS; i++) {
            // Only Refunded bidders
            address bidder = bidders[i];
            vm.prank(bidder);
            vesting.claimRefund(PROJECT_ID, BIDDER_USD, bidderProofs[bidder]);

            // Verify USDT was returned
            assertEq(usdt.balanceOf(bidders[i]), BIDDER_USD);
        }

        vm.prank(projectCreator);
        A26ZDividendDistributor DividendDistributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), block.timestamp, block.timestamp + 365 days, address(token));
        
        deal(address(usdt), projectCreator, 15e18);

        vm.startPrank(projectCreator);
        usdt.transfer(address(DividendDistributor), 15e18);
        vm.stopPrank();

        address bidder = bidders[1];
        uint256 nftId = nftIds[1];


        // user will get reverted setting the unclaimed amount on dividend contract
        vm.prank(bidder);
        DividendDistributor.getUnclaimedAmounts(nftId);
        
        // 11. Fast forward time to simulate vesting period
        vm.warp(block.timestamp + 365 days);


        // 12. Users claim tokens after vesting period
        for (uint256 i = 0; i < 15; i++) {
            address bidder = bidders[i];
            //uint256 nftId = bidderNFTIds[bidder];

            uint256 tokenBalanceBefore = token.balanceOf(bidder);

            // console.log(tokenBalanceBefore);

            vm.prank(bidder);
            vesting.claimTokens(PROJECT_ID, nftIds[i]);

            uint256 tokenBalanceAfter = token.balanceOf(bidder);

            // console.log(tokenBalanceAfter);
            assertTrue(tokenBalanceAfter > tokenBalanceBefore, "No tokens claimed");
        }

    }

}
```

Here is the cleaned-up version:

---

The results show that the dynamic arrays do not appear when accessed using this approach. Even other fields, such as `token`, are missing due to this unsupported access pattern. This behavior is consistent with the issue described in the Solidity reference: [https://github.com/argotorg/solidity/issues/12792](https://github.com/argotorg/solidity/issues/12792).

```bash
SNIP
    ├─ [5823] A26ZDividendDistributor::getUnclaimedAmounts(2)
    │   ├─ [3648] ERC1967Proxy::fallback(2) [staticcall]
    │   │   ├─ [2911] AlignerzVesting::allocationOf(2) [delegatecall]
    │   │   │   └─ ← [Return] false, Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 1
    │   │   └─ ← [Return] false, Aligners26: [0x2e234DAe75C793f67A35089C9d99245E1C58470b], 1
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert

Backtrace:
  at A26ZDividendDistributor.getUnclaimedAmounts
  at AlignerzVestingProtocolTest.test_DynamicArrayInsideStructAccessBug

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 5.16s (33.88ms CPU time)

Ran 1 test suite in 5.50s (5.16s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: EvmError: Revert] test_DynamicArrayInsideStructAccessBug() (gas: 22138564)
```


### Mitigation

Based on the reference [https://github.com/argotorg/solidity/issues/12792](https://github.com/argotorg/solidity/issues/12792), the recommended mitigation is to create a dedicated getter function that explicitly returns the struct, as shown in the snippet below.

```solidity
    function getAllocation(uint256 nftId) external view returns (Allocation memory) {
        return allocationOf[nftId];
    }
```
  