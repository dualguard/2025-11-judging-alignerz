# [000923] [M-3] Vesting Period Can Be Set to Zero in Reward Projects (Instant Unlock / Division by Zero)
  
  ### Summary

The setTVSAllocation function lets the owner configure the vesting period for KOL reward projects, but it lacks a minimum check. Setting vestingPeriod to 0 can either cause division-by-zero errors during claims or enable immediate full token release, bypassing the intended vesting schedule.

### Root Cause

The function setTVSAllocation does not enforce any minimum value for vestingPeriod, allowing the owner to set it to zero. This absence of validation leads to potential division by zero or instant full claim in the vesting calculation.

Code Link : https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L458-L477

### Internal Pre-conditions

_No response_

### External Pre-conditions

_No response_

### Attack Path


1. The owner calls `setTVSAllocation()` and sets `vestingPeriod = 0` because the function performs no validation.

2. The zero value is stored directly in `rewardProject.vestingPeriod`.

3. When a KOL later claims rewards, the claim logic computes `secondsPassed / vestingPeriod`. With vestingPeriod equal to zero, this results in a division‑by‑zero revert or the system treating the vesting as fully completed instantly.

4. As a consequence, a KOL can immediately claim the full allocation without any vesting delay.

5. This behavior breaks the intended vesting schedule and compromises the integrity of token distribution.


### Impact

If the owner sets vestingPeriod = 0, tokens can be claimed instantly, bypassing the intended vesting schedule. This may trigger division-by-zero errors during claims, break the vesting logic, and allow immediate full token release. Such behavior misaligns incentives, undermines token lock-up mechanisms, and disrupts expected reward distribution.


### PoC

1. Create a AlignerzVesting.t.sol file inside Test folder .
2.  Run the test using:

```solidity
forge test --match-test test_ZeroVestingPeriodCausesDivByZeroRevert --match-path test/AlignerzVesting.t.sol -vvvv
```

```solidity


// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import "../src/contracts/vesting/AlignerzVesting.sol";
import {IERC20} from "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import {ERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";

contract MockNFT {
    uint256 private _nextTokenId = 1;

    function mint(address to) external returns (uint256) {
        uint256 tokenId = _nextTokenId++;
        return tokenId;
    }

    function extOwnerOf(uint256 tokenId) external view returns (address) {
        return address(this); 
    }
}

contract AlignerzVesting_PoC is Test {
    AlignerzVesting vesting;
    MockERC20 token;
    MockERC20 stablecoin;
    MockNFT nft;

    address owner = address(this);
    address kol = makeAddr("kol");

    uint256 constant REWARD_PROJECT_ID = 0;
    uint256 constant TVS_AMOUNT = 1000 ether;
    uint256 constant START_TIME = 1000000;
    uint256 constant CLAIM_WINDOW = 30 days;

    function setUp() public {
        token = new MockERC20("TVS", "TVS");
        stablecoin = new MockERC20("USDT", "USDT");
        nft = new MockNFT();

        vesting = new AlignerzVesting();
        vesting.initialize(address(nft));

        token.mint(owner, TVS_AMOUNT);
        token.approve(address(vesting), TVS_AMOUNT);

        vesting.launchRewardProject(address(token), address(stablecoin), START_TIME, CLAIM_WINDOW);
    }

    function test_ZeroVestingPeriodCausesDivByZeroRevert() public {
        uint256 vestingPeriod = 0; 

        address[] memory kols = new address[](1);
        kols[0] = kol;

        uint256[] memory amounts = new uint256[](1);
        amounts[0] = TVS_AMOUNT;

        vesting.setTVSAllocation(REWARD_PROJECT_ID, TVS_AMOUNT, vestingPeriod, kols, amounts);

        vm.warp(START_TIME + 1);

        vm.prank(kol);
        vesting.claimRewardTVS(REWARD_PROJECT_ID);

        uint256 nftId = 1; 

        vm.expectRevert(); 
        vesting.claimTokens(REWARD_PROJECT_ID, nftId);

    }
}

contract MockERC20 is ERC20 {
    constructor(string memory name, string memory symbol) ERC20(name, symbol) {}

    function mint(address to, uint256 amount) external {
        _mint(to, amount);
    }
}



```

### Mitigation

Enforce a minimum vesting period, e.g., require(vestingPeriod > 0, "Vesting period must be > 0");

  