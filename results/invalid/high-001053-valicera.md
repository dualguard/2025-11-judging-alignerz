# [001053] Missing onlyOwner modifier allows anyone to distribute RewardTVS and StablecoinAllocation
  
  ### Summary

The lack of access control on `AlignerzVesting::distributeRewardTVS` and `AlignerzVesting::distributeStablecoinAllocation` will allow anyone to trigger distribution after the claim window.
`distributeStablecoinAllocation` calls _claimStablecoinAllocation which immediately executes `rewardProject.stablecoin.safeTransfer(kol, amount)`. Anyone can initiate distributeStablecoinAllocation with the proper kol list and cause the contract to transfer the stored stablecoins to those kol addresses. Funds are transferred out of the protocol - direct financial action that the owner should control.
`distributeRewardTVS` calls `_claimRewardTVS` which mints NFTs and assigns allocations. While it does not immediately transfer ERC20 TVS tokens, it mints on-chain NFTs and sets up allocations so KOLs can claim later. Allowing anyone to mint these NFTs bypasses intended owner control over distribution timing.
```solidity
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external /* @> missing onlyOwner */ {
        // ....
    }
```
```solidity
  function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external /* @> missing onlyOwner */ {
        // ....
    }
```
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L550

### Root Cause

`AlignerzVesting::distributeRewardTVS` and `AlignerzVesting::distributeStablecoinAllocation` are supposed to be called by the owner.

### Internal Pre-conditions

1. Reward project should be launched.
2. TVS and Stablecoin Allocations should be set.
3. Claim window must have passed.

### External Pre-conditions

N/A

### Attack Path

1. Attacker calls distributeStablecoinAllocation after claim deadline.
2. Attacker calls distributeRewardTVS after claim deadline.

### Impact

Protocol suffers unexpected fund distribution and nft allocation.

### PoC

Add test in `AlignerzVestingProtocolTest` and run `forge test --mt test_NonOwnerCanCall_DistributeRewardTVS_distributeStablecoinAllocation -vvvv`

```solidity
    function test_NonOwnerCanCall_DistributeRewardTVS_distributeStablecoinAllocation() public {
        // prepare allocation for 5 KOLs TVS/stablecoin amounts
        address[] memory kols = new address[](5);
        uint256[] memory tvsAndStableAmounts = new uint256[](5);
        for (uint256 i = 0; i < kols.length; i++) {
            kols[i] = bidders[i];
            tvsAndStableAmounts[i] = BIDDER_USD;
        }
        uint256 totalTvsAndStableCoinAllocation = BIDDER_USD * kols.length;
        // projectCreator has token and approved vesting in setUp
        usdt.mint(projectCreator, totalTvsAndStableCoinAllocation);
        vm.startPrank(projectCreator);
        // ensure projectCreator has USDT and approve vesting contract
        usdt.approve(address(vesting), totalTvsAndStableCoinAllocation);
        // launch a reward project with 1 day claim window
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 1 days);
        vesting.setTVSAllocation(PROJECT_ID, totalTvsAndStableCoinAllocation, 90 days, kols, tvsAndStableAmounts);
        vesting.setStablecoinAllocation(PROJECT_ID, totalTvsAndStableCoinAllocation, kols, tvsAndStableAmounts);
        vm.stopPrank();

        // fast-forward past claim deadline
        vm.warp(block.timestamp + 2 days);

        // call distributeRewardTVS and distributeStablecoinAllocationas a non-owner kols[1]
        vm.startPrank(kols[1]);
        vesting.distributeRewardTVS(PROJECT_ID, kols);
        vesting.distributeStablecoinAllocation(PROJECT_ID, kols);
        vm.stopPrank();

        // the KOL should have received an NFT and stablecoin - assertions
        assertEq(nft.extOwnerOf(1), kols[0]);
        assertEq(nft.extOwnerOf(2), kols[1]);
        assertEq(nft.extOwnerOf(3), kols[2]);
        assertEq(nft.extOwnerOf(4), kols[3]);
        assertEq(nft.extOwnerOf(5), kols[4]);
        assertEq(usdt.balanceOf(kols[0]), BIDDER_USD * 2);
        assertEq(usdt.balanceOf(kols[1]), BIDDER_USD * 2);
        assertEq(usdt.balanceOf(kols[2]), BIDDER_USD * 2);
        assertEq(usdt.balanceOf(kols[3]), BIDDER_USD * 2);
        assertEq(usdt.balanceOf(kols[4]), BIDDER_USD * 2);
        assertEq(usdt.balanceOf(projectCreator), 0);
        console.log("Total nft minted: ", nft.getTotalMinted());
        console.log("USD supply on KOL1: ", usdt.balanceOf(kols[1]));
    }
```

### Mitigation

Add access control - mark both functions with onlyOwner modifier
  