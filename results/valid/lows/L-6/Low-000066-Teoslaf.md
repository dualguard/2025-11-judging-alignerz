# [000066] Broken Batching in `distributeRewardTVS()` Causes Array Out-of-Bounds Revert
  
  ## Summary

The [`distributeRewardTVS()`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525) function is supposed to allow distributing funds to a subset of KOLs (batch processing), but it's completely broken. When you try to distribute to only some KOLs instead of all of them, the transaction reverts with an array out-of-bounds error. This means funds can become permanently locked if there are too many KOLs to process in a single transaction.

## Vulnerability Details

### Root Cause

The `distributeRewardTVS()` function accepts a `kols` array parameter intended for batch processing, but the implementation has a critical flaw: it gets the loop length from the stored array (`rewardProject.kolTVSAddresses.length`) but loops through the input parameter array (`kol[i]`):

```solidity
function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
        // ...
        // bug: Gets length from stored array (e.g., 20 KOLs)
        uint256 len = rewardProject.kolTVSAddresses.length;

        // But loops through input array kol[] which might only have 5 elements
        for (uint256 i; i < len;) {
            _claimRewardTVS(rewardProjectId, kol[i]); // Out of bounds when i >= kol.length
            unchecked {
                ++i;
            }
        }
}
```

The Bug Explained:

- If the stored array has 20 KOLs but you pass in only 5 KOLs for batch processing
- The loop runs 20 times (using `len = 20`)
- But `kol[]` only has 5 elements
- When `i` reaches 5, accessing `kol[5]` causes an out-of-bounds error

## Impact

- Batching Completely Non-Functional: The batch parameter cannot be used to process subsets of KOLs
- Permanent Fund Lock: If the number of KOLs exceeds what can be processed in a single transaction, funds become permanently stuck

## Proof of Concept

Add this test in /AlignerzVetingProtocolTest.t.sol

```solidity
    function test_BrokenBatching_distributeRewardTVS() public {
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

        // projectCreator already has tokens from setUp()
        vm.startPrank(projectCreator);
        vesting.setTVSAllocation(0, 20000 ether, 30 days, allKols, amounts);
        vm.stopPrank();

        vm.warp(block.timestamp + 61);

        address[] memory batch = new address[](5);
        for (uint256 i = 0; i < 5; i++) {
            batch[i] = allKols[i];
        }
        vm.expectRevert();
        vesting.distributeRewardTVS(0, batch);
    }
```

And run it with:

```bash
forge test --match-test test_BrokenBatching_distributeRewardTVS -vvvv
```

Output:

```bash
    │   │   ├─ emit RewardTVSClaimed(projectId: 0, kol: kol4: [0x47e859c2901Edc4852600B0dd034170849CeF1B2], nftId: 5, amount: 1000000000000000000000 [1e21], vestingPeriod: 2592000 [2.592e6])
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 2.58s (2.31ms CPU time)

Ran 1 test suite in 2.59s (2.58s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

## Recommended Mitigation

The function must stop looping over the stored KOL list and instead loop only over the KOLs provided in the batch input.

  