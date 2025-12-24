# [000716] Dividend setup skips vested token, leading to zero allocations
  
  
## summary

A26ZDividendDistributor incorrectly skips TVSs whose token matches the distributor’s configured token, causing unclaimed amounts to be zero and breaking dividend setup.

## Finding Description

The getUnclaimedAmounts function in A26ZDividendDistributor  returns 0 whenever the vesting allocation’s token equals the distributor’s token. This inverts the intended match: TVSs that should be included are excluded.The protocol distributes stablecoin dividends via A26ZDividendDistributor, which reads each NFT’s vesting allocation from IAlignerzVesting and computes per-NFT unclaimed amounts. The flow is: getUnclaimedAmounts() → getTotalUnclaimedAmounts() → setAmounts() → setDividends(). The distributor uses IAlignerzNFT.getTotalMinted() and extOwnerOf() to scan holders, aggregates unclaimed amounts, and apportions stablecoinAmountToDistribute proportionally to each NFT’s unclaimed vesting flows.

At the core, getUnclaimedAmounts() should include allocations whose token matches the distributor’s token (the normal case). Instead, it excludes them by returning 0 when the tokens are equal, which inverts the intended logic. As a result, getTotalUnclaimedAmounts() sums to 0, and setDividends() divides by totalUnclaimedAmounts, triggering a revert or yielding no allocations. The erroneous equality check is at protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:141. The dependent totals and distribution code paths that rely on non-zero totals are at protocol/src/contracts/A26ZDividendDistributor/[A26ZDividendDistributor.sol:206–211](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207-L211) and [protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:214–219](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L220).

[affected function code](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141)

```solidity

function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {

if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;

uint256[] memory amounts = vesting.allocationOf(nftId).amounts;

uint256[] memory claimedSeconds = vesting.allocationOf(nftId).claimedSeconds;

uint256[] memory vestingPeriods = vesting.allocationOf(nftId).vestingPeriods;

bool[] memory claimedFlows = vesting.allocationOf(nftId).claimedFlows;

uint256 len = vesting.allocationOf(nftId).amounts.length;

for (uint i; i < len;) {

if (claimedFlows[i]) continue;

if (claimedSeconds[i] == 0) {

amount += amounts[i];

continue;

}

uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];

uint256 unclaimedAmount = amounts[i] - claimedAmount;

amount += unclaimedAmount;

unchecked { ++i; }

}

unclaimedAmountsIn[nftId] = amount;

}

```

Root cause: the equality check in getUnclaimedAmounts is inverted; it returns for equal tokens instead of skipping for mismatched tokens. Highest-impact scenario: the distributor is configured to reward the exact vesting token (normal operation). Every TVS then returns `0` unclaimed amount, totalUnclaimedAmounts becomes `0`, and _setDividends attempts to divide by totalUnclaimedAmounts, causing a hard revert or no allocations.

Locations:

- [`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:141`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L141)

- [`protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:206–211`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207-L211) (dependent totals)

- `protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol:214–219` (distribution relies on totals)
In the highest-impact scenario, the distributor is configured with the vesting token (standard operation). Every NFT’s vesting Allocation.token then equals the distributor’s token, so getUnclaimedAmounts() always returns 0, getTotalUnclaimedAmounts() becomes 0, and setDividends() divides by zero, reverting the setup or producing zero allocations.
## Attack Path
- Configure A26ZDividendDistributor with the vesting token used by allocationOf(nftId).token.
- Call getUnclaimedAmounts or getTotalUnclaimedAmounts; both produce zero due to the inverted equality check.
- Call setDividends; it divides by totalUnclaimedAmounts == 0 and reverts, blocking all distribution. An operator can trigger this unintentionally by f
## Impact

High. Normal dividend setup either reverts on division by zero or results in zero allocations.
## poc
```solidity
pragma solidity =0.8.29;

import {Test} from "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract MockNFT is IAlignerzNFT {
    address _owner;
    constructor(address owner_) { _owner = owner_; }
    function mint(address) external returns (uint256) { return 0; }
    function extOwnerOf(uint256) external view returns (address) { return _owner; }
    function burn(uint256) external {}
    function getTotalMinted() external view returns (uint256) { return 1; }
}

contract MockVesting is IAlignerzVesting {
    IERC20 _token;
    constructor(address token_) { _token = IERC20(token_); }
    function allocationOf(uint256) external view returns (Allocation memory alloc) {
        alloc.amounts = new uint256[](1);
        alloc.amounts[0] = 100e18;
        alloc.vestingPeriods = new uint256[](1);
        alloc.vestingPeriods[0] = 90 days;
        alloc.vestingStartTimes = new uint256[](1);
        alloc.vestingStartTimes[0] = block.timestamp;
        alloc.claimedSeconds = new uint256[](1);
        alloc.claimedSeconds[0] = 0;
        alloc.claimedFlows = new bool[](1);
        alloc.claimedFlows[0] = false;
        alloc.isClaimed = false;
        alloc.token = _token;
        alloc.assignedPoolId = 0;
    }
}

contract A26ZDividendDistributorBugTest is Test {
    Aligners26 erc20;
    MockNFT nft;
    MockVesting vesting;
    A26ZDividendDistributor distributor;
    MockUSD musd;

    function setUp() public {
        musd = new MockUSD();
        erc20 = new Aligners26("26Aligners", "A26Z");
        address user = address(0xBEEF);
        nft = new MockNFT(user);
        vesting = new MockVesting(address(erc20));

        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(musd),
            block.timestamp,
            90 days,
            address(erc20)
        );

        musd.mint(address(this), 1000e6);
        musd.transfer(address(distributor), 1000e6);
    }

    function testDividendSetupRevertsDueToZeroTotalsWhenTokenMatches() public {
        distributor.setAmounts();
        assertEq(distributor.totalUnclaimedAmounts(), 0);
        vm.expectRevert();
        distributor.setDividends();
    }

    function testGetUnclaimedAmountsReturnsZeroForMatchingToken() public {
        uint256 amount = distributor.getUnclaimedAmounts(0);
        assertEq(amount, 0);
    }

    function testGetTotalUnclaimedAmountsReturnsZero() public {
        uint256 total = distributor.getTotalUnclaimedAmounts();
        assertEq(total, 0);
    }
}
```
## Recommendation

Invert the token mismatch condition so only non-matching tokens are skipped.



```diff

- if (address(token) == address(vesting.allocationOf(nftId).token)) return 0;

+ if (address(token) != address(vesting.allocationOf(nftId).token)) return 0;

```
  