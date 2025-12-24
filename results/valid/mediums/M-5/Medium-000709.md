# [000709] Future-start flows revert claimTokens after a merge
  
  
## summary  
claimTokens iterates all flows; getClaimableAmountAndSeconds assumes block.timestamp >= vestingStartTime, so if a merged NFT contains a flow whose start time is still in the future, the subtraction block.timestamp - vestingStartTime underflows and the entire claim reverts.

## Finding Description  
`claimTokens()` loops over every flow inside `allocation.vestingStartTimes`. When one flow has a start time > `block.timestamp`—a realistic scenario if users merge/split TVS from different projects—the helper `getClaimableAmountAndSeconds()` calculates `secondsPassed = block.timestamp - vestingStartTime` inside the `else` branch. Because that value is negative, the subtraction underflows and the transaction reverts before the caller can claim any flow, including those that are already active. Since mergers/splits concatenate flows from arbitrary pools, an attacker can always create an NFT containing at least one future-start flow to lock the entire NFT’s claim.  

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L989
```solidity
  secondsPassed = block.timestamp - vestingStartTime;
```
https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L980-L996


```solidity
    function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
        uint256 secondsPassed;
        uint256 claimedSeconds = allocation.claimedSeconds[flowIndex];
        uint256 vestingPeriod = allocation.vestingPeriods[flowIndex];
        uint256 vestingStartTime = allocation.vestingStartTimes[flowIndex];
        uint256 amount = allocation.amounts[flowIndex];
        if (block.timestamp > vestingPeriod + vestingStartTime) {
            secondsPassed = vestingPeriod;
        } else {
            secondsPassed = block.timestamp - vestingStartTime;
        }


        claimableSeconds = secondsPassed - claimedSeconds;
        claimableAmount = (amount * claimableSeconds) / vestingPeriod;
        require(claimableAmount > 0, No_Claimable_Tokens());
        return (claimableAmount, claimableSeconds);
    }
```

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L941-L975
```solidity
function claimTokens(uint256 projectId, uint256 nftId)

```
** Attack Path  **
Mint two NFTs whose vesting start times differ. Merge or split them so the resulting NFT inherits a future-start flow (common when combining reward + bidding allocations). Call `claimTokens()` while the future flow is not yet active. The call immediately reverts because `block.timestamp - vestingStartTime` underflows, so the attacker freezes every other unlocked flow inside that NFT until the future `vestingStartTime` passes.

## Impact  
High. A single merged NFT can halt every claim forever until the future-start flow unlocks, allowing the attacker to deny their previously vested tokens (and any future dividends tied to that NFT) for an extended period. This is a denial-of-service on claims for that NFT.

## Likelihood Explanation  
Medium. While requiring a merged or split TVS that combines flows with different start times, this is still a supported feature. Once a single such NFT exists (owner can intentionally create one), it permanently blocks `claimTokens()` until the latest `vestingStartTime` elapses, making the threat easy to trigger.

## poc 
run command 
cd protocol
forge clean
forge build
forge test --mt test_ClaimTokens_Reverts_For_FutureStartTime -vvv
```solidity
function test_ClaimTokens_Reverts_For_FutureStartTime() public {
        // Owner configures a reward project whose vesting starts in the future
        vm.startPrank(projectCreator);
        uint256 futureStartTime = block.timestamp + 30 days;
        uint256 claimWindow = 60 days;
        vesting.launchRewardProject(address(token), address(usdt), futureStartTime, claimWindow);

        uint256 rewardProjectId = 0;
        uint256 totalTVSAllocation = 1_000 ether;
        address kol = bidders[0];

        address[] memory kolTVS = new address[](1);
        kolTVS[0] = kol;
        uint256[] memory TVSamounts = new uint256[](1);
        TVSamounts[0] = totalTVSAllocation;

        // Set a single TVS allocation for the KOL
        vesting.setTVSAllocation(rewardProjectId, totalTVSAllocation, 90 days, kolTVS, TVSamounts);
        vm.stopPrank();

        // KOL claims the reward TVS, receiving an NFT whose vestingStartTime is in the future
        vm.prank(kol);
        vesting.claimRewardTVS(rewardProjectId);

        uint256 nftId = nft.getTotalMinted() - 1;

        // At this point block.timestamp < vestingStartTime, so claimTokens() underflows
        vm.prank(kol);
        vm.expectRevert();
        vesting.claimTokens(rewardProjectId, nftId);
    }
```
## Recommendation  
Guard the subtraction by skipping future flows or returning zero claimable amount instead of subtracting. For example:

```solidity 
function getClaimableAmountAndSeconds(Allocation memory allocation, uint256 flowIndex) public view returns(uint256 claimableAmount, uint256 claimableSeconds) {
    uint256 vestingStartTime = allocation.vestingStartTimes[flowIndex];
    if (block.timestamp <= vestingStartTime) {
        return (0, 0);
    }
    uint256 secondsPassed = Math.min(block.timestamp - vestingStartTime, allocation.vestingPeriods[flowIndex]);
    ...
}
```
  