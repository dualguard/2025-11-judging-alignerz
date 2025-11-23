# [000740] Uninitialized Memory Arrays in AlignerzVesting::splitTVS Cause Permanent DoS of Splitting Feature
  
  ### Summary

The `AlignerzVesting::splitTVS` function is permanently broken and unusable. The internal helper function `_computeSplitArrays` attempts to populate dynamic arrays within a memory struct without first initializing them. In Solidity, declaring a struct with dynamic arrays in memory does not allocate space for those arrays (they default to length 0). Any attempt to write data into these arrays results in an "Array Out Of Bounds " panic (0x32), causing the transaction to fail 100% of the time

### Root Cause

Solidity does not automatically allocate memory for dynamic arrays nested inside a struct. The execution attempts to access index `j` (starting at 0) on an array of size 0, triggering a Panic `0x32`
```solidity
function _computeSplitArrays( //@> function will revert 100% of the time due to uininitialized dynamic arrays
        Allocation storage allocation,
        uint256 percentage,
        uint256 nbOfFlows
    )
        internal
        view
        returns (
            Allocation memory alloc
        )
    {
        uint256[] memory baseAmounts = allocation.amounts;
        uint256[] memory baseVestings = allocation.vestingPeriods;
       // .........
        alloc.token = allocation.token;
        for (uint256 j; j < nbOfFlows;) { // @> j starts at 0, but the dynamic arrays in alloc are uninitialized, leading to panic 0x32
          //...........
            alloc.claimedFlows[j] = baseClaimedFlows[j];
            unchecked {
                ++j;
            }
        }
    }
```

### Internal Pre-conditions

1. The `splitTVS function` is called by a user who owns a valid Alignerz NFT (TVS).

2. The input parameters (percentages) are valid.

### External Pre-conditions

None

### Attack Path

1. A user (e.g., a KOL) has been allocated a TVS and holds the corresponding NFT.

2. The user calls `splitTVS(projectId, percentages, nftId)`to split their allocation.

3. The contract validates ownership and parameters.

4. The contract calls` _computeSplitArrays` to calculate the new allocation values.

5. The loop inside` _computeSplitArrays` attempts to assign `alloc.amounts[0].`

6. The EVM detects an out-of-bounds write operation and throws `Panic(0x32)`.

7.  The transaction reverts, and the user is unable to split their TVS.

### Impact

No user can ever successfully split a TVS. While funds are not directly stolen, the protocol fails to deliver advertised functionality, locking users into their original allocation structures.

### PoC

The following test was ran with the command: 
```bash
 forge test --match-test testSplitTVSReverts --force -vvvv
``` 
And produced this panic logs: 
```bash
Ran 1 test for test/PoC.t.sol:SplitTVSPoC
[PASS] testSplitTVSReverts() (gas: 725718)
Logs:
  npm warn exec The following package was not found and will be installed: @openzeppelin/upgrades-core@1.44.2

  splitTVS reverted as expected due to uninitialized arrays
///....................................................
 ├─ [0] VM::prank(kol: [0x33d2D4463aa670248f3281346574C17bC52b5C4C])
    │   └─ ← [Return] 
    ├─ [0] VM::expectRevert(custom error 0xf4844814)
    │   └─ ← [Return] 
    ├─ [9877] ERC1967Proxy::splitTVS(0, [5000, 5000], 1)
    │   ├─ [9533] AlignerzVesting::splitTVS(0, [5000, 5000], 1) [delegatecall]
    │   │   ├─ [1626] AlignerzNFT::extOwnerOf(1) [staticcall]
    │   │   │   └─ ← [Return] kol: [0x33d2D4463aa670248f3281346574C17bC52b5C4C]
    │   │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    │   └─ ← [Revert] panic: array out-of-bounds access (0x32)
    ├─ [0] console::log("splitTVS reverted as expected due to uninitialized arrays") [staticcall]
    │   └─ ← [Stop] 
    └─ ← [Return] 

Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 8.24s (613.30µs CPU time)
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
  
contract SplitTVSPoC is Test {  
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
          
        // Deploy vesting contract as proxy  
        address payable proxy = payable(Upgrades.deployUUPSProxy(  
            "AlignerzVesting.sol",  
            abi.encodeCall(AlignerzVesting.initialize, (address(nft)))  
        ));  
        vesting = AlignerzVesting(proxy);  
          
        // Set NFT minter to vesting contract  
        nft.addMinter(address(vesting));  
          
        // Set treasury  
        vesting.setTreasury(address(1));  
          
        // Owner already has tokens from constructor, just approve them  
        token.approve(address(vesting), TVS_AMOUNT);  
    }  
      
    function testSplitTVSReverts() public {  
        // 1. Launch a reward project  
        uint256 startTime = block.timestamp + 1 days;  
        uint256 claimWindow = 30 days;  
        vesting.launchRewardProject(  
            address(token),  
            address(usdt),  
            startTime,  
            claimWindow  
        );  
          
        // 2. Set TVS allocation for a KOL  
        address[] memory kols = new address[](1);  
        kols[0] = kol;  
        uint256[] memory amounts = new uint256[](1);  
        amounts[0] = TVS_AMOUNT;  
          
        vesting.setTVSAllocation(  
            REWARD_PROJECT_ID,  
            TVS_AMOUNT,  
            VESTING_PERIOD,  
            kols,  
            amounts  
        );  
          
        // 3. KOL claims their TVS allocation (mints NFT)  
        vm.prank(kol);  
        vesting.claimRewardTVS(REWARD_PROJECT_ID);  
          
        // 4. Get the NFT ID 
        uint256 nftId = 1;  // For ERC721A, first minted NFT has ID 1
          
        // 5. Attempt to split the TVS - this will revert with panic  
        uint256[] memory percentages = new uint256[](2);  
        percentages[0] = 5000; // 50% in basis points  
        percentages[1] = 5000; // 50% in basis points  
          
        vm.prank(kol);  
        // Expect a panic (array out of bounds or similar)  
        vm.expectRevert();  
        vesting.splitTVS(REWARD_PROJECT_ID, percentages, nftId);  
          
        console.log("splitTVS reverted as expected due to uninitialized arrays");  
    }  
}
```

### Mitigation

In  `_computeSplitArrays`, allocate memory for all dynamic arrays inside the `alloc` struct before entering the loop.
  