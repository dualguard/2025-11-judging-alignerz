# [000692] Missing Zero-Amount Check in `claimDividends` Causes DoS for USDT Users
  
  ### Summary

The missing zero-amount validation in `claimDividends()` before calling `safeTransfer` will cause a denial of service for TVS holders as any user calling `claimDividends()` at `startTime` or in the same block as a previous claim will trigger a zero-value transfer that reverts with USDT.

### Root Cause

In [A26ZDividendDistributor.sol:187-204](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L187-L204) there is a missing check for `claimableAmount > 0` before calling `stablecoin.safeTransfer()`:

```solidity
function claimDividends() external {
    address user = msg.sender;
    uint256 totalAmount = dividendsOf[user].amount;
    uint256 claimedSeconds = dividendsOf[user].claimedSeconds;
    uint256 secondsPassed;
    if (block.timestamp >= vestingPeriod + startTime) {
        secondsPassed = vestingPeriod;
        dividendsOf[user].amount = 0;
        dividendsOf[user].claimedSeconds = 0;
    } else {
        secondsPassed = block.timestamp - startTime;
        dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
    }
    uint256 claimableSeconds = secondsPassed - claimedSeconds;
    uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
    stablecoin.safeTransfer(user, claimableAmount); // @audit No check if claimableAmount > 0
    emit dividendsClaimed(user, claimableAmount);
}
```

When `claimableSeconds = 0` (e.g., at `startTime` or same block as previous claim), `claimableAmount` calculates to 0, and USDT's `transfer()` function reverts on zero-value transfers.

### Internal Pre-conditions

1. Owner needs to call `setUpTheDividends()` to set `dividendsOf[user].amount` to be greater than 0
2. The stablecoin used needs to be USDT (which reverts on zero-value transfers)

### External Pre-conditions

None required. USDT's behavior of reverting on zero-value transfers is a known, permanent characteristic.


### Attack Path

1. Owner deploys `A26ZDividendDistributor` with USDT as the stablecoin
2. Owner transfers USDT to the contract and calls `setUpTheDividends()` to allocate dividends to TVS holders
3. User (KOL) calls `claimDividends()` immediately at `startTime`
4. Calculation results in `claimableAmount = totalAmount * 0 / vestingPeriod = 0`
5. `stablecoin.safeTransfer(user, 0)` reverts because USDT rejects zero-value transfers
6. User cannot claim their dividends until time passes

Alternative path (repeated claim):
1. User successfully claims dividends after some time has passed
2. User calls `claimDividends()` again in the same block
3. `claimableSeconds = secondsPassed - claimedSeconds = 0` because `claimedSeconds` was just updated
4. Transfer of 0 reverts

### Impact

The TVS holders suffer a temporary denial of service and cannot claim dividends when:
- Calling at `startTime` (before any vesting time has elapsed)
- Calling twice in the same block
- When integer division rounds `claimableAmount` to 0

The protocol documentation states USDT is a supported stablecoin for bidding and rewards. While funds are not permanently lost (users can wait and retry), this creates poor UX, wastes gas on failed transactions, and may cause users to believe the contract is broken.


### PoC

Save this POC to `protocol/test/M03_NoZeroAmountCheckClaimDividends.t.sol`

Run with: `forge test --match-contract M03_NoZeroAmountCheckClaimDividendsTest -vv`

### POC Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockUSDTRevertsOnZero is ERC20 {
    constructor() ERC20("Mock USDT", "USDT") {
        _mint(msg.sender, 1_000_000_000 * 10 ** decimals());
    }

    function decimals() public pure override returns (uint8) {
        return 6;
    }

    function transfer(address to, uint256 amount) public override returns (bool) {
        require(amount > 0, "USDT: transfer amount must be greater than zero");
        return super.transfer(to, amount);
    }

    function transferFrom(address from, address to, uint256 amount) public override returns (bool) {
        require(amount > 0, "USDT: transfer amount must be greater than zero");
        return super.transferFrom(from, to, amount);
    }
}

contract VulnerableDividendDistributorM03 is Ownable {
    using SafeERC20 for IERC20;

    struct Dividend {
        uint256 amount;
        uint256 claimedSeconds;
    }

    uint256 public startTime;
    uint256 public vestingPeriod;
    IERC20 public stablecoin;

    mapping(address => Dividend) public dividendsOf;

    constructor(
        address _stablecoin,
        uint256 _startTime,
        uint256 _vestingPeriod
    ) Ownable(msg.sender) {
        stablecoin = IERC20(_stablecoin);
        startTime = _startTime;
        vestingPeriod = _vestingPeriod;
    }

    function setDividendsFor(address user, uint256 amount) external onlyOwner {
        dividendsOf[user].amount = amount;
        dividendsOf[user].claimedSeconds = 0;
    }

    function claimDividends() external {
        address user = msg.sender;
        uint256 totalAmount = dividendsOf[user].amount;
        uint256 claimedSeconds = dividendsOf[user].claimedSeconds;
        uint256 secondsPassed;

        if (block.timestamp >= vestingPeriod + startTime) {
            secondsPassed = vestingPeriod;
            dividendsOf[user].amount = 0;
            dividendsOf[user].claimedSeconds = 0;
        } else {
            secondsPassed = block.timestamp - startTime;
            dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
        }

        uint256 claimableSeconds = secondsPassed - claimedSeconds;
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;

        stablecoin.safeTransfer(user, claimableAmount);
    }
}

