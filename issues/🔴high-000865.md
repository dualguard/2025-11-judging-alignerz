# [000865] Owner is unable to set amounts and dividends inside A26ZDividendDistributor contract
  
  ### Summary

When the owner attempts to set distribution amounts using `setAmounts()` or `setUpTheDividends()`, the call eventually reaches `getUnclaimedAmounts()`. This function reverts due to an incorrect assumption about the default getter of the `allocationOf` mapping. Specifically, the default getter does not return all elements of the struct; mappings and arrays (except bytes) are omitted. As a result, the owner cannot initialize dividend distributions.

### Root Cause

In [getUnclaimedAmounts()](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L146) it is assumed that the default getter will return the entire struct. According to [Solidity documentation](https://docs.soliditylang.org/en/latest/contracts.html#getter-functions), default getters for mappings that point to structs omit arrays and mappings inside the struct, making it impossible to retrieve all necessary data. This leads to reverts when setting amounts

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

None

### Impact

- Owner cannot configure dividend distribution.
- Dividends cannot be initialized or updated, breaking core functionality of the A26ZDividendDistributor contract.

### PoC

Consider adding the following import to the top of the test file.
`import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";`

Additionaly, create a public variable:
`A26ZDividendDistributor distributor;`

Modifty the `setUp()` function:
```solidity
...
vesting = AlignerzVesting(proxy);
 vm.prank(projectCreator);
        distributor = new A26ZDividendDistributor(address(vesting), address(nft),address(usdt), block.timestamp, 7 days, address(token));
...
```
Run the test using:
`forge test --mt testDoSForDividends -vvv`

```solidity
function testDoSForDividends() public {
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
        uint256[] memory percentages = new uint256[](10);
        for (uint8 i = 0; i < 10; i++)
        {
            percentages[i] = 1000;
        }
        vm.prank(bidder);
        (uint256 oldNft, uint256[] memory newNfts) = vesting.splitTVS(PROJECT_ID, percentages, nftId);
        
        
        vm.prank(projectCreator);
        distributor.setAmounts();

    }
```

Which will give the following outputs:
```bash
A26ZDividendDistributor::setAmounts()
    │   ├─ [3284] MockUSD::balanceOf(A26ZDividendDistributor: [0x1422476607EB46CFb4925de78e2915E2A15701e9]) [staticcall]
    │   │   └─ ← [Return] 0
    │   ├─ [1018] AlignerzNFT::getTotalMinted() [staticcall]
    │   │   └─ ← [Return] 10
    │   ├─ [1680] AlignerzNFT::extOwnerOf(0) [staticcall]
    │   │   └─ ← [Revert] OwnerQueryForNonexistentToken()
    │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   └─ ← [Return] bidder0: [0xf0Ad0970EE36C6FaF41575f3736D081ceEb36083]
    │   ├─ [2325] ERC1967Proxy::fallback(1) [staticcall]
    │   │   ├─ [1596] AlignerzVesting::fallback(1) [delegatecall]
    │   │   │   └─ ← [Revert] EvmError: Revert
    │   │   └─ ← [Revert] EvmError: Revert
    │   └─ ← [Revert] EvmError: Revert
    └─ ← [Revert] EvmError: Revert

Suite result: FAILED. 0 passed; 1 failed; 0 skipped; finished in 1.35s (6.34ms CPU time)

Ran 1 test suite in 1.37s (1.35s CPU time): 0 tests passed, 1 failed, 0 skipped (1 total tests)

Failing tests:
Encountered 1 failing test in test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[FAIL: EvmError: Revert] testDoSForDividends()
```

### Mitigation

Create a custom getter that returns the entire struct, including arrays and mappings:
```solidity
function allocationOfGetter(uint256 id) external view returns (Allocation memory) {
        return allocationOf[id];
    }
```
Use this function to retrieve all necessary data safely.
  