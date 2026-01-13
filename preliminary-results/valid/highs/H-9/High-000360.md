# [000360] H15 Interface Mismatch DoS
  
  ## Summary

There is a mismatch between the `IAlignerzVesting` interface and the `AlignerzVesting` implementation regarding the `allocationOf` function. The interface expects a full `Allocation` struct (including arrays), but the implementation's public mapping getter returns a tuple that omits the arrays. This causes all calls from `A26ZDividendDistributor` to `vesting.allocationOf` to revert.

## Root Cause

In `IAlignerzVesting.sol`:
```solidity
    function allocationOf(uint256 nftId) external view returns (Allocation memory);
```

In `AlignerzVesting.sol`:
```solidity
    mapping(uint256 => Allocation) public allocationOf;
```

Solidity's auto-generated getter for public mappings with structs containing arrays does not return the arrays. It returns `(bool isClaimed, IERC20 token, uint256 assignedPoolId)`. The distributor expects the full struct, leading to a decoding error.

## Internal Pre-Conditions

None.

## External Pre-Conditions

None.

## Impact

**Critical**. The `A26ZDividendDistributor` is completely non-functional. Any attempt to calculate or distribute dividends will revert.

## PoC

See `test/InterfaceMismatchTest.t.sol`.

```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "forge-std/Test.sol";
import "../src/contracts/vesting/AlignerzVesting.sol";
import "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import "../src/MockUSD.sol";
import "../src/contracts/nft/AlignerzNFT.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";

contract InterfaceMismatchTest is Test {
    AlignerzVesting vesting;
    AlignerzNFT nft;
    A26ZDividendDistributor distributor;
    MockUSD token;
    MockUSD stablecoin;

    address owner = address(0x1);
    address user = address(0x2);

    function setUp() public {
        vm.startPrank(owner);

        token = new MockUSD();
        stablecoin = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "ANFT", "https://example.com/");

        AlignerzVesting vestingImpl = new AlignerzVesting();
        ERC1967Proxy proxy = new ERC1967Proxy(
            address(vestingImpl),
            abi.encodeWithSelector(
                AlignerzVesting.initialize.selector,
                address(nft)
            )
        );
        vesting = AlignerzVesting(payable(address(proxy)));

        nft.addMinter(address(vesting));
        nft.transferOwnership(address(vesting));

        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            block.timestamp,
            100 days,
            address(token)
        );

        vm.stopPrank();
    }

    function testInterfaceMismatchRevert() public {
        vm.startPrank(owner);

        // 1. Launch Project
        uint256 projectId = vesting.rewardProjectCount();
        token.mint(owner, 1000 ether);
        token.approve(address(vesting), 1000 ether);
        vesting.launchRewardProject(
            address(token),
            address(stablecoin),
            block.timestamp,
            1 days
        );

        // 2. Allocate
        address[] memory kols = new address[](1);
        kols[0] = user;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 100 ether;
        vesting.setTVSAllocation(projectId, 100 ether, 100 days, kols, amounts);

        vm.stopPrank();

        vm.startPrank(user);
        vesting.claimRewardTVS(projectId);
        uint256 nftId = nft.getTotalMinted() - 1;
        vm.stopPrank();

        // 3. Call getUnclaimedAmounts
        // This calls vesting.allocationOf(nftId).
        // The interface expects a struct (with arrays).
        // The contract returns a tuple (without arrays).
        // This causes a decoding error/revert.

        // vm.expectRevert(); // We expect it to revert due to the mismatch
        distributor.getUnclaimedAmounts(nftId);
    }
}

```

## Mitigation

Implement a custom getter function in `AlignerzVesting` that returns the full struct.

```solidity
    function getAllocation(uint256 nftId) external view returns (Allocation memory) {
        return allocationOf[nftId];
    }
```
And update the interface and distributor to use this function.

  