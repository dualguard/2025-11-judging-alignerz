# [000007] missing acces control on  distribution functions let KOLs bypass reward claim deadline
  
  ### Summary

Missing access control on the post-deadline reward distribution helpers [`AlignerzVesting::distributeRewardTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525) and [`AlignerzVesting::distributeStablecoinAllocation`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540) allows KOLs to bypass the configured reward claim window, as any caller can trigger `_claimRewardTVS` / `_claimStablecoinAllocation` after `claimDeadline` and still receive their allocations even though the direct claim functions revert once the deadline has passed.

### Root Cause

The reward-project claim deadline is documented as a hard cutoff after which users should no longer be able to claim TVS or refunds:

```solidity
struct RewardProject {
    // ...
@>  uint256 claimDeadline; // deadline after which it's impossible for users to claim TVS or refund
}
```

Direct KOL claim functions correctly enforce this window by reverting once `block.timestamp` is greater than or equal to `claimDeadline`:

```solidity
function claimRewardTVS(uint256 rewardProjectId) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
@>  require(
@>      block.timestamp < rewardProject.claimDeadline,
@>      Deadline_Has_Passed()
@>  );
    address kol = msg.sender;
    _claimRewardTVS(rewardProjectId, kol);
}

function claimStablecoinAllocation(uint256 rewardProjectId) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
@>  require(
@>      block.timestamp < rewardProject.claimDeadline,
@>      Deadline_Has_Passed()
@>  );
    address kol = msg.sender;
    _claimStablecoinAllocation(rewardProjectId, kol);
}
```

However, the post-deadline helpers that are documented as owner-only distribution mechanisms are implemented as unrestricted `external` functions without `onlyOwner`, and they themselves call the same internal claim logic after the deadline:

```solidity
function distributeRewardTVS(
    uint256 rewardProjectId,
    address[] calldata kol
) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
@>  require(
@>      block.timestamp > rewardProject.claimDeadline,
@>      Deadline_Has_Not_Passed()
@>  );
    uint256 len = rewardProject.kolTVSAddresses.length;
    for (uint256 i; i < len; ) {
@>      _claimRewardTVS(rewardProjectId, kol[i]); // anyone can trigger post-deadline claims
        unchecked {
            ++i;
        }
    }
}

function distributeStablecoinAllocation(
    uint256 rewardProjectId,
    address[] calldata kol
) external {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
@>  require(
@>      block.timestamp > rewardProject.claimDeadline,
@>      Deadline_Has_Not_Passed()
@>  );
    uint256 len = rewardProject.kolStablecoinAddresses.length;
    for (uint256 i; i < len; ) {
@>      _claimStablecoinAllocation(rewardProjectId, kol[i]); // same pattern for stablecoin allocations
        unchecked {
            ++i;
        }
    }
}
```

This is inconsistent with every other post-deadline path in the contract, which is explicitly restricted to `onlyOwner` while using the same `block.timestamp > claimDeadline` gating. For reward projects, the “remaining” distribution helpers are owner-only:

```solidity
function distributeRemainingRewardTVS(
    uint256 rewardProjectId
) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
@>  require(
@>      block.timestamp > rewardProject.claimDeadline,
@>      Deadline_Has_Not_Passed()
@>  );
    // ...
}

function distributeRemainingStablecoinAllocation(
    uint256 rewardProjectId
) external onlyOwner {
    RewardProject storage rewardProject = rewardProjects[rewardProjectId];
@>  require(
@>      block.timestamp > rewardProject.claimDeadline,
@>      Deadline_Has_Not_Passed()
@>  );
    // ...
}
```

Given this consistent pattern, the lack of `onlyOwner` on `distributeRewardTVS` and `distributeStablecoinAllocation` strongly indicates an implementation bug rather than an intentional design choice, and it is what allows arbitrary callers to reopen post-deadline claims for specific KOLs.

Because these functions do not restrict `msg.sender`, any KOL (or third party acting for them) can simply call `distributeRewardTVS` / `distributeStablecoinAllocation` after `claimDeadline`, include the KOL’s address in the `kol` array, and cause `_claimRewardTVS` / `_claimStablecoinAllocation` to mint the TVS NFT or transfer the stablecoins to that KOL. This directly contradicts the stated invariant that, after `claimDeadline`, it should be impossible for users to claim TVS or refunds, and effectively extends the claimability of allocations beyond the configured `claimWindow`.

### Internal Pre-conditions

1. A reward project is launched via `AlignerzVesting::launchRewardProject` with a non-zero `claimWindow`, so `RewardProject.claimDeadline` is set to `startTime + claimWindow`.
2. KOL allocations are set for that reward project via `AlignerzVesting::setTVSAllocation` and/or `AlignerzVesting::setStablecoinAllocation`, leaving non-zero entries in `RewardProject.kolTVSRewards` and/or `RewardProject.kolStablecoinRewards`.
3. At least one KOL with a non-zero allocation does not call `AlignerzVesting::claimRewardTVS` / `AlignerzVesting::claimStablecoinAllocation` before `RewardProject.claimDeadline` is reached.

### External Pre-conditions

1. None; any EOA or contract can call `AlignerzVesting::distributeRewardTVS` or `AlignerzVesting::distributeStablecoinAllocation`.

### Attack Path

1. A reward project is created and KOL allocations are funded; some KOL (the attacker) has a non-zero TVS and/or stablecoin allocation recorded but chooses not to claim during the normal claim window.
2. After `RewardProject.claimDeadline` has passed (so `claimRewardTVS` / `claimStablecoinAllocation` now revert), the attacker prepares an array `kol` that includes their own address (and optionally other KOL addresses).
3. The attacker (or any cooperating third party) calls `AlignerzVesting::distributeRewardTVS(rewardProjectId, kol)` and/or `AlignerzVesting::distributeStablecoinAllocation(rewardProjectId, kol)`.
4. The `require(block.timestamp > claimDeadline, Deadline_Has_Not_Passed())` check now passes, and the loop invokes `_claimRewardTVS` / `_claimStablecoinAllocation` for each entry in `kol`, minting TVS NFTs or transferring stablecoins to the attacker and other listed KOLs despite the claim deadline having expired.

### Impact
A core invariant is broken and the affected party is the protocol / reward issuer, which can no longer rely on `claimDeadline` to enforce a hard expiry of unclaimed TVS or stablecoin allocations: any KOL with a leftover allocation can still obtain their reward indefinitely by calling the public distribution helpers after the deadline. This breaks the documented invariant that post-deadline user claims are impossible, undermines any economic or governance assumptions built around reclaiming or repurposing unclaimed rewards after `claimDeadline`, and can lead to unexpected token outflows relative to off-chain accounting that assumes strict expiry of claims.

### PoC

Not provided.

### Mitigation

Restrict the post-deadline distribution helpers to the owner so that only protocol-controlled operations can trigger claims after `claimDeadline`, making the claim window a real cutoff for users:

```solidity
function distributeRewardTVS(
    uint256 rewardProjectId,
    address[] calldata kol
) external onlyOwner {
    // ...
}

function distributeStablecoinAllocation(
    uint256 rewardProjectId,
    address[] calldata kol
) external onlyOwner {
    // ...
}
```



  