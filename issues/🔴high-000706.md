# [000706] Uninitialized Memory Array in `calculateFeeAndNewAmountForOneTVS` Causes Complete DoS of Merge and Split TVS Functions
  
  ### Summary

In `FeesManager.sol:169-174`, the `calculateFeeAndNewAmountForOneTVS` function returns an uninitialized memory array `newAmounts` which will cause a complete denial of service for users as any call to `mergeTVS()` or `splitTVS()` will revert with an out-of-bounds array access error.


### Root Cause

In [`FeesManager.sol:169-174`](https://github.com/dualguard/2025-11-alignerz/blob/main/protocol/src/contracts/vesting/feesManager/FeesManager.sol#L169-L174), the `newAmounts` return variable is declared but never initialized with a size before being accessed:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    for (uint256 i; i < length;) {
        feeAmount += calculateFeeAmount(feeRate, amounts[i]);
        newAmounts[i] = amounts[i] - feeAmount;  // @audit newAmounts is never initialized - reverts here
    }
}
```

The function attempts to write to `newAmounts[i]` but `newAmounts` has length 0 (uninitialized), causing a `panic: array out-of-bounds access (0x32)` error.

Additionally, the loop is missing the increment statement `unchecked { ++i; }`, which would cause an infinite loop even if the array was properly initialized.


### Internal Pre-conditions

1. User needs to have a valid TVS NFT with at least one allocation flow (which is the normal state after `claimNFT()`)

Note: The vulnerability triggers regardless of fee rate values - even with `splitFeeRate = 0` or `mergeFeeRate = 0`, the function still reverts because it attempts to access an uninitialized array.


### External Pre-conditions

None required.


### Attack Path

This is a vulnerability path (DoS), not an attack path:

1. User calls `claimNFT()` to receive their TVS NFT after bidding closes
2. User attempts to call `splitTVS()` to split their vesting position into multiple NFTs
3. `splitTVS()` internally calls `calculateFeeAndNewAmountForOneTVS()` at line 1069
4. Function attempts to access `newAmounts[0]` on an uninitialized array
5. Transaction reverts with `panic: array out-of-bounds access (0x32)`

Alternative path for merge:
1. User owns multiple TVS NFTs from the same project
2. User attempts to call `mergeTVS()` to consolidate their vesting positions
3. `mergeTVS()` internally calls `calculateFeeAndNewAmountForOneTVS()` at line 1013
4. Function attempts to access `newAmounts[0]` on an uninitialized array
5. Transaction reverts with `panic: array out-of-bounds access (0x32)`

### Impact

The users cannot execute `mergeTVS()` or `splitTVS()` functions. This represents a complete denial of service for two core protocol features:

- Split TVS: Users cannot divide their vesting position into smaller positions (e.g., to sell partial positions on secondary markets)
- Merge TVS: Users cannot consolidate multiple vesting positions into a single NFT for easier management

This affects 100% of users who attempt to use these features, regardless of their allocation size or project participation.


### PoC

Save the following test file to `protocol/test/C01_UninitializedMemoryArray.t.sol` and run with:
```bash
forge test --match-contract C01_UninitializedMemoryArrayTest -vvv
```

```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {Merkle} from "murky/src/Merkle.sol";

contract MockERC20 is IERC20 {
    string public name;
    string public symbol;
    uint8 public decimals;
    uint256 public totalSupply;
    mapping(address => uint256) public balanceOf;
    mapping(address => mapping(address => uint256)) public allowance;

    constructor(string memory _name, string memory _symbol, uint8 _decimals) {
        name = _name;
        symbol = _symbol;
        decimals = _decimals;
    }

    function mint(address to, uint256 amount) external {
        balanceOf[to] += amount;
        totalSupply += amount;
    }

    function transfer(address to, uint256 amount) external returns (bool) {
        balanceOf[msg.sender] -= amount;
        balanceOf[to] += amount;
        emit Transfer(msg.sender, to, amount);
        return true;
    }

    function approve(address spender, uint256 amount) external returns (bool) {
        allowance[msg.sender][spender] = amount;
        emit Approval(msg.sender, spender, amount);
        return true;
    }

    function transferFrom(address from, address to, uint256 amount) external returns (bool) {
        allowance[from][msg.sender] -= amount;
        balanceOf[from] -= amount;
        balanceOf[to] += amount;
        emit Transfer(from, to, amount);
        return true;
    }
}

