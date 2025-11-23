# [000952] distributeRewardTVS / distributeStablecoinAllocation Logic & Access Control Are Inconsistent With Documentation
  
  ### Summary

The functions `distributeRewardTVS` and `distributeStablecoinAllocation` have documentation stating they are owner-only (`/// @notice Allows the owner to distribute ...`), but they lack the `onlyOwner` modifier, making them callable by anyone. Additionally, the implementation has a fragile design where the loop iterates based on the storage array length but reads from a caller-provided array, leading to potential out-of-bounds errors or ignored addresses.

### Root Cause

The root cause is in 
[`distributeRewardTVS`](AlignerzVesting.sol) 
[`distributeStablecoinAllocation`](AlignerzVesting.sol):

```solidity
/// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
/// @param rewardProjectId Id of the rewardProject
/// @param kol addresses of the KOLs who chose to be rewarded in stablecoin that have not claimed their tokens during the claimWindow
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {  // Missing onlyOwner
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(block.timestamp > rewardProject.claimDeadline, Deadline_Has_Not_Passed());
    uint256 len = rewardProject.kolTVSAddresses.length;  // Uses storage array length
    for (uint256 i; i < len;) {
        _claimRewardTVS(rewardProjectId, kol[i]);  // Reads from calldata array
        unchecked {
            ++i;
        }
    }
}
```

Issues:
1. **Missing access control**: No `onlyOwner` modifier despite documentation
2. **Array mismatch vulnerability**: 
   - Loop length based on `kolTVSAddresses.length`
   - But reads `kol[i]` from calldata
   - If `kol.length < len`: out-of-bounds revert
   - If `kol.length > len`: extra addresses ignored
3. **Inconsistent design**: Why accept external array if storage array already exists?


### Internal Pre-conditions

1. Owner must call `launchRewardProject` with:
   - `startTime` in the past, OR
   - `startTime + claimWindow <= block.timestamp`

### External Pre-conditions

_No response_

### Attack Path

**Scenario 1: Unauthorized distribution**
1. Project has 3 unclaimed KOL allocations: `[Alice, Bob, Charlie]`
2. Claim deadline passes
3. Malicious actor (not owner) calls `distributeRewardTVS(0, [Alice, Bob, Charlie])`
4. Function executes successfully, distributing NFTs to KOLs
5. This may be unintended if owner wanted to control distribution timing

### Impact

1. **Access control bypass**: Anyone can trigger distribution after deadline (may be unintended)

### PoC

```solidity
function testUnauthorizedDistribution() public {
    vm.startPrank(owner);
    vesting.launchRewardProject(address(token), address(stablecoin), block.timestamp, 1000);
    
    token.approve(address(vesting), 1000 ether);
    address[] memory kols = new address[](2);
    kols[0] = alice;
    kols[1] = bob;
    uint256[] memory amounts = new uint256[](2);
    amounts[0] = 500 ether;
    amounts[1] = 500 ether;
    
    vesting.setTVSAllocation(0, 1000 ether, 1000, kols, amounts);
    vm.stopPrank();
    
    // Wait for deadline to pass
    vm.warp(block.timestamp + 2000);
    
    // Malicious actor (not owner) can call distributeRewardTVS
    vm.prank(address(0x999)); // Random address
    vesting.distributeRewardTVS(0, kols); // Should revert with onlyOwner but doesn't
    
    // Distribution succeeds even though caller is not owner
}

function testArrayLengthMismatch() public {
    // ... setup same as above ...
    
    vm.warp(block.timestamp + 2000);
    
    // Provide array with wrong length
    address[] memory wrongKols = new address[](1);
    wrongKols[0] = alice;
    
    vm.prank(owner);
    vm.expectRevert(); // Out of bounds access
    vesting.distributeRewardTVS(0, wrongKols); // len=2 but array has 1 element
}
```

### Mitigation

Add access control and remove external array parameter
  