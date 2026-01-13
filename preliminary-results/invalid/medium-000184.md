# [000184] `setTVSAllocation` and `setStablecoinAllocation` functions might run out of gas  and cannot be called multiple times for a given reward project.
  
  ### Summary

The `setTVSAllocation` and `setStablecoinAllocation` functions are extremely similar and are used to set allocations for all KOLs in the context of a reward project.

Let's take the `setTVSAllocation` function as an example:

```solidity
    function setTVSAllocation(
        uint256 rewardProjectId,
        uint256 totalTVSAllocation,
        uint256 vestingPeriod,
        address[] calldata kolTVS,
        uint256[] calldata TVSamounts
    ) external onlyOwner {
        RewardProject storage rewardProject = rewardProjects[rewardProjectId];
        rewardProject.vestingPeriod = vestingPeriod;
        uint256 length = kolTVS.length;
        require(length == TVSamounts.length, Array_Lengths_Must_Match());
        uint256 totalAmount;
        for (uint256 i = 0; i < length; i++) {
            address kol = kolTVS[i];
            rewardProject.kolTVSAddresses.push(kol);
            uint256 amount = TVSamounts[i];
            rewardProject.kolTVSRewards[kol] = amount;
            rewardProject.kolTVSIndexOf[kol] = i;
            totalAmount += amount;
            emit TVSAllocated(rewardProjectId, kol, amount, vestingPeriod);
        }
        require(totalTVSAllocation == totalAmount, Amounts_Do_Not_Add_Up_To_Total_Allocation());
        rewardProject.token.safeTransferFrom(msg.sender, address(this), totalTVSAllocation);
    }
```

It takes an array of KOL addresses and an array of amounts, iterates over the length of the arrays, and sets these values in the corresponding mappings (`kolTVSIndexOf[kol] ` and `kolTVSRewards[kol]`) and pushes addresses in `kolTVSAddresses` array.

This means that for each iteration, multiple `SSTORE` instructions are used, costing gas.

The problem is that if too many KOL addresses and amounts are passed to `setTVSAllocation` or `setStablecoinAllocation` functions, calling the function will exceed the block gas limit of 45 millions gas.

### Root Cause

The root cause is that it is not possible to call multiple times these 2 functions in order to set allocations for multiple batches if there are too many KOLs.
Indeed,  the increment `i` variable is used to assign the index in the array:

```solidity
            rewardProject.kolTVSIndexOf[kol] = i;
```

This will entirely break the claim mechanism. 

### Impact

The impact of this issue is medium as it results in an incapacity for the protocol to set allocations when there are too many eligible KOLs.
With the current implementation, 5000 KOLs would exceed the block gas limit.

### PoC

Please copy paste the following test in the test file:

```solidity
  function testOutOfGasSetters() external {
        vm.startPrank(projectCreator);

        uint256[] memory TVSAmounts = new uint256[](5000);
        for (uint256 i = 0; i < 5000; i++) {
            TVSAmounts[i] = 1;
        }
        address[] memory kolTVS = new address[](5000);

        vesting.setTVSAllocation(0, 5000, 1, kolTVS, TVSAmounts); // same for setStablecoinAllocation

        // 100  : 1_029_585
        // 1000 : 9_309_811
        // 5000 : 46_876_436 - exceeds block gas limit
    }
```

For the test to successfully run and to simplify it, just comment out the `transferFrom`  call at the end of the `setTVSAllocation` function:

```solidity
        // rewardProject.token.safeTransferFrom(msg.sender, address(this), totalTVSAllocation);

```

This test shows that:
- 100 KOLs costs around 1 million gas
- 1000 KOLs costs 9 million gas
- 5000 KOLs exceeds 45 million gas

The test is simplified and includes the gas consumption of setting all values of `TVSAmounts`. On the other hand, it excludes the consumption of the `transferFrom`. So overall, it is negligible. 

### Mitigation

Modify the current implementation so that it is possible to call these 2 functions multiple times to set allocations for multiple batches.
  