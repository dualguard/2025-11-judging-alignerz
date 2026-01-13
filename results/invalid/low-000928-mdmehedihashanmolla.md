# [000928] [L-1] Early claimDividends() call causes underflow revert before startTime
  
  ### Summary

Calling claimDividends() before startTime causes an underflow in the seconds calculation, reverting the transaction. This prevents users from claiming dividends early and may confuse users or integrations, though no funds are lost.

### Root Cause

The function claimDividends() does not check if block.timestamp is earlier than startTime before subtracting, causing an underflow in Solidity 0.8+ when claims are attempted before vesting starts.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L187-L204

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path



1. A user calls **claimDividends()** before the vesting **startTime**.
2. The function executes `block.timestamp - startTime`, where `block.timestamp` is lower than `startTime`.
3. Solidity 0.8+ detects the **underflow** in this subtraction and **reverts** the entire transaction.
4. The user cannot claim dividends until `startTime` is reached, resulting in a temporary DoS of the claim function.



### Impact

Temporary denial of service for dividend claims if called before startTime. No funds are lost, but users may be confused, locked out, or waste gas on failed transactions.

### PoC

1. Navigate to test/AlignerzVestingProtocolTest.t.sol.

2.  Run the test using:

```solidity
forge test --match-test test_UnderflowReversionOnEarlyClaims --match-path test/AlignerzVestingProtocolTest.t.sol -vvvv
```

```solidity
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
    IAlignerzVesting public vesting;
    IAlignerzNFT public nft;
    IERC20 stablecoin;
    IERC20 token;
    mapping(address => Dividend) public dividendsOf;
    mapping(uint256 => uint256) unclaimedAmountsIn;

    event dividendsClaimed(address user, uint256 amountClaimed);

    error Zero_Address();

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
    mapping(uint256 => IAlignerzVesting.Allocation) public allocations;

    function setAllocation(
        uint256 nftId,
        address token,
        uint256[] memory amounts,
        uint256[] memory claimedSeconds,
        uint256[] memory vestingPeriods,
        bool[] memory claimedFlows
    ) external {
        allocations[nftId] = IAlignerzVesting.Allocation({
            token: IERC20(token),
            amounts: amounts,
            claimedSeconds: claimedSeconds,
            vestingPeriods: vestingPeriods,
            claimedFlows: claimedFlows
        });
    }

    function allocationOf(uint256 nftId) external view returns (IAlignerzVesting.Allocation memory) {
        return allocations[nftId];
    }
}

contract A26ZDividendDistributorPoC is Test {
    A26ZDividendDistributorMock distributor;
    MockERC20 stablecoin;
    MockERC20 a26zToken;
    MockNFT nft;
    MockVesting vesting;

    address owner = address(this);
    address user = makeAddr("user");

    uint256 constant START_TIME = 1000000;
    uint256 constant VESTING_PERIOD = 90 days;

    function setUp() public {
        stablecoin = new MockERC20("USDT", "USDT");
        a26zToken = new MockERC20("A26Z", "A26Z");
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

        stablecoin.mint(address(distributor), 1000 ether);

    }

    function test_UnderflowReversionOnEarlyClaims() public {
        vm.prank(user);
        vm.expectRevert();  
        distributor.claimDividends();

        vm.warp(START_TIME + 1);
        vm.prank(user);
        distributor.claimDividends();  
    }
}

```

### Mitigation

Add a simple check at the start of claimDividends() to prevent claims before startTime. For example, if (block.timestamp < startTime) return; or revert with an error. This stops the underflow and avoids failed transactions.
  