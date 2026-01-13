# [000993] Users will be unable to split TVS positions due to denial of service [High]
  
  ### Summary

The uninitialized dynamic arrays in `_computeSplitArrays()` will cause a complete denial of service for TVS splitting functionality as any user will attempt to write to unallocated memory array indices causing a `Panic(0x32)` revert.

### Root Cause

In [AlignerzVesting.sol#L1113-L1141](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L1113-L1141) the `_computeSplitArrays()` function attempts to write to dynamic array elements in the `Allocation memory alloc` struct without first initializing the arrays with proper lengths, causing out-of-bounds access when writing to `alloc.amounts[j]`, `alloc.vestingPeriods[j]`, `alloc.vestingStartTimes[j]`, `alloc.claimedSeconds[j]`, and `alloc.claimedFlows[j]`.

### Internal Pre-conditions

1. User needs to own a TVS NFT with at least one flow (which occurs for all TVSs created through normal reward claiming or bidding processes)

### External Pre-conditions

None required

### Attack Path

1. User calls `splitTVS()` with valid percentages and owned NFT ID
2. The function calls `_computeSplitArrays()` which attempts to write to uninitialized dynamic arrays in the `alloc` memory struct
3. Writing to unallocated array indices triggers a `Panic(0x32)` revert due to out-of-bounds access

### Impact

The users cannot split any TVS positions at all, preventing important portfolio management operations and breaking the composability features advertised by the protocol. While no funds are lost due to the atomic revert behavior, the core splitting functionality remains completely unusable.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {stdError} from "forge-std/StdError.sol";
import {Aligners26} from "../../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../../src/MockUSD.sol";
import {AlignerzVesting} from "../../src/contracts/vesting/AlignerzVesting.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

// PoC: splitTVS reverts due to uninitialized dynamic arrays
// in both calculateFeeAndNewAmountForOneTVS and _computeSplitArrays.
contract SplitTVS_DoS is Test {
    Aligners26 internal token;
    MockUSD internal stable;
    AlignerzNFT internal nft;
    AlignerzVesting internal vesting;

    address internal owner;
    address internal kol;

    function setUp() public {
        owner = address(this);
        kol = makeAddr("kol");

        // Deploy mock assets
        token = new Aligners26("26Aligners", "A26Z");
        stable = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        // Deploy vesting as UUPS proxy and initialize with NFT
        address payable proxy = payable(
            Upgrades.deployUUPSProxy(
                "AlignerzVesting.sol",
                abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
            )
        );
        vesting = AlignerzVesting(proxy);

        // Grant vesting contract minter role on NFT and set treasury
        nft.addMinter(proxy);
        vesting.setTreasury(makeAddr("treasury"));

        // Transfer ownership stays with this test contract by default
    }

    // Creates a reward project and mints a TVS NFT for `kol`.
    function _mintRewardTVS() internal returns (uint256 nftId) {
        // Launch reward project (id = 0)
        vesting.launchRewardProject(address(token), address(stable), block.timestamp, 1 days);

        // Fund vesting with project tokens and set KOL allocation
        uint256 amount = 100 ether;
        token.approve(address(vesting), amount);
        address[] memory addrs = new address[](1);
        addrs[0] = kol;
        uint256[] memory amts = new uint256[](1);
        amts[0] = amount;
        vesting.setTVSAllocation(0, amount, 90 days, addrs, amts);

        // kol claims the TVS NFT
        vm.prank(kol);
        vesting.claimRewardTVS(0);

        // The first minted tokenId is 1 with this ERC721A implementation
        // getTotalMinted() == last minted id because _startTokenId() == 1
        nftId = nft.getTotalMinted();
        assertEq(nftId, 1, "unexpected nftId");
        assertEq(nft.ownerOf(nftId), kol, "KOL should own TVS");
    }

    function test_splitTVS_reverts_with_indexOOB_panic() public {
        uint256 nftId = _mintRewardTVS();

        // Attempt to split the TVS into two 50/50 parts.
        // Due to missing memory allocations in split helpers, this MUST revert with Panic(0x32).
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; // 50%
        percentages[1] = 5000; // 50%

        // Expect an index out-of-bounds panic from dynamic memory array writes
        vm.startPrank(kol);
        vm.expectRevert(stdError.indexOOBError);
        vesting.splitTVS(0, percentages, nftId);
        vm.stopPrank();
    }
}
```

### Mitigation

Consider initializing the dynamic arrays in the `Allocation memory alloc` struct before attempting indexed writes. The arrays should be allocated with the proper length at the beginning of the `_computeSplitArrays()` function by adding memory allocations for all dynamic array fields before the loop that attempts to populate them.
  