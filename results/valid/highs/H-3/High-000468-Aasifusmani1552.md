# [000468] The `_computeSplitArrays` function in `AlignerzVesting` contract tries to access elements in array without allocating memory leading to transaction revert with out of bound array access error.
  
  ### Summary

The `splitTVS` function in `AlignerzVesting` contract is used to split TVS (NFTs) into many TVSs. The `splitTVS` function uses other internal functions to correctly split TVSs. 
The `splitTVS` function use internal `_computeSplitArrays` function to correctly split TVS's allocation and its all flows into specified percentages. But there is a flaw in `_computeSplitArrays` function.
The function `_computeSplitArrays` when building `Allocation` struct writes into its dynamic memory arrays (`alloc.amounts`, `alloc.vestingPeriods`, `alloc.vestingStartTimes`, `alloc.claimedSeconds`, `alloc.claimedFlows`) without allocating them, **causing out-of-bounds (OOB) memory writes and guaranteed reverts**.
As this function is used by `splitTVS` function, this makes the complete split tvs functionality unusable.  

### Root Cause

The root cause of this issue is not allocating memory to the dynamic array of `Allocation` struct to be created by the function, here:-

```solidity
    function _computeSplitArrays(Allocation storage allocation, uint256 percentage, uint256 nbOfFlows)
        internal
        view
        returns (Allocation memory alloc)
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
        uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
        uint256[] memory baseClaimed = allocation.claimedSeconds;
        bool[] memory baseClaimedFlows = allocation.claimedFlows;
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) {
@>            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
@>            alloc.vestingPeriods[j] = baseVestings[j];
@>            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
@>            alloc.claimedSeconds[j] = baseClaimed[j];
@>            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

### Internal Pre-conditions

N/A

### External Pre-conditions

N/A

### Attack Path

Not an attack.

### Impact

1. **Function unusable** — `_computeSplitArrays` always reverts due to **OOB writes**, making it impossible to construct valid split allocations.
2. **Denial of Service** — any feature that relies on splitting allocations (e.g., NFT split) becomes non-functional.
3. **Breaks contract invariants** — upstream logic assumes this function returns a valid **Allocation struct**; instead, it halts execution.

**Severity - HIGH**.

### PoC

- To show the root cause of issue, i have added some `console.log` statements in the `AlignerzVesting` contract in the `_computeSplitArrays` function, please add the same. First import console from forge standard library as follows in `AlignerzVesting` contract:-
```solidity
import {console} from "forge-std/console.sol";
```
- Add the following console statements:-
```solidity
function _computeSplitArrays(..)
        internal
        view
        returns (Allocation memory alloc)
    {
         //rest of the logic
        for (uint256 j; j < nbOfFlows;) {
@>            console.log("before OOB error (This will get printed)");
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
@>            console.log("after OOB error (This won't get printed)");
            alloc.vestingPeriods[j] = baseVestings[j];
            alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
            alloc.claimedSeconds[j] = baseClaimed[j];
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

- To run the PoC, copy / paste the following test in `AlignerzVestingProtocolTest.t.sol` test suite and run the PoC with command `forge test --mt test_OutOfBoundArrayAccessInSplitTvsPoC -vvvv`.
**PoC**:-
```solidity
    function test_OutOfBoundArrayAccessInSplitTvsPoC() public {
        address thomas = makeAddr("Thomas");
        address bob = makeAddr("Bob");
        //deal tokens to the users, 1 million tokens to each
        deal(address(usdt), thomas, 1000_000e6);
        deal(address(usdt), bob, 1000_000e6);
        vm.prank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // Setup the project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token), address(usdt), block.timestamp, block.timestamp + 1000_000 days, "0x0", false
        );
        ///creating a pool with 2 million tokens and 1 usdt price per token
        vesting.createPool(0, 2_000_000 ether, 1e6, false);
        vm.stopPrank();

        /// Place bids for the project
        vm.startPrank(thomas);
        usdt.approve(address(vesting), 1000_000e6);
        vesting.placeBid(0, 1000_000e6, 180 days);
        vm.stopPrank();

        vm.startPrank(bob);
        usdt.approve(address(vesting), 1000_000e6);
        vesting.placeBid(0, 1000_000e6, 90 days);
        vm.stopPrank();
        //Create bind info to generate merkle proofs
        BidInfo[] memory allBids = new BidInfo[](2);

        /// Simulate off-chain allocation process
        // For simplicity, all bids are the same amount
        allBids[0] =
            BidInfo({bidder: thomas, amount: 1000_000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});
        allBids[1] = BidInfo({bidder: bob, amount: 1000_000 ether, vestingPeriod: 90 days, poolId: 0, accepted: true});

        bytes32[] memory poolRoot = new bytes32[](1);
        poolRoot[0] = generateMerkleProofs(allBids, 0);

        //finalize bids
        vm.startPrank(projectCreator);
        vesting.finalizeBids(0, bytes32(0), poolRoot, 30 days);
        vm.stopPrank();

        ///users cliam nfts
        vm.prank(thomas);
        uint256 nftIdThomas = vesting.claimNFT(0, 0, 1000_000 ether, bidderProofs[thomas]);
        assertEq(nft.ownerOf(nftIdThomas), thomas);
        vm.prank(bob);
        uint256 nftIdBob = vesting.claimNFT(0, 0, 1000_000 ether, bidderProofs[bob]);
        assertEq(nft.ownerOf(nftIdBob), bob);

        /*Now here happens the issue when thomas tries to split his nft.
        In the _computeSplitArrays function, it tries to access alloc
        struct's array's elements without allocating memory. Because of 
        this, the transaction revert with panic(0x32) out of bound array
        access. */

        //creating paramters required to split
        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000; //50%
        percentages[1] = 5000; //50%

        vm.prank(thomas);
        vm.expectRevert("panic: array out-of-bounds access (0x32)");
        vesting.splitTVS(0, percentages, nftIdThomas);
    }
```

**Logs**:-

```solidity
├─ [45563] ERC1967Proxy::fallback(0, [5000, 5000], 1)
    │   ├─ [44801] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [4318] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] Thomas: [0x8Bd5D9e4435b94CDBCF6842F520Bd2B8525c3269]
    │   │   ├─ [9944] Aligners26::transfer(ECRecover: [0x0000000000000000000000000000000000000001], 0)
    │   │   │   ├─ emit Transfer(from: ERC1967Proxy: [0xc7183455a4C133Ae270771860664b6B7ec320bB1], to: ECRecover: [0x0000000000000000000000000000000000000001], value: 0)
    │   │   │   └─ ← [Return] true
    │   │   ├─ [0] console::log("before OOB error (This will get printed)") [staticcall]
    │   │   │   └─ ← [Stop]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    └─ ← [Return]

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.89s (4.23ms CPU time)
```

### Mitigation

To mitigate the issue, first allocate memory to the dynamic arrays before accessing them, like :-
```solidity
function _computeSplitArrays(
    Allocation storage allocation,
    uint256 percentage,
    uint256 nbOfFlows
) internal view returns (Allocation memory alloc) {
    uint256[] memory baseAmounts = allocation.amounts;
    uint256[] memory baseVestings = allocation.vestingPeriods;
    uint256[] memory baseVestingStartTimes = allocation.vestingStartTimes;
    uint256[] memory baseClaimed = allocation.claimedSeconds;
    bool[] memory baseClaimedFlows = allocation.claimedFlows;

    require(
        nbOfFlows == baseAmounts.length &&
        nbOfFlows == baseVestings.length &&
        nbOfFlows == baseVestingStartTimes.length &&
        nbOfFlows == baseClaimed.length &&
        nbOfFlows == baseClaimedFlows.length,
        "Invalid nbOfFlows"
    );

    alloc.amounts = new uint256[](nbOfFlows);
    alloc.vestingPeriods = new uint256[](nbOfFlows);
    alloc.vestingStartTimes = new uint256[](nbOfFlows);
    alloc.claimedSeconds = new uint256[](nbOfFlows);
    alloc.claimedFlows = new bool[](nbOfFlows);

    alloc.assignedPoolId = allocation.assignedPoolId;
    alloc.token = allocation.token;

    for (uint256 j; j < nbOfFlows; ) {
        alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
        alloc.vestingPeriods[j] = baseVestings[j];
        alloc.vestingStartTimes[j] = baseVestingStartTimes[j];
        alloc.claimedSeconds[j] = baseClaimed[j];
        alloc.claimedFlows[j] = baseClaimedFlows[j];
        unchecked { ++j; }
    }
}

```
  