# [000700] Treasury Not Initialized Causes DoS for Fee-Related Functions
  
  ### Summary

In `AlignerzVesting.sol:352-359` the `initialize()` function never sets the `treasury` address, leaving it as `address(0)`. This will cause a denial of service for users and the protocol as any function that transfers tokens to `treasury` will revert when ERC20 tokens reject transfers to `address(0)`.


### Root Cause


In [AlignerzVesting.sol:352-359](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L352-L359) the `treasury` state variable is never initialized:

```solidity
function initialize(address _nftContract) public initializer {
    __Ownable_init(msg.sender);
    __FeesManager_init();
    __WhitelistManager_init();
    require(_nftContract != address(0), Zero_Address());
    nftContract = IAlignerzNFT(_nftContract);
    vestingPeriodDivisor = 2_592_000;
    // treasury is NEVER initialized - remains address(0)
}
```

The following functions transfer tokens to `treasury` and will revert:
- [Line 728](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L728): `placeBid()` when `bidFee > 0`
- [Line 770](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L770): `updateBid()` when `updateBidFee > 0`
- [Line 919](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L919): `withdrawPostDeadlineProfit()`
- [Line 931](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L931): `_withdrawPostDeadlineProfit()`
- [Line 1023](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1023): `mergeTVS()`
- [Line 1071](https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L1071): `splitTVS()`


### Internal Pre-conditions

1. Admin needs to deploy contract and call `initialize()` without subsequently calling `setTreasury()`
2. Admin needs to call `setBidFee()` to set `bidFee` to be greater than 0 (for placeBid DoS)
3. Admin needs to call `setUpdateBidFee()` to set `updateBidFee` to be greater than 0 (for updateBid DoS)

### External Pre-conditions

None required.

### Attack Path


Scenario 1: User Bidding DoS
1. Admin deploys and initializes the contract without setting treasury
2. Admin sets `bidFee` to a non-zero value (e.g., 0.1 USDT)
3. Admin launches a bidding project and creates a pool
4. User calls `placeBid()` with valid parameters
5. Transaction reverts because `safeTransferFrom(msg.sender, treasury, bidFee)` attempts transfer to `address(0)`
6. All users are blocked from participating in the bidding project

Scenario 2: Protocol Funds Locked
1. Admin deploys and initializes the contract without setting treasury
2. Admin launches a bidding project with `bidFee = 0` (so users can bid)
3. Users successfully place bids, depositing stablecoins into the contract
4. After the claim deadline passes, admin calls `withdrawPostDeadlineProfit()`
5. Transaction reverts because `safeTransfer(treasury, amount)` attempts transfer to `address(0)`
6. Protocol profits remain permanently locked in the contract until `setTreasury()` is called

### Impact

User Impact:
- Users cannot place bids when `bidFee > 0` - complete DoS for bidding functionality
- Users cannot update bids when `updateBidFee > 0`

Protocol Impact:
- Protocol profits (stablecoins from successful bids) are locked in the contract
- Owner cannot withdraw any profits via `withdrawPostDeadlineProfit()` or `withdrawAllPostDeadlineProfits()`
- `mergeTVS()` and `splitTVS()` functions are blocked when merge/split fees are enabled

The issue is recoverable by calling `setTreasury()`, but until then all affected functionality is completely blocked. If the admin is unaware of this requirement, significant user funds and protocol profits could be temporarily inaccessible.

### PoC

Save this POC to `protocol/test/M07_TreasuryNotInitialized.t.sol`

Run with: `forge test --match-contract M07_TreasuryNotInitializedTest -vvv`

### POC Code

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {CompleteMerkle} from "murky/src/CompleteMerkle.sol";

