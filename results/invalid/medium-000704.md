# [000704] Missing Project ID Validation in claimTokens Burns NFT With Zero Tokens
  
  ### Summary

Missing validation that an NFT belongs to the specified `projectId` in `claimTokens()` will cause permanent loss of vesting tokens for users as providing an incorrect `projectId` results in the NFT being burned with zero tokens transferred.

### Root Cause


In [AlignerzVesting.sol:941-975](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975) there is no validation that the `nftId` actually belongs to the specified `projectId`. The function accepts both parameters independently but only validates NFT ownership, not the project association.

```solidity
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    bool isBiddingProject = NFTBelongsToBiddingProject[nftId];
    // @audit No validation that nftId belongs to projectId
    (Allocation storage allocation, IERC20 token) = isBiddingProject ?
    (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token) :
    (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
    uint256 nbOfFlows = allocation.vestingPeriods.length;
    // ...
    uint256 flowsClaimed;
    for (uint256 i; i < nbOfFlows; i++) {
        // Loop doesn't execute when nbOfFlows = 0
    }
    if (flowsClaimed == nbOfFlows) {  // 0 == 0 = true
        nftContract.burn(nftId);  // NFT BURNED!
        allocation.isClaimed = true;
    }
    token.safeTransfer(msg.sender, claimableAmounts);  // Transfers 0 tokens
}
```

When the wrong `projectId` is provided:
1. The allocation lookup returns empty data (`nbOfFlows = 0`)
2. The loop executes 0 times, so `flowsClaimed = 0`
3. The condition `flowsClaimed == nbOfFlows` evaluates to `0 == 0 = true`
4. The NFT is burned via `nftContract.burn(nftId)`
5. Zero tokens are transferred to the user


### Internal Pre-conditions

1. User needs to have claimed an NFT from a bidding or reward project (PROJECT_0)
2. A second project (PROJECT_1) needs to exist with the same token type (to avoid revert on token transfer)

### External Pre-conditions

None required.

### Attack Path

1. User claims NFT from PROJECT_0 with a vesting allocation of X tokens
2. User mistakenly calls `claimTokens(PROJECT_1, nftId)` with wrong project ID
3. The allocation lookup returns empty data from `biddingProjects[PROJECT_1].allocations[nftId]`
4. Since `nbOfFlows = 0` and `flowsClaimed = 0`, the condition `flowsClaimed == nbOfFlows` is true
5. The NFT is permanently burned
6. User receives 0 tokens
7. User has permanently lost their X token vesting allocation


### Impact

The user suffers a 100% loss of their vesting allocation. The tokens remain locked in the contract forever as the NFT representing the claim has been burned. This is not a griefing attack but rather a vulnerability that can be triggered by user error (incorrect projectId input).

For example:
- User has NFT with 5,000 project tokens vesting allocation in PROJECT_0
- User accidentally calls `claimTokens(1, nftId)` instead of `claimTokens(0, nftId)`
- NFT is burned, user receives 0 tokens
- 5,000 project tokens are permanently locked in the contract


### PoC

Add the following test file to `protocol/test/M06_MissingProjectIdValidation.t.sol`

