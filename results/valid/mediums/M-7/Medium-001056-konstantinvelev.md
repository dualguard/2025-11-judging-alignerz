# [001056] [H] Front-run of `finalizeBids()` make opportunity for last-second bid.
  
  ### Summary
The protocol closes bidding via `finalizeBids(projectId, refundRoot, merkleRoots, claimWindow)`. By design, the owner may pass empty Merkle roots at close, emit `BiddingClosed`, and only later (e.g., ~5 minutes) set the real Merkle roots with `updateProjectAllocations` after off-chain processing. Any bid mined on-chain right before `finalizeBids`and setting `biddingProject.closed = true;` is possible and it's becomes eligible for inclusion in that off-chain snapshot. This creates a last-second bidding window (MEV/front‑run) that gives timing advantage to bidders who can *submit* or *update* a “better” bid immediately before closure.

What makes the issue possible is that we are emiting even for every places and updated bid. So the attaker can simply observe or check logs to see which bid is the highes one and basically beat it by checking the mempool and do the frontrun `finalizeBids()` transaction.


```solidity
    placeBid:
        emit BidPlaced(projectId, msg.sender, amount, vestingPeriod);
```
```solidity
    updateBid:
        emit BidUpdated(projectId, msg.sender, oldAmount, newAmount, oldVestingPeriod, newVestingPeriod);
```


The execution flow was shared to the Guards and it is as follows:

```
⚠️ Attention @Guards
Clearing confusion about finalizeBids(...) and updateProjectAllocations(...).
In finalizeBids(...) Owner inputs an array of empty bytes of a lenght equal to the nb of pools. To close the project and be sure bids stopped completely. finalizeBids(...) will then emit a BiddingClosed event that will be listened to by the backend to do the calculations for refund amountd and TVS amounts for each user depending on their bid. Then 5minutes later, after the off-chain calculations are done. Owner will call updateProjectAllocations(...) passing the merkleroots generated off-chain post-calculations. The backend will listen to AllPoolAllocationsSet to allow the users to be able to call claimRefund(...) or claimNFT(...). This is desing choice, the updateProjectAllocations will not be called mid claim, only once, right after finalizeBids(...) which I agree can be refactored. This is a design choice.
```
Also in private thred it was confirmed that the merkle root is created after `finalizeBids()`
```
0xjarix

 — Yesterday at 7:23 PM
he calls finalizeBids when he wants when there are enough bids. The merkle roots will be created after finalizeBids() not before
```

### Code context (what on-chain enforces)
- Bids are accepted while: `startTime <= block.timestamp <= endTime` and `!closed`.
Its important to note that the `endTime` would be a huge number which basically will be unachivable which exactly makes the bidding unpredictable.
- The project is “closed” only when the owner calls `finalizeBids`:
```826:849:protocol/src/contracts/vesting/AlignerzVesting.sol
function finalizeBids(uint256 projectId, bytes32 refundRoot, bytes32[] calldata merkleRoots, uint256 claimWindow)
    external
    onlyOwner
{
    ...
    // Can be called with empty roots to hard-close now; real roots come later
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
- Winners/refunds are then pushed off-chain via `updateProjectAllocations` by setting Merkle roots after close.

### Impact
The impact should be High because this totaly compromize the Fairness in the bidding. Compromizing this part of the protocol can give a chance to attacker to be winner every time and claim the win NFT.

### PoC 
I cannot demonstrate the full issue because the winner is calculated off-chain, but from what the protocol is intended to do is basically take the best bid and makes him a winner.

```solidity
 function test_Bidding_LastSecondBidWindow_Frontrun_PoC() public {
        // Setup a bidding project with 1 pool
        vm.startPrank(projectCreator);
        uint256 nowTs = block.timestamp;
        uint256 startTime = nowTs;
        uint256 endTime = nowTs + 2 days;
        vesting.launchBiddingProject(address(token), address(usdt), startTime, endTime, bytes32(0), false);
        // projectCreator already approved token in setUp
        vesting.createPool(PROJECT_ID, 100_000 ether, 1e16, false); // price=0.01 in 18-dec (arbitrary)
        vm.stopPrank();

        // Two regular bidders place bids early
        address bidder0 = bidders[0];
        address bidder1 = bidders[1];
        vm.startPrank(bidder0);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 90 days);
        vm.stopPrank();
        vm.startPrank(bidder1);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 120 days);
        vm.stopPrank();

        // Attacker Frontrun the finalize call and places a bid at the very last moment before finalize
        // Use a long enough vesting that makes him win the whole bidding and claim the Winner NFT after some time.
        address attacker = bidders[2];
        vm.warp(endTime - 1);
        vm.startPrank(attacker);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 150 days); // last-second bid
        vm.stopPrank();

        // Owner immediately finalizes with empty roots (design choice: backend computes after close)
        vm.startPrank(projectCreator);
        bytes32[] memory roots = new bytes32[](1);
        roots[0] = bytes32(0); // close now, set real roots later via updateProjectAllocations
        vesting.finalizeBids(PROJECT_ID, bytes32(0), roots, 3 days);
        vm.stopPrank();
    }
