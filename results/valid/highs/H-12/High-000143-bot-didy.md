# [000143] Dividend setup reverts when TVS token matches distributor token
  
  ### Summary

Inverted token filtering in the dividend distributor causes `totalUnclaimedAmounts` to be zero when the distributor’s configured token equals the vesting allocation token, so `_setDividends` divides by zero and the owner cannot initialize dividends.

### Root Cause

In A26ZDividendDistributor.sol, `getUnclaimedAmounts` starts with `if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;`, which excludes the intended TVS token from unclaimed `totals. _setDividends` then computes `... / totalUnclaimedAmounts` and reverts if it is zero.

### Internal Pre-conditions

1. Distributor `token` equals the vesting allocation token for NFTs.
2. At least one NFT exists/owned; the distributor loops over minted `IDs`.
3. Owner calls `setUpTheDividends` (or `setAmounts + setDividends`).

### External Pre-conditions

none

### Attack Path

1. `getUnclaimedAmounts` returns 0 for matching-token NFTs.
2. `totalUnclaimedAmounts` remains 0 in `_setAmounts`.
3. `_setDividends` divides by zero (`stablecoinAmountToDistribute / totalUnclaimedAmounts`) and reverts, blocking dividend setup.

### Impact

dividends cannot be initialized (or are misallocated) when using the intended TVS token; core distribution flow breaks.

### PoC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import {Test, stdError} from "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IERC721Receiver} from "@openzeppelin/contracts/token/ERC721/IERC721Receiver.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";

/// @dev PoC: getUnclaimedAmounts returns 0 when distributor.token == vesting allocation token, making totalUnclaimedAmounts=0 and dividends division revert.
contract DividendTokenFilterBugTest is Test, IERC721Receiver {
    AlignerzVesting vesting;
    AlignerzNFT nft;
    Aligners26 token;
    MockUSD stable;
    A26ZDividendDistributor distributor;

    function setUp() public {
        // Deploy vesting stack.
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "uri/");
        token = new Aligners26("Aligners", "ALN");
        stable = new MockUSD();
        // Deploy vesting via proxy to mirror protocol setup.
        address payable proxy = payable(
            Upgrades.deployUUPSProxy("AlignerzVesting.sol", abi.encodeCall(AlignerzVesting.initialize, (address(nft))))
        );
        vesting = AlignerzVesting(proxy);
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1));

        // Launch reward project and allocate TVS in the same token as distributor.token.
        vesting.launchRewardProject(address(token), address(stable), block.timestamp, 1 days);
        token.approve(address(vesting), 1 ether);
        address[] memory kols = new address[](1);
        kols[0] = address(this);
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1 ether;
        vesting.setTVSAllocation(0, 1 ether, 30 days, kols, amounts);
        vesting.claimRewardTVS(0); // Mint NFT id 1 to this contract.

        // Bump totalMinted so the distributor loop reaches a real tokenId (loop iterates 0..len-1).
        nft.mint(address(this)); // Mint tokenId 2; totalMinted now 2.

        // Deploy distributor with same token as vesting allocation.
        distributor = new A26ZDividendDistributor(address(vesting), address(nft), address(stable), block.timestamp, 90 days, address(token));
        // Fund distributor with some stablecoin to distribute.
        stable.mint(address(distributor), 1000e6);
    }

    function test_SetDividendsRevertsDueToZeroTotalUnclaimed() public {
        // Division by zero when setting dividends because totalUnclaimedAmounts stays 0 (token matches allocation token).
        vm.expectRevert();
        distributor.setUpTheDividends();
    }

    // Implement IERC721Receiver so vesting can mint to this contract.
    function onERC721Received(address, address, uint256, bytes calldata) external pure override returns (bytes4) {
        return IERC721Receiver.onERC721Received.selector;
    }
}
```

### Mitigation

_No response_
  