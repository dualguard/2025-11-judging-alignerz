# [000251] Missing Access Control in Reward Distribution Functions
  
  ### Summary

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L525-L535

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L540-L550

Two administrative functions responsible for distributing unclaimed rewards, `distributeRewardTVS` and `distributeStablecoinAllocation`, lack any form of access control. As a result, any address can call these functions and trigger reward distribution logic intended only for the contract owner or administrators.

### Root Cause

Both functions are declared `external` but do not implement ownership checks, role-based authorization, or any mechanism restricting who may call them. Also the natspec clearly indicated that they should be only owners function:
```solidity
/// @notice Allows the owner to distribute the TVS that have not been claimed yet to the KOLs
    function distributeRewardTVS(uint256 rewardProjectId, address[] calldata kol) external {
        ........
    }

/// @notice Allows the owner to distribute the stablecoin tokens that have not been claimed yet to the KOLs
    function distributeStablecoinAllocation(uint256 rewardProjectId, address[] calldata kol) external {
        ........
    }
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1.  `claimDeadline` expires.
2.  Attacker prepares arbitrary `kol` address list.
3.  Attacker calls either distribution function.
4.  Contract executes internal reward claiming logic without verifying
    caller legitimacy.

### Impact

-   Unauthorized callers can trigger reward distribution.
-   Execution order and business logic assumptions may be violated.
-   Gas griefing vectors introduced.
-   Potential for inconsistent distribution states.

### PoC

```solidity
    function test_DistributeRewardTVS_NoAccessControl() public {
        uint256 projectId = 0;
        address attacker = address(0xBADOO);

        address[] memory kolList = new address[](1);
        kolList[0] = address(0xCAFED00D);

        vm.prank(attacker);
        vesting.distributeRewardTVS(projectId, kolList);
    }
```

### Mitigation

- Add proper access control:
```solidity
    function distributeRewardTVS(...) external onlyOwner { ... }
    function distributeStablecoinAllocation(...) external onlyOwner { ... }
```
  