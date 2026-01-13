# [000434] Forced ``mergeFee`` for users who get allocated to ``two`` different pools.
  
  ### Summary

Forced ``mergeFee`` for users who get allocated to ``two`` different pools.

### Root Cause

```solidity
./AlignerzVesting.sol

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

        claimedNFT[leaf] = true;

        uint256 nftId = nftContract.mint(msg.sender);
...
```
``claimNft()`` is used to claim the ``TVS NFT`` for allocated ``pool`` and amount. But, let's say, you are allocated to ``two different pools``.

> Once the bidding ends, the longest vesting schedules are served first until the pool is
depleted.
> If a pool is depleted, remaining bids are served from the next pools.

You will have to claim twice with ``two`` positions. And now if you want to ``merge them``, you need to pay the ``mergeFee`` on top of that  which can be ``2%``.

> mergeFeeRate and splitFeeRate can be set to 2% at most

But the ``user`` didn't opt in for ``two nftIds`` and ``mergeFee`` on top of that.

```solidity
./AlignerzVesting.sol

    function mergeTVS(uint256 projectId, uint256 mergedNftId, uint256[] calldata projectIds, uint256[] calldata nftIds) external returns(uint256) {
        address nftOwner = nftContract.extOwnerOf(mergedNftId);
        require(msg.sender == nftOwner, Caller_Should_Own_The_NFT());
        
        bool isBiddingProject = NFTBelongsToBiddingProject[mergedNftId];
        (Allocation storage mergedTVS, IERC20 token) = isBiddingProject ?
        (biddingProjects[projectId].allocations[mergedNftId], biddingProjects[projectId].token) :
        (rewardProjects[projectId].allocations[mergedNftId], rewardProjects[projectId].token);

        uint256[] memory amounts = mergedTVS.amounts;
        uint256 nbOfFlows = mergedTVS.amounts.length;
        (uint256 feeAmount, uint256[] memory newAmounts) = calculateFeeAndNewAmountForOneTVS(mergeFeeRate, amounts, nbOfFlows);
        mergedTVS.amounts = newAmounts;
...
```

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path

1. A user gets allocated to two different pools.

### Impact

1. It causes ``inconvenience`` to the user as apart from claiming ``twice``, now the user needs to ``merge()`` again.
2. Extra forced ``mergeFee`` is charged to the user.
3. User needs to submit ``three`` transcations. Extra gas costs too.

### PoC

_No response_

### Mitigation

``merge`` it inside the ``claimNft()`` with the internal ``_merge()`` function.
  