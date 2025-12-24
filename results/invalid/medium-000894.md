# [000894] Wrong projectId burns vesting NFT with zero payout
  
  
## Summary
`claimTokens(projectId, nftId)` trusts the caller’s `projectId` and only derives type from `NFTBelongsToBiddingProject[nftId]`. If the projectId doesn’t actually own that NFT’s allocation, the lookup returns an empty allocation (0 flows). The function then satisfies `flowsClaimed == nbOfFlows` (0 == 0), burns the NFT, marks it claimed, and transfers 0 tokens. One mis-specified parameter permanently destroys the user’s vesting position.

## Root Cause
In `protocol/src/contracts/vesting/AlignerzVesting.sol`, `claimTokens` fetches the allocation using the caller-supplied `projectId` without any binding to the NFT’s originating project and without a bounds check. For a mismatched pair `(projectId, nftId)`, `allocation.vestingPeriods.length == 0`, so the post-loop burn path is taken. No diagnosis or revert is thrown.
The same missing binding affects `mergeTVS`/`splitTVS` (they also read/write allocations using caller-supplied projectId), and even an out-of-range projectId resolves to the zero slot and can trigger the burn path.

## Internal Preconditions
1) An NFT has been minted from a project (bidding or reward), so it represents a real allocation.  
2) The caller owns the NFT (ownership check passes).  
3) The caller supplies a `projectId` that does not match the NFT’s originating project (or is out of range).

## Attack Path
1) User holds a valid vesting NFT from project A.  
2) User (or buggy UI) calls `claimTokens(projectId = B, nftId = theirNFT)`.  
3) Allocation lookup returns empty; loop skips; `flowsClaimed == nbOfFlows` triggers NFT burn; no tokens are sent. The valid allocation in project A is now unrecoverable.

## Impact
- Loss of funds / position: The vesting NFT is burned and its real allocation becomes inaccessible; payout is 0.
- Triggerable by the rightful NFT owner via UI bug or user error; no attacker privileges required beyond calling `claimTokens` with a wrong `projectId`.
 - Related side effects: merge/split can act on the wrong project slot for an nftId, causing confusion or corrupted state if tokens match.

## Likelihood
Medium (high-risk footgun). Any wallet or frontend that misroutes `projectId` can brick user positions in one call.

## Proof of Concept
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";

/// @notice Demonstrates that calling claimTokens with the wrong projectId burns the NFT and pays out 0
contract AlignerzProjectBindingBurnBugTest is Test {
    MockUSD internal tvs;
    MockUSD internal stable;
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;

    address internal user = address(0xBEEF);

    uint256 internal constant TVS_AMOUNT = 100_000_000; // 100 tokens with 6 decimals

    function setUp() public {
        tvs = new MockUSD();
        stable = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        nft.addMinter(address(vesting));
        vesting.setTreasury(address(0x1));

        tvs.approve(address(vesting), type(uint256).max);
        stable.approve(address(vesting), type(uint256).max);
    }

    function test_BurnsNFTWithZeroPayout_WhenClaimingUsingWrongProjectId() public {
        // Launch two reward projects so projectId=1 is valid but unrelated
        uint256 start = block.timestamp;
        vesting.launchRewardProject(address(tvs), address(stable), start, 30 days);
        vesting.launchRewardProject(address(tvs), address(stable), start, 30 days);

        address[] memory kols = new address[](1);
        kols[0] = user;
        uint256[] memory tvsAllocAmounts = new uint256[](1);
        tvsAllocAmounts[0] = TVS_AMOUNT;

        vesting.setTVSAllocation(0, TVS_AMOUNT, 30 days, kols, tvsAllocAmounts);

        vm.prank(user);
        vesting.claimRewardTVS(0);

        uint256 nftId = _findUserNftId(user);
        assertEq(nft.ownerOf(nftId), user, "user owns freshly minted NFT");

        // Wrong projectId (1) has no allocation for this NFT; call still succeeds but burns the NFT and transfers 0
        vm.prank(user);
        vesting.claimTokens(1, nftId);

        // NFT is burned
        vm.expectRevert();
        nft.ownerOf(nftId);

        // No payout was received
        assertEq(tvs.balanceOf(user), 0);
    }

    function _findUserNftId(address owner) internal view returns (uint256) {
        uint256 minted = nft.getTotalMinted();
        for (uint256 tokenId; tokenId <= minted; tokenId++) {
            try nft.ownerOf(tokenId) returns (address currentOwner) {
                if (currentOwner == owner) return tokenId;
            } catch {
                continue;
            }
        }
        revert("NFT not found");
    }

}
```
Behavior:
1) Mint a reward NFT in project 0.  
2) Call `claimTokens(1, nftId)` (project 1 has no allocation for this NFT).  
3) The call succeeds, burns the NFT, and transfers 0. The original allocation is lost.

## Recommendation
- Bind NFTs to their originating project on-chain (e.g., `projectOf[nftId]`) and require `projectId` to match; add `projectId` bounds checks.
- Alternatively derive the project internally, removing `projectId` as user input in claim/merge/split paths.
- Add a revert guard for zero-flow allocations (e.g., require `allocation.vestingPeriods.length > 0`).



  