# [000702] Bid State Not Cleared After claimNFT Leads to Stale Storage and State Inconsistency
  
  ### Summary

In `AlignerzVesting.sol:860-891`, the `claimNFT()` function never clears the user's bid state after successfully minting the NFT and creating the allocation. This will cause state inconsistency and wasted storage for the protocol as users who claim their NFT will have their bid data (`amount` and `vestingPeriod`) remain in storage indefinitely.

### Root Cause


In [AlignerzVesting.sol:860-891](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860-L891), the `claimNFT()` function reads the bid data to verify the user has placed a bid and uses `bid.vestingPeriod` for the allocation, but never deletes the bid struct after successful NFT claim:

```solidity
function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof)
    external
    returns (uint256)
{
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());

    Bid storage bid = biddingProject.bids[msg.sender];
    require(bid.amount > 0, No_Bid_Found());  // Checks bid exists

    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(!claimedNFT[leaf], Already_Claimed());
    require(MerkleProof.verify(merkleProof, biddingProject.vestingPools[poolId].merkleRoot, leaf), Invalid_Merkle_Proof());

    claimedNFT[leaf] = true;

    uint256 nftId = nftContract.mint(msg.sender);
    biddingProject.allocations[nftId].amounts.push(amount);
    biddingProject.allocations[nftId].vestingPeriods.push(bid.vestingPeriod);
    biddingProject.allocations[nftId].vestingStartTimes.push(biddingProject.endTime);
    biddingProject.allocations[nftId].claimedSeconds.push(0);
    biddingProject.allocations[nftId].claimedFlows.push(false);
    biddingProject.allocations[nftId].assignedPoolId = poolId;
    biddingProject.allocations[nftId].token = biddingProject.token;
    NFTBelongsToBiddingProject[nftId] = true;
    allocationOf[nftId] = biddingProject.allocations[nftId];

    emit NFTClaimed(projectId, msg.sender, nftId, poolId, amount);

    return nftId;
    // @audit bid data is NEVER cleared - missing: delete biddingProject.bids[msg.sender];
}
```

### Internal Pre-conditions

1. Owner needs to call `launchBiddingProject()` to create a bidding project
2. Owner needs to call `createPool()` to create at least one vesting pool
3. User needs to call `placeBid()` to place a bid with `amount > 0`
4. Owner needs to call `finalizeBids()` to set the merkle roots and close bidding


### External Pre-conditions

None required.

### Attack Path

This is a vulnerability path rather than an attack path:

1. User calls `placeBid()` with `amount = 10,000 USDC` and `vestingPeriod = 30 days`
2. Bid state is stored: `biddingProject.bids[user].amount = 10,000` and `biddingProject.bids[user].vestingPeriod = 30 days`
3. Owner finalizes bids with valid merkle roots
4. User calls `claimNFT()` with valid merkle proof
5. NFT is minted, allocation is created, `claimedNFT[leaf] = true` is set
6. Bid state remains unchanged: `biddingProject.bids[user].amount` still equals `10,000` and `vestingPeriod` still equals `30 days`
7. 2 storage slots per user are permanently wasted and state is inconsistent

### Impact

The protocol suffers from:
1. Wasted storage: 2 storage slots (64 bytes) per user who claims an NFT are never reclaimed
2. State inconsistency: `bid.amount > 0` check would pass even after NFT is claimed, which could mislead external integrations or off-chain systems querying bid status
3. Code maintainability: Future code that relies on `bid.amount == 0` to check if a user has already claimed will behave incorrectly

Note: There is no direct fund loss risk because the `claimedNFT[leaf]` mapping prevents double-claiming. The merkle proof mechanism serves as the primary protection against exploitation.


### PoC

Save the following test file to `protocol/test/L02_BidStateNotClearedAfterClaimNFT.t.sol`.

Run with:
```bash
forge test --match-test test_L02_BidStateNotClearedAfterClaimNFT -vv
```

### POC Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Merkle} from "murky/src/Merkle.sol";

contract MockERC20 is IERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }
}

contract AlignerzVestingHarness is AlignerzVesting {
    function getBid(uint256 projectId, address bidder) external view returns (uint256 amount, uint256 vestingPeriod) {
        Bid storage bid = biddingProjects[projectId].bids[bidder];
        return (bid.amount, bid.vestingPeriod);
    }
}

