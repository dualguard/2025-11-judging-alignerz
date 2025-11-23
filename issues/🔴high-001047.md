# [001047] calculateFeeAndNewAmountForOneTVS will revert due to OOG
  
  ### Summary

The function `FeesManager::calculateFeeAndNewAmountForOneTVS` always reverts due to out-of-gas.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174

### Root Cause

The loop index i is never incremented, the loop becomes non-terminating whenever length > 0: i stays 0, the condition i < length remains true forever, and the transaction will run until it runs out of gas and reverts.

```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
>>      for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
        }
    }
```
`length` are the nbOfFlows which is equal to the amount of tokens for allocation for all flows. So by splitting or merging 2 TVS flows are meant to be increased by the amounts. Therefore length will be > 1

### Internal Pre-conditions

1. User needs to have 2 TVS either from allocation or splitting TVS

### External Pre-conditions

N/A

### Attack Path

N/A

### Impact

Users are not able to merge. Merging is denied due to reverting OOG, which is core functionality of the protocol.

### PoC

in `AlignerzVestingProtocolTest` add the following test and run `forge test --mt test_mergeTVS_Panic_Reverts -vvvv`

```solidity
    function test_mergeTVS_Reverts_OutOfGas() public {
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

Modify the loop to increment i
  