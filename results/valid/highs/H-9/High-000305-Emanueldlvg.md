# [000305] Function signature collision for allocationOf function on AlignerzVesting.sol contract, lead to getUnclaimedAmounts on A26ZDividendDistributor.sol contract always revert
  
  ### Summary

On `AlignerzVesting.sol` contract there is a public mapping, `allocationOf` :

```solidity
mapping(uint256 => Allocation) public allocationOf;
```

This will generate auto getter functions :

```solidity
function allocationOf(uint256) external view returns (Allocation memory);
```

if we see on `IAlignerzVesting.sol` contract, there is same function :

```solidity
function allocationOf(uint256 nftId) external view returns (Allocation memory);
```

This will make a signature collision issue.

The problem arises when `allocationOf()` function called from `A26ZDividendDistributor.sol` , more precisely on the `getUnclaimedAmounts()` function. This is main function to set up the dividends. 

Thus, this has an impact that dividends cannot be distributed to TVS holders because the main function does not work at all.

### Root Cause

In [AlignerzVesting.sol:114](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L114) & in [IAlignerzVesting.sol:67](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/interfaces/IAlignerzVesting.sol#L67) `allocationOf()` function make signature collision issue

### Internal Pre-conditions

-

### External Pre-conditions

-

### Attack Path

1. There are some TVS holder
2. Owner transfer reward token to `A26ZDividendDistributor` contract
3. Owner call `setUpTheDividends()` and end up revert because of the issue explained above

### Impact

This has an impact that dividends cannot be distributed to TVS holders because the main function does not work at all.

### PoC

1. Add this on AlignerzVestingProtocolTest.t.sol 

```solidity
import {A26ZDividendDistributor} from "src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
```

2. Add this too

```solidity
A26ZDividendDistributor public dividenDistributor;
```

3. Add this to `setUp()` function 

```solidity
dividenDistributor = new A26ZDividendDistributor(address(vesting), address(nft), address(usdt), block.timestamp, 1 days, address(token));
```

4. Add test and run `forge test --match-test testFunctionSignatureCollision`

```solidity
function testFunctionSignatureCollision() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(address(token), address(usdt), block.timestamp, block.timestamp + 1_000_000, "0x0", true);

        // 2. Create multiple pools with different prices
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.02 ether, false);
        vesting.createPool(PROJECT_ID, 4_000_000 ether, 0.03 ether, false);

        // 3. Place bids from different whitelisted users
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

        // 4. Update some bids
        vm.prank(bidders[0]);
        vesting.updateBid(PROJECT_ID, BIDDER_USD, 180 days);

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
        // 9. Users claim NFTs with proofs
        for (uint256 i = 0; i < 15; i++) {
            // Only accepted bidders
            address bidder = bidders[i];
            uint256 poolId = bidderPoolIds[bidder];

            vm.prank(bidder);
            uint256 nftId = vesting.claimNFT(PROJECT_ID, poolId, BIDDER_USD, bidderProofs[bidder]);
            nftIds[i] = nftId;
            vm.prank(bidder);
            bidderNFTIds[bidder] = nftId;

            // Verify NFT ownership
            assertEq(nft.ownerOf(nftId), bidder);
        }

        // 10. Some users try to claim refunds
        for (uint256 i = 15; i < NUM_BIDDERS; i++) {
            // Only Refunded bidders
            address bidder = bidders[i];
            vm.prank(bidder);
            vesting.claimRefund(PROJECT_ID, BIDDER_USD, bidderProofs[bidder]);

            // Verify USDT was returned
            assertEq(usdt.balanceOf(bidders[i]), BIDDER_USD);
        }

        // 11. Fast forward time to simulate vesting period
        vm.warp(block.timestamp + 60 days);

        usdt.mint(address(dividenDistributor), 1000);

        vm.prank(owner);
        vm.expectRevert();
        dividenDistributor.setUpTheDividends();

    }
```

Result :

```solidity
Ran 1 test for test/AlignerzVestingProtocolTest.t.sol:AlignerzVestingProtocolTest
[PASS] testFunctionSignatureCollision() (gas: 19516125)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 6.68s (12.78ms CPU time)

Ran 1 test suite in 6.73s (6.68s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```

### Mitigation

_No response_
  