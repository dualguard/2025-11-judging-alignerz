# [000396] Hardcoded poolId = 0 in claimRefund() causes refund loss for all non-pool-0 users
  
  ### Summary

The `claimRefund()` function is responsible for refunding users whose bids were rejected.
Refund eligibility is proven via a Merkle proof generated off-chain.

However, the function hardcodes poolId = 0 when constructing the Merkle leaf:
```solidity
uint256 poolId = 0;
bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
```

This means only users whose bids belong to pool 0 can ever produce a valid Merkle leaf. Users from pools 1-9 can never satisfy the Merkle proof, even if they are legitimately included in the refund tree.
Thus, all users bidding in pools other than pool 0 will lose their refundable stablecoins.

### Root Cause

1. Incorrect hardcoding of poolId

Refund logic assumes:
```solidity
uint256 poolId = 0;
```
Instead of deriving the user's actual pool from storage or accepting it as an input.

2. Merkle tree mismatch

Off-chain refund tree likely encodes:
```solidity
(msg.sender, amount, projectId, actualPoolId)
```

But on-chain code forces:
```solidity
(msg.sender, amount, projectId, 0)
```
Thus, the computed leaf will never match.

3. No verification of user’s actual pool

The contract does not track or use the user’s real poolId during refund. This makes the refund logic fundamentally broken.

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

.

### Impact

- Refund proofs for pools 1–9 cannot ever be claimed, even if valid.
- All non-pool-0 users will be unable to claim refunds → "loss of funds"


### PoC

_No response_

### Mitigation

1. Use the actual user pool: Retrieve bid.poolId from storage and use it:
    ```solidity
       uint256 poolId = bid.poolId;
      bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
    ```

2. Add poolId as a function parameter and validate it matches internal records:
    ```solidity
       function claimRefund(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata proof) external {
       require(biddingProjects[projectId].bids[msg.sender].poolId == poolId, Invalid_Pool());
       bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));
       ...
      }
    ```
  