```

Test results:

```solidity

    ├─ [26927] MockUSD::approve(ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], 1000000000000000000000 [1e21])
    │   ├─ emit Approval(owner: bidder2: [0x0259A6Af7B531933469a03A3257a2135Fc188978], spender: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], value: 1000000000000000000000 [1e21])
    │   └─ ← [Return] true
    ├─ [72682] ERC1967Proxy::fallback(0, 1000000000000000000000 [1e21], 12960000 [1.296e7])
    │   ├─ [71945] AlignerzVesting::placeBid(0, 1000000000000000000000 [1e21], 12960000 [1.296e7]) [delegatecall]
    │   │   ├─ [15622] MockUSD::transferFrom(bidder2: [0x0259A6Af7B531933469a03A3257a2135Fc188978], ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], 1000000000000000000000 [1e21])
    │   │   │   ├─ emit Transfer(from: bidder2: [0x0259A6Af7B531933469a03A3257a2135Fc188978], to: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], value: 1000000000000000000000 [1e21])
    │   │   │   └─ ← [Return] true
    │   │   ├─ emit BidPlaced(projectId: 0, user: bidder2: [0x0259A6Af7B531933469a03A3257a2135Fc188978], amount: 1000000000000000000000 [1e21], vestingPeriod: 12960000 [1.296e7])
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    ├─ [0] VM::startPrank(projectCreator: [0x3e221db247A1Dd4A0724cf57b6b05304A2DcA513])
    │   └─ ← [Return]
    ├─ [55366] ERC1967Proxy::fallback(0, 0x0000000000000000000000000000000000000000000000000000000000000000, [0x000000000000000000000000000000000000000000000000    │   └─ ← [Return]
    ├─ [55366] ERC1967Proxy::fallback(0, 0x0000000000000000000000000000000000000000000000000000000000000000, [0x000000000000000000000000000000000000000000000000    ├─ [55366] ERC1967Proxy::fallback(0, 0x0000000000000000000000000000000000000000000000000000000000000000, [0x0000000000000000000000000000000000000000000000000000000000000000], 259200 [2.592e5])
0000000000000000], 259200 [2.592e5])
    │   ├─ [54611] AlignerzVesting::finalizeBids(0, 0x0000000000000000000000000000000000000000000000000000000000000000, [0x0000000000000000000000000000000000000000000000000000000000000000], 259200 [2.592e5]) [delegatecall]
000000000000000000000000000], 259200 [2.592e5]) [delegatecall]
    │   │   ├─ emit PoolAllocationSet(projectId: 0, poolId: 0, merkleRoot: 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   │   ├─ emit PoolAllocationSet(projectId: 0, poolId: 0, merkleRoot: 0x0000000000000000000000000000000000000000000000000000000000000000)
    │   │   ├─ emit BiddingClosed(projectId: 0)
    │   │   └─ ← [Return]
    │   └─ ← [Return]
    ├─ [0] VM::stopPrank()
    │   └─ ← [Return]
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 5.57s (1.22ms CPU time)

Ran 1 test suite in 5.58s (5.57s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Recommended mitigations
I can mitigate this by implementing a request–approve pattern. Instead of allowing bidders to submit bids directly, they will first submit a request to place a bid. The system (or contract owner) will then review and explicitly approve or reject each request before the actual bidding is allowed. This prevents unauthorized or front-running attempts because no bid can be executed without prior approval.
  