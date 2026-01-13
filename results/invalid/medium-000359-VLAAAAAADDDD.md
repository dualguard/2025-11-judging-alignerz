# [000359] A project can have more than 10 pools
  
  ### Summary

##  Description and impact

Within ` AlignerzVesting.createPool() ` and more exactly here https://github.com/dualguard/2025-11-alignerz/blob/f7eeed88d91356484c02af6f38b71f27b790828c/protocol/src/contracts/vesting/AlignerzVesting.sol#L689
The implementation says ` require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project()); ` 
Even the error says that a project can have a max number of 10 pools but it is wrongly implemented and there can be 11 pools created .
Because ` createPool() ` is ` onlyOwner ` and owner is trusted, there is not an actual impact here but it is good to have the system does what is intended to do every single time . 
I created a test to demonstrate the existence of this issue .


### Root Cause

Everything has already been said within ` Summary `

### Internal Pre-conditions

Everything has already been said within ` Summary `

### External Pre-conditions

Everything has already been said within ` Summary `

### Attack Path

Everything has already been said within ` Summary `

### Impact

Everything has already been said within ` Summary `

### PoC




##  PoC




How to execute the PoC ?


- Add the test within  ' test/AlignerzVestingProtocolTest.t.sol '  .


- Execute the test using  '  forge  test  --match-test  test____  -vv  ' .




```

function test____() public {



    vm.startPrank(projectCreator);

    vesting.launchBiddingProject(
        address(token), 
        address(usdt), 
        block.timestamp, 
        block.timestamp + 1_000_000, 
        "0x0", 
        false
    );

    vm.stopPrank();



    vm.startPrank(projectCreator);

    token.approve(address(vesting), 30_000_000 ether);

    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);
    vesting.createPool(0, 1_000_000 ether, 0.01 ether, false);


    vm.stopPrank();



    (, , , uint256 poolCount, , , , , , ) = vesting.biddingProjects(0);

    assertEq(poolCount, 11);



}

```

### Mitigation

##  Recommended mitigation

From 

` require(biddingProject.poolCount <= 10, Cannot_Exceed_Ten_Pools_Per_Project()); ` 

to

` require(biddingProject.poolCount < 10, Cannot_Exceed_Ten_Pools_Per_Project()); ` 

  