# [001048] calculateFeeAndNewAmountForOneTVS reverts due to unallocated memory writes
  
  ### Summary

`FeesManager::calculateFeeAndNewAmountForOneTVS` function always reverts due to writing to unallocated memory array.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174


### Root Cause

The function  `FeesManager::calculateFeeAndNewAmountForOneTVS` returns uint256[] memory newAmounts but never assigns newAmounts = new uint256[](length). Writing to newAmounts[i] therefore performs indexed writes into an uninitialized pointer and panic reverts - out-of-bounds memory access.

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```

### Internal Pre-conditions

1. User should have 2 NFTs either through allocation in different projects, owner distribution, or tvs splitting.

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Due to `calculateFeeAndNewAmountForOneTVS` always reverting users are denied from merging TVS, which breaks the core functionality of the protocol.

### PoC

In `AlignerzVestingProtocolTest` add the following test and run `forge test --mt test_mergeTVS_Panic_Reverts -vvvv`

```solidity
    function test_mergeTVS_Panic_Reverts() public {
        address kol = bidders[0];

        // projectCreator launches TWO reward projects and allocates one TVS to the same kol in each
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Reward project 0
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 days);
        address[] memory kols0 = new address[](1);
        kols0[0] = kol;
        uint256[] memory amts0 = new uint256[](1);
        amts0[0] = 1_000 ether;
        vesting.setTVSAllocation(0, amts0[0], 90 days, kols0, amts0);

        // Reward project 1
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 days);
        address[] memory kols1 = new address[](1);
        kols1[0] = kol;
        uint256[] memory amts1 = new uint256[](1);
        amts1[0] = 1_000 ether;
        vesting.setTVSAllocation(1, amts1[0], 90 days, kols1, amts1);
        vm.stopPrank();

        // kol claims both TVS allocations and receives two NFTs (one per project)
        vm.prank(kol);
        vesting.claimRewardTVS(0);
        uint256 nft1 = nft.getTotalMinted();

        vm.prank(kol);
        vesting.claimRewardTVS(1);
        uint256 nft2 = nft.getTotalMinted();

        // ensure kol owns both NFTs
        assertEq(nft.extOwnerOf(nft1), kol);
        assertEq(nft.extOwnerOf(nft2), kol);

        // Attempt to merge nft2 (from project 1) into nft1 (from project 0)
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = 1; // merging NFT from project 1
        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nft2;

        vm.prank(kol);
        vm.expectRevert();
        vesting.mergeTVS(0, nft1, projectIds, nftIds);
    }
```

### Mitigation

Allocate the return array `newAmounts` before writing to it.
newAmounts = new uint256 [] ();
Further enforce require(length <= amounts.length) at the start of the function.
  