contract L02_BidStateNotClearedAfterClaimNFTTest is Test {
    AlignerzVestingHarness public vesting;
    AlignerzVestingHarness public vestingImpl;
    AlignerzNFT public nft;
    MockERC20 public projectToken;
    MockERC20 public stablecoin;
    Merkle public merkle;

    address public owner = address(0x1);
    address public treasury = address(0x2);
    address public user1 = address(0xBEEF);
    address public user2 = address(0xCAFE);

    uint256 public constant PROJECT_ID = 0;
    uint256 public constant POOL_ID = 0;
    uint256 public constant BID_AMOUNT = 10_000e6;
    uint256 public constant TOKEN_ALLOCATION = 100_000e18;
    uint256 public constant TOKEN_PRICE = 1e17;
    uint256 public constant VESTING_PERIOD = 30 days;
    uint256 public constant CLAIM_WINDOW = 7 days;

    function setUp() public {
        vm.startPrank(owner);

        nft = new AlignerzNFT("Alignerz TVS", "ATVS", "https://api.alignerz.com/metadata/");

        vestingImpl = new AlignerzVestingHarness();
        bytes memory initData = abi.encodeWithSelector(AlignerzVesting.initialize.selector, address(nft));
        ERC1967Proxy proxy = new ERC1967Proxy(address(vestingImpl), initData);
        vesting = AlignerzVestingHarness(payable(address(proxy)));

        nft.addMinter(address(vesting));

        vesting.setTreasury(treasury);
        vesting.setVestingPeriodDivisor(1);

        projectToken = new MockERC20("Project Token", "PROJ", 18);
        stablecoin = new MockERC20("USD Coin", "USDC", 6);

        projectToken.mint(owner, 1_000_000e18);
        stablecoin.mint(user1, 100_000e6);
        stablecoin.mint(user2, 100_000e6);

        merkle = new Merkle();

        vm.stopPrank();
    }

    function test_L02_BidStateNotClearedAfterClaimNFT() public {
        vm.startPrank(owner);

        uint256 startTime = block.timestamp;
        uint256 endTime = block.timestamp + 7 days;
        bytes32 endTimeHash = keccak256(abi.encodePacked(endTime));

        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin),
            startTime,
            endTime,
            endTimeHash,
            false
        );

        projectToken.approve(address(vesting), TOKEN_ALLOCATION);
        vesting.createPool(PROJECT_ID, TOKEN_ALLOCATION, TOKEN_PRICE, false);

        vm.stopPrank();

        vm.startPrank(user1);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(user2);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        (uint256 bidAmountBefore, uint256 vestingPeriodBefore) = vesting.getBid(PROJECT_ID, user1);
        assertEq(bidAmountBefore, BID_AMOUNT, "Bid amount should be set before claim");
        assertEq(vestingPeriodBefore, VESTING_PERIOD, "Vesting period should be set before claim");

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(user1, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));
        leaves[1] = keccak256(abi.encodePacked(user2, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));

        bytes32 root = merkle.getRoot(leaves);

        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = root;
        bytes32 refundRoot = bytes32(0);

        vm.startPrank(owner);
        vesting.finalizeBids(PROJECT_ID, refundRoot, merkleRoots, CLAIM_WINDOW);
        vm.stopPrank();

        bytes32[] memory proof = merkle.getProof(leaves, 0);

        vm.startPrank(user1);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, POOL_ID, TOKEN_ALLOCATION / 2, proof);
        vm.stopPrank();

        assertEq(nft.ownerOf(nftId), user1, "NFT should be minted to user1");

        (uint256 bidAmountAfter, uint256 vestingPeriodAfter) = vesting.getBid(PROJECT_ID, user1);

        assertEq(bidAmountAfter, BID_AMOUNT, "VULNERABILITY: Bid amount still exists after NFT claim");
        assertEq(vestingPeriodAfter, VESTING_PERIOD, "VULNERABILITY: Vesting period still exists after NFT claim");

        assertTrue(bidAmountAfter > 0, "Bid state was NOT cleared - this is the vulnerability");
    }

    function test_L02_StaleStateCanCauseConfusion() public {
        vm.startPrank(owner);

        uint256 startTime = block.timestamp;
        uint256 endTime = block.timestamp + 7 days;
        bytes32 endTimeHash = keccak256(abi.encodePacked(endTime));

        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin),
            startTime,
            endTime,
            endTimeHash,
            false
        );

        projectToken.approve(address(vesting), TOKEN_ALLOCATION);
        vesting.createPool(PROJECT_ID, TOKEN_ALLOCATION, TOKEN_PRICE, false);

        vm.stopPrank();

        vm.startPrank(user1);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(user2);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(user1, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));
        leaves[1] = keccak256(abi.encodePacked(user2, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));

        bytes32 root = merkle.getRoot(leaves);

        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = root;
        bytes32 refundRoot = bytes32(0);

        vm.startPrank(owner);
        vesting.finalizeBids(PROJECT_ID, refundRoot, merkleRoots, CLAIM_WINDOW);
        vm.stopPrank();

        bytes32[] memory proof = merkle.getProof(leaves, 0);

        vm.startPrank(user1);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, POOL_ID, TOKEN_ALLOCATION / 2, proof);
        vm.stopPrank();

        (uint256 bidAmount,) = vesting.getBid(PROJECT_ID, user1);
        assertTrue(bidAmount > 0, "Stale bid data exists - could mislead external integrations");

        assertTrue(vesting.claimedNFT(keccak256(abi.encodePacked(user1, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID))));
        assertTrue(nft.ownerOf(nftId) == user1);
    }

    function test_L02_GasWastedOnStaleStorage() public {
        vm.startPrank(owner);

        uint256 startTime = block.timestamp;
        uint256 endTime = block.timestamp + 7 days;
        bytes32 endTimeHash = keccak256(abi.encodePacked(endTime));

        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin),
            startTime,
            endTime,
            endTimeHash,
            false
        );

        projectToken.approve(address(vesting), TOKEN_ALLOCATION);
        vesting.createPool(PROJECT_ID, TOKEN_ALLOCATION, TOKEN_PRICE, false);

        vm.stopPrank();

        vm.startPrank(user1);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(user2);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(user1, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));
        leaves[1] = keccak256(abi.encodePacked(user2, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));

        bytes32 root = merkle.getRoot(leaves);

        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = root;
        bytes32 refundRoot = bytes32(0);

        vm.startPrank(owner);
        vesting.finalizeBids(PROJECT_ID, refundRoot, merkleRoots, CLAIM_WINDOW);
        vm.stopPrank();

        bytes32[] memory proof = merkle.getProof(leaves, 0);

        vm.startPrank(user1);
        vesting.claimNFT(PROJECT_ID, POOL_ID, TOKEN_ALLOCATION / 2, proof);
        vm.stopPrank();

        (uint256 bidAmount, uint256 vestingPeriod) = vesting.getBid(PROJECT_ID, user1);

        uint256 slotsWasted = 0;
        if (bidAmount > 0) slotsWasted++;
        if (vestingPeriod > 0) slotsWasted++;

        assertEq(slotsWasted, 2, "2 storage slots wasted per unclaimed bid");
    }

    function test_L02_MerkleProofPreventsDoubleClaim() public {
        vm.startPrank(owner);

        uint256 startTime = block.timestamp;
        uint256 endTime = block.timestamp + 7 days;
        bytes32 endTimeHash = keccak256(abi.encodePacked(endTime));

        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin),
            startTime,
            endTime,
            endTimeHash,
            false
        );

        projectToken.approve(address(vesting), TOKEN_ALLOCATION);
        vesting.createPool(PROJECT_ID, TOKEN_ALLOCATION, TOKEN_PRICE, false);

        vm.stopPrank();

        vm.startPrank(user1);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(user2);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(user1, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));
        leaves[1] = keccak256(abi.encodePacked(user2, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));

        bytes32 root = merkle.getRoot(leaves);

        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = root;
        bytes32 refundRoot = bytes32(0);

        vm.startPrank(owner);
        vesting.finalizeBids(PROJECT_ID, refundRoot, merkleRoots, CLAIM_WINDOW);
        vm.stopPrank();

        bytes32[] memory proof = merkle.getProof(leaves, 0);

        vm.startPrank(user1);
        vesting.claimNFT(PROJECT_ID, POOL_ID, TOKEN_ALLOCATION / 2, proof);
        vm.stopPrank();

        (uint256 bidAmount,) = vesting.getBid(PROJECT_ID, user1);
        assertTrue(bidAmount > 0, "Bid state still exists - vulnerable code path");

        vm.startPrank(user1);
        vm.expectRevert(AlignerzVesting.Already_Claimed.selector);
        vesting.claimNFT(PROJECT_ID, POOL_ID, TOKEN_ALLOCATION / 2, proof);
        vm.stopPrank();
    }
}

