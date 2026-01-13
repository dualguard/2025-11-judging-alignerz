# [000675] [M-3] Reward distribution trusts user array lengths and reverts when they mismatch storage
  
  ### Summary

`distributeRewardTVS` and `distributeStablecoinAllocation` iterate from `i = 0` to `len = rewardProject.kol...Addresses.length` but index into the user-provided `kol` calldata array. If the caller passes a shorter array than the stored length (or an empty array), the loops attempt to read beyond `kol.length`, causing an immediate revert and making these admin helpers unusable unless operators supply perfectly matching arrays.

### Root Cause

In `AlignerzVesting.sol:522-549`, both functions compute `len` from the storage array but access `kol[i]` (calldata). Solidity bounds-checks calldata reads, so when `i` reaches `kol.length`, the call reverts with an out-of-bounds error. There is no validation that `kol.length == len`, nor do the functions fall back to storage iteration.

### Internal Pre-conditions

  1. Reward project has `kolTVSAddresses` / `kolStablecoinAddresses` entries awaiting distribution.
  2. Caller mistakenly provides a shorter `kol` array (e.g., omits an address or uses an empty list).

### External Pre-conditions

None.

### Attack Path

  1. Admin calls `distributeRewardTVS(projectId, new address[](0))`.
  2. `len` equals the storage array length (say 10), so the `for` loop runs from 0 to 9.
  3. On the first iteration, the code tries to read `kol[0]` but the calldata array is empty, so the transaction reverts. All distributions are blocked until the caller supplies an exact-length array.

### Impact

Admin convenience functions become fragile and revert on minor caller mistakes, preventing bulk distributions and leaving allocations stuck. While not a direct fund loss, it prevents operational flows unless perfect inputs are supplied.

### PoC

```
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {stdError} from "forge-std/StdError.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";

contract RewardDistributionMismatchTest is Test {
    AlignerzVesting private vesting;
    address private constant KOL1 = address(0x1111);
    address private constant KOL2 = address(0x2222);

    function setUp() public {
        AlignerzNFT nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "uri");
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vesting.transferOwnership(address(this));

        Aligners26 token = new Aligners26("Aligners", "A26Z");
        MockUSD stable = new MockUSD();

        vesting.launchRewardProject(address(token), address(stable), block.timestamp, 1 days);

        token.approve(address(vesting), type(uint256).max);
        address[] memory kols = new address[](2);
        kols[0] = KOL1;
        kols[1] = KOL2;
        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 10 ether;
        amounts[1] = 20 ether;
        vesting.setTVSAllocation(0, 30 ether, 30 days, kols, amounts);

        vm.warp(block.timestamp + 1 days + 1);
    }

    function test_distributeRewardTVS_revertsOnMismatch() public {
        address[] memory kolSubset = new address[](1);
        kolSubset[0] = KOL1;

        vm.expectRevert(stdError.indexOOBError);
        vesting.distributeRewardTVS(0, kolSubset);
    }
}
```

### Mitigation

 Ignore the calldata array entirely and iterate over storage addresses, or add `require(kol.length == len, "Mismatched array length")` before looping. Alternatively, treat the calldata list as authoritative and iterate based on `kol.length` while verifying each address exists in storage.

  