contract M03_NoZeroAmountCheckClaimDividendsTest is Test {
    VulnerableDividendDistributorM03 public distributor;
    MockUSDTRevertsOnZero public usdt;

    address public kol1;
    address public kol2;
    address public kol3;

    uint256 constant DIVIDEND_AMOUNT = 100_000e6;
    uint256 constant VESTING_PERIOD = 90 days;

    function setUp() public {
        kol1 = makeAddr("kol1");
        kol2 = makeAddr("kol2");
        kol3 = makeAddr("kol3");
        usdt = new MockUSDTRevertsOnZero();
    }

    function _setupDistributor(uint256 startTime, uint256 vestingPeriod) internal {
        distributor = new VulnerableDividendDistributorM03(
            address(usdt),
            startTime,
            vestingPeriod
        );
        usdt.transfer(address(distributor), DIVIDEND_AMOUNT);
    }

    function test_ImmediateClaimAtStartTime_ZeroTransfer_Reverts() public {
        _setupDistributor(block.timestamp, VESTING_PERIOD);
        distributor.setDividendsFor(kol1, DIVIDEND_AMOUNT);

        (uint256 kolAmount,) = distributor.dividendsOf(kol1);
        assertEq(kolAmount, DIVIDEND_AMOUNT);

        vm.prank(kol1);
        vm.expectRevert("USDT: transfer amount must be greater than zero");
        distributor.claimDividends();
    }

    function test_AllUsersDoS_100kUSDT_LockedInContract() public {
        _setupDistributor(block.timestamp, VESTING_PERIOD);

        distributor.setDividendsFor(kol1, DIVIDEND_AMOUNT / 3);
        distributor.setDividendsFor(kol2, DIVIDEND_AMOUNT / 3);
        distributor.setDividendsFor(kol3, DIVIDEND_AMOUNT / 3);

        uint256 contractBalanceBefore = usdt.balanceOf(address(distributor));
        assertEq(contractBalanceBefore, DIVIDEND_AMOUNT);

        vm.prank(kol1);
        vm.expectRevert("USDT: transfer amount must be greater than zero");
        distributor.claimDividends();

        vm.prank(kol2);
        vm.expectRevert("USDT: transfer amount must be greater than zero");
        distributor.claimDividends();

        vm.prank(kol3);
        vm.expectRevert("USDT: transfer amount must be greater than zero");
        distributor.claimDividends();

        assertEq(usdt.balanceOf(address(distributor)), DIVIDEND_AMOUNT);
        assertEq(usdt.balanceOf(kol1), 0);
        assertEq(usdt.balanceOf(kol2), 0);
        assertEq(usdt.balanceOf(kol3), 0);
    }

    function test_RepeatedClaimSameBlock_ZeroTransfer_Reverts() public {
        _setupDistributor(block.timestamp, VESTING_PERIOD);
        distributor.setDividendsFor(kol1, DIVIDEND_AMOUNT);

        vm.warp(block.timestamp + 30 days);

        vm.prank(kol1);
        distributor.claimDividends();

        vm.prank(kol1);
        vm.expectRevert("USDT: transfer amount must be greater than zero");
        distributor.claimDividends();
    }

    function test_NormalFlow_AfterTimePassed_Succeeds() public {
        _setupDistributor(block.timestamp, VESTING_PERIOD);
        distributor.setDividendsFor(kol1, DIVIDEND_AMOUNT);

        vm.warp(block.timestamp + 30 days);

        uint256 balanceBefore = usdt.balanceOf(kol1);
        vm.prank(kol1);
        distributor.claimDividends();
        uint256 balanceAfter = usdt.balanceOf(kol1);

        assertGt(balanceAfter, balanceBefore);
    }
}

```

### Mitigation

Add a zero-amount check before the transfer:

```diff
function claimDividends() external {
    address user = msg.sender;
    uint256 totalAmount = dividendsOf[user].amount;
    uint256 claimedSeconds = dividendsOf[user].claimedSeconds;
    uint256 secondsPassed;
    if (block.timestamp >= vestingPeriod + startTime) {
        secondsPassed = vestingPeriod;
        dividendsOf[user].amount = 0;
        dividendsOf[user].claimedSeconds = 0;
    } else {
        secondsPassed = block.timestamp - startTime;
        dividendsOf[user].claimedSeconds += (secondsPassed - claimedSeconds);
    }
    uint256 claimableSeconds = secondsPassed - claimedSeconds;
    uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
+   if (claimableAmount == 0) revert Zero_Value();
    stablecoin.safeTransfer(user, claimableAmount);
    emit dividendsClaimed(user, claimableAmount);
}
```

  