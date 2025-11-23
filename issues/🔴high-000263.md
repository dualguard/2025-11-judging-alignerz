# [000263] Owner Can Prematurely Finalize Bids and Cause Denial of Service
  
  ### Summary

The finalizeBids function allows the owner to prematurely end a bidding round and immediately close the auction, regardless of whether the original bidding window has elapsed. The absence of a time-based check in finalizeBids allows the owner to terminate the bidding process at any moment, causing a denial of service for users and undermining the fairness and predictability of the auction mechanism.

The function performs the following state transitions:

```solidity
biddingProject.closed = true;
biddingProject.endTime = block.timestamp;
biddingProject.claimDeadline = block.timestamp + claimWindow;
```


Both placeBid and updateBid contain the requirement:`require(!biddingProject.closed, ...);`
As a result, once finalizeBids is called, all bidding activity is permanently disabled, even if the original endTime is still in the future.

This creates a Denial of Service (DoS) for legitimate users who expect to be able to bid or update bids until the predefined bidding period ends.

### Root Cause

The finalizeBids function lacks a time-based guard that enforces the natural end of the bidding period. Specifically, the function does not check that: `block.timestamp >= biddingProject.endTime` .

```function finalizeBids(uint256 projectId, bytes32 refundRoot, bytes32[] calldata merkleRoots, uint256 claimWindow)
        external
        onlyOwner
    {
        require(projectId < biddingProjectCount, Invalid_Project_Id());
        BiddingProject storage biddingProject = biddingProjects[projectId];
        require(!biddingProject.closed, Project_Already_Closed());


        uint256 nbOfPools = biddingProject.poolCount;
        require(merkleRoots.length == nbOfPools, Invalid_Merkle_Roots_Length());


        // Set merkle root for each pool
        for (uint256 poolId = 0; poolId < nbOfPools; poolId++) {
            require(biddingProject.vestingPools[poolId].merkleRoot == bytes32(0), Merkle_Root_Already_Set());
            biddingProject.vestingPools[poolId].merkleRoot = merkleRoots[poolId];
            emit PoolAllocationSet(projectId, poolId, merkleRoots[poolId]);
        }


        biddingProject.closed = true;
        biddingProject.endTime = block.timestamp;
        biddingProject.claimDeadline = block.timestamp + claimWindow;
        biddingProject.refundRoot = refundRoot;
        emit BiddingClosed(projectId);
    }
```

### Internal Pre-conditions

The contract must have an active bidding round with a non-zero endTime already set. finalizeBids must be callable by the privileged role (e.g., onlyOwner) without any temporal restriction

### External Pre-conditions

_No response_

### Attack Path

- The bidding round is active, and participants are still within the intended bidding window (block.timestamp < endTime).

- The owner (or any address with the privileged onlyOwner role) invokes finalizeBids().
 
- The function executes without verifying that the auction has actually reached its natural end time.
 
- finalizeBids() forcefully updates key state variables: 
```closed = true
  endTime = block.timestamp
  claimDeadline = block.timestamp + claimWindow
``` 
- With closed set to true, all user-side interactions relying on !biddingProject.closed immediately begin to fail.

- Any attempt to call placeBid() or updateBid() now reverts, even though the intended bidding period has not elapsed.
- 
- The premature finalization freezes activity and denies all remaining bidders the opportunity to submit or adjust their bids, effectively enabling a DoS on the bidding phase.

### Impact

The premature finalization freezes activity and denies all remaining bidders the opportunity to submit or adjust their bids, effectively enabling a DoS on the bidding phase.

### PoC

_No response_

### Mitigation

Enforce that finalizeBids can only be executed after the bidding period has ended:

```solidity
require(
    block.timestamp >= biddingProject.endTime,
    Bidding_Period_Not_Ended()
);
```

This ensures the auction lifecycle is deterministic and protects users from premature closure.
  