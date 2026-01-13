# [000729] Hardcoded Pool ID in claimRefund Breaks Refunds for Multi-Pool Projects
  
  ### Summary

The `claimRefund function` in `AlignerzVesting.sol `contains a logic error where the `poolId` used for Merkle Leaf generation is hardcoded to 0. However, the protocol explicitly supports creating multiple vesting pools per project via `createPool`. This hardcoding prevents users who participated in any pool other than "Pool 0" from claiming refunds, as the on-chain leaf calculation will not match the off-chain Merkle Proof generated for the correct pool ID.

### Root Cause

```solidity
function claimRefund(uint256 projectId, uint256 amount, bytes32[] calldata merkleProof) external {
    BiddingProject storage biddingProject = biddingProjects[projectId];
    // ... checks ...

    // @> poolId is strictly hardcoded to 0, If the user was in Pool 1, the off-chain tree likely used '1' and merkle verification will fail due to mismatch
    uint256 poolId = 0; 
```

### Internal Pre-conditions

1. The Project Owner creates multiple pools (e.g., Pool 0 and Pool 1) using `createPool`.
2. A user places a bid that is assigned to Pool 1 (or any index > 0).
3. The user's bid is rejected, necessitating a refund.

### External Pre-conditions

The off-chain backend generates the Merkle Tree for refunds using the correct `poolId` (e.g., 1) to maintain data integrity and separate refund logic per pool.

### Attack Path

1.  A project is launched with two pools. User A bids in Pool 1.
2. The project ends. User A is not selected for allocation and is owed a refund.
3. The backend generates a Merkle Root for refunds. User A's leaf is `Keccak(UserA, Amount, ProjectID, 1)`.
4.  User A calls `claimRefund(...)` with the valid proof.
5. The contract calculates the leaf as `Keccak(UserA, Amount, ProjectID, 0)`.
6. This hash is different from the one in the Merkle Tree.
7. `MerkleProof.verify` returns false.
8. The transaction reverts, and User A cannot retrieve their funds.

### Impact

*  Users in secondary pools cannot claim refunds 

### PoC

The following test was run and produced these logs: 
```bash
Ran 1 test for test/PoC.t.sol:RefundPoolIdPoC
[PASS] testRefundFailsForPoolOne() (gas: 423580)
Logs:
  npm warn exec The following package was not found and will be installed: @openzeppelin/upgrades-core@1.44.2

  Refund failed as expected due to hardcoded Pool ID 0.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.88s (7.99ms CPU time)

Ran 1 test suite in 4.89s (4.88s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
The test:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

contract RefundPoolIdPoC is Test {
    AlignerzVesting public vesting;
    AlignerzNFT public nft;
    Aligners26 public token;
    MockUSD public usdt;
    
    address public user;
    uint256 constant PROJECT_ID = 0;

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
        vesting.setTreasury(address(1));
        
        // Fund User with Stablecoin (for bidding)
        usdt.mint(user, 2000 ether);
        vm.prank(user);
        usdt.approve(address(vesting), type(uint256).max);

        // --- FIX START: Setup Owner funds for createPool ---
        // The owner (this contract) needs Project Tokens to create a pool.
       
        token.approve(address(vesting), type(uint256).max);
        // --- FIX END ---
    }
    
    function testRefundFailsForPoolOne() public {
        // 1. Launch Project
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp + 1 days,
            block.timestamp + 7 days,
            bytes32(0),
            false
        );

        // Create two pools to justify having Pool IDs > 0
        // Pool 0: Main
        vesting.createPool(PROJECT_ID, 10000 ether, 1 ether, false);
        // Pool 1: Side Pool
        vesting.createPool(PROJECT_ID, 10000 ether, 1 ether, false);

        // Warp to bidding
        vm.warp(block.timestamp + 1 days + 1);

        // 2. User Bids
        uint256 bidAmount = 1000 ether;
        vm.prank(user);
        vesting.placeBid(PROJECT_ID, bidAmount, 30 days);

        // Warp to end
        vm.warp(block.timestamp + 7 days + 1);

        // 3. Finalize Bids (Simulating Backend)
        // Scenario: The user is rejected from Pool 1 and needs a refund.
        // The backend generates the Merkle Leaf using 'poolId = 1' to track this correctly.
        
        uint256 backendPoolId = 1;
        
        // Leaf Construction: Hash(user, amount, project, poolId)
        bytes32 leaf = keccak256(abi.encodePacked(user, bidAmount, PROJECT_ID, backendPoolId));
        
        // For a 1-item tree, Root == Leaf
        bytes32 refundRoot = leaf;
        
        // We pass this root to the contract
        bytes32[] memory poolRoots = new bytes32[](2);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 30 days);

        // 4. Attempt Refund
        vm.startPrank(user);
        
        // Generate proof (empty for single item)
        bytes32[] memory proof = new bytes32[](0);
        
        // The user tries to claim the refund.
        // The contract will calculate the leaf using 'poolId = 0' (Hardcoded).
        // Contract Leaf: Hash(user, amount, project, 0)
        // Merkle Root:   Hash(user, amount, project, 1)
        // Result: Mismatch -> Revert
        
        vm.expectRevert(AlignerzVesting.Invalid_Merkle_Proof.selector);
        vesting.claimRefund(PROJECT_ID, bidAmount, proof);
        
        console.log("Refund failed as expected due to hardcoded Pool ID 0.");
        
        vm.stopPrank();
    }
}
```

### Mitigation

Update `claimRefund` to accept `poolId` as a parameter, allowing the user to specify which pool they are claiming a refund from.

  