# [000077] Users in Pools with hasExtraRefund Cannot Claim Refunds
  
  ### Summary

The `claimRefund` function hardcodes `poolId=0` when constructing merkle leaves, preventing users allocated to pools with `hasExtraRefund=true` (poolId > 0) from claiming their partial refunds, as the backend will generate refund proofs with their actual poolId but the contract always verifies against `poolId=0`. As if the code is disregard `hasExtraRefunded`, and not having any fucntionality to claim if present in some pools.

### Root Cause

In[ AlignerzVesting.sol:841-843](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L841-L843), the `claimRefund` function constructs the merkle leaf with a hardcoded `poolId` of 0, regardless of which pool the user was actually allocated to:
This means users in pools 1, 2, 3, etc. with `hasExtraRefund=true` cannot claim refunds because their merkle proof will be generated with their actual poolId, but the contract will always try to verify it against `poolId=0`.

NOTE: The whitepaper of the project show a exmaple about different pools and in the white paper it show that poolId = 1 and poolId = 5 will have both refunds (althoug differnt but both pools will have refund)

Also, the comment line about `claimRefund()`[  /// @notice Allows users to claim refunds for rejected bids](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L831), clearly show that this function in only for refund the rejected bids.  And, `claimNFT()` for winner bids only provide NFT wihout refund and there is not functionality to get this refund.

### Internal Pre-conditions

1. Admin needs to call `createPool()` to create a pool with hasExtraRefund set to true and `poolId > 0`
2. Admin needs to call `closeBidding()` to set the refundRoot merkle root that includes users from pools with `poolId > 0`

### External Pre-conditions

None

### Attack Path

1. User places a bid and gets allocated to a pool with `poolId=1` and `hasExtraRefund=true` 
2. Backend generates a merkle proof for the user's refund with leaf: `keccak256(abi.encodePacked(userAddress, refundAmount, projectId, 1))`
3. User calls `claimRefund()` with their merkle proof
4. Contract constructs leaf as: `keccak256(abi.encodePacked(msg.sender, amount, projectId, 0))` , where poolId is hardcoded to zero
5. Merkle proof verification fails because the leaf hashes don't match (poolId=1 vs poolId=0)
6. User cannot claim their refund

### Impact

Users allocated to pools with `hasExtraRefund=true` and poolId > 0 cannot claim their partial refunds, resulting in a complete loss of their entitled refund amount. The protocol loses user trust and users lose funds they should be entitled to receive.

### PoC

N/A

### Mitigation

Modify `claimRefund()` to accept poolId as a parameter and use it when constructing the merkle leaf:

```diff
function claimRefund(  
    uint256 projectId,  
+    uint256 poolId,  
    uint256 amount,  
    bytes32[] calldata merkleProof  
) external {  
    // ... existing checks ...  
      
    bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));  
    // Use the provided poolId instead of hardcoded 0  
      
    // ... rest of function ...  
}
```
  