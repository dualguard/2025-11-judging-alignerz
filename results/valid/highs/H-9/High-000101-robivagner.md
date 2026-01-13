# [000101] `allocationOf` will not return the arrays inside `Allocation` struct for external calls, breaking divindend distributor contract
  
  ### Summary

The way to get an allocation of an NFT/TVS in the vesting contract is by calling [`allocationOf`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L114) mapping. The problem is that solidity auto-generated getter functions will only return non-array fields: `(bool isClaimed, IERC20 token, uint256 assignedPoolId)`, so 3 values instead of 8 values.
The `A26ZDividendDistributor` contract calls `allocationOf` inside [`_setAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207), which is a core functionality.
```solidity
    function _setAmounts() internal {
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
@>        totalUnclaimedAmounts = getTotalUnclaimedAmounts();
        emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
    }
    ...
    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
@>        uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
@>        uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
@>        uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
@>        bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
       ...
```
Function [`getUnclaimedAmounts`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140) will always revert.

### Root Cause

Calling the auto-generated getter function inside `A26ZDividendDistributor` since there is no getter function inside `AlignerzVesting` for the allocations will always revert.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. `A26ZDividendDistributor` contract owner calls `setUpTheDividends` function.
2. It will always revert since it calls `_setAmounts`.

### Impact

The `A26ZDividendDistributor` is rendered useless since a core functionality of the contract `setUpTheDividends` always reverts.

### PoC

```solidity
    function test_PoC_getUnclaimedAmountsReverts() public {
        vm.startPrank(projectCreator);
        A26ZDividendDistributor distributor = new A26ZDividendDistributor(
            address(vesting), address(nft), address(usdt), block.timestamp, 30 days, address(token)
        );

        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);

        address[] memory kols = new address[](10);
        for (uint256 i = 0; i < 10; i++) {
            kols[i] = address(uint160(i + 1));
        }

        uint256[] memory amounts = new uint256[](10);
        for (uint256 i = 0; i < 10; i++) {
            amounts[i] = 1_000e18;
        }

        vesting.setTVSAllocation(0, 10_000e18, 30 days, kols, amounts);
        vm.stopPrank();

        for (uint256 i = 0; i < 10; i++) {
            vm.prank(kols[i]);
            vesting.claimRewardTVS(0);
        }

        vm.prank(projectCreator);
        vm.expectRevert();
        distributor.setUpTheDividends();
    }
```

### Mitigation

Add a getter function for the `allocationOf` mapping inside the vesting contract.
  