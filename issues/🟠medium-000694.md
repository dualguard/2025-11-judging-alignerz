# [000694] Division by Zero in `A26ZDividendDistributor` Causes Permanent DoS and Trapped Funds
  
  ### Summary

Missing zero-value validation for `vestingPeriod` and `totalUnclaimedAmounts` in `A26ZDividendDistributor.sol` will cause permanent denial of service and trapped funds for KOL dividend recipients as the owner can deploy a distributor with `vestingPeriod = 0` causing all `claimDividends()` calls to revert due to division by zero.


### Root Cause


In [`A26ZDividendDistributor.sol:201`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L201) there is a division by `vestingPeriod` without checking if it's zero:
```solidity
uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
```

In [`A26ZDividendDistributor.sol:218`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L218) there is a division by `totalUnclaimedAmounts` without checking if it's zero:
```solidity
dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
```

In [`A26ZDividendDistributor.sol:153`](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L153) there is a division by `vestingPeriods[i]` without checking if it's zero:
```solidity
uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
```

The constructor at line 70-80 does not validate that `_vestingPeriod > 0`, allowing deployment with zero vesting period.

### Internal Pre-conditions

1. Owner needs to deploy `A26ZDividendDistributor` with `_vestingPeriod` set to exactly `0`
2. Owner needs to call `setUpTheDividends()` to configure dividend distribution
3. Stablecoin balance in the contract needs to be at least `1` (any non-zero amount)

### External Pre-conditions

None required.

### Attack Path

1. Owner deploys `A26ZDividendDistributor` with `vestingPeriod = 0` (either maliciously or by mistake)
2. Owner transfers 100,000 USDC to the distributor contract
3. Owner calls `setUpTheDividends()` which successfully sets up dividend allocations for KOL users
4. Time passes and KOL users attempt to claim their dividends
5. KOL user calls `claimDividends()` which calculates `claimableAmount = totalAmount * claimableSeconds / vestingPeriod`
6. Since `vestingPeriod = 0`, the transaction reverts with "division by zero" panic
7. All KOL users are permanently unable to claim their dividends
8. The 100,000 USDC remains trapped in the contract forever (unless owner rescues via `withdrawStuckTokens`)

### Impact

The KOL dividend recipients suffer a complete loss of their allocated dividends (up to 100,000 USDC in the demonstrated scenario). Users cannot claim any dividends as every call to `claimDividends()` reverts. The funds remain permanently trapped in the contract. Only the owner can rescue the funds via `withdrawStuckTokens()`, but legitimate users have no recourse to claim their entitled dividends.

### PoC

Save the following test in `protocol/test/M01_DivisionByZeroDividendDistributor.t.sol`

Run with:
```bash
cd protocol && forge test --match-contract M01_DivisionByZeroDividendDistributorTest -vvv
```

### POC Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract MockVestingForDivByZero {
    mapping(uint256 => IAlignerzVesting.Allocation) public allocations;
    uint256 public totalMinted;

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
        if (nftId >= totalMinted) {
            totalMinted = nftId + 1;
        }
    }

    function allocationOf(uint256 nftId) external view returns (IAlignerzVesting.Allocation memory) {
        return allocations[nftId];
    }
}