contract M07_TreasuryNotInitializedTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;

    address public owner;
    address public user1;
    address public user2;

    uint256 constant TOKEN_AMOUNT = 10_000_000 ether;
    uint256 constant BID_AMOUNT = 1000e6;
    uint256 constant PROJECT_ID = 0;
    uint256 constant POOL_ID = 0;

    function setUp() public {
        owner = address(this);
        user1 = makeAddr("user1");
        user2 = makeAddr("user2");

        vm.deal(owner, 100 ether);
        vm.deal(user1, 10 ether);
        vm.deal(user2, 10 ether);

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        AlignerzVesting impl = new AlignerzVesting();
        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        ERC1967Proxy proxy = new ERC1967Proxy(address(impl), initData);
        vesting = AlignerzVesting(payable(address(proxy)));

        nft.addMinter(address(vesting));

        token.transfer(owner, TOKEN_AMOUNT);
        token.approve(address(vesting), TOKEN_AMOUNT);

        usdt.mint(user1, BID_AMOUNT * 10);
        usdt.mint(user2, BID_AMOUNT * 10);
    }

    function getLeaf(
        address user,
        uint256 amount,
        uint256 projectId,
        uint256 poolId
    ) internal pure returns (bytes32) {
        return keccak256(abi.encodePacked(user, amount, projectId, poolId));
    }

    function test_TreasuryNotInitialized_RootCause() public view {
        assertEq(
            vesting.treasury(),
            address(0),
            "Treasury is address(0) - never initialized in initialize()"
        );
    }

    function test_TreasuryNotInitialized_PlaceBidDoS() public {
        assertEq(vesting.treasury(), address(0));

        vesting.setVestingPeriodDivisor(1);
        vesting.setBidFee(1e5);

        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            bytes32(0),
            false
        );

        vesting.createPool(PROJECT_ID, 5_000_000 ether, 0.01 ether, true);

        vm.startPrank(user1);
        usdt.approve(address(vesting), BID_AMOUNT + 1e5);
        vm.expectRevert();
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, 90 days);
        vm.stopPrank();
    }

    function test_TreasuryNotInitialized_FundsStuckInContract() public {
        assertEq(vesting.treasury(), address(0));

        vesting.setVestingPeriodDivisor(1);

        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            bytes32(0),
            false
        );

        vesting.createPool(PROJECT_ID, 5_000_000 ether, 0.01 ether, true);

        vm.startPrank(user1);
        usdt.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, 90 days);
        vm.stopPrank();

        CompleteMerkle merkle = new CompleteMerkle();
        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = getLeaf(user1, BID_AMOUNT, PROJECT_ID, POOL_ID);
        leaves[1] = getLeaf(user2, 0, PROJECT_ID, POOL_ID);
        bytes32 root = merkle.getRoot(leaves);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;

        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 1 days);

        vm.warp(block.timestamp + 2 days);

        uint256 contractBalance = usdt.balanceOf(address(vesting));
        assertEq(contractBalance, BID_AMOUNT, "Contract holds user funds");

        vm.expectRevert();
        vesting.withdrawPostDeadlineProfit(PROJECT_ID);

        vm.expectRevert();
        vesting.withdrawAllPostDeadlineProfits();
    }

    function test_TreasuryNotInitialized_FixRestoresFunctionality() public {
        assertEq(vesting.treasury(), address(0));

        vesting.setVestingPeriodDivisor(1);

        vesting.launchBiddingProject(
            address(token),
            address(usdt),
            block.timestamp,
            block.timestamp + 1_000_000,
            bytes32(0),
            false
        );

        vesting.createPool(PROJECT_ID, 5_000_000 ether, 0.01 ether, true);

        vm.startPrank(user1);
        usdt.approve(address(vesting), BID_AMOUNT * 5);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, 90 days);
        vm.stopPrank();

        CompleteMerkle merkle = new CompleteMerkle();
        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = getLeaf(user1, BID_AMOUNT, PROJECT_ID, POOL_ID);
        leaves[1] = getLeaf(user2, 0, PROJECT_ID, POOL_ID);
        bytes32 root = merkle.getRoot(leaves);

        bytes32[] memory poolRoots = new bytes32[](1);
        poolRoots[0] = root;

        vesting.finalizeBids(PROJECT_ID, bytes32(0), poolRoots, 1 days);

        vm.warp(block.timestamp + 2 days);

        vm.expectRevert();
        vesting.withdrawPostDeadlineProfit(PROJECT_ID);

        address treasury = makeAddr("treasury");
        vesting.setTreasury(treasury);

        vesting.withdrawPostDeadlineProfit(PROJECT_ID);

        assertEq(usdt.balanceOf(treasury), BID_AMOUNT, "Treasury receives funds after fix");
        assertEq(usdt.balanceOf(address(vesting)), 0, "Contract balance is now 0");
    }
}

```

### Mitigation


Initialize treasury in the `initialize()` function or require it as a parameter:

```solidity
function initialize(address _nftContract, address _treasury) public initializer {
    __Ownable_init(msg.sender);
    __FeesManager_init();
    __WhitelistManager_init();
    require(_nftContract != address(0), Zero_Address());
    require(_treasury != address(0), Zero_Address());
    nftContract = IAlignerzNFT(_nftContract);
    treasury = _treasury;
    vestingPeriodDivisor = 2_592_000;
}
```

Alternatively, add a check in functions that use treasury to ensure it's set:

```solidity
modifier treasurySet() {
    require(treasury != address(0), "Treasury not initialized");
    _;
}
```
  