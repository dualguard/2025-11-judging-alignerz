# [001085] Amounts and Dividends cant be set in A26ZDividendDistributor because of ABI Mismatch
  
  ### Summary

The function `A26ZDividendDistributor::getUnclaimedAmounts` uses the `IAlignerzNFT` interface to call `AlignerzNFT::allocationOf`. However, `allocationOf` is a public state variable in AlignerzVesting, not an actual function.
In Solidity, when a `public` struct variable contains dynamic types, the automatically generated getter will only return the static fields of the struct.

In this case, the full Allocation struct is:
```solidity
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
```
But the auto-generated getter for allocationOf will return only the static fields:

```solidity
(bool isClaimed,IERC20 token,uint256 assignedPoolId)
```

When `A26ZDividendDistributor` calls this function through the `IAlignerzNFT` interface, it expects the full struct, including all dynamic arrays. Instead, it receives only the static subset.

Because the ABI returned by the actual contract does not match the ABI expected by the interface, the call **fails during decoding**, causing the function in `A26ZDividendDistributor` to revert.

### Root Cause

The root cause is an **ABI mismatch** between what the interface expects and what the actual contract returns.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/interfaces/IAlignerzVesting.sol#L67

The interface defines `allocationOf` as a function that returns the entire `Allocation` struct, including all dynamic arrays.

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L114

However, in the actual implementation, `allocationOf` is not a function. It is a public state variable.
Solidity auto-generates a getter for public structs, but this getter only returns the static fields of the struct.
For Allocation, that means the getter returns only:
```solidity
(bool isClaimed, IERC20 token, uint256 assignedPoolId)
```

https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol#L142

The `A26ZDividendDistributor` contract tries to decode this returned tuple as if it were the full struct, starting with dynamic arrays (e.g., `uint256[] amounts`).
Since the actual returned values do not match the expected ABI layout, decoding fails, causing the call to **revert**.

### Internal Pre-conditions

1. The owner must transfer the dividend amount to the deployed `A26ZDividendDistributor` contract.
2. Owner must to call `A26ZDividendDistributor::setUpTheDividends` or `A26ZDividendDistributor::setAmounts` 

### External Pre-conditions

1. AlignerzVesting must be deployed, and at least one project must be created.
2. Users must submit bids for that project, the project owner must finalize all bids, and atleast one user must claim their NFTs.
3. The project must announce dividends and deploy an instance of `A26ZDividendDistributor` with the correct parameters for that specific project.

### Attack Path

1. The owner calls `setUpTheDividends` or `setAmounts` on `A26ZDividendDistributor`.
2. These functions call `vesting.allocationOf(nftId)` expecting the full Allocation struct.
3. But `allocationOf` is a public state variable, so it only returns the static fields `(isClaimed, token, assignedPoolId)`.
4. The `A26ZDividendDistributor` tries to decode this as the full struct, leading to an ABI mismatch revert, preventing dividend setup.

### Impact

**Dividend distribution** cannot be set up because the contract always reverts when reading vesting data. This makes dividend distribution **impossible**, and any tokens sent to the distributor get **stuck** until the owner **manually** withdraws them. The team must **redeploy** a fixed distributor contract for dividends to work.


### PoC

