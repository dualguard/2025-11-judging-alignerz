# [000772] allocationOf ABI / interface mismatch breaks dividend distributor reads
  
  ### Summary

The mismatch between the `allocationOf` public mapping getter and the `IAlignerzVesting.allocationOf` interface will cause dividend calculations and any external reads of allocations to revert or decode incorrectly, as external modules (like the dividend distributor) call `allocationOf` via the interface expecting a full `Allocation` struct while the implementation’s auto‑generated getter does not return it in that shape.



### Root Cause

In `AlignerzVesting`, `allocationOf` is declared as a public mapping of a struct with dynamic arrays:

```solidity
// File: AlignerzVesting.sol

struct Allocation {
    uint256[] amounts;
    uint256[] vestingPeriods;
    uint256[] vestingStartTimes;
    uint256[] claimedSeconds;
    bool[] claimedFlows;
    bool isClaimed;
    IERC20 token;
    uint256 assignedPoolId;
}

mapping(uint256 => Allocation) public allocationOf;
```

Solidity generates a public getter whose ABI returns the components in a flat tuple form, not as a single `Allocation` struct, and does not expose the dynamic arrays in the way the interface expects.

By contrast, the interface declares:

```solidity
// File: IAlignerzVesting.sol

interface IAlignerzVesting {
    struct Allocation {
        uint256[] amounts;
        uint256[] vestingPeriods;
        uint256[] vestingStartTimes;
        uint256[] claimedSeconds;
        bool[] claimedFlows;
        bool isClaimed;
        IERC20 token;
        uint256 assignedPoolId;
    }

>>  function allocationOf(uint256 nftId) external view returns (Allocation memory);
}
```

External callers compiled against `IAlignerzVesting` (like the dividend distributor) expect `allocationOf` to return a full `Allocation` struct, but the actual implementation’s getter has a different ABI, so the return data cannot be decoded as `Allocation` and the call reverts when the consumer tries to use it.

The dividend distributor relies on this interface:

```solidity
// File: A26ZDividendDistributor.sol

function getUnclaimedAmounts(uint256 nftId) public returns (uint256 amount) {
>>  IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId);

    if (address(token) != address(alloc.token)) {
        return 0;
    }

    uint256 len = alloc.amounts.length;
    for (uint256 i; i < len; ) {
        if (alloc.claimedFlows[i]) {
            unchecked { ++i; }
            continue;
        }

        if (alloc.claimedSeconds[i] == 0) {
            amount += alloc.amounts[i];
        } else {
            uint256 claimedAmount = alloc.claimedSeconds[i] * alloc.amounts[i] / alloc.vestingPeriods[i];
            uint256 unclaimedAmount = alloc.amounts[i] - claimedAmount;
            amount += unclaimedAmount;
        }

        unchecked { ++i; }
    }

    unclaimedAmountsIn[nftId] = amount;
}
```

Because `vesting.allocationOf(nftId)` does not conform to the ABI that `IAlignerzVesting.Allocation` expects, `getUnclaimedAmounts` fails when decoding `alloc` (as observed in the POC).



### Internal Pre-conditions

1. A TVS NFT has been minted and has a valid allocation stored in the vesting contract (via `claimRewardTVS`, `claimNFT` or `setTVSAllocation` → `claimRewardTVS`), so `rewardProjects[projectId].allocations[nftId]` is non‑empty.  
2. External modules interacting with vesting (e.g. `A26ZDividendDistributor`) are compiled against `IAlignerzVesting` and call `vesting.allocationOf(nftId)` through that interface.


### External Pre-conditions

1. The dividend distributor (or any other external protocol) is deployed with a `vesting` address pointing to the `AlignerzVesting` implementation / proxy and uses `IAlignerzVesting` for type information.  
2. The distributor’s logic calls `getUnclaimedAmounts(nftId)` or any other function that reads `vesting.allocationOf(nftId)` and expects a full `Allocation` struct.


### Attack / Failure Path

1. Admin deploys `AlignerzVesting` and `A26ZDividendDistributor`, configuring the distributor with the vesting contract and TVS token via its constructor.  
2. A KOL (`kol1`) receives a TVS allocation (e.g. 100 TVS) and calls `claimRewardTVS`, which mints `nftId1` and populates `rewardProjects[projectId].allocations[nftId1]` and `allocationOf[nftId1]`.  
3. At some later point, the dividend distributor calls `getUnclaimedAmounts(nftId1)`.  
4. Inside `getUnclaimedAmounts`, the distributor executes `IAlignerzVesting.Allocation memory alloc = vesting.allocationOf(nftId1);` expecting to decode a full `Allocation` struct.  
5. The underlying implementation’s public `allocationOf` getter, however, returns data in a different shape (auto-generated mapping getter for a struct with dynamic arrays), so the ABI decoder in the distributor cannot interpret the return data as `Allocation` and the call **reverts**.  
6. As a result, `getUnclaimedAmounts`, `getTotalUnclaimedAmounts`, and any dividend setup (`setAmounts`, `setDividends`, etc.) that depends on them fail, preventing the dividend system from functioning for all NFTs, regardless of whether merges happened.


