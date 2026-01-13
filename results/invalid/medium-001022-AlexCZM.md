# [001022] Updating the project allocations may allow users to double claim
  
  ### Summary

Both [AlignerzVesting::claimNft()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L860) and [claimRefund()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L835C14-L835C25) check if the refund respectively if the NFT was already claimed.
When the admin updates the project allocations by calling `updateProjectAllocations()`, a claiming Tx may be executed between the moment the Merkle roots have been calculated (off-chain action) and the admin calls the update function. After that, since the roots have been updated, the user whose Tx was executed may claim again. 

### Root Cause

The Merkle roots are computed offchain. For this it needs the latest on chain state (who claimed what). 
The problem is that, between the moment the on-chain data is read and new roots value are set, the on chain state may have been changed. 
If either `amount`, `poolId` or a bid is moved between refundRoot and a pool's merkleRoot, the `leaf` is changed and the `claimedNFT[leaf]` check (or `claimedRefund[leaf]` in `claimRefund` case) is not enough to protect against a double claiming. 

```solidity
    function claimNFT(uint256 projectId, uint256 poolId, uint256 amount, bytes32[] calldata merkleProof)
        external
        returns (uint256)
    {
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(biddingProject.claimDeadline > block.timestamp, Deadline_Has_Passed());

        Bid storage bid = biddingProject.bids[msg.sender];
        require(bid.amount > 0, No_Bid_Found());

        // Verify merkle proof
        bytes32 leaf = keccak256(abi.encodePacked(msg.sender, amount, projectId, poolId));

        require(!claimedNFT[leaf], Already_Claimed()); 
        require(MerkleProof.verify(merkleProof, biddingProject.vestingPools[poolId].merkleRoot, leaf), Invalid_Merkle_Proof());
// ...
```

### Internal Pre-conditions

- Admin must call `updateProjectAllocations()` to update the Merkle trees. 
- At least one claiming (refund or NFT) was executed before the `updateProjectAllocations()` call. 

### External Pre-conditions

None

### Attack Path

1. Admin calls `finalizeBids()` and Alice is included in `poolID 0`
2. Admin wants to update the Alice allocation - pool Id, amount or even to move it to `refundRoot`. Let's suppose Alice is moved from `poolID 0` to `poolID 1`: 
- admin interrogates the smart contract state to compute the new roots;
- admin calls `updateProjectAllocations()` with the new roots;
- due to block order inclusion, Alice's  `claimNFT()` Tx is executed first. 
3. Since now Alice is included in `poolID 1` she can claim again, resulting in a double claiming. 

This is not a front-running type of attack; the [L2 sequencers](https://docs.arbitrum.io/learn-more/faq#do-i-need-to-pay-a-tip-or-priority-fee-for-my-arbitrum-transactions)  includes the received tx on a first came, first served basis.
```text
Transaction processing occurs in the order that the Sequencer receives them
```

### Impact

Users can claim twice their bid value, at the expense of the protocol (if second claim is refund) or at the expense of the other users (if second claim is claim NFT). 

### PoC

_No response_

### Mitigation

Consider inheriting the [Pausable](https://github.com/OpenZeppelin/openzeppelin-contracts-upgradeable/blob/master/contracts/utils/PausableUpgradeable.sol) contract. 
Apply the `whenNotPaused()` modifier to both `claimNFT()` and `claimRefund()` functions. 
Pause the contract before updating the allocation and un-pause it after.

In this way users can't claim and admin can compute and update the Merkle roots using an on-chain frozen state. 
  