contract C01_UninitializedMemoryArrayTest is Test {
    AlignerzVesting public vesting;
    AlignerzVesting public vestingImpl;
    AlignerzNFT public nft;
    MockERC20 public projectToken;
    MockERC20 public stablecoin;
    Merkle public merkle;

    address public owner = address(0x1);
    address public treasury = address(0x2);
    address public user1 = address(0x3);
    address public user2 = address(0x4);

    uint256 public constant PROJECT_ID = 0;
    uint256 public constant POOL_ID = 0;
    uint256 public constant BID_AMOUNT = 1000e6;
    uint256 public constant TOKEN_ALLOCATION = 10000e18;
    uint256 public constant TOKEN_PRICE = 1e17;
    uint256 public constant VESTING_PERIOD = 30 days;
    uint256 public constant CLAIM_WINDOW = 7 days;

    function setUp() public {
        vm.startPrank(owner);

        nft = new AlignerzNFT("Alignerz TVS", "ATVS", "https://api.alignerz.com/metadata/");

        vestingImpl = new AlignerzVesting();
        bytes memory initData = abi.encodeWithSelector(AlignerzVesting.initialize.selector, address(nft));
        ERC1967Proxy proxy = new ERC1967Proxy(address(vestingImpl), initData);
        vesting = AlignerzVesting(payable(address(proxy)));

        nft.addMinter(address(vesting));

        vesting.setTreasury(treasury);
        vesting.setFees(0, 0, 100, 100);

        projectToken = new MockERC20("Project Token", "PROJ", 18);
        stablecoin = new MockERC20("USD Coin", "USDC", 6);

        projectToken.mint(owner, 1_000_000e18);
        stablecoin.mint(user1, 100_000e6);
        stablecoin.mint(user2, 100_000e6);

        merkle = new Merkle();

        vm.stopPrank();
    }

    function _setupBiddingProject() internal returns (uint256 nftId1, uint256 nftId2) {
        vm.startPrank(owner);

        uint256 startTime = block.timestamp;
        uint256 endTime = block.timestamp + 7 days;
        bytes32 endTimeHash = keccak256(abi.encodePacked(endTime));

        vesting.launchBiddingProject(
            address(projectToken),
            address(stablecoin),
            startTime,
            endTime,
            endTimeHash,
            false
        );

        projectToken.approve(address(vesting), TOKEN_ALLOCATION);
        vesting.createPool(PROJECT_ID, TOKEN_ALLOCATION, TOKEN_PRICE, false);

        vm.stopPrank();

        vm.startPrank(user1);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        vm.startPrank(user2);
        stablecoin.approve(address(vesting), BID_AMOUNT);
        vesting.placeBid(PROJECT_ID, BID_AMOUNT, VESTING_PERIOD);
        vm.stopPrank();

        bytes32[] memory leaves = new bytes32[](2);
        leaves[0] = keccak256(abi.encodePacked(user1, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));
        leaves[1] = keccak256(abi.encodePacked(user2, TOKEN_ALLOCATION / 2, PROJECT_ID, POOL_ID));

        bytes32 root = merkle.getRoot(leaves);

        bytes32[] memory merkleRoots = new bytes32[](1);
        merkleRoots[0] = root;
        bytes32 refundRoot = bytes32(0);

        vm.startPrank(owner);
        vesting.finalizeBids(PROJECT_ID, refundRoot, merkleRoots, CLAIM_WINDOW);
        vm.stopPrank();

        bytes32[] memory proof1 = merkle.getProof(leaves, 0);
        bytes32[] memory proof2 = merkle.getProof(leaves, 1);

        vm.startPrank(user1);
        nftId1 = vesting.claimNFT(PROJECT_ID, POOL_ID, TOKEN_ALLOCATION / 2, proof1);
        vm.stopPrank();

        vm.startPrank(user2);
        nftId2 = vesting.claimNFT(PROJECT_ID, POOL_ID, TOKEN_ALLOCATION / 2, proof2);
        vm.stopPrank();
    }

    function test_C01_MergeTVS_RevertsOnUninitializedArray() public {
        (uint256 nftId1, uint256 nftId2) = _setupBiddingProject();

        vm.prank(user2);
        nft.transferFrom(user2, user1, nftId2);

        vm.startPrank(user1);

        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = PROJECT_ID;

        uint256[] memory nftIds = new uint256[](1);
        nftIds[0] = nftId2;

        vm.expectRevert();
        vesting.mergeTVS(PROJECT_ID, nftId1, projectIds, nftIds);

        vm.stopPrank();
    }

    function test_C01_SplitTVS_RevertsOnUninitializedArray() public {
        (uint256 nftId1,) = _setupBiddingProject();

        vm.startPrank(user1);

        uint256[] memory percentages = new uint256[](2);
        percentages[0] = 5000;
        percentages[1] = 5000;

        vm.expectRevert();
        vesting.splitTVS(PROJECT_ID, percentages, nftId1);

        vm.stopPrank();
    }

    function test_C01_DirectFunctionCall_RevertsOnUninitializedArray() public {
        uint256[] memory amounts = new uint256[](3);
        amounts[0] = 1000e18;
        amounts[1] = 2000e18;
        amounts[2] = 3000e18;

        vm.expectRevert();
        vesting.calculateFeeAndNewAmountForOneTVS(100, amounts, 3);
    }

    function test_C01_SingleElementArray_StillReverts() public {
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = 1000e18;

        vm.expectRevert();
        vesting.calculateFeeAndNewAmountForOneTVS(100, amounts, 1);
    }
}

```

### Mitigation

Initialize the `newAmounts` array with proper size and add the missing loop increment:

```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
    newAmounts = new uint256[](length);  // Initialize array with proper size
    for (uint256 i; i < length;) {
        uint256 fee = calculateFeeAmount(feeRate, amounts[i]);
        feeAmount += fee;
        newAmounts[i] = amounts[i] - fee;  // Deduct individual fee, not cumulative
        unchecked { ++i; }  // Add missing increment
    }
}
```

Note: The fix also corrects the fee deduction logic - the original code incorrectly deducted the cumulative `feeAmount` instead of the individual `fee` for each element.
  