### Impact

The dividend distributor (and any other external module compiled against `IAlignerzVesting`) cannot safely read TVS allocations via `allocationOf`: calls to `vesting.allocationOf(nftId)` revert or decode incorrectly, breaking `getUnclaimedAmounts` and all higher-level dividend calculations. Practically, this means TVS holders cannot receive dividends through the current distributor implementation, as the core read of vesting state fails.


### PoC
Please insert below as `AllocationOfMergeBug.t.sol` under `test` folder and run as `forge test --mt test_allocationOf_getter_used_by_dividend_distributor -vvv`


```solidity
// SPDX-License-Identifier: MIT
pragma solidity =0.8.29;

import "forge-std/Test.sol";
import {Aligners26} from "../src/contracts/token/Aligners26.sol";
import {AlignerzNFT} from "../src/contracts/nft/AlignerzNFT.sol";
import {MockUSD} from "../src/MockUSD.sol";
import {AlignerzVesting} from "../src/contracts/vesting/AlignerzVesting.sol";
import {ERC1967Proxy} from "@openzeppelin/contracts/proxy/ERC1967/ERC1967Proxy.sol";
import {A26ZDividendDistributor} from "../src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";

contract AllocationOfMergeBugTest is Test {
    AlignerzVesting public vesting;
    Aligners26 public token;
    AlignerzNFT public nft;
    MockUSD public usdt;
    A26ZDividendDistributor public distributor;

    address public owner;

    function setUp() public {
        owner = address(this);

        usdt = new MockUSD();
        token = new Aligners26("26Aligners", "A26Z");
        nft = new AlignerzNFT("AlignerzNFT", "AZNFT", "https://nft.alignerz.bid/");

        AlignerzVesting implementation = new AlignerzVesting();
        
        bytes memory initData = abi.encodeCall(AlignerzVesting.initialize, (address(nft)));
        ERC1967Proxy proxy = new ERC1967Proxy(address(implementation), initData);
        vesting = AlignerzVesting(payable(address(proxy)));

        nft.addMinter(address(proxy));
        vesting.setTreasury(address(1));

        token.approve(address(vesting), type(uint256).max);

        distributor = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(usdt),       
            block.timestamp,
            30 days,
            address(token)     
        );
    }

    function test_allocationOf_getter_used_by_dividend_distributor() public {
        address kol1 = makeAddr("kol1");
        address kol2 = makeAddr("kol2");

        uint256 rewardProjectId = vesting.rewardProjectCount();
        uint256 startTime = block.timestamp;
        uint256 claimWindow = 30 days;

        vesting.launchRewardProject(address(token), address(usdt), startTime, claimWindow);

        address[] memory kolTVS = new address[](2);
        kolTVS[0] = kol1;
        kolTVS[1] = kol2;

        uint256[] memory TVSamounts = new uint256[](2);
        TVSamounts[0] = 100 ether;
        TVSamounts[1] = 200 ether;

        uint256 totalTVSAllocation = TVSamounts[0] + TVSamounts[1];
        uint256 vestingPeriod = 30 days;

        token.transfer(address(vesting), totalTVSAllocation);
        vesting.setTVSAllocation(rewardProjectId, totalTVSAllocation, vestingPeriod, kolTVS, TVSamounts);

        vm.prank(kol1);
        vesting.claimRewardTVS(rewardProjectId);
        uint256 nftId1 = nft.getTotalMinted();

        // When the dividend module asks for unclaimed amounts of nftId1, it calls
        // vesting.allocationOf(nftId1) through the IAlignerzVesting interface.
        //
        // Due to the mismatch between the public mapping getter (which only returns
        // the static tail of the struct) and the interface (which expects the full
        // Allocation with dynamic arrays), the ABI decoding in getUnclaimedAmounts
        // reverts.
        vm.expectRevert();
        distributor.getUnclaimedAmounts(nftId1);
    }
}
```

Result:
```bash
Ran 1 test for test/AllocationOfMergeBug.t.sol:AllocationOfMergeBugTest
[PASS] test_allocationOf_getter_used_by_dividend_distributor() (gas: 882648)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 1.70ms (425.17µs CPU time)

Ran 1 test suite in 86.97ms (1.70ms CPU time): 1 tests passed, 0 failed, 0 skipped (1 total tests)
```


### Mitigation

Align the interface with the actual ABI and update `IAlignerzVesting` and `A26ZDividendDistributor` to call `getAllocation` instead of relying on the auto-generated mapping getter.



  