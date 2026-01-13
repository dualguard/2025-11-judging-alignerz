# [000183] `distributeRewardTVS` and `distributeStablecoinAllocation` functions iterate over the wrong length, leading to impossibility to distribute to a batch of addresses, and ultimately rewards lost.
  
  ### Summary

The `distributeRewardTVS` and `distributeStablecoinAllocation` functions are defined as follows:

```solidity
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        // @audit HIGH should be len = kol.length
        uint256 len = rewardProject.kolTVSAddresses.length;
        for (uint256 i; i < len;) {
            _claimRewardTVS(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```


```solidity
    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
        // @audit HIGH should be len = kol.length
        uint256 len = rewardProject.kolStablecoinAddresses.length;
        for (uint256 i; i < len;) {
            _claimStablecoinAllocation(rewardProjectId, kol[i]);
            unchecked {
                ++i;
            }
        }
    }
```

Both functions are used to distribute allocation / TVS to a batch of KOL addresses. These functions are especially important because if too many addresses didn't claim by themselves, `distributeRemainingRewardTVS` and `distributeRemainingStablecoinAllocation` functions (which distribute allocations / TVS to all remaining addresses)  won't be callable as they will run out of gas.

The problem is that there is an error in the implementation of `distributeRewardTVS` and `distributeStablecoinAllocation` functions.

### Root Cause

The root cause lies in the same line in both functions:

```solidity
        uint256 len = rewardProject.kolTVSAddresses.length;
```
and 
```solidity
        uint256 len = rewardProject.kolStablecoinAddresses.length;
```

The loop iterates over all remaining KOL addresses that didn't claim, instead of iterating over the provided array in the arguments.

Hence, these functions cannot distribute TVS or allocations to only a batch of addresses.

### Attack Path

If too many KOL didn't claim their TVS or allocation, it might be impossible to distribute to them.

### Impact

The impact of this issue is high as it may lead to users unable to receive their rewards if too many people didn't claim the NFT / stablecoin allocation by themselves.

### PoC

Please copy paste the following test in the test file:

```solidity
function testGas_DistributeRemainingRewardTVS_SingleCall() public {
        uint256 kolCount = 100;
        uint256 projectId = 0;

        // Setup project
        vm.startPrank(projectCreator);
        vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 3600);

        // Prepare KOL addresses and amounts
        address[] memory kolAddresses = new address[](kolCount);
        uint256[] memory tvsAmounts = new uint256[](kolCount);

        for (uint256 j = 0; j < kolCount; j++) {
            kolAddresses[j] = address(uint160(1000 + j));
            tvsAmounts[j] = 1000 ether;
        }

        // Set TVS allocations
        vesting.setTVSAllocation(projectId, kolCount * 1000 ether, 30 days, kolAddresses, tvsAmounts);

        // Fast forward past claim deadline
        vm.warp(block.timestamp + 3601);

        // Measure gas
        uint256 gasStart = gasleft();
        vesting.distributeRemainingRewardTVS(projectId);
        uint256 gasUsed = gasStart - gasleft();

        vm.stopPrank();

        console.log("Gas used for %d KOLs: %d", kolCount, gasUsed); // 49_959_737 -> exceeds block gas limit
        console.log("Gas per KOL: %d", gasUsed / kolCount); // 499_597
    }
```

This test highlights that with only 100 KOL addresses who didn't claim their TVS, it is impossible to distribute them.

### Mitigation

Make sure to correctly implement `distributeRewardTVS` and `distributeStablecoinAllocation` functions and iterate using the correct length:

```solidity
        uint256 len = kol.length;
```

  