```solidity
//SPDX-License-Identifier: MIT
pragma solidity 0.8.29;


import {Test,console} from "forge-std/Test.sol";
import {ERC20,IERC20} from "@openzeppelin/contracts/token/ERC20/ERC20.sol";
import {Aligners26} from "src/contracts/token/Aligners26.sol";
import {AlignerzVesting} from "src/contracts/vesting/AlignerzVesting.sol";
import {AlignerzNFT} from "src/contracts/nft/AlignerzNFT.sol";
import {A26ZDividendDistributor} from "src/contracts/A26ZDividendDistributor/A26ZDividendDistributor.sol";
import {MockUSD} from "src/MockUSD.sol";
import {IAlignerzVesting} from "src/interfaces/IAlignerzVesting.sol";
contract TestA26ZDividendDistributor is Test {

    MockUSD public USD;
    Aligners26 public project1;
    Aligners26 public project2;
    AlignerzVesting public vesting;
    AlignerzNFT public nft;
    A26ZDividendDistributor public dividendDistributorProject1;
    address OWNER=makeAddr("OWNER");
    address USER1=makeAddr("USER1");
    address USER2=makeAddr("USER2");

    function setUp()public{
        startHoax(OWNER);
        USD=new MockUSD();
        project1=new Aligners26("PROJECT-1","PROJECT-1");
        project2=new Aligners26("PROJECT-2","PROJECT-2");
        vesting = new AlignerzVesting();
        nft=new AlignerzNFT("Aligner-NFT","NFT","NO-URI");
        vesting.initialize(address(nft));
        nft.addMinter(address(vesting));
        vm.stopPrank();
    }

    //@note Iam taking reward projects for making the tests simpler
    function createProjectsAndSetAllocation()internal{
        startHoax(OWNER);
        project1.approve(address(vesting), type(uint256).max);
        project2.approve(address(vesting), type(uint256).max);
        uint256 startTime=block.timestamp+10;
        uint256 claimWindow=block.timestamp+110;
        uint256 totalTVSAllocation=1000e18;
        vesting.launchRewardProject(address(project1), address(USD), startTime, claimWindow);
        vesting.launchRewardProject(address(project2), address(USD), startTime, claimWindow);
        address[] memory kolTVS1=new address[](1);
        uint256[] memory TVSamounts=new uint256[](1);
        kolTVS1[0]=USER1;
        TVSamounts[0]=totalTVSAllocation;
        
        vesting.setTVSAllocation(0, totalTVSAllocation, 100, kolTVS1, TVSamounts);
        address[] memory kolTVS2=new address[](1);
        kolTVS1[0]=USER2;
        vesting.setTVSAllocation(1, totalTVSAllocation, 100, kolTVS1, TVSamounts);
        vm.warp(block.timestamp+110);
        vm.stopPrank();
        hoax(USER1);
        vesting.claimRewardTVS(0);
        hoax(USER2);
        vesting.claimRewardTVS(1);
    }

    function createDividendDistributorForProject1() public {
        startHoax(OWNER);
        uint256 startTime = 0;
        uint256 vestingPeriod = 100;
        dividendDistributorProject1 = new A26ZDividendDistributor(
            address(vesting),
            address(nft),
            address(USD),
            startTime,
            vestingPeriod,
            address(project1)
        );
        USD.transfer(address(dividendDistributorProject1), 1000e6);
        vm.stopPrank();
    }

    function setupDividends()internal{
        hoax(OWNER);
        dividendDistributorProject1.setUpTheDividends();
    }
    

    function test_Amounts_And_Dividends_Can_Never_Set_Due_To_Abi_Mismatch() public {
        createProjectsAndSetAllocation();
        createDividendDistributorForProject1();

        // Because the A26ZDividendDistributor relies on IAlignerzVesting.Allocation,
        // and the implementation of allocationOf() returns a tuple instead of a struct,
        // setUpTheDividends() must revert due to an ABI mismatch.
        vm.expectRevert();
        setupDividends();
        // This works because the concrete implementation returns (bool, IERC20, uint256),
        // and Solidity can unpack the tuple directly into these variables.
        (bool isClaimed1, IERC20 token1, uint256 assignedPoolId1) = vesting.allocationOf(1);
        (bool isClaimed2, IERC20 token2, uint256 assignedPoolId2) = vesting.allocationOf(2);

        // This fails at compile-time because the struct type does not match the actual return type.
        // IAlignerzVesting.Allocation memory allocation1 = vesting.allocationOf(1); // compile error

        // Wrapping the contract address into the IAlignerzVesting interface bypasses the compiler error.
        // However, the implementation does NOT return an Allocation struct.
        // => This causes an ABI decoding mismatch and MUST revert.
        vm.expectRevert();
        (bool ok, ) = address(vesting).call(
            abi.encodeWithSelector(IAlignerzVesting.allocationOf.selector, 1)
        );
    }
}
```

1. Add this test in `test/A26ZDividendDistributor/TestA26ZDividendDistributor.t.sol` .
2. Run the test using:
```bash
forge test --mt test_Amounts_And_Dividends_Can_Never_Set_Due_To_Abi_Mismatch
```
3. You should see an output similar to:
```bash
Ran 1 test for test/A26ZDividendDistributor/TestA26ZDividendDistributor.t.sol:TestA26ZDividendDistributor
[PASS] test_Amounts_And_Dividends_Can_Never_Set_Due_To_Abi_Mismatch() (gas: 3462177)
Suite result: ok. 1 passed; 0 failed; 0 skipped; finished in 4.49ms (1.96ms CPU time)
```

### Mitigation

- Instead of depending on the state variable use a helper function.

  