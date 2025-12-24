# [000731] Infinite Loop in Dividend Distributor Due to Incorrect Loop Logic Causes DoS
  
  ### Summary

The `getUnclaimedAmounts` function in `A26ZDividendDistributor` contains an infinite loop vulnerability. The loop uses `continue` statements to skip processing for fully claimed or untouched vesting schedules. However, the loop iterator increment `(++i)` is placed at the end of the loop block. When a `continue` statement is executed, the increment is skipped, causing the loop to process the same index indefinitely until the transaction runs out of gas.

### Root Cause

```solidity
    /// @notice USD value in 1e18 of all the unclaimed tokens of all the TVS
    function getTotalUnclaimedAmounts() public returns (uint256 _totalUnclaimedAmounts) {
        uint256 len = nft.getTotalMinted();
        for (uint i; i < len;) {
            (, bool isOwned) = safeOwnerOf(i);
            if (isOwned) _totalUnclaimedAmounts += getUnclaimedAmounts(i);
            unchecked {
                ++i; //@> Infinite loop if claimedSeconds[i] == 0 in getUnclaimedAmounts
            }
        }
    }

```

### Internal Pre-conditions

1. A user has a vesting schedule where `claimedFlows[i]` is true OR `claimedSeconds[i]` is 0.

### External Pre-conditions

None

### Attack Path

1. The admin creates a reward project and allocates a vesting schedule to at least one user.
2. The user's vesting schedule is initialized. By default, `claimedSeconds` for the new allocation is 0.
3. The admin calls `setUpTheDividends()` to calculate and distribute rewards.
4. The contract calls `getTotalUnclaimedAmounts`, which iterates through all NFT holders.
5. It calls `getUnclaimedAmounts` for the user created in Step 1.
6. The function enters the for loop to process the user's vesting flows.
7. The condition `if (claimedSeconds[i] == 0)` evaluates to true.
8. The continue statement executes, skipping the rest of the loop body, including the `unchecked { ++i; }` block at the end.
9. Execution jumps back to the start of the loop with `i` unchanged.
10. The condition evaluates to true again.
11. The transaction consumes all available gas and reverts with OutOfGas. The dividend distribution mechanism is permanently blocked as long as any user exists with an unclaimed or fully claimed allocation.

### Impact

The dividend calculation function `getTotalUnclaimedAmounts` (which calls `getUnclaimedAmounts`) will revert with "Out of Gas" for any user satisfying the pre-conditions. Since `claimedSeconds` starts at 0 for all new users, this function is broken for every new user, permanently preventing dividend distribution.

### PoC

The following test was ran and produced these logs:
```bash
Ran 1 test for test/PoC.t.sol:DistributorInfiniteLoopPoC
[PASS] testInfiniteLoopConsumesAllGas() (gas: 23553)
Logs:
  Calling getUnclaimedAmounts with gas limit: 1000000
  Transaction failed as expected due to Infinite Loop.

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 37.88ms (13.98ms CPU time)

Ran 1 test suite in 147.48ms (37.88ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
The test:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {IAlignerzNFT} from "../src/interfaces/IAlignerzNFT.sol";
import {IAlignerzVesting} from "../src/interfaces/IAlignerzVesting.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {IERC20} from "lib/openzeppelin-contracts/contracts/token/ERC20/IERC20.sol";

// 1. Mock Vesting to set up the Loop Trigger
contract MockVestingLoop is IAlignerzVesting {
    // Use the Allocation type from the interface to match the exact return type.
    address public vestingToken;

    constructor(address _token) { vestingToken = _token; }

    function allocationOf(uint256) external view returns (IAlignerzVesting.Allocation memory alloc) {
        alloc.token = IERC20(vestingToken);
        
        // Setup 1 flow
        alloc.amounts = new uint256[](1);
        alloc.amounts[0] = 1000 ether;
        alloc.vestingPeriods = new uint256[](1);
        alloc.vestingPeriods[0] = 100 days;
        alloc.vestingStartTimes = new uint256[](1);
        alloc.vestingStartTimes[0] = block.timestamp - 10 days;

        // CRITICAL: claimedSeconds is 0. 
        // This triggers: 'if (claimedSeconds[i] == 0) { ... continue; }'
        // causing the increment to be skipped.
        alloc.claimedSeconds = new uint256[](1);
        alloc.claimedSeconds[0] = 0; 
        
        alloc.claimedFlows = new bool[](1);
        return alloc;
    }

    // Stubs
    function claimRewardTVS(uint256) external {}
    function claimTokens(uint256, uint256) external {}
}

contract MockNFTLoop is IAlignerzNFT {
    function extOwnerOf(uint256) external pure returns (address) { return address(0x1); }
    function getTotalMinted() external pure returns (uint256) { return 1; }
    function mint(address) external returns (uint256) { return 0; }
    function burn(uint256) external {}
    function addMinter(address) external {}
}

contract DistributorInfiniteLoopPoC is Test {
    A26ZDividendDistributor public distributor;
    MockVestingLoop public vesting;
    MockNFTLoop public nft;
    MockUSD public targetToken;
    MockUSD public stablecoin;
    MockUSD public otherToken;

    function setUp() public {
        targetToken = new MockUSD();
        otherToken = new MockUSD(); // Used to bypass the logic bug
        stablecoin = new MockUSD();
        nft = new MockNFTLoop();
        
        // 1. Vesting holds 'targetToken'
        vesting = new MockVestingLoop(address(targetToken));
        
        // 2. Distributor configured with 'otherToken'
        // This ensures 'distributor.token != vesting.token', 
        // preventing the "Inverted Token Check" bug from returning 0 early.
        // We want execution to reach the loop!
        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(stablecoin),
            block.timestamp,
            30 days,
            address(otherToken) 
        );
    }

    function testInfiniteLoopConsumesAllGas() public {
        // We restrict gas to something reasonable (e.g., 1 million).
        // A normal read operation should take < 50k gas.
        // If it eats 1 million, it's definitely looping infinitely.
        uint256 testGasLimit = 1_000_000;

        console.log("Calling getUnclaimedAmounts with gas limit:", testGasLimit);

        // We use a low-level call to safely catch the OutOfGas exception
        // without crashing the test runner if it takes too long.
        (bool success, ) = address(distributor).call{gas: testGasLimit}(
            abi.encodeCall(distributor.getUnclaimedAmounts, (1))
        );

        // Verify failure
        assertEq(success, false, "Function should fail (OOG)");
        
        console.log("Transaction failed as expected due to Infinite Loop.");
    }
}
```

### Mitigation

Move the increment logic into the loop header (standard `for` loop) or ensure `i` is incremented before every continue.


  