contract MockNFTForDivByZero {
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

contract VulnerableDividendDistributorForM01 is Ownable {
    using SafeERC20 for IERC20;

    struct Dividend {
        uint256 amount;
        uint256 claimedSeconds;
    }

    uint256 public startTime;
    uint256 public vestingPeriod;
    uint256 public totalUnclaimedAmounts;
    uint256 public stablecoinAmountToDistribute;

    MockVestingForDivByZero public vesting;
    MockNFTForDivByZero public nft;
    IERC20 public stablecoin;
    IERC20 public token;

    mapping(address => Dividend) public dividendsOf;
    mapping(uint256 => uint256) public unclaimedAmountsIn;

    constructor(
        address _vesting,
        address _nft,
        address _stablecoin,
        uint256 _startTime,
        uint256 _vestingPeriod,
        address _token
    ) Ownable(msg.sender) {
        vesting = MockVestingForDivByZero(_vesting);
        nft = MockNFTForDivByZero(_nft);
        stablecoin = IERC20(_stablecoin);
        startTime = _startTime;
        vestingPeriod = _vestingPeriod;
        token = IERC20(_token);
    }

    function setUpTheDividends() external onlyOwner {
        _setAmounts();
        _setDividends();
    }

    function claimDividends() external {
        address user = msg.sender;
        uint256 totalAmount = dividendsOf[user].amount;
        uint256 claimedSecs = dividendsOf[user].claimedSeconds;
        uint256 secondsPassed;
        if (block.timestamp >= vestingPeriod + startTime) {
            secondsPassed = vestingPeriod;
            dividendsOf[user].amount = 0;
            dividendsOf[user].claimedSeconds = 0;
        } else {
            secondsPassed = block.timestamp - startTime;
            dividendsOf[user].claimedSeconds += (secondsPassed - claimedSecs);
        }
        uint256 claimableSeconds = secondsPassed - claimedSecs;
        uint256 claimableAmount = totalAmount * claimableSeconds / vestingPeriod;
        stablecoin.safeTransfer(user, claimableAmount);
    }

    function _setAmounts() internal {
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
        totalUnclaimedAmounts = _getTotalUnclaimedAmounts();
    }

    function _getTotalUnclaimedAmounts() internal returns (uint256 total) {
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len; i++) {
            try nft.extOwnerOf(i) returns (address) {
                total += _getUnclaimedAmounts(i);
            } catch {}
        }
    }

    function _getUnclaimedAmounts(uint256 nftId) internal returns (uint256 amount) {
        IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId);
        if (address(token) != address(alloc.token)) return 0;

        uint256 len = alloc.amounts.length;
        for (uint256 i; i < len; i++) {
            if (alloc.claimedFlows[i]) continue;
            if (alloc.claimedSeconds[i] == 0) {
                amount += alloc.amounts[i];
                continue;
            }
            uint256 claimedAmt = alloc.claimedSeconds[i] * alloc.amounts[i] / alloc.vestingPeriods[i];
            amount += alloc.amounts[i] - claimedAmt;
        }
        unclaimedAmountsIn[nftId] = amount;
    }

    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint256 i; i < len; i++) {
            try nft.extOwnerOf(i) returns (address owner) {
                dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            } catch {}
        }
    }

    function withdrawStuckTokens(address _token, uint256 amount) external onlyOwner {
        IERC20(_token).safeTransfer(msg.sender, amount);
    }
}

