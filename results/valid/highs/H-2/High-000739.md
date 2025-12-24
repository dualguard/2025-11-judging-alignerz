# [000739] Uninitialized Memory in FeesManager.sol Causes Permanent DoS of mergeTVS
  
  ### Summary

The `mergeTVS` functionality is permanently inoperable due to implementation errors within the `FeesManager.sol` contract. Specifically, the helper function `calculateFeeAndNewAmountForOneTVS` attempts to write to a dynamic memory array that has not been allocated (initialized), resulting in a deterministic EVM `Panic (0x32)` error.

### Root Cause

The function defines `newAmounts` as a named return parameter of type `uint256[] memory`. In Solidity, declaring a dynamic array in memory does not allocate space for it; the array initializes with a length of 0.
```solidity
function calculateFeeAndNewAmountForOneTVS(uint256 feeRate, uint256[] memory amounts, uint256 length) public pure returns (uint256 feeAmount, uint256[] memory newAmounts) {
        for (uint256 i; i < length;) { //@> Uninitialized newAmounts array will cause panic 0x32
            feeAmount += calculateFeeAmount(feeRate, amounts[i]);
            newAmounts[i] = amounts[i] - feeAmount; 
        }
    }
```

### Internal Pre-conditions

1. The user attempts to use the `mergeTVS` function.

2. The input arrays passed to the function have a length of at least 1 (which is required for any meaningful merge operation).

### External Pre-conditions

None

### Attack Path

1. A user calls `AlignerzVesting::mergeTVS`to consolidate multiple vesting NFTs into one.

2. The vesting contract delegates fee calculation to `FeesManager::calculateFeeAndNewAmountForOneTVS`

3. The EVM attempts to execute the assignment `newAmounts[0] = ....`

4. The EVM checks the bounds of `newAmounts` and finds the length is 0.

5. The EVM throws `Panic(0x32) (Array Accessed Out of Bounds)`

 6. The entire transaction is reverted, and the user is unable to perform the merge.

### Impact

1. Loss of Functionality: The `mergeTVS` feature, a core utility for managing user positions, is completely broken.

### PoC

The following test was ran and produced these logs: 
```bash
Ran 1 test for test/PoC.t.sol:MergeTVSBrokenPoC
[PASS] testMergeTVSRevertsDueToFeesManagerBug() (gas: 1199648)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 7.30s (4.11ms CPU time)

Ran 1 test suite in 7.31s (7.30s CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```
The test:
```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {Upgrades} from "openzeppelin-foundry-upgrades/Upgrades.sol";
import "forge-std/console.sol";

contract MergeTVSBrokenPoC is Test {
    AlignerzVesting public vesting;
    AlignerzNFT public nft;
    Aligners26 public token;
    MockUSD public usdt;
    
    address public owner;
    address public kol;
    
    uint256 constant REWARD_PROJECT_ID = 0;
    uint256 constant TVS_AMOUNT = 1000 ether; 
    uint256 constant VESTING_PERIOD = 365 days;
    
    function setUp() public {
        owner = address(this);
        kol = makeAddr("kol");
        
        // Deploy contracts
        token = new Aligners26("26Aligners", "A26Z");
        usdt = new MockUSD();
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");
        
        // Deploy vesting contract
        address payable proxy = payable(Upgrades.deployUUPSProxy(
            "AlignerzVesting.sol",
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))
        ));
        vesting = AlignerzVesting(proxy);
        
        // Setup permissions and treasury
        nft.addMinter(address(vesting));
        vesting.setTreasury(address(1));
        
        // Owner approves vesting contract
        token.approve(address(vesting), type(uint256).max);
    }
    
    function testMergeTVSRevertsDueToFeesManagerBug() public {
        // 1. Launch a reward project
        vesting.launchRewardProject(
            address(token),
            address(usdt),
            block.timestamp + 1 days,
            30 days
        );

        // 2. Setup allocations to generate 2 separate NFTs for the KOL
        // We need arrays for the allocation function
        address[] memory kols = new address[](1);
        kols[0] = kol;
        uint256[] memory amounts = new uint256[](1);
        amounts[0] = TVS_AMOUNT;

        // --- Mint 1st NFT ---
        vesting.setTVSAllocation(REWARD_PROJECT_ID, TVS_AMOUNT, VESTING_PERIOD, kols, amounts);
        vm.prank(kol);
        vesting.claimRewardTVS(REWARD_PROJECT_ID); // NFT ID 1

        // --- Mint 2nd NFT ---
        vesting.setTVSAllocation(REWARD_PROJECT_ID, TVS_AMOUNT, VESTING_PERIOD, kols, amounts);
        vm.prank(kol);
        vesting.claimRewardTVS(REWARD_PROJECT_ID); // NFT ID 2

        // 3. Prepare for Merge
        // Merging NFT #2 into NFT #1
        uint256 targetNftId = 1;
        uint256[] memory idsToBurn = new uint256[](1);
        idsToBurn[0] = 2;
        
        uint256[] memory projectIds = new uint256[](1);
        projectIds[0] = REWARD_PROJECT_ID;

        // 4. Execute Merge
        // We expect this to fail with panic code 0x32 (Array Out of Bounds)
        // because FeesManager.sol does not initialize the `newAmounts` memory array.
        
        vm.prank(kol);
        vm.expectRevert(stdError.indexOOBError); // Expects Panic 0x32
        vesting.mergeTVS(REWARD_PROJECT_ID, targetNftId, projectIds, idsToBurn);
        
        
    }
}
```

### Mitigation

Update `FeesManager.sol` to explicitly allocate memory for the return array
  