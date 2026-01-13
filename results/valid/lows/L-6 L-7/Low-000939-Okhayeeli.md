# [000939] Array Index Mismatch in `distributeRewardTVS` and `distributeStablecoinAllocation`
  
  ### Summary

The array index mismatch in `distributeRewardTVS` and `distributeStablecoinAllocation` will cause transaction reverts  as the functions loop using storage array length but access the input parameter array,halting batch processing.

###  Vulnerability Description

 In [AlignerzVesting.sol:524-535](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535) ands [AlignerzVesting.sol:540-550](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L550) it uses  `rewardProject.kolTVSAddresses.length` (storage array) as the loop bound but access `kol[i]` (input parameter array) inside the loop. When the input array is shorter than the storage array, the function reverts with index out of bounds.

Scenerio
- Owner calls distributeRewardTVS(projectId, [addr1, addr2, ...]) with 10 addresses
- Storage array kolTVSAddresses contains 100 unclaimed KOLs
- Loop sets len = 100 at line 528[ AlignerzVesting.sol:528](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L528)
- Loop attempts to access kol[10] when i = 10
- Transaction reverts with "Index out of bounds" because input array only has 10 elements


### POC
```solidity
function test_DistributeRewardTVS_OutOfBoundsRevert() public {  
    // Setup: Create a reward project with 100 KOLs  
    vm.startPrank(projectCreator);  
    vesting.launchRewardProject(address(token), address(usdt), block.timestamp, 7 days);  
      
    // Create 100 KOL addresses  
    address[] memory allKols = new address[](100);  
    uint256[] memory amounts = new uint256[](100);  
    for (uint256 i = 0; i < 100; i++) {  
        allKols[i] = makeAddr(string.concat("kol", vm.toString(i)));  
        amounts[i] = 1000 ether;  
    }  
      
    // Set TVS allocations for all 100 KOLs  
    vesting.setTVSAllocation(0, 100_000 ether, 90 days, allKols, amounts);  
    vm.stopPrank();  
      
    // Wait for claim deadline to pass  
    vm.warp(block.timestamp + 8 days);  
      
    // Try to distribute to only 10 KOLs (batch processing attempt)  
    address[] memory batchKols = new address[](10);  
    for (uint256 i = 0; i < 10; i++) {  
        batchKols[i] = allKols[i];  
    }  
      
    // This should revert with "Index out of bounds" when i reaches 10  
    // because the loop runs 100 times (storage array length) but tries to access batchKols[10+]  
    vm.startPrank(projectCreator);  
    vm.expectRevert();  
    vesting.distributeRewardTVS(0, batchKols);  
    vm.stopPrank();  
}
```
 
### Mitigation:

Use kol.length as the loop bound and access kol[i] consistently
  