contract M01_DivisionByZeroDividendDistributorTest is Test {
    VulnerableDividendDistributorForM01 public distributor;
    MockVestingForDivByZero public mockVesting;
    MockNFTForDivByZero public mockNft;
    Aligners26 public token;
    MockUSD public stablecoin;

    address public owner;
    address public kol1;
    address public kol2;
    address public kol3;

    uint256 constant DIVIDEND_AMOUNT = 100_000e6;
    uint256 constant TOKEN_ALLOCATION = 10_000 ether;
    uint256 constant VESTING_PERIOD = 90 days;

    function setUp() public {
        owner = address(this);
        kol1 = makeAddr("kol1");
        kol2 = makeAddr("kol2");
        kol3 = makeAddr("kol3");

        token = new Aligners26("A26Z Token", "A26Z");
        stablecoin = new MockUSD();

        mockVesting = new MockVestingForDivByZero();
        mockNft = new MockNFTForDivByZero();
    }

    function _setupDistributor(uint256 startTime, uint256 vestingPeriod) internal {
        distributor = new VulnerableDividendDistributorForM01(
            address(mockVesting),
            address(mockNft),
            address(stablecoin),
            startTime,
            vestingPeriod,
            address(token)
        );
    }

    function _createAllocation(
        uint256 nftId,
        address kolAddr,
        uint256 amount,
        uint256 vestingPeriod,
        uint256 claimedSecs
    ) internal {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = amount;

        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = vestingPeriod;

        uint256[] memory vestingStartTimes = new uint256[](1);
        vestingStartTimes[0] = block.timestamp;

        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = claimedSecs;

        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        mockNft.setOwner(nftId, kolAddr);
        mockVesting.setAllocation(
            nftId,
            amounts,
            vestingPeriods,
            vestingStartTimes,
            claimedSeconds,
            claimedFlows,
            address(token)
        );
    }

    function test_DivisionByZero_Line153_ZeroVestingPeriodInAllocation() public {
        _createAllocation(0, kol1, TOKEN_ALLOCATION, 0, 1000);
        _setupDistributor(block.timestamp, VESTING_PERIOD);

        stablecoin.transfer(address(distributor), DIVIDEND_AMOUNT);

        vm.expectRevert();
        distributor.setUpTheDividends();
    }

    function test_DivisionByZero_Line218_ZeroTotalUnclaimedAmounts() public {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = TOKEN_ALLOCATION;
        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = VESTING_PERIOD;
        uint256[] memory vestingStartTimes = new uint256[](1);
        vestingStartTimes[0] = block.timestamp;
        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 0;
        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        mockNft.setOwner(0, kol1);
        Aligners26 differentToken = new Aligners26("Different", "DIFF");
        mockVesting.setAllocation(0, amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows, address(differentToken));

        _setupDistributor(block.timestamp, VESTING_PERIOD);

        stablecoin.transfer(address(distributor), DIVIDEND_AMOUNT);

        vm.expectRevert();
        distributor.setUpTheDividends();
    }

    function test_DivisionByZero_Line201_ZeroVestingPeriod_ClaimReverts() public {
        _createAllocation(0, kol1, TOKEN_ALLOCATION, VESTING_PERIOD, 0);
        _setupDistributor(block.timestamp, 0);

        stablecoin.transfer(address(distributor), DIVIDEND_AMOUNT);

        distributor.setUpTheDividends();

        vm.warp(block.timestamp + 1 days);

        vm.prank(kol1);
        vm.expectRevert();
        distributor.claimDividends();
    }

    function test_DivisionByZero_MaxImpact_100kUSDC_TrappedForever() public {
        _createAllocation(0, kol1, TOKEN_ALLOCATION, VESTING_PERIOD, 0);
        _createAllocation(1, kol2, TOKEN_ALLOCATION, VESTING_PERIOD, 0);
        _createAllocation(2, kol3, TOKEN_ALLOCATION, VESTING_PERIOD, 0);

        _setupDistributor(block.timestamp, 0);

        stablecoin.transfer(address(distributor), DIVIDEND_AMOUNT);

        distributor.setUpTheDividends();

        assertEq(distributor.stablecoinAmountToDistribute(), DIVIDEND_AMOUNT);

        vm.warp(block.timestamp + 30 days);

        vm.prank(kol1);
        vm.expectRevert();
        distributor.claimDividends();

        vm.prank(kol2);
        vm.expectRevert();
        distributor.claimDividends();

        vm.prank(kol3);
        vm.expectRevert();
        distributor.claimDividends();

        vm.warp(block.timestamp + 365 days);

        vm.prank(kol1);
        vm.expectRevert();
        distributor.claimDividends();

        assertEq(stablecoin.balanceOf(address(distributor)), DIVIDEND_AMOUNT);
        assertEq(stablecoin.balanceOf(kol1), 0);
        assertEq(stablecoin.balanceOf(kol2), 0);
        assertEq(stablecoin.balanceOf(kol3), 0);
    }
}

```

### Mitigation

Add zero-value validation in the constructor and setter functions:

```solidity
constructor(
    address _vesting,
    address _nft,
    address _stablecoin,
    uint256 _startTime,
    uint256 _vestingPeriod,
    address _token
) Ownable(msg.sender) {
    require(_vesting != address(0), Zero_Address());
    require(_nft != address(0), Zero_Address());
    require(_stablecoin != address(0), Zero_Address());
    require(_vestingPeriod > 0, Zero_Value());  // Add this check
    // ...
}

function setVestingPeriod(uint256 _vestingPeriod) external onlyOwner {
    require(_vestingPeriod > 0, Zero_Value());  // Add this check
    vestingPeriod = _vestingPeriod;
    emit vestingPeriodSet(_vestingPeriod);
}

function _setDividends() internal {
    require(totalUnclaimedAmounts > 0, Zero_Value());  // Add this check
    // ...
}
```

Additionally, add validation in `getUnclaimedAmounts()` for `vestingPeriods[i]`:

```solidity
if (vestingPeriods[i] == 0) continue;  // Skip entries with zero vesting period
uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
```

  