Run with:
```bash
forge test --match-test test_M06_WrongProjectIdBurnsNFTWithZeroTokens -vvv
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

contract M06_MissingProjectIdValidationTest is Test {
    AlignerzVesting public vesting;
    AlignerzVesting public vestingImpl;
    AlignerzNFT public nft;
    MockERC20 public projectToken;
    MockERC20 public stablecoin;
    Merkle public merkle;

    address public owner = address(0x1);
    address public treasury = address(0x2);
    address public victim = address(0x3);
    address public user2 = address(0x4);

    uint256 public constant PROJECT_0 = 0;
    uint256 public constant PROJECT_1 = 1;
    uint256 public constant POOL_ID = 0;
    uint256 public constant BID_AMOUNT = 1000e6;
    uint256 public constant TOKEN_ALLOCATION = 10000e18;
    uint256 public constant TOKEN_PRICE = 1e17;
    uint256 public constant VESTING_PERIOD = 30 days;
    uint256 public constant CLAIM_WINDOW = 7 days;

    function setUp() public {
        vm.startPrank(owner);

        nft = new AlignerzNFT("Alignerz TVS", "ATVS", "https://api.alignerz.com/metadata/");

        vestingImpl = new AlignerzVesting();
        bytes memory initData = abi.encodeWithSelector(AlignerzVesting.initialize.selector, address(nft));
        ERC1967Proxy proxy = new ERC1967Proxy(address(vestingImpl), initData);
        vesting = AlignerzVesting(payable(address(proxy)));

        nft.addMinter(address(vesting));
        vesting.setTreasury(treasury);

        projectToken = new MockERC20("Project Token", "PROJ", 18);
        stablecoin = new MockERC20("USD Coin", "USDC", 6);

        projectToken.mint(owner, 1_000_000e18);
        stablecoin.mint(victim, 100_000e6);
        stablecoin.mint(user2, 100_000e6);

        merkle = new Merkle();

        vm.stopPrank();
    }

    function _setupTwoProjects() internal returns (uint256 victimNftId) {
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
        vesting.createPool(PROJECT_0, TOKEN_ALLOCATION, TOKEN_PRICE, false);

        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin),
            startTime,
            endTime,
            endTimeHash,
            false
        );

        projectToken.approve(address(vesting), TOKEN_ALLOCATION);
        vesting.createPool(PROJECT_1, TOKEN_ALLOCATION, TOKEN_PRICE, false);

        vm.stopPrank();

        vm.startPrank(victim);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_0, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(user2);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_0, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(victim, TOKEN_ALLOCATION / 2, PROJECT_0, POOL_ID));
        leaves[1] = keccak256(abi.encodePacked(user2, TOKEN_ALLOCATION / 2, PROJECT_0, POOL_ID));

        bytes32 root = merkle.getRoot(leaves);

        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = root;

        vm.startPrank(owner);
        vesting.finalizeBids(PROJECT_0, bytes32(0), merkleRoots, CLAIM_WINDOW);
        vm.stopPrank();

        bytes32[] memory proof = merkle.getProof(leaves, 0);

        vm.startPrank(victim);
        victimNftId = vesting.claimNFT(PROJECT_0, POOL_ID, TOKEN_ALLOCATION / 2, proof);
        vm.stopPrank();

        bytes32[] memory emptyMerkleRoots = new bytes32[](1);
        emptyMerkleRoots[0] = bytes32(0);

        vm.startPrank(owner);
        vesting.finalizeBids(PROJECT_1, bytes32(0), emptyMerkleRoots, CLAIM_WINDOW);
        vm.stopPrank();

        return victimNftId;
    }

    function test_M06_WrongProjectIdBurnsNFTWithZeroTokens() public {
        uint256 victimNftId = _setupTwoProjects();

        assertEq(nft.ownerOf(victimNftId), victim);

        uint256 tokenBalanceBefore = projectToken.balanceOf(victim);

        vm.warp(block.timestamp + VESTING_PERIOD + 1);

        vm.startPrank(victim);
        vesting.claimTokens(PROJECT_1, victimNftId);
        vm.stopPrank();

        uint256 tokenBalanceAfter = projectToken.balanceOf(victim);
        assertEq(tokenBalanceAfter, tokenBalanceBefore);

        vm.expectRevert();
        nft.ownerOf(victimNftId);
    }

    function test_M06_CorrectProjectIdClaimsTokensProperly() public {
        uint256 victimNftId = _setupTwoProjects();

        assertEq(nft.ownerOf(victimNftId), victim);

        uint256 tokenBalanceBefore = projectToken.balanceOf(victim);

        vm.warp(block.timestamp + VESTING_PERIOD + 1);

        vm.startPrank(victim);
        vesting.claimTokens(PROJECT_0, victimNftId);
        vm.stopPrank();

        uint256 tokenBalanceAfter = projectToken.balanceOf(victim);

        assertGt(tokenBalanceAfter, tokenBalanceBefore);

        vm.expectRevert();
        nft.ownerOf(victimNftId);
    }

    function test_M06_NonExistentProjectIdCausesRevert() public {
        uint256 victimNftId = _setupTwoProjects();

        assertEq(nft.ownerOf(victimNftId), victim);

        uint256 NON_EXISTENT_PROJECT = 999;

        vm.warp(block.timestamp + VESTING_PERIOD + 1);

        vm.startPrank(victim);
        vm.expectRevert();
        vesting.claimTokens(NON_EXISTENT_PROJECT, victimNftId);
        vm.stopPrank();

        assertEq(nft.ownerOf(victimNftId), victim);
    }

    function test_M06_MaxImpactTokenLoss() public {
        uint256 victimNftId = _setupTwoProjects();

        vm.warp(block.timestamp + VESTING_PERIOD + 1);

        uint256 tokensLost = TOKEN_ALLOCATION / 2;

        vm.startPrank(victim);
        vesting.claimTokens(PROJECT_1, victimNftId);
        vm.stopPrank();

        vm.expectRevert();
        nft.ownerOf(victimNftId);

        uint256 tokensReceived = projectToken.balanceOf(victim);
        assertEq(tokensReceived, 0);

        assertEq(tokensLost, TOKEN_ALLOCATION / 2);
    }
}

```

### Mitigation


Store and validate the project ID association with each NFT:

```solidity
// Add state variable
mapping(uint256 => uint256) public nftProjectId;

// Update claimNFT to store association
function claimNFT(...) external returns (uint256) {
    // ... existing code ...
    uint256 nftId = nftContract.mint(msg.sender);
    nftProjectId[nftId] = projectId;  // Store association
    // ... rest of code ...
}

// Validate in claimTokens
function claimTokens(uint256 projectId, uint256 nftId) external {
    address nftOwner = nftContract.extOwnerOf(nftId);
    require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
    require(nftProjectId[nftId] == projectId, "NFT does not belong to this project");
    // ... rest of code ...
}
```

Alternatively, remove the `projectId` parameter from `claimTokens()` entirely and derive it from the stored mapping.

  