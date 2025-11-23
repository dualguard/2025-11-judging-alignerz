# [000920] [H-6] No Refund for Winners in "Extra Refund" Pools
  
  ### Summary

Pools created with hasExtraRefund = true are intended to refund winners’ bids. However, claimNFT() ignores this flag, so winners receive the NFT but no refund. The owner can then withdraw the full bid amount via withdrawPostDeadlineProfit(), resulting in total loss for winning users.

### Root Cause

The claimNFT() function does not check the hasExtraRefund flag in the pool. As a result, winning users are never refunded their bid, and the full bid amount remains in the contract, allowing the owner to withdraw it later.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860-L891

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path



1. **Pool Creation:**

   * The owner creates a pool via `createPool()` with `hasExtraRefund = true`.
   * This signals that winning bidders should receive their bid back after claiming their NFT.

2. **User Bids:**

   * A user (e.g., Alice) places a bid using `placeBid()` and locks their stablecoins in the contract.

3. **Finalize Bids:**

   * The owner calls `finalizeBids()` to close the bidding period and set merkle roots for allocation verification.

4. **NFT Claim:**

   * The winner calls `claimNFT()` to claim their NFT.
   * **Bug:** The function mints the NFT but **does not refund the bid**, ignoring the `hasExtraRefund` flag.

5. **Owner Withdraws Funds:**

   * After the claim deadline, the owner can call `withdrawPostDeadlineProfit()` or `_withdrawPostDeadlineProfit()` to withdraw the total stablecoin balance of the pool.
   * Because the winner never received a refund, the owner effectively takes the user's bid.

6. **Result:**

   * Winning users lose their entire bid.
   * Owner receives all bid funds as profit.


### Impact


1. Users in “extra refund” pools lose their full bid amount even after winning.

2. The contract owner can later withdraw these funds  direct theft of user funds.

3. This is a critical severity issue, as it breaks the intended economic guarantees of the vesting system.




### PoC


1. Create a AlignerzVesting.t.sol file inside Test folder .
2.  Run the test using:

```solidity
forge test --match-test test_ExtraRefundNotSentToWinners --match-path test/AlignerzVesting.t.sol -vvvv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/vesting/AlignerzVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";



contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {
        _mint(msg.sender, 1000000e18);
    }
}

contract MockNFT {
    uint256 public nextId = 1;
    mapping(uint256 => address) public ownerOf;

    function mint(address to) external returns (uint256) {
        uint256 id = nextId++;
        ownerOf[id] = to;
        return id;
    }
}

contract ExtraRefundPoC is Test {
    AlignerzVesting vesting;
    MockERC20 projectToken;
    MockERC20 stablecoin;
    MockNFT nft;
    
    address alice = makeAddr("alice");
    address treasury = makeAddr("treasury");
    address owner;
    
    uint256 constant PROJECT_ID = 0;
    uint256 constant POOL_ID = 0;
    uint256 constant BID_AMOUNT = 100e18;

    function setUp() public {
        owner = address(this);
        
        projectToken = new MockERC20("Project Token", "PROJ");
        stablecoin = new MockERC20("Stablecoin", "STABLE");
        nft = new MockNFT();
        
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft)); 
        
        vesting.setTreasury(treasury);
        
        stablecoin.transfer(alice, 1000e18);
        
        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin), 
            block.timestamp,
            block.timestamp + 1 days,
            bytes32(0),
            false
        );

        projectToken.approve(address(vesting), 1000e18);
        vesting.createPool(PROJECT_ID, 1000e18, 1e18, true); 

        vm.prank(alice);
        stablecoin.approve(address(vesting), BID_AMOUNT);
    }

    function test_ExtraRefundNotSentToWinners() public {
        
        vm.prank(alice);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, 30 days);
        
        uint256 aliceInitialStable = stablecoin.balanceOf(alice);
        uint256 contractInitialStable = stablecoin.balanceOf(address(vesting));
        uint256 treasuryInitialStable = stablecoin.balanceOf(treasury);
        
        console.log("Alice initial stable balance:", aliceInitialStable / 1e18);
        console.log("Contract initial stable balance:", contractInitialStable / 1e18);
        console.log("Treasury initial stable balance:", treasuryInitialStable / 1e18);

        bytes32 aliceLeaf = keccak256(abi.encodePacked(alice, BID_AMOUNT, PROJECT_ID, POOL_ID));
        bytes32[] memory roots = new bytes32[](1);
        roots[0] = aliceLeaf;
        vesting.finalizeBids(PROJECT_ID, bytes32(0), roots, 1 days);

        bytes32[] memory emptyProof = new bytes32[](0);
        
        uint256 aliceBalanceBeforeClaim = stablecoin.balanceOf(alice);
        vm.prank(alice);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, POOL_ID, BID_AMOUNT, emptyProof);
        uint256 aliceBalanceAfterClaim = stablecoin.balanceOf(alice);
        
        console.log("Alice claimed NFT #", nftId);

        uint256 contractFinalStable = stablecoin.balanceOf(address(vesting));
        console.log("Alice balance before claim:", aliceBalanceBeforeClaim / 1e18);
        console.log("Alice balance after claim:", aliceBalanceAfterClaim / 1e18);
        console.log("Contract final stable balance:", contractFinalStable / 1e18);

        uint256 expectedRefund = BID_AMOUNT;
        uint256 actualRefund = aliceBalanceAfterClaim - aliceBalanceBeforeClaim;
        
        console.log("Alice should have received refund:", expectedRefund / 1e18);
        console.log("Alice actually received refund:", actualRefund / 1e18);
        console.log("Missing refund:", (expectedRefund - actualRefund) / 1e18);

        vm.warp(block.timestamp + 2 days);

        uint256 treasuryBalanceBefore = stablecoin.balanceOf(treasury);
        vesting.withdrawPostDeadlineProfit(PROJECT_ID);
        uint256 treasuryBalanceAfter = stablecoin.balanceOf(treasury);
        uint256 treasuryProfit = treasuryBalanceAfter - treasuryBalanceBefore;

        console.log("Treasury received:", treasuryProfit / 1e18, "stablecoins as 'profit'");

        assertEq(actualRefund, 0, "Alice received 0 refund instead of her bid amount");
        assertEq(treasuryProfit, BID_AMOUNT, "Treasury got exactly Alice's bid amount");
        assertEq(aliceBalanceAfterClaim, aliceBalanceBeforeClaim, "Alice's balance didn't change - no refund");       
    }

}
```

### Mitigation

To address the issue, the claimNFT() function should be updated to honor the hasExtraRefund flag. Specifically, when a pool is created with hasExtraRefund = true, the contract should automatically refund the bidder’s amount upon claiming their NFT. This can be implemented by checking the flag within claimNFT() and transferring the bid back to the user, while also decrementing the project’s totalStablecoinBalance accordingly. Additionally, it is recommended to emit a dedicated event to track these refunds, ensuring transparency and easier auditing. Comprehensive unit tests should be implemented to confirm that winners in extra-refund pools always receive their bids back. Finally, for clarity and to avoid developer confusion, consider renaming the hasExtraRefund flag to a more descriptive name such as refundBidOnClaim.
  