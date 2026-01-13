# [000005] Admin will be unable to selectively airdrop unclaimed TVS to KOLs
  
  
### Summary

The mismatch between `kolTVSAddresses.length` and the user-provided `kol` array in [`AlignerzVesting::distributeRewardTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L528) will cause post-deadline TVS redistributions to revert for reward projects as the admin will trigger an array-out-of-bounds panic whenever they try to airdrop only a subset of KOLs, effectively forcing them to pass (at least) all stored KOL addresses for the call to succeed.

### Root Cause

In reward projects, the post-deadline TVS distribution helper iterates based on the length of the internal `kolTVSAddresses` array, but indexes into the user-supplied `kol` calldata array:

```solidity
function distributeRewardTVS(
    uint256 rewardProjectId,
    address[] calldata kol
) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(
        block.timestamp > rewardProject.claimDeadline,
        Deadline_Has_Not_Passed()
    );
@>  uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i; i < len; ) {
@>      _claimRewardTVS(rewardProjectId, kol[i]);
        unchecked {
            ++i;
        }
    }
}
```

Because the loop bound is `rewardProject.kolTVSAddresses.length` but the index expression is `kol[i]`, any call where `kol.length < rewardProject.kolTVSAddresses.length` will attempt to read past the end of the calldata array and revert with `Panic(0x32)`.

This has two important consequences:

- The only way to avoid a revert is to pass a `kol` array whose length is at least `rewardProject.kolTVSAddresses.length`; in practice, this means including all KOLs that still have pending TVS allocations.
- The function cannot be safely used to selectively airdrop leftover TVS to a subset of KOLs, and for large `kolTVSAddresses` arrays the forced full-iteration also increases the risk of out-of-gas aborts when `_claimRewardTVS` is called for every stored KOL.

The same pattern appears in `AlignerzVesting::distributeStablecoinAllocation`, which uses `kolStablecoinAddresses.length` as the loop bound while indexing into the user-provided `kol` array.

### Internal Pre-conditions

1. A reward project has been launched via `AlignerzVesting::launchRewardProject`, and `RewardProject.claimDeadline` has passed so that post-deadline helpers are available.
2. `AlignerzVesting::setTVSAllocation` has been called at least once, populating `rewardProject.kolTVSAddresses` with one or more KOL addresses and leaving non-zero `kolTVSRewards` balances for them.
3. Some KOLs still have unclaimed TVS allocations when the admin decides to redistribute them using `AlignerzVesting::distributeRewardTVS`.

### External Pre-conditions

1. None beyond the ability of the admin (or any caller) to invoke `AlignerzVesting::distributeRewardTVS` with an arbitrary `kol` array after `claimDeadline`.

### Attack Path

This is a liveness / usability vulnerability rather than a profit-maximizing attack, but the failure mode can be described as follows:

1. A reward project is created, KOL TVS allocations are configured via `AlignerzVesting::setTVSAllocation`, and multiple KOL addresses are stored in `rewardProject.kolTVSAddresses`.
2. The claim window elapses; `RewardProject.claimDeadline` has passed, so direct user claims revert while the admin plans to redistribute leftover TVS using `AlignerzVesting::distributeRewardTVS`.
3. The admin prepares a `kol` calldata array that contains only a subset of the stored KOLs (for example, only active or whitelisted KOLs) and calls `distributeRewardTVS(rewardProjectId, kol)`.
4. Inside `distributeRewardTVS`, `len` is set to `rewardProject.kolTVSAddresses.length`, but the loop indexes `kol[i]`; once `i` exceeds `kol.length - 1`, the EVM attempts to read an out-of-bounds element from the calldata array and the transaction reverts with `Panic(0x32)`.
5. The only non-reverting configuration is to pass a `kol` array of length at least `len` whose early entries correspond to KOLs with non-zero allocations, effectively requiring the admin to include (and process) the full internal list rather than a chosen subset.

### Impact
A core functionality, the ability of the admin to airdrop to specific addresess post-deadline, is broken.
The affected party is the protocol / reward issuer, which cannot reliably use `AlignerzVesting::distributeRewardTVS` (and the analogous stablecoin helper) for targeted post-deadline redistributions:

- Any attempt to pass only a subset of stored KOL addresses causes the helper to revert with `Panic(0x32)`, preventing selective airdrops or partial cleanups of leftover TVS allocations.
- For projects with many KOLs, the forced full-iteration over `kolTVSAddresses` increases the chance that `distributeRewardTVS` and `_claimRewardTVS` will hit block gas limits and fail, effectively making the post-deadline redistribution path unusable in high-cardinality scenarios.


### PoC

add the following Foundry test  to `AlignerzVestingProtocolTest.t.sol` and it demonstrates that calling `AlignerzVesting::distributeRewardTVS` with fewer KOL addresses than are stored in `rewardProject.kolTVSAddresses` always reverts with `Panic(0x32)`, while providing the full list succeeds:

```solidity
    function test_POC_distributeRewardTVS_requires_full_address_list() public {
        address[] memory kols = new address[](2);
        kols[0] = makeAddr("kol-one");
        kols[1] = makeAddr("kol-two");

        uint256[] memory amounts = new uint256[](2);
        amounts[0] = 100 ether;
        amounts[1] = 200 ether;
        uint256 totalAllocation = amounts[0] + amounts[1];
        uint256 startTime = block.timestamp + 10;
        uint256 claimWindow = 1 days;

        vm.startPrank(projectCreator);
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            startTime,
            claimWindow
        );
        vesting.setTVSAllocation(
            0,
            totalAllocation,
            30 days,
            kols,
            amounts
        );
        vm.stopPrank();

        vm.warp(startTime + claimWindow + 1);

        address[] memory subset = new address[](1);
        subset[0] = kols[0];

        bytes memory panic = abi.encodeWithSignature("Panic(uint256)", 0x32);
        vm.expectRevert(panic);
      vesting.distributeRewardTVS(0, subset); // reverts: subset shorter than kolTVSAddresses

      vesting.distributeRewardTVS(0, kols);   // succeeds: kol.length matches kolTVSAddresses.length
    }
```

Run this PoC from the `protocol` directory:

```bash
forge clean && forge test --match-test test_POC_distributeRewardTVS_requires_full_address_list -vv
```

### Mitigation

Update the distribution helpers to iterate over the user-provided `kol` array length instead of the internal `kolTVSAddresses` / `kolStablecoinAddresses` arrays, so that the admin can safely choose which KOLs to process and avoid out-of-bounds reads:

```solidity
function distributeRewardTVS(
    uint256 rewardProjectId,
    address[] calldata kol
) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
    require(
        block.timestamp > rewardProject.claimDeadline,
        Deadline_Has_Not_Passed()
    );
@>  uint256 len = kol.length; // iterate over the provided list
    for (uint256 i; i < len; ) {
        _claimRewardTVS(rewardProjectId, kol[i]);
        unchecked {
            ++i;
        }
    }
}
```

Apply the same pattern to `AlignerzVesting::distributeStablecoinAllocation`, and consider adding explicit sanity checks (e.g., that each `kol[i]` has a non-zero allocation) if the protocol wants to fail-fast when passed extraneous or misaligned addresses.


  