```





### Mitigation


Clear the bid state after successful NFT claim by adding a delete statement:

```solidity
function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof)
    external
    returns (uint256)
{
    BiddingProject storage biddingProject = biddingProjects[projectId];
    require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());

    Bid storage bid = biddingProject.bids[msg.sender];
    require(bid.amount > 0, No_Bid_Found());

    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    require(!claimedNFT[leaf], Already_Claimed());
    require(MerkleProof.verify(merkleProof, biddingProject.vestingPools[poolId].merkleRoot, leaf), Invalid_Merkle_Proof());

    claimedNFT[leaf] = true;

    uint256 nftId = nftContract.mint(msg.sender);
    biddingProject.allocations[nftId].amounts.push(amount);
    biddingProject.allocations[nftId].vestingPeriods.push(bid.vestingPeriod);
    biddingProject.allocations[nftId].vestingStartTimes.push(biddingProject.endTime);
    biddingProject.allocations[nftId].claimedSeconds.push(0);
    biddingProject.allocations[nftId].claimedFlows.push(false);
    biddingProject.allocations[nftId].assignedPoolId = poolId;
    biddingProject.allocations[nftId].token = biddingProject.token;
    NFTBelongsToBiddingProject[nftId] = true;
    allocationOf[nftId] = biddingProject.allocations[nftId];

    // Clear the bid state
    delete biddingProject.bids[msg.sender];

    emit NFTClaimed(projectId, msg.sender, nftId, poolId, amount);

    return nftId;
}
```

This also provides a gas refund for clearing storage slots.

  