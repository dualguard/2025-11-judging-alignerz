# [000065] Broken Batching in `distributeStablecoinAllocation()` Causes Array Out-of-Bounds Revert
  
  ## Summary

The [`distributeStablecoinAllocation()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540)function accepts a batch parameter to allow processing KOLs incrementally, but this functionality is completely broken. Attempting to distribute stablecoins to a subset of KOLs causes an array out-of-bounds panic, making it impossible to use batching. On gas-constrained chains, this can permanently lock stablecoin funds if the total number of KOLs exceeds what can be processed in one transaction.

## Vulnerability Details

### Root Cause

The `distributeStablecoinAllocation()` function accepts a `kols` array parameter intended for batch processing, but the implementation has a critical flaw: it gets the loop length from the stored array (`rewardProject.kolStablecoinAddresses.length`) but loops through the input parameter array (`kol[i]`):

```solidity
function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        // bug: Gets length from stored array (e.g., 20 KOLs)
        uint256 len = rewardProject.kolStablecoinAddresses.length;

        // But loops through input array kol[] which might only have 5 elements
        for (uint256 i; i < len;) {
            _claimStablecoinAllocation(rewardProjectId, kol[i]); // Out of bounds when i >= kol.length
            unchecked {
                ++i;
            }
        }
}
```

Why This Happens:

Consider a project with 20 KOLs who haven't claimed their stablecoins. The admin wants to distribute in batches of 5 to manage gas costs:

1. Admin calls `distributeStablecoinAllocation(0, [kol0, kol1, kol2, kol3, kol4])`
2. Function reads `len = rewardProject.kolStablecoinAddresses.length` which equals 20
3. Loop attempts to iterate 20 times: `for (uint256 i; i < 20; i++)`
4. But the input array `kol[]` only has 5 elements
5. At iteration 5, accessing `kol[5]` triggers panic: array out-of-bounds (0x32)
6. Transaction reverts after successfully processing only the first 5 KOLs


## Impact

Batching is Completely Broken: Despite accepting a batch array parameter, the function cannot process subsets of KOLs. Every call must include all unclaimed KOLs or it will revert.

Permanent Fund Lock on Gas-Constrained Chains: Projects deployed on chains with lower gas limits, face permanent fund lock if they have too many KOLs. The function cannot process them in batches, and processing all at once exceeds the gas limit.


## Proof of Concept

Add this test in /AlignerzVetingProtocolTest.t.sol

```solidity
function test_BrokenBatching_distributeStablecoinAllocation() public {
    vm.startPrank(projectCreator);
    
    // Launch reward project
    vesting.launchRewardProject(
        address(token),
        address(usdt),
        block.timestamp,
        60
    );
    vm.stopPrank();
    address[] memory allKols = new address[](20);
    uint256[] memory amounts = new uint256[](20);
    
    for (uint256 i = 0; i < 20; i++) {
        allKols[i] = makeAddr(string.concat("kol", vm.toString(i)));
        amounts[i] = 1000 ether;
    }
    usdt.mint(projectCreator, 20000 ether);
    
    vm.startPrank(projectCreator);
    usdt.approve(address(vesting), 20000 ether);
    vesting.setStablecoinAllocation(0, 20000 ether, allKols, amounts);
    vm.stopPrank();

    vm.warp(block.timestamp + 61);
    
    address[] memory batch = new address[](5);
    for (uint256 i = 0; i < 5; i++) {
        batch[i] = allKols[i];
    }
    vm.expectRevert();
    vesting.distributeStablecoinAllocation(0, batch);
}
```

```bash
forge test --match-test test_BrokenBatching_distributeStablecoinAllocation -vvvv
```

Output:

```bash
    │   │   ├─ emit StablecoinAllocationClaimed(projectId: 0, kol: kol4: [0x47e859c2901Edc4852600B0dd034170849CeF1B2], amount: 1000000000000000000000 [1e21])
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.59s (2.16ms CPU time)

Ran 1 test suite in 2.60s (2.59s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommended Mitigation

The function must stop looping over the stored KOL list and instead loop only over the KOLs provided in the batch input.


  