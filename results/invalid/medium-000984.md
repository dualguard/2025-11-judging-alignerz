# [000984] Attacker will cause denial of service for post-deadline stablecoin distribution [Medium]
  
  Attacker will cause denial of service for post-deadline stablecoin distribution [Medium]

### Summary

The missing validation in `setStablecoinAllocation()` allows invalid recipients to be added to the tail of the allocation list, which will cause a permanent denial of service for post-deadline stablecoin distributions as the `distributeRemainingStablecoinAllocation()` function will revert when attempting to transfer to failing addresses and cannot make progress through the recipient list.

### Root cause

In [AlignerzVesting.sol:484-501](https://github.com/dualguard/2025-11-alignerz-i-am-zcai/blob/main/protocol/src/contracts/vesting/AlignerzVesting.sol#L484-L501) the `setStablecoinAllocation()` function accepts any address without validation for zero addresses or potential blacklist status. Additionally, in `AlignerzVesting.sol:581-596` the `distributeRemainingStablecoinAllocation()` function processes recipients from the tail of the array and only removes entries after successful transfers, creating a dependency where progress requires all transfers to succeed.

### Internal pre-conditions

1. Admin needs to call `setStablecoinAllocation()` to set an invalid recipient (such as `address(0)` or a blacklisted address) at the tail position of `kolStablecoinAddresses`
2. Claim deadline needs to pass to enable post-deadline distribution functionality

### External pre-conditions

1. Stablecoin token needs to implement transfer restrictions (such as USDC blacklist functionality) that cause `safeTransfer()` to revert for certain addresses

### Attack path

1. Admin calls `setStablecoinAllocation()` with valid recipients followed by an invalid address (such as `address(0)`)
2. The claim deadline passes, enabling post-deadline distribution
3. Owner calls `distributeRemainingStablecoinAllocation()` which starts processing from the tail of the allocation list
4. The function attempts to transfer to the invalid recipient at the tail position and reverts on `safeTransfer()`
5. Since the transfer fails, the `pop()` operation is never reached, keeping the problematic address at the end
6. Subsequent calls continue to hit the same failing recipient, preventing any progress

### Impact

The protocol suffers a permanent denial of service for post-deadline stablecoin distributions when any invalid recipient exists at the tail position. Valid recipients cannot receive their allocations through the automated distribution mechanism, and funds remain stuck in the contract until manual administrative intervention using generic rescue functions. This breaks the intended on-chain distribution protocol flow.

### POC

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";

import {AlignerzVesting} from "src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "src/MockUSD.sol";

contract DoSDistributeRemainingStablecoinAllocationTest is Test {
    AlignerzVesting vesting;
    AlignerzNFT nft;
    MockUSD stablecoin;

    address alice = makeAddr("alice"); // valid recipient intended to receive stablecoin
    address zero = address(0); // invalid recipient that will cause transfer to revert

    function setUp() public {
        // Deploy NFT and vesting, initialize vesting
        nft = new AlignerzNFT("AlignerzNFT", "ANFT", "https://api/metadata/");
        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));
        // Allow vesting to mint/burn if needed (not required for this PoC path but keeps system wiring realistic)
        nft.addMinter(address(vesting));

        // Deploy mock stablecoin and fund the test owner
        stablecoin = new MockUSD();

        // Launch a reward project where stablecoin allocations will be set
        // startTime arbitrary; claimWindow short so we can advance past it
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 1 hours;
        vesting.launchRewardProject(address(0xdead), address(stablecoin), startTime, claimWindow);

        // Configure two recipients: a valid address first, then a failing tail at zero address
        address[] memory kolAddrs = new address[](2);
        kolAddrs[0] = alice;
        kolAddrs[1] = zero; // tail element that will always fail on transfer

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 100e6; // 100 USDC-style decimals (MockUSD has 6 decimals)
        amounts[1] = 50e6;  // 50

        uint256 total = amounts[0] + amounts[1];

        // Approve vesting to pull stablecoins from the owner during allocation setup
        stablecoin.approve(address(vesting), total);

        // Set allocations and fund vesting with the stablecoin total
        vesting.setStablecoinAllocation(0, total, kolAddrs, amounts);

        // Sanity: all tokens moved into vesting
        assertEq(stablecoin.balanceOf(address(vesting)), total, "vesting should hold all stablecoins");
    }

    function test_DoS_DistributeRemainingStablecoinAllocation_tailReverts_blocksAll() public {
        // Move time forward beyond the claim deadline so only owner-driven distribution is possible
        vm.warp(block.timestamp + 2 hours);

        // Attempt to distribute remaining stablecoin allocations.
        // The function iterates from the tail and tries to transfer to zero address first, which reverts.
        vm.expectRevert();
        vesting.distributeRemainingStablecoinAllocation(0);

        // Ensure no progress was made: vesting still holds all funds and Alice received nothing.
        uint256 total = 150e6;
        assertEq(stablecoin.balanceOf(address(vesting)), total, "funds remain stuck in vesting");
        assertEq(stablecoin.balanceOf(alice), 0, "alice did not receive allocation due to tail DoS");

        // Retrying continues to hit the same failing tail element and reverts again, proving permanent blockage.
        vm.expectRevert();
        vesting.distributeRemainingStablecoinAllocation(0);

        // As an additional check, attempting the alternative public batch function also cannot bypass the failure.
        // Passing both recipients will revert on the zero-address transfer during processing.
        address[] memory batch = new address[](2);
        batch[0] = alice;
        batch[1] = zero;
        vm.expectRevert();
        vesting.distributeStablecoinAllocation(0, batch);
    }
}
```

### Mitigation

Consider implementing a failure-resilient distribution mechanism that can handle individual transfer failures without blocking the entire process. One approach could be to modify the distribution logic to track and skip problematic addresses while continuing to process valid recipients. The remediation strategy should include recipient validation during the allocation setup phase to prevent obviously invalid addresses from entering the system. Additionally, consider implementing a mechanism to mark failed recipients for later manual intervention while allowing the automated distribution to continue processing the remaining recipients rather than reverting the entire transaction on the first failure.
  