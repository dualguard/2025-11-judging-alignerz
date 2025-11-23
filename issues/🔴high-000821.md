# [000821] Solidity's storage getter doesn't return the full dynamic arrays, causing dividend system to always revert due to never being able to fetch the arrays of `Allocation` struct
  
  ### Summary

The dividend distributor attempts to access dynamic arrays from `allocationOf(nftId)`, but Solidity's auto-generated getter for public mappings only returns simple value types, causing all dividend operations to revert.

### Root Cause


The `allocationOf` mapping is declared as `public`:

```solidity
struct Allocation {
    uint256[] amounts;
    uint256[] vestingPeriods;
    uint256[] vestingStartTimes;
    uint256[] claimedSeconds;
    bool[] claimedFlows;
    bool isClaimed;
    IERC20 token;
    uint256 assignedPoolId;
}

mapping(uint256 => Allocation) public allocationOf;
```

Solidity's auto-generated getter for `public` mappings of structs **only returns simple value types** (like `bool`, `address`, `uint256`). It does not return arrays or nested mappings. The actual signature of the auto-generated getter is:

```solidity
function allocationOf(uint256 nftId) external view returns (
    bool isClaimed,
    IERC20 token,
    uint256 assignedPoolId
);
```

However, in [A26ZDividendDistributor.sol:140-161](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L140-L161), the code attempts to access the dynamic arrays directly:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    //...
    uint256[] memory amounts = vesting.allocationOf(nftId).amounts;
    uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;
    uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;
    bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;
    uint256 len = vesting.allocationOf(nftId).amounts.length;
    // ...
}
```

Since the auto-generated getter doesn't return these arrays, attempting to access `.amounts`, `.claimedSeconds`, `.vestingPeriods`, or `.claimedFlows` will revert.

### Internal Pre-conditions

None

### External Pre-conditions

None

### Attack Path

1. Vesting system operates normally and NFTs are minted for the dividend token
2. Owner deploys the dividend distributor contract and calls `setUpTheDividends()` to initialize dividend distribution
3. `setUpTheDividends()` calls `_setAmounts()`, which calls `getTotalUnclaimedAmounts()`
4. `getUnclaimedAmounts()` attempts to access `vesting.allocationOf(nftId).amounts`
5. Transaction reverts because the auto-generated getter doesn't return the `amounts` array
6. Dividend distribution system cannot be initialized and remains permanently unusable

### Impact

The entire dividend distribution system is permanently broken and cannot be used. No dividends can ever be distributed to TVS holders because `setAmounts()` and `getTotalUnclaimedAmounts()` will always revert.

### PoC

```solidity
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

contract AlignerzVestingProtocolTest is Test {
    //...
    function test_poc() public {
        vm.startPrank(projectCreator);
        vesting.setVestingPeriodDivisor(1);

        // 1. Launch project
        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            "0x0",
            false
        );

        // 2. Create pool
        vesting.createPool(PROJECT_ID, 10_000 ether, 0.02 ether, true);
        vesting.createPool(PROJECT_ID, 10_000 ether, 0.03 ether, false);
        vm.stopPrank();

        // 3. Place bids
        vm.deal(bidders[0], 10 ether);
        usdt.mint(bidders[0], BIDDER_USD);

        vm.startPrank(bidders[0]);
        usdt.approve(address(vesting), BIDDER_USD);
        vesting.placeBid(PROJECT_ID, BIDDER_USD, 30 days);
        vm.stopPrank();

        // 4. Prepare bid allocations
        BidInfo[] memory allBidsPool0 = new BidInfo[](2);
        allBidsPool0[0] = BidInfo({
            bidder: bidders[0],
            amount: BIDDER_USD,
            vestingPeriod: 30 days,
            poolId: 0,
            accepted: true
        });

        BidInfo[] memory allBidsPool1 = new BidInfo[](2);
        allBidsPool1[1] = BidInfo({
            bidder: bidders[0],
            amount: BIDDER_USD,
            vestingPeriod: 30 days,
            poolId: 1,
            accepted: true
        });

        // 5. Generate merkle root
        bytes32[] memory poolRoots = new bytes32[](2);
        poolRoots[0] = generateMerkleProofs(allBidsPool0, 0);
        poolRoots[1] = generateMerkleProofs(allBidsPool1, 1);

        // 6. Finalize project
        vm.prank(projectCreator);
        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 60);

        // 7. Bidder claim NFTs
        vm.prank(bidders[0]);
        vesting.claimNFT(PROJECT_ID, 0, BIDDER_USD, bidderProofs[bidders[0]]);
        vm.prank(bidders[0]);
        vesting.claimNFT(PROJECT_ID, 1, BIDDER_USD, bidderProofs[bidders[0]]);

        //8. Owner setup dividends
        vm.startPrank(projectCreator);
        A26ZDividendDistributor distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),
            block.timestamp,
            30 days,
            address(token)
        );

        vm.expectRevert();
        distributor.setUpTheDividends();
        vm.stopPrank();
    }
}
```

Run:

```bash
forge clean && forge build --force
forge test --mt test_poc -vv
```


### Mitigation


Add explicit getter functions in `AlignerzVesting.sol` to return the dynamic arrays:

```solidity
function getAllocationAmounts(uint256 nftId) external view returns (uint256[] memory) {
    return allocationOf[nftId].amounts;
}

function getAllocationClaimedSeconds(uint256 nftId) external view returns (uint256[] memory) {
    return allocationOf[nftId].claimedSeconds;
}

function getAllocationVestingPeriods(uint256 nftId) external view returns (uint256[] memory) {
    return allocationOf[nftId].vestingPeriods;
}

function getAllocationClaimedFlows(uint256 nftId) external view returns (bool[] memory) {
    return allocationOf[nftId].claimedFlows;
}
```

Then update `A26ZDividendDistributor.sol` to